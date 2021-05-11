---
title: MySQL 事务提交过程
date: 2020-12-05 19:44:18
tags:
- 事务
categories: 
- MySQL

---

关于 MySQL 的事务提交过程，其实已经有很多文章进行过介绍，本文之所以还想再总结一遍是因为之前的文章都是从某一个或者某几个角度进行介绍，本文尝试从：binlog、gtid、semi-sync、undo、redo、trx等多个角度一起，将 MySQL 事务提交过程串联起来。

<!-- more -->

## 整体入门

> 一条 SQL 的执行：https://time.geekbang.org/column/article/68633
>
> 日志相关问题：https://time.geekbang.org/column/article/73161
>
> 如何保证数据不丢：https://time.geekbang.org/column/article/76161
>
> 如何保证主备一致：https://time.geekbang.org/column/article/76446

上面的文章是丁奇《MySQL实战45讲》中的部分内容，作者深入浅出的对 MySQL 的基本原理进行了介绍，非常适合对于 MySQL 的整体入门，本文中使用的图片也全部来自上述文章。通过上面的文章，需要对下面的问题有一个基本的概念：

1. 事务提交的基本流程
2. 为什么需要两阶段提交协议
3. Crash Safe 是如何保障的
4. 主备间如何进行同步
5. GTID 的作用

本文主要就上面几个问题分别进行展开说明。

## 事务提交的基本流程

以最简单的单行更新操作为例，数据的更新过程如下图：

<img src="/images/server-trx-1.png"  width="380px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

如上图所示，一个简单的更新操作包含了一下几个部分：

1. 修改内存中的数据页。对于 InnoDB 来说，所有的内存页都是由 Buffer Pool 进行管理，修改数据页之前需要先记录 undo。undo 页和数据页的修改又都需要通过 redo 进行保护。
2. 提交修改。不管是 auto-commit 模式还是手动提交，提交过程都需要完成以下几件事情：

- - 将步骤 1 中生成的 redo 持久化（WAL）
  - 记录 binlog
  - 修改事务状态

从图中可以看到，MySQL 内部将事务提交的过程分成了 3 个部分：

1. 写 redo，修改事务状态为 prepared
2. 写 binlog
3. 修改事务状态为 commited

关于提交过程，更详细的过程如下图：

<img src="/images/server-trx-2.png"  width="380px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

前面提到过，undo 页和数据页在内存中的数据结构都是通过 Buffer Pool 进行管理的；但是 redo log 和 binlog 使用了操作系统的 page cache 机制，即，write 操作只写入到 page cache 中，sync 操作才会持久化到磁盘。通过上图可以看到，在真实的提交过程中，通过拆解 redo log prepare 的过程来提高写入的性能，这里面涉及到了组提交（group commit）的概念，详细的过程后续再做介绍。下面结合代码进行分析：

```c++
|--> ha_commit_trans
|    |--> tc_log->prepare  // MYSQL_BIN_LOG::prepare
|    |    |--> ha_prepare_low
|    |    |    |--> innobase_xa_prepare
|    |
|    |--> tc_log->commit  // MYSQL_BIN_LOG::commit
|    |    |--> cache_mngr->trx_cache.finalize  // XID event
|    |    |
|    |    |--> ordered_commit
|    |    |    |    // step 1
|    |    |    |--> process_flush_stage_queue
|    |    |    |    |--> assign_automatic_gtids_to_flush_group
|    |    |    |    |    |--> generate_automatic_gtid
|    |    |    |    |    |    |--> acquire_ownership
|    |    |    |    |    |    |    |--> add_gtid_owner
|    |    |    |    |--> flush_thread_caches
|    |    |    |    |    |..> MYSQL_BIN_LOG::write_gtid  // GTID event
|    |    |    |    |    |..> MYSQL_BIN_LOG::write_cache
|    |    |    |--> flush_cache_to_file
|    |    |    |
|    |    |    |    // step 2
|    |    |    |--> sync_binlog_file
|    |    |    |--> update_binlog_end_pos
|    |    |    |
|    |    |    |    // step 3
|    |    |    |--> call_after_sync_hook
|    |    |    |--> process_commit_stage_queue
|    |    |    |    |--> ha_commit_low
|    |    |    |    |    |--> innobase_commit
|    |    |    |--> process_after_commit_stage_queue
|    |    |    |
|    |    |    |--> finish_commit
|    |    |    |    |--> gtid_state->update_on_commit
|    |    |    |    |    |--> update_gtids_impl
```

