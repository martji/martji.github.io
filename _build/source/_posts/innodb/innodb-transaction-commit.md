---
title: InnoDB 事务提交过程
date: 2020-12-05 21:29:37
tags:
- 事务
categories: 
- InnoDB

---

前面已经介绍过 MySQL 在 Server 层的事务提交过程，本文主要对 InnoDB 层的事务提交过程进行介绍。

<!-- more -->

## Server 层两阶段提交过程

```c++
|--> trans_commit
|    |--> ha_commit_trans
|    |    |--> tc_log->prepare
|    |    |    |--> ha_prepare_low
|    |    |    |    |--> innobase_xa_prepare
|    |    |    |    |    |--> trx_prepare_for_mysql
|    |    |    |    |    |    |--> trx_prepare
|    |    |    |    |    |    |    |--> trx_prepare_low
|    |    |    |    |    |    |    |    |--> trx_undo_set_state_at_prepare
|    |    |    |    |    |    |    |    |    |--> trx_undo_write_xid
|    |    |
|    |    |--> tc_log->commit
|    |    |    |--> ha_commit_low
|    |    |    |    |--> innobase_commit
|    |    |    |    |    |--> innobase_commit_low
|    |    |    |    |    |    |--> trx_commit_for_mysql
|    |    |    |    |    |    |    |--> trx_undo_gtid_add_update_undo
|    |    |    |    |    |    |    |--> trx_undo_gtid_flush_prepare
|    |    |    |    |    |    |    |--> trx_commit
|    |    |    |    |    |    |    |    |--> trx_commit_low
|    |    |    |    |    |
|    |    |    |    |    |--> trx_deregister_from_2pc
|    |    |    |    |    |
|    |    |    |    |    |--> trx_commit_complete_for_mysql
|    |    |    |    |    |    |--> trx_flush_log_if_needed
```

## InnoDB 层提交过程

结合代码分析一下事务提交的过程：

```c++
|--> trx_commit_low
|    |--> trx_write_serialisation_history
|    |    |--> trx_serialisation_number_get
|    |    |    |--> trx_sys_get_new_trx_id  // trx_no
|    |    |    |--> UT_LIST_ADD_LAST(trx_sys->serialisation_list)
|    |    |    |--> purge_sys->purge_queue->push  // purge_sys 维护
|    |    |
|    |    |--> trx_undo_set_state_at_finish
|    |    |--> trx_undo_update_clean
|    |    |    |--> trx_purge_add_update_undo_to_history
|    |    |    |    |--> flst_add_first(rseg_header + TRX_RSEG_HISTORY, undo_header + TRX_UNDO_HISTORY_NODE, mtr)
|    |    |
|    |    |--> trx_sys_update_mysql_binlog_offset
|    |
|    |--> trx_commit_in_memory
|    |    |--> trx_release_impl_and_expl_locks
|    |    |    |--> trx_erase_lists
|    |    |    |    |--> UT_LIST_REMOVE(trx_sys->serialisation_list)
|    |    |    |    |--> trx_sys->rw_trx_ids.erase
|    |    |    |    |--> UT_LIST_REMOVE(trx_sys->rw_trx_list)
|    |    |    |    |--> trx_sys->mvcc->view_close
|    |    |    |    |--> trx_sys->rw_trx_set.erase
|    |    |    |    |--> trx_sys->min_active_id.store
|    |    |    |
|    |    |    |--> lock_trx_release_locks
|    |    |    |    |--> lock_release
|    |    |    |    |    |--> lock_rec_dequeue_from_page
|    |    |    |    |    |    |--> lock_rec_discard
|    |    |    |    |    |    |--> lock_rec_grant // 遍历 page，查看是否有需要唤醒的锁等待
|    |    |    |    |    |    |    |--> lock_grant_or_update_wait_for_edge_if_waiting
|    |    |    |    |    |    |    |    |--> lock_grant_or_update_wait_for_edge
|    |    |    |    |    |    |    |    |    |--> lock_has_to_wait_in_queue // 检查是否需要继续等待
|    |    |    |    |    |    |    |    |    |--> lock_grant // 解锁，唤醒 trx->lock->wait_thr
|    |    |    |    |    |    |    |    |    |    |--> lock_reset_wait_and_release_thread_if_suspended
|    |    |    |    |    |    |    |    |    |    |    |--> lock_reset_lock_and_trx_wait
|    |    |    |    |    |    |    |    |    |    |    |--> lock_wait_release_thread_if_suspended
|    |    |    |    |    |    |    |    |    |    |    |     |--> que_thr_end_lock_wait
|    |    |    |    |    |    |    |    |    |    |    |     |--> os_event_set(thr->slot->event)
|    |    |    |    |    |    |    |    |    |--> lock_update_wait_for_edge 
|    |    |    |    |    |    |    |
|    |    |    |    |    |    |    |--> lock_grant_cats
|    |    |    |    |    |
|    |    |    |    |    |--> lock_table_dequeue
|    |    |
|    |    |--> trx_sys->mvcc->view_close
|    |    |--> trx_undo_insert_cleanup
|    |    |--> srv_active_wake_master_thread
```

