---
title: MySQL 词法&语法分析过程
date: 2020-12-05 19:19:25
categories: 
- MySQL

---

MySQL 的词法分析部分采用了自己实现的方式，语法分析部分使用了 Yacc。关于 Yacc 的基本语法规则，可以先看看前面的参考文献。

<!-- more -->

## 基本流程

首先以 MySQL 8.0 版本为例，介绍一下一条 SQL 在进行解析时候的基本流程：

```c++
1. mysql_parse(thd, &parser_state)

2. parse_sql(thd, parser_state, NULL)

3. thd->sql_parser()

4. MYSQLparse(this, &root)

    4.1. MYSQLlex

    4.2. lex_one_token

    4.3. find_key_word

    4.4. consume_optimizer_hints

    4.5. HINT_PARSER_parse

5. lex->make_sql_cmd(root)

    5.1. m_sql_cmd = root->make_cmd(thd)

    5.2. opt_hints->contextualize(&pc)
```

Server 层调用词法&语法分析的入口在 `mysql_parse ` 方法中，最终调用 `MYSQLparse` 方法，生成 `Parse_tree_root` 对象。根据 sql_yacc.cc 文件头部的宏定义可以知道，`MYSQLparse` 对应的就是 sql_yacc.yy 中定义的语法分析规则。

```c++
// sql_yacc.cc

#define yyparse         MYSQLparse
#define yylex           MYSQLlex
#define yylval          MYSQLlval
#define yylloc          MYSQLlloc

# define YYLEX yylex (&yylval, &yylloc, YYTHD)

int
yyparse (class THD *YYTHD, class Parse_tree_root **parse_tree)
{
  ...

  /* YYCHAR is either YYEMPTY or YYEOF or a valid lookahead symbol.  */
  if (yychar == YYEMPTY)
  {
    YYDPRINTF ((stderr, "Reading a token: "));
    yychar = YYLEX;  //调用词法分析
  }

  ...
}
```

前面已经提到，MySQL 中的词法分析没有采用 Lex 进行实现，而是自己手写了词法分析的逻辑，具体的代码在 sql_lex.cc 中。基本的逻辑如下：

```c++
int MYSQLlex(YYSTYPE *yacc_yylval, YYLTYPE *yylloc, THD *thd) {
  auto *yylval = reinterpret_cast<Lexer_yystype *>(yacc_yylval);
  Lex_input_stream *lip = &thd->m_parser_state->m_lip;
  int token;
    
  ...

  token = lex_one_token(yylval, thd);

  ...
}

static int lex_one_token(Lexer_yystype *yylval, THD *thd) {
  ...
  uchar c = 0;
  enum my_lex_states state;
  Lex_input_stream *lip = &thd->m_parser_state->m_lip;
  lip->yylval = yylval;
  ...
      
  lip->start_token();
  state = lip->next_state;
  lip->next_state = MY_LEX_START;
  for (;;) {
    switch (state) {
      case MY_LEX_START:  // Start of token
        // Skip starting whitespace
        while (state_map[c = lip->yyPeek()] == MY_LEX_SKIP) {
          if (c == '\n') lip->yylineno++;

          lip->yySkip();
        }

        /* Start of real token */
        lip->restart_token();
        c = lip->yyGet();  // 此处读取到第1个有效字符，进入后续处理逻辑
        state = state_map[c];
        break;
      ...
    }
  }
  
  ...
}
```

以一个标识符的处理为例：

