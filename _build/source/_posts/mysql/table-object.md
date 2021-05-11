---
title: MySQL 中的 TABLE 对象
date: 2020-12-05 20:07:49
categories: 
- MySQL

---

MySQL 中的 TABLE 是最核心的数据结构之一，包含的类包括：TABLE、TABLE_LIST、TABLE_SHARE 等，下面结合 MySQL 的代码进行分析说明。

<!-- more -->

## 基本说明

- TABLE_LIST：词法&语法分析后生成的对象，常见的生成方式为 add_table_to_list
- TABLE_SHARE：server 层表定义缓存，一个 table 实例对应一个 TABLES_SHARE 对象，可以将TABLE_SHRAE 对象看成是表的物理文件（frm）在内存中的映射
- TABLE：每一个查询 sql 中的表建一个 TABLE 对象

<img src="/images/table-object-1.png" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

TABLE_LIST 只是一个表的描述信息，包括各种名称、锁类型等；

TABLE_SHARE 可以看作是完整的表定义信息，在 8.0 以前的版本中需要通过读取 frm 文件获取，8.0 版本中引入了 DD，可以直接通过 DD 获取到表定义信息。Server 层有一个全局的 Table_definition_cache 缓存对象，保存已经打开的 TABLE_SHARE 对象；

TABLE 对象为查询最后使用的 TABLE 信息，该对象上关联了引擎层的 hander，所有的引擎层的操作都需要通过此 hander 进行。Server 层同样有一个全局的 Table_cache_manager，然而 TABLE 的缓存和 TABLE_SHARE 的缓存不同，由于每个 THD 上的 TABLE 对象都不相同，所有同一个表会在 Table_cache_manager 保存多份。

关于 Table_cache_manager，有两个参数进行控制：

```sql
MySQL [none]> show variables like "%table_open%";
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| table_open_cache           | 500   |
| table_open_cache_instances | 16    |
+----------------------------+-------+
2 rows in set (0.01 sec)
```

其中：

- table_open_cache_instances 表示将 Table_cache_manager 分为多少个部分；
- table_open_cache 表示 TABLE 对象的阈值，超过此值后会开始进行无用对象的回收；

## MySQL 8.0

```c++
|--> open_tables
|    |--> open_and_process_table
|    |    |--> open_table  // 打开表
|    |    |    |--> thd->mdl_context.acquire_lock  // 获取 MDL 锁
|    |    |    |--> open_table_get_mdl_lock
|    |    |    |    |--> thd->mdl_context.acquire_lock
|    |    |    |
|    |    |    |--> check_if_table_exists
|    |    |    |    |--> dd::table_exists
|    |    |    |    |    |--> client->acquire // dd::Abstract_table
|    |    |    |
|    |    |    |--> tc->get_table  //  从 Table_cache_manager 查找
|    |    |    |
|    |    |    |--> get_table_share_with_discover  // 构建 TABLE_SHARE
|    |    |    |    |--> get_table_share
|    |    |    |    |    |--> table_def_cache->find  // 从 Table_definition_cache 查找
|    |    |    |    |    |--> process_found_table_share
|    |    |    |    |    |
|    |    |    |    |    |--> alloc_table_share
|    |    |    |    |    |--> table_def_cache->emplace  // 维护 Table_definition_cache
|    |    |    |    |    |
|    |    |    |    |    |--> thd->dd_client()->acquire
|    |    |    |    |    |--> open_table_def
|    |    |    |
|    |    |    |--> open_table_from_share  // 通过 TABLE_SHARE 打开 TABLE
|    |    |    |    |--> ha_open
|    |    |    |    |    |--> ha_innobase::open // 打开一个 innodb 表
|    |    |    |    |    |    |--> lookup_table_handler
|    |    |    |    |    |    |--> dict_table_check_if_in_cache_low
|    |    |    |    |    |    |--> dd_open_table
|    |    |    |    |    |    |    |--> dd_open_table_one
|    |    |    |    |    |    |
|    |    |    |    |    |    |--> row_create_prebuilt
|    |    |    |    |    |    |--> info(...)
|    |    |    |    |    |    |    |--> info_low
|    |    |    |    |    |    |    |    |--> update_thd
|    |    |    |    |    |    |    |    |    |--> row_update_prebuilt_trx
|    |    |    |
|    |    |    |--> tc->add_used_table  // 添加到 Table_cache_manager
|    |    |    |
|    |    |    |--> thd->set_open_tables
|    |    |    |--> table->init



|--> close_thread_tables
|    |--> mysql_unlock_tables
|    |    |--> unlock_external
|    |    |    |--> ha_external_lock
|    |    |    |    |--> ha_innobase::external_lock
|    |
|    |--> close_open_tables
|    |    |--> close_thread_table
|    |    |    |--> release_or_close_table
|    |    |    |    |--> intern_close_table
|    |    |    |    |--> tc->remove_table
|    |    |    |    |
|    |    |    |    |--> tc->release_table
```

## MySQL 5.7

### 表打开过程