回到事务提交的过程，整个事务提交的过程中有以下参与方：

1. binlog ：逻辑日志
2. innodb ：存储引擎
3. XA ：XA 事务
4. GTID：全局事务 ID

这些内容会在后面部分分别进行详细介绍。

## 为什么需要两阶段提交协议

为什么需要两阶段提交协议？这个问题相信很多阅读 MySQL 代码的同学都会问到。从结果上来说，两阶段提交协议保证了 binlog 和 innodb 数据的完整性。换个角度，如果没有两阶段提交协议，可能会出现什么结果？可能的情况无非下面两种：

1. 先写 binlog 成功，后写 redo 失败。
2. 先写 redo 成功，后写 binlog 失败。

对于第一种情况，若本机的实例 crash 后重启，需要通过 redo 进行恢复，由于写 redo 的时候失败了，所以无法进行恢复；对于第二种情况，备库接收到的 binlog 不完整，导致主备不一致。同样，利用 binlog 进行恢复时也会造成数据不完整。

然而，两阶段提交过程本身真的是必须的吗？前面讨论两阶段提交必要性的时候，一个前提条件是：binlog 和 redo 的完整性。binlog 日志是 server 层记录的逻辑日志，提供归档能力；redo 是 innodb 层的物理日志，提供 crash-safe 能力。这个限制也是 MySQL 设计之初带来的。如果抛开这个限制，两阶段提交是否还是必须的？当前比较流性的存储-计算分离型数据库，包括：Aurora、PolarDB 等都采用了物理复制的方案，直接通过 redo 进行主备的同步。

## Crash Safe 是如何保障的

关于 crash-safe 的保障，再次引用一张图片进行说明：

<img src="/images/server-trx-3.png"  width="380px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

图中标注了系统可能出现故障的点，下面分别进行分析：

1. 时刻 A 发生故障，此时事务状态还未修改完成，binlog 也未写入，恢复时对应的事务可以安全回滚；
2. 时刻 B 发生故障，恢复时需要结合 binlog 的状态进行判断：如果 binlog 里面的内容完整，那么事务可以正常提交；否则需要进行回滚。如何判断 binlog 的内容是否完整呢？这里需要借助前面提到的 XID，server 层在记录 binlog 时，会在事务提交阶段写入一个 XID event，里面包含了当前事务的 XID 信息。如果在重放 redo 的过程中（redo 中无 commit 记录），在 binlog 中能够找到对应的 XID event，则说明对应的 binlog 内容完整，事务可以提交。

一个简单的例子：

```bash
MySQL [(none)]> show master status;
+------------------+----------+--------------+------------------+-------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                         |
+------------------+----------+--------------+------------------+-------------------------------------------+
| mysql-bin.000002 |      195 |              |                  | f778c1cc-24a9-11eb-9911-b8599f3009a8:1-15 |
+------------------+----------+--------------+------------------+-------------------------------------------+
1 row in set (0.00 sec)

MySQL [mytest]> update t2 set c2 = 10 where id = 1;
Query OK, 1 row affected (11.62 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MySQL [mytest]> show binlog events in 'mysql-bin.000002';
+------------------+-----+----------------+------------+-------------+--------------------------------------------------------------------+
| Log_name         | Pos | Event_type     | Server_id  | End_log_pos | Info                                                               |
+------------------+-----+----------------+------------+-------------+--------------------------------------------------------------------+
| mysql-bin.000002 |   4 | Format_desc    | 2762931378 |         124 | Server ver: 8.0.18-rds-dev-debug, Binlog ver: 4                    |
| mysql-bin.000002 | 124 | Previous_gtids | 2762931378 |         195 | f778c1cc-24a9-11eb-9911-b8599f3009a8:1-15                          |
| mysql-bin.000002 | 195 | Gtid           | 2762931378 |         274 | SET @@SESSION.GTID_NEXT= 'f778c1cc-24a9-11eb-9911-b8599f3009a8:16' |
| mysql-bin.000002 | 274 | Query          | 2762931378 |         360 | BEGIN                                                              |
| mysql-bin.000002 | 360 | Table_map      | 2762931378 |         412 | table_id: 87 (mytest.t2)                                           |
| mysql-bin.000002 | 412 | Update_rows_v1 | 2762931378 |         472 | table_id: 87 flags: STMT_END_F                                     |
| mysql-bin.000002 | 472 | Xid            | 2762931378 |         503 | COMMIT /* xid=14 */                                                |
+------------------+-----+----------------+------------+-------------+--------------------------------------------------------------------+
7 rows in set (0.00 sec)

MySQL [mytest]> show master status;
+------------------+----------+--------------+------------------+-------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                         |
+------------------+----------+--------------+------------------+-------------------------------------------+
| mysql-bin.000002 |      503 |              |                  | f778c1cc-24a9-11eb-9911-b8599f3009a8:1-16 |
+------------------+----------+--------------+------------------+-------------------------------------------+
1 row in set (0.00 sec)
```

