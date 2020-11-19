---
title: MySQL Online DDL
date: 2020-11-18 22:26:22
tags:
- MySQL
categories: 
- 数据库


---

关于 MySQL 的 Online DDL，网上有很多文章进行介绍，但是笔者浏览了部分文章，发现很多介绍都比较模糊，理解起来比较费劲。本文试图从使用者的角度对 MySQL Online DDL 进行一个直观的介绍。

<!-- more -->

## 各版本支持情况

> https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl-operations.html
>
> https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-operations.html
>
> https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html

截止 20201118 社区各个版本的支持情况：

<img src="/images/online-ddl-1.jpg" width="680px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

<img src="/images/online-ddl-2.jpg" width="680px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

<img src="/images/online-ddl-3.jpg" width="680px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

结合上面的表格，对 MySQL 当前 DDL 的执行模式总结如下：

1. Instance DDL 是 MySQL 8.0 引入的新功能，当前支持的范围较小，包括：

   - 修改二级索引的类型
   - 新增列
   - 修改列的默认值
   - 重命名表

2. 在执行 DDL 操作时，MySQL 内部对于 algorithm 的选择策略是：如果用户显式指定了 algorithm，那么使用用户指定的选项；如果用户未指定，那么如果该操作支持 inplace 则优先选择 inplace，否则选择 copy；当前不支持 inplace 的操作主要有：

   - 删除主键
   - 修改列的数据类型
   - 修改表的字符集

3. 我们常说的 Online DDL，其实是从 DML 操作的角度描述的，如果 DDL 操作不阻塞 DML 操作，那么这个 DDL 就是 Online 的。当前非 Online 的 DDL 其实已经比较少了，主要有：

   - 新增全文索引
   - 新增空间索引
   - 删除主键
   - 指定表的字符集
   - 修改表的字符集

更多详细的示例请参考上面的官方文档的地址。

## 两个问题

最后讨论两个非常容易混淆的问题：

1. Online DDL 不会锁表，可以随意的执行。
2. 对于支持 inplace 算法的 DDL，DDL 操作是原地修改数据，不需要额外的数据空间。

### Q1: Online DDL 会不会锁表

Online DDL 会不会锁表？要回答这个问题，首先要明确“锁表”的含义。很多 MySQL 用户经常在表无法正常的进行 DML 时就觉得是锁表了，这种说法其实过于宽泛，实际上能够影响 DML 操作的锁至少包括以下几种（默认为 InnoDB 表）：

- MDL 锁
- 表锁
- 行锁
- GAP 锁

其中除了 MDL 锁是在 Server 层加的之外，其它三种都是在 InnoDB 层加的。具体的加锁逻辑不在此进行展开，但是需要明确一点：所有的操作（不管是 DDL 还是 DML 还是查询语句）都需要先拿 Server 层的 MDL 锁，然后再去拿 InnoDB 层的某个需要的锁。一个 DDL 的基本过程是这样的：

1. 首选，在开始进行 DDL 时，需要拿到对应表的 MDL X 锁，然后进行一系列的准备工作；
2. 然后将 MDL X 锁降级为 MDL S 锁，进行真正的 DDL 操作；
3. 最后再次将 MDL S 锁升级为 MDL X 锁，完成 DDL 操作，释放 MDL 锁；

所以在真正执行 DDL 操作期间，确实是不会“锁表”的，但是如果在第一阶段拿 MDL X 锁时无法正常获取，那就可能真的会“锁表了”。一个简单的例子如下：

```sql
# session 1
select sleep(300) from mytest.t1;

# session 2
optimize table mytest.t1;

# session 3
select * from mytest.t1;
```

session 1 模拟了一个慢查询，然后 session 2 开始进行 DDL 操作，无法拿到 MDL X 锁，处于等到中。此时 session 3 需要执行一个查询，发现无法执行。实际上，在 session 1 介绍前，表 t1 的所有操作都无法进行了，也可以说表 t1 “锁表”了。

明确了上面的概念之后，再回到我们的问题，Online DDL 是不是不锁表？如果非要回答，那么只能说，Online DDL 并不是绝对安全，更不是可以随意的执行。线上操作还是需要在业务低峰期谨慎操作。

### Q2: inplace DDL 需不需要额外的数据空间

前面我们提到过，MySQL 内部对于 DDL 的 algorithm 有两种选择：inplace 和 copy。copy 算法理解起来相对简单一点：创建一张临时表，然后将原表的数据 copy 到临时表中，最后再用临时表替换原表。对于上面的步骤，由于需要将原表的数据 copy 到临时表中，所以肯定需要消耗额外的数据空间。

那么对于支持 inplace 算法的 DDL，是不是不需要额外的数据空间？答案是：需要。其实之所以会问这个问题，还是因为对 inplace 本身的理解出现了偏差。简单来说：inplace 描述的是表，而不是数据文件。只要不创建临时表，那么都是 inplace 的。

实际上，很多 inplace DDL 都会创建临时的数据文件，所以都会需要额外的数据空间，例如：

- 添加主键
- 删除并重新添加主键
- 新增列（8.0 支持 instance DDL，不需要）
- 删除列
- 重新排序列
- 删除列的默认值
- 增加列的默认值
- 修改表的 row_format
- optimize table

以上只是对 MySQL Online DDL 的一个简单的总结，更多细节的内容还需要大家去发掘。