```c++
bool open_table(THD *thd, TABLE_LIST *table_list, Open_table_context *ot_ctx)
    |
    |
    |--> TABLE* Table_cache::get_table(THD *thd, my_hash_value_type hash_value,
    |                                  const char *key, size_t key_length,
    |                                  TABLE_SHARE **share)
    |
    |
    |--> TABLE_SHARE *
    |    get_table_share_with_discover(THD *thd, TABLE_LIST *table_list,
    |                                  const char *key, size_t key_length,
    |                                  uint db_flags, int *error,
    |                                  my_hash_value_type hash_value)
    |        |
    |        |
    |        |--> TABLE_SHARE *get_table_share(THD *thd, TABLE_LIST *table_list,
    |        |                                 const char *key, size_t key_length,
    |        |                                 uint db_flags, int *error,
    |        |                                 my_hash_value_type hash_value)
    |        |        |
    |        |        |
    |        |        |--> uchar* my_hash_search_using_hash_value(const HASH *hash, 
    |        |        |                                            my_hash_value_type hash_value,
    |        |        |                                            const uchar *key,
    |        |        |                                            size_t length)
    |        |        |
    |        |        |
    |        |        |--> int open_table_def(THD *thd, TABLE_SHARE *share, uint db_flags)
    |        |        |        |
    |        |        |        |
    |        |        |        |--> static int open_binary_frm(THD *thd, TABLE_SHARE *share, uchar *head,
    |        |        |        |                               File file)
    |        |        |        |        |
    |        |        |        |        |
    |        |        |        |        |--> handler *get_new_handler(TABLE_SHARE *share, MEM_ROOT *alloc,
    |        |        |        |        |                             handlerton *db_type)
    |
    |
    |--> int open_table_from_share(THD *thd, TABLE_SHARE *share, const char *alias,
    |                              uint db_stat, uint prgflag, uint ha_open_flags,
    |                              TABLE *outparam, bool is_create_table)
    |        |
    |        |
    |        |--> handler *get_new_handler(TABLE_SHARE *share, MEM_ROOT *alloc,
    |        |                             handlerton *db_type)
    |        |
    |        |
    |        |--> int handler::ha_open(TABLE *table_arg, const char *name, int mode,
                                       int test_if_locked)
    
```

### 表创建过程

```c++
TABLE_LIST *
st_select_lex::add_table_to_list(THD *thd,
                                 Table_ident *table,
                                 LEX_STRING *alias,
                                 ulong table_options,
                                 thr_lock_type lock_type,
                                 enum_mdl_type mdl_type,
                                 List<Index_hint> *index_hints_arg,
                                 List<String> *partition_names,
                                 LEX_STRING *option,
                                 Sequence_scan_mode seq_scan_mode)


bool mysql_create_table(THD *thd, TABLE_LIST *create_table,
                        HA_CREATE_INFO *create_info,
                        Alter_info *alter_info)
    |
    |
    |--> bool mysql_create_table_no_lock(THD *thd,
    |                                   const char *db, const char *table_name,
    |                                   HA_CREATE_INFO *create_info,
    |                                   Alter_info *alter_info,
    |                                   uint select_field_count,
    |                                   bool *is_trans)
    |        |
    |        |
    |        |--> static
    |        |    bool create_table_impl(THD *thd,
    |        |                           const char *db, const char *table_name,
    |        |                           const char *error_table_name,
    |        |                           const char *path,
    |        |                           HA_CREATE_INFO *create_info,
    |        |                           Alter_info *alter_info,
    |        |                           bool internal_tmp_table,
    |        |                           uint select_field_count,
    |        |                           bool no_ha_table,
    |        |                           bool *is_trans,
    |        |                           KEY **key_info,
    |        |                           uint *key_count)
    |        |        |
    |        |        |
    |        |        |--> int rea_create_table(THD *thd, const char *path,
    |        |        |                         const char *db, const char *table_name,
    |        |        |                         HA_CREATE_INFO *create_info,
    |        |        |                         List<Create_field> &create_fields,
    |        |        |                         uint keys, KEY *key_info, handler *file,
    |        |        |                         bool no_ha_table)
    
```



```c++
/* create table 过程 */

|--> mysql_create_table
|    |--> open_tables
|    |    |--> open_and_process_table
|    |    |    |--> open_table
|    |    |    |    |--> check_if_table_exists
|    |
|    |--> mysql_create_table_no_lock
|    |    |--> create_table_impl
|    |    |    |--> mysql_prepare_create_table
|    |    |    |
|    |    |    |--> rea_create_table
|    |    |    |    |--> mysql_create_frm // 创建 frm 文件
|    |    |    |    |
|    |    |    |    |--> ha_create_table
|    |    |    |    |    |--> open_table_def
|    |    |    |    |    |    |--> mysql_file_open
|    |    |    |    |    |    |--> open_binary_frm
|    |    |    |    |    |    |--> mysql_file_close
|    |    |    |    |    |
|    |    |    |    |    |--> open_table_from_share
|    |    |    |    |    |
|    |    |    |    |    |--> table.file->ha_create



|--> ha_innobase::create
|    |--> info.prepare_create_table
|    |--> row_mysql_lock_data_dictionary
|    |
|    |--> info.create_table
|    |    |--> create_table_def
|    |    |    |--> dict_mem_table_create
|    |    |    |--> row_create_table_for_mysql
|    |    |    |    |--> dict_build_table_def_step
|    |    |    |    |    |--> dict_build_tablespace_for_table
|    |    |    |    |    |    |--> fil_ibd_create // 创建 ibd 文件
|    |    |    |    |    |--> dict_create_sys_tables_tuple
|    |    |    |    |    |--> ins_node_set_new_row // 插入系统表
|    |    |
|    |    |--> create_index
|    |
|    |--> row_mysql_unlock_data_dictionary
```

## 参考文献

> http://mysql.taobao.org/monthly/2015/08/10/
>
> http://liuyangming.tech/10-2019/information_schema-exploration.html
>
> https://www.cnblogs.com/xpchild/p/3780625.html