## 读写事务维护

### 插入 rw_trx_xxx

```c++
|--> trx_set_rw_mode
|    |--> trx_assign_rseg_durable
|    |    |--> get_next_redo_rseg
|    |    |    |--> get_next_redo_rseg_from_trx_sys  // 系统表空间
|    |    |    |
|    |    |    |--> get_next_redo_rseg_from_undo_spaces  // 独立 undo 表空间
|    |
|    |--> trx_sys_get_new_trx_id
|    |--> trx_sys->rw_trx_ids.push_back
|    |--> trx_sys->rw_trx_set.insert
|    |--> UT_LIST_ADD_FIRST(trx_sys->rw_trx_list)
```

MySQL 在 trx_sys 中引入了好几个数据结构来维护读写事务集合，这些数据结构的修改都需要在 trx_sys->mutex 的保护下进行：

- rw_trx_ids ：仅 trx_id 的集合
- rw_trx_set ：仅 trx 的集合
- rw_trx_list ：trx 的有序列表（事务开始时间有序）

对于显示启动的事务，MySQL 默认只会开启一个只读事务，直到事务中有 DML 操作时才会升级为读写事务。通过上面的代码可以看到，不管是 undo 还是 trx_id 都是在升级为读写事务时才会分配。

判断一个 trx_id 是否是活跃的读写事务：

```c++
trx_t *trx_rw_is_active(trx_id_t trx_id, ibool *corrupt, bool do_ref_count) {
  trx_t *trx;

  if (trx_sys->min_active_id.load() > trx_id) {
    return (NULL);
  }

  ...
    
  trx = trx_rw_is_active_low(trx_id, corrupt);

  return (trx);
}
```

### 移除 rw_trx_xxx

在事务提交的时候，需要从读写事务对应的数据结构中进行移除：

```c++
|--> trx_commit_in_memory
|    |--> trx_release_impl_and_expl_locks
|    |    |--> trx_erase_lists
|    |    |    |--> UT_LIST_REMOVE(trx_sys->serialisation_list)
|    |    |    |--> trx_sys->rw_trx_ids.erase
|    |    |    |--> UT_LIST_REMOVE(trx_sys->rw_trx_list)
|    |    |    |--> trx_sys->mvcc->view_close
|    |    |    |--> trx_sys->rw_trx_set.erase
|    |    |    |--> trx_sys->min_active_id.store
```

## purge_sys 维护

<img src="/images/innodb-trx-1.png" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

如图所示是一个简单的 purge_sys 相关的类图，purge_sys 是一个全局对象，所有 purge 相关的信息全部在其中维护。关于 purge_sys 主要包括了一下几项：

### 插入 purge_sys

对于一个读写事务，如果事务中包含了更新操作，则需要将更新操作先关的 undo 插入到 purge_sys（如果事务中只有插入操作，不需要维护 purge_sys，因为不需要利用 undo 构建历史版本）。purge_sys 利用一个优先队列 `purge_queue` 保存待 purge 的 undo，准确的说应该是回滚段，一个回滚段中包含了多个 undo，但是如果该回滚段之前已经加入过队列，则不需要继续添加。