```c++
{
    ...
    case MY_LEX_IDENT:
      const char *start;
      if (use_mb(cs)) {
        result_state = IDENT_QUOTED;
        switch (my_mbcharlen(cs, lip->yyGetLast())) {
          case 1:
            break;
          case 0:
            if (my_mbmaxlenlen(cs) < 2) break;
            /* else fall through */
          default:
            int l =
                my_ismbchar(cs, lip->get_ptr() - 1, lip->get_end_of_query());
            if (l == 0) {
              state = MY_LEX_CHAR;
              continue;
            }
            lip->skip_binary(l - 1);
        }
        while (ident_map[c = lip->yyGet()]) {
          switch (my_mbcharlen(cs, c)) {
            case 1:
              break;
            case 0:
              if (my_mbmaxlenlen(cs) < 2) break;
              /* else fall through */
            default:
              int l;
              if ((l = my_ismbchar(cs, lip->get_ptr() - 1,
                                   lip->get_end_of_query())) == 0)
                break;
              lip->skip_binary(l - 1);
          }
        }
      } else {
        for (result_state = c; ident_map[c = lip->yyGet()]; result_state |= c)
          ;
        /* If there were non-ASCII characters, mark that we must convert */
        result_state = result_state & 0x80 ? IDENT_QUOTED : IDENT;
      }
      length = lip->yyLength();
      start = lip->get_ptr();
      if (lip->ignore_space) {
        /*
          If we find a space then this can't be an identifier. We notice this
          below by checking start != lex->ptr.
        */
        for (; state_map[c] == MY_LEX_SKIP; c = lip->yyGet()) {
          if (c == '\n') lip->yylineno++;
        }
      }
      if (start == lip->get_ptr() && c == '.' && ident_map[lip->yyPeek()])
        lip->next_state = MY_LEX_IDENT_SEP;
      else {  // '(' must follow directly if function
        lip->yyUnget();
        if ((tokval = find_keyword(lip, length, c == '('))) {
          lip->next_state = MY_LEX_START;  // Allow signed numbers
          return (tokval);                 // Was keyword
        }
        lip->yySkip();  // next state does a unget
      }
      yylval->lex_str = get_token(lip, 0, length);
      
      /*
         Note: "SELECT _bla AS 'alias'"
         _bla should be considered as a IDENT if charset haven't been found.
         So we don't use MYF(MY_WME) with get_charset_by_csname to avoid
         producing an error.
      */
      
      if (yylval->lex_str.str[0] == '_') {
        auto charset_name = yylval->lex_str.str + 1;
        const CHARSET_INFO *cs =
            get_charset_by_csname(charset_name, MY_CS_PRIMARY, MYF(0));
        if (cs) {
          lip->warn_on_deprecated_charset(cs, charset_name);
          if (cs == &my_charset_utf8mb4_0900_ai_ci) {
            /*
              If cs is utf8mb4, and the collation of cs is the default
              collation of utf8mb4, then update cs with a value of the
              default_collation_for_utf8mb4 system variable:
            */
            cs = thd->variables.default_collation_for_utf8mb4;
          }
          yylval->charset = cs;
          lip->m_underscore_cs = cs;
      
          lip->body_utf8_append(lip->m_cpp_text_start,
                                lip->get_cpp_tok_start() + length);
          return (UNDERSCORE_CHARSET);
        }
      }
      
      lip->body_utf8_append(lip->m_cpp_text_start);
      
      lip->body_utf8_append_literal(thd, &yylval->lex_str, cs,
                                    lip->m_cpp_text_end);
      
      return (result_state);  // IDENT or IDENT_QUOTED
    ...
}
```

以一个简单的 update 语句为例：

```sql
update t1 set age = 10 where id = 1;
```

在 sql_yacc.yy 中，对于 update 语句的处理模式如下：

```bash
update_stmt:
          opt_with_clause
          UPDATE_SYM            /* #1 */
          opt_low_priority      /* #2 */
          opt_ignore            /* #3 */
          table_reference_list  /* #4 */
          SET_SYM               /* #5 */
          update_list           /* #6 */
          opt_where_clause      /* #7 */
          opt_order_clause      /* #8 */
          opt_simple_limit      /* #9 */
          {
            $$= NEW_PTN PT_update($1, $2, $3, $4, $5, $7.column_list, $7.value_list,
                                  $8, $9, $10);
          }
        ;
```

一条 update 语句被切分为 10 个部分，对应生成 10 个参数：

1. opt_with_clause: with子句
2. UPDATE_SYM: update关键字
3. opt_low_priority: 
4. opt_ignore: 
5. table_reference_list: 表信息
6. SET_SYM: set关键字
7. update_list: 更新内容
8. opt_where_clause: where子句
9. opt_order_clause: order子句
10. opt_simple_limit: limit子句

每一个部分经过处理都会生成一个 YYSTYPE 类型对象，YYSTYPE 是一个集合类，具体定义可以查看 parser_yystype.h，摘要部分信息如下：

```c++
union YYSTYPE {
  Lexer_yystype lexer;  // terminal values from the lexical scanner
  ...

  Mem_root_array_YY<PT_table_reference *> table_reference_list;
  ...  

  Item *item;
  ...

  struct {
    class PT_item_list *column_list;
    class PT_item_list *value_list;
  } column_value_list_pair;
  ...
}
```

匹配到 update 语句模式的，经过语法分析，最终会生成一个 PT_update 类型的 Parse_tree_root。

## 语法扩展

## 参考文献

> Yacc介绍：https://www.ibm.com/developerworks/cn/linux/sdk/lex/index.html
>
> 词法分析：https://www.cnblogs.com/nocode/archive/2011/08/03/2126726.html
>
> 语法分析：https://www.cnblogs.com/nocode/archive/2011/08/09/2132814.html
>
> http://mysql.taobao.org/monthly/2017/02/04/
>
> http://mysql.taobao.org/monthly/2017/04/02/
>
> SQL解析：https://tech.meituan.com/2018/05/20/sql-parser-used-in-mtdp.html
>
> http://www.orczhou.com/index.php/2012/11/mysql-innodb-source-code-optimization-1/
>
> https://www.jiqizhixin.com/articles/2018-12-12-17