对应的 binlog 文件内容：

```bash
$/flash3/guoqing.mgq/lizard-2.0/bin/mysqlbinlog -vvv mysql-bin.000002
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#201116  9:45:22 server id 2762931378  end_log_pos 124 CRC32 0x8169654a     Start: binlog v 4, server v 8.0.18-rds-dev-debug created 201116  9:45:22 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
BINLOG '
stmxXw+y/K6keAAAAHwAAAABAAQAOC4wLjE4LXJkcy1kZXYtZGVidWcAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAACy2bFfEwANAAgAAAAABAAEAAAAYAAEGggAAAAICAgCAAAACgoKKioAEjQA
CgFKZWmB
'/*!*/;
# at 124
#201116  9:45:22 server id 2762931378  end_log_pos 195 CRC32 0x2fdbb3d5     Previous-GTIDs
# f778c1cc-24a9-11eb-9911-b8599f3009a8:1-15
# at 195
#201116  9:49:07 server id 2762931378  end_log_pos 274 CRC32 0xd877c4e7     GTID    last_committed=0    sequence_number=1   rbr_only=yes    original_committed_timestamp=1605491358802675   immediate_commit_timestamp=1605491358802675 transaction_length=308
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1605491358802675 (2020-11-16 09:49:18.802675 CST)
# immediate_commit_timestamp=1605491358802675 (2020-11-16 09:49:18.802675 CST)
/*!80001 SET @@session.original_commit_timestamp=1605491358802675*//*!*/;
/*!80014 SET @@session.original_server_version=80018*//*!*/;
/*!80014 SET @@session.immediate_server_version=80018*//*!*/;
SET @@SESSION.GTID_NEXT= 'f778c1cc-24a9-11eb-9911-b8599f3009a8:16'/*!*/;
# at 274
#201116  9:49:07 server id 2762931378  end_log_pos 360 CRC32 0x9d456972     Query   thread_id=8 exec_time=0 error_code=0
SET TIMESTAMP=1605491347/*!*/;
SET @@session.pseudo_thread_id=8/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=0/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
/*!80011 SET @@session.default_collation_for_utf8mb4=255*//*!*/;
BEGIN
/*!*/;
# at 360
#201116  9:49:07 server id 2762931378  end_log_pos 412 CRC32 0xb6d1f3c1     Table_map: `mytest`.`t2` mapped to number 87
# at 412
#201116  9:49:07 server id 2762931378  end_log_pos 472 CRC32 0xb9c2d35e     Update_rows_v1: table id 87 flags: STMT_END_F

BINLOG '
k9qxXxOy/K6kNAAAAJwBAAAAAFcAAAAAAAEABm15dGVzdAACdDIAAwMDAwAGAQEAwfPRtg==
k9qxXxiy/K6kPAAAANgBAAAAAFcAAAAAAAEAA///AAEAAAABAAAAAQAAAAABAAAAAQAAAAoAAABe
08K5
'/*!*/;
### UPDATE `mytest`.`t2`
### WHERE
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2=1 /* INT meta=0 nullable=1 is_null=0 */
###   @3=1 /* INT meta=0 nullable=1 is_null=0 */
### SET
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2=1 /* INT meta=0 nullable=1 is_null=0 */
###   @3=10 /* INT meta=0 nullable=1 is_null=0 */
# at 472
#201116  9:49:07 server id 2762931378  end_log_pos 503 CRC32 0x027351e3     Xid = 14
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

## 主备间如何进行同步

MySQL 通过 binlog 实现主备之间的同步，基本流程如下图：

<img src="/images/server-trx-4.png"  width="520px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

简单来说，MySQL 内部通过三个独立的线程来保证主备之间的 binlog 同步：

1. dump 线程（主）：在事务提交过程中，写完 binlog 后会去更新当前 binlog 的位置，dump 线程根据该位置，向备节点发送新的 binlog；
2. io 线程（备）：备节点上的 io 线程接收到主节点发来的 binlog，保存到本地的 relaylog，并向主节点进行回复；
3. ack 线程（主）：接收备节点返回的确认消息，确定事务是否可以提交（WAIT_AFTER_SYNC 模式下）；

在实际的同步过程中，主要通过以下的参数控制主备间的同步效率：

- rpl_semi_sync_master_wait_point ：[WAIT_AFTER_COMMIT | WAIT_AFTER_SYNC] 同步等待的位点
- rpl_semi_sync_master_timeout ：同步等待超时时间
- sync_binlog ：binlog 刷盘策略
- sync_relay_log ：relay log 刷盘策略

## GTID 的作用

前面已经提到了，MySQL 通过 binlog 来实现主备节点间的同步。在早期的版本中，进行同步前，需要在备节点上指定主节点的 binlog 位点信息，基本的语法如下：

```sql
CHANGE MASTER TO master_host='127.0.0.1',
master_port=3932,
master_user='replicator',
master_password='123456',
master_log_file='mysql-binlog.000001',
master_log_pos=155;
```

备节点根据指定的位点信息从主节点进行同步。这种方式的主要问题是：当主备节点发生切换时，需要手动的修改 binlog 位点，运维比较麻烦，GTID 应运而生。关于 GTID 的入门，可以参考[这篇文章](https://dbaplus.cn/news-11-857-1.html)，本文尝试用比较简单的方式进行归纳。首先，GTID 作为全局事务 ID，有 source_id:transaction_id 组成，可以简单的理解为某个 MySQL 节点上的某个事务。在前面的例子中也可以看到，在每个事务（读写事务）执行时，binlog 都会记录一个类型为 gtid 的 event 用于指定接下来要执行的事务的 GTID。与此同时，MySQL 内部有一个专门的线程 clone_gtid_thread 负责将已经执行的 GTID 进行持久化，保存到 mysql 库下的 gtid_executed 表中。

```c++
|--> srv_start_threads_after_ddl_recovery
|    |--> gtid_persistor.start()
|    |    |--> os_thread_create(clone_gtid_thread)