purge_sys 中维护了一个按照 trx_no 有序的优先队列 purge_queue，在事务提交的过程中（commit阶段）会将事务对应的回滚段添加到 purge_queue 中。前面已经说过，并不是每次事务提交的时候都会将回滚段添加到 purge_queue 中，只有当 redo_rseg->last_page_no 为空时（即回滚段上之前不存在未 puge 的 undo），才会添加到 purge_queue 中。若 redo_rseg->last_page_no 非空，则说明该回滚段上存在未 purg 的 undo，此时只会更新回滚段的 TRX_RSEG_HISTORY 列表，将新的 undo 添加到 TRX_RSEG_HISTORY 列表的头部。

为了方便理解，结合回滚段的物理结构进行分析：

<img src="/images/innodb-trx-2.png" width="280px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

新产生的 undo 都需要被链接到 TRX_RSEG_HISTORY 上。

注：回滚段中的 undo 通过一个有序列表进行组织，先生成的 undo 肯定会先被 purge。

在前面的代码中可以看到，在事务提交的过程中，需要调用 `trx_serialisation_number_get` 将需要 purge 的 undo 插入到 purge_sys 的优先队列中。

### purge 推进

InnoDB 内部通过单独的后台线程周期性的进行 undo 的 purge，purge 推进的基本原则是：尽可能的将不再使用的 undo 内容 purge 掉。具体的实现上，基本步骤如下：

```c++
/* srv_start_purge_threads */

|--> srv_purge_coordinator_thread
|--> srv_worker_thread


/* srv_purge_coordinator_thread::trx_purge */
|--> trx_sys->mvcc->clone_oldest_view  // 确认当前最老的 read view
|
|--> trx_purge_attach_undo_recs // 收集指定数据的 undo pages
|    |--> trx_purge_fetch_next_rec // loop
|    |    |--> trx_purge_choose_next_log // !purge_sys->next_stored，寻找下一个 log
|    |    |    |--> purge_sys->rseg_iter->set_next() // 更新 purge_sys->rseg 位置
|    |    |    |--> trx_purge_read_undo_rec // 更新 purge_sys->iter 信息
|    |    |    |    |--> trx_undo_get_first_rec // 总是从 resg->last_page_no 开始，next_stored 置为 true
|    |    |
|    |    |--> trx_undo_build_roll_ptr
|    |    |--> trx_purge_get_next_rec // 寻找下一个 record
|    |    |    |--> trx_undo_page_get_next_rec // 从当前 page 寻找下一个 record
|    |    |    |--> trx_undo_get_next_rec
|    |    |    |    |--> trx_undo_page_get_next_rec
|    |    |    |    |--> trx_undo_get_next_rec_from_next_page // 从下一个 page 寻找，TRX_UNDO_PAGE_NODE
|    |    |    |--> trx_purge_rseg_get_next_history_log // next_stored 置为 false，查找 TRX_UNDO_HISTORY_NODE, 找到下一个 hdr
|
|--> que_fork_scheduler_round_robin
|--> srv_que_task_enqueue_low
|--> que_run_threads
|--> trx_purge_wait_for_workers_to_complete
|
|--> trx_purge_truncate // 清理 undo 文件
|    |--> trx_purge_truncate_history // 遍历 rsegs
|    |    |--> trx_purge_truncate_rseg_history // 遍历 TRX_RSEG_HISTORY


/* srv_worker_thread::::row_purge_step */

|--> row_purge
|    |--> row_purge_parse_undo_rec
|    |--> row_purge_record
|    |    |--> row_purge_record_func
|    |    |    |--> row_purge_del_mark
|    |    |    |--> row_purge_upd_exist_or_extern
```

undo 的 purge 由后台线程完成，后台线程分为两类：coordinator 线程和 worker 线程。coordinator 线程负责收集和分发 undo log，worker 线程负责实际的 purge 工作（coordinator 线程本身也会参与 purge）。

coordinator 线程按照以下的逻辑收集 undo：

1. 从 purge_queue 里面弹出一个元素 TrxUndoRsegs，m_trx_undo_rsegs 指向弹出的元素；
2. 继续从 purge_queue 弹出元素，若弹出元素的 trx_no 和 m_trx_undo_rsegs 的 trx_no 相同，则直接进行合并，否则退出循环；
3. m_iter 指向 m_trx_undo_rsegs，开始遍历，purg_sys->rseg 指向待处理的回滚段；
4. 从待处理的回滚段中依次取 undo_record，若无法取到下一个 undo_record，则需要根据TRX_UNDO_HISTORY_NODE 取下一个 undo，并将 trx_rseg_t 重新 放入 purge_queue 中；
5. m_iter 正常遍历，当遍历到 m_trx_undo_rsegs 尾部时，重复 1；

由于这个逻辑实在有点绕，下面结合 undo page 的内部结构再分析一下：

<img src="/images/innodb-trx-3.png" width="520px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

基本的逻辑如下：

1. trx_rseg_t 上的 last_page_no、last_offset、last_trx_no 指向的是该回滚到对应的第一个未 purge 的 undo，也可以理解为 TRX_RSEG_HISTORY 最尾部（添加时都是添加到头部）的的一个 undo；
2. 拿到这个 undo 后，先在当前页去获取 undo record，如果无法获取，那么就需要通过 TRX_UNDO_PAGE_NODE 找到下一个 page；
3. 如果无法继续找到 undo record，那么说明当前 undo 已经结束了，那么需要根据 TRX_UNDO_HOSTORY_NODE 向前（添加时都是添加到头部）找到下一个 undo，这个时候找到的 undo 不能直接去获取 undo record，而是需要将 trx_rseg_t 指向该 page，然后重新放回优先队列；

### 记录清除

准确的说 purge 过程包括了两个部分：清除记录和清除 undo。在 purge 推进过程中收集到的 undo rec 会被组装成 purge_node_t，通过后台线程进行清理。

<img src="/images/innodb-trx-4.png" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

基本的执行路径如下：

```c++
|--> row_purge_step
|    |--> row_purge
|    |    |--> row_purge_record  // row_purge_record_func, loop
|    |    |    |--> row_purge_del_mark
|    |    |    |    |--> row_purge_remove_multi_sec_if_poss
|    |    |    |    |--> row_purge_remove_clust_if_poss
|    |    |    |    |    |--> row_purge_remove_clust_if_poss_low
|    |    |    |    |    |    |--> btr_cur_optimistic_delete
|    |    |    |    |    |    |--> btr_cur_pessimistic_delete
```

## Undo 管理

### Undo 分配

关于 Undo 的分配，参考之前的文章。

### Undo 回收

前面在介绍 purge_sys 的时候，主要分析了 purge 过程中如何收集需要被 purge 的 undo 内容，这部分重点介绍 undo 的具体回收过程。

```c++
|--> trx_purge_truncate // 清理 undo 文件
|    |--> trx_purge_truncate_history // 遍历 rsegs
|    |    |--> trx_purge_truncate_rseg_history // 遍历 TRX_RSEG_HISTORY
|    |    |    |--> trx_undo_truncate_start  // 正在处理的 trx_no
|    |    |    |    |--> trx_undo_empty_header_page
|    |    |    |    |--> trx_undo_free_page
|    |    |    |
|    |    |    |--> trx_purge_free_segment  // 已经处理完的 trx_no
|    |    |    |    |--> fseg_free_step_not_header // 清理除 header 外的内容
|    |    |    |    |    |--> fseg_free_page_low
|    |    |    |    |--> trx_purge_remove_log_hdr // 从回滚段 TRX_RSEG_HISTORY 移除
|    |    |    |    |--> fseg_free_step // 清理 page
|    |    |    |    |    |--> fseg_free_page_low
|    |    |    |    |    |--> fsp_free_seg_inode
|    |    |    |
|    |    |    |--> trx_purge_remove_log_hdr
```

## 参考文献

> http://mysql.taobao.org/monthly/2015/04/01/