|--> clone_gtid_thread
|    |--> persist_gtid->periodic_write
|    |    |--> flush_gtids  // loop
|    |    |    |--> gtid_table_persistor->fetch_gtids
|    |    |    |
|    |    |    |--> write_to_table  // 持久化到 gtid_executed 表
|    |    |    |    |--> gtid_table_persistor->save
|    |    |    |    |    |--> write_row
|    |    |    |
|    |    |    |--> trx_sys_oldest_trx_no
|    |    |    |--> update_gtid_trx_no
|    |    |    |    |--> trx_sys_persist_gtid_num  // 持久化到 trx_sys page
|    |    |    |    |--> srv_purge_wakeup
```

接下来，备节点在收到对应的 binlog 后，会分别统计已接收到的 gtid 集合和已执行的 gtid 集合。备节点与主节点进行同步时，也会将上述 gtid 的集合发送到主节点，主节点根据备节点上已有的 gtid 信息，确定还需要发送哪些 binlog 给备节点，不再需要手动的修改同步位点。

```sql
CHANGE MASTER TO master_host='127.0.0.1',
master_port=5932,
master_user='replicator',
master_password='123456',
master_auto_position=1;
```

## 参考文献

XA 与两阶段提交

> http://mysql.taobao.org/monthly/2020/05/07/

Binlog

> http://mysql.taobao.org/monthly/2020/02/06/

半同步复制

> http://mysql.taobao.org/monthly/2017/04/01/
>
> http://mysql.taobao.org/monthly/2016/08/01/

GTID

> https://dbaplus.cn/news-11-857-1.html
>
> https://keithlan.github.io/2016/06/23/gtid/
>
> http://mysql.taobao.org/monthly/2020/05/09/
>
> https://www.cnblogs.com/MYSQLZOUQI/p/3850578.html

InnoDB 事务子系统

> http://mysql.taobao.org/monthly/2015/12/01/
>
> http://mysql.taobao.org/monthly/2017/12/01/