---
title: Undo Log 的一些基本概念
date: 2020-12-05 21:17:42
categories: 
- InnoDB

---

本文主要对 InnoDB 的 Undo Log 进行介绍。

<!-- more -->

## 逻辑结构

<img src="/images/innodb-undo-log-1.png" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

## 物理结构

### 回滚段

<img src="/images/innodb-undo-log-2.png" width="680px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

InnoDB会为每一个读写事务分配回滚段，临时表有单独的回滚段。每一个回滚段内有两个重要的数据结构：

1. Undo slots, 可以理解为undo page的槽位，1个回滚段内部有1024个槽位，每个槽位可以指向`FIL_NULL`或者指向一个真实的undo page地址；回滚段为每个读写事务分配槽位时，只能分配指向`FIL_NULL`的槽位；
2. TRX_RSEG_HISTORY，维护了一个undo log链表，与undo log中的TRX_UNDO_HISTORY_NODE关联，添加时加到头部；

### undo page

<img src="/images/innodb-undo-log-3.png" width="680px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

按照上文的描述，InnoDB为每1个读写事务分配一个undo slot槽位，对应的槽位会指向一个undo page。undo page可以分为两个部分：undo page header 和 undo log。undo page header是一个公共的区域，一个undo page内部可能会包含多个undo log，还可能出现undo page的复用。对于undo page header，主要的信息包括：

1. TRX_UNDO_PAGE_FREE：指向当前page的空闲位置
2. TRX_UNDO_LAST_LOG：指向上一个undo log的开始位置
3. TRX_UNDO_STATE：当前undo page状态
4. TRX_UNDO_PAGE_LIST：维护一个undo page的链表，与TRX_UNDO_PAGE_NODE关联，添加时加到尾部

### undo log

<img src="/images/innodb-undo-log-4.png" width="680px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

undo内部包括两个部分：undo log header 和 undo  log record，其中undo log header主要信息包括：

1. TRX_UNDO_LOG_START：当前undo log内容(不含header)的开始位置，undo->hdr_offset指向的是包含了header的开始位置，undo->top_offset指向的是log record(不含header)的开始位置

## 生命周期

在介绍undo log的生命周期前，先回答下面几个问题。

1）什么时候申请回滚段？

A：这个问题前面已经提到，当事务升级为读写事务时，会自动的为事务分配回滚段。

2）undo log的写入过程？

A：读写事务分配好回滚段只是相当于做了一个占位，当有IDU操作发生时，才会开始记录undo。记录undo前，首先需要为事务绑定undo slot，寻找undo slot前，首先会去查看回滚段上是否有被cached的undo page，如果有的话，则可以直接拿来使用；如果没有则需要找到一个指向`FIL_NULL`的，然后生成一个新的undo page。

不管是cached的undo page还是新生成的undo page，在写入真实的undo log record前，会提前生成undo log header。然后根据具体的操作，记录相应的record。最后生成roll_ptr，最终绑定到数据行记录上。

3）undo log的回收过程？

A：对于undo log的回收，插入操作和非插入操作的处理流程不同，对于插入操作，可以在事务提交之后，直接清理掉undo log(或者进入cached队列)；对于更新/删除操作，需要后台的purge线程进行删除，事务提交时仅将undo log加入到回滚段的history列表中(或者进入cached队列)，并且会插入一个元素到purge_sys内的最小堆中。purge线程会异步的进行删除。

### 初始化

undo的初始化在innodb启动的时候进行，基本的入口在 srv_start() ：

```c++
|--> srv_start
|    |    // Init
|    |--> srv_undo_tablespaces_init  // create
|    |    |--> srv_undo_tablespaces_create
|    |    |    |--> srv_undo_tablespace_create
|    |    |    |--> srv_undo_tablespace_open
|    |--> trx_sys_create_sys_pages
|    |--> trx_sys_init_at_db_start
|    |    |--> trx_rsegs_init
|    |    |    |--> trx_rseg_mem_create
|    |--> trx_purge_sys_create
|    |
|    |    // Start
|    |--> recv_recovery_from_checkpoint_start
|    |--> recv_recovery_from_checkpoint_finish
|    |--> srv_undo_tablespaces_init  // open
|    |    |--> srv_undo_tablespaces_open
|    |    |    |--> srv_undo_tablespace_open_by_num
|    |    |    |    |--> srv_undo_tablespace_open
|    |--> trx_sys_init_at_db_start  
|    |--> trx_purge_sys_create
|    |
|    |--> trx_rseg_adjust_rollback_segments
|    |    |--> trx_rseg_add_rollback_segments
|    |    |    |--> trx_rseg_create
|    |    |    |--> trx_rseg_mem_create
  
```

### 数据写入

主要代码入口如下：

```c++
// 申请回滚段
void trx_assign_rseg_durable(trx_t *trx) {
  ut_ad(trx->rsegs.m_redo.rseg == nullptr);

  trx->rsegs.m_redo.rseg = srv_read_only_mode ? nullptr : get_next_redo_rseg();
}


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



```c++
// 写入undo log
dberr_t trx_undo_report_row_operation(...) {
  ...
  err = trx_undo_assign_undo(trx, undo_ptr, TRX_UNDO_UPDATE);
  
  ...
  offset = trx_undo_page_report_modify(undo_page, trx, index, rec, offsets,
                                       update, cmpl_info, clust_entry, &mtr);
  
  ...
  *roll_ptr = trx_undo_build_roll_ptr(op_type == TRX_UNDO_INSERT_OP,
                                      undo_ptr->rseg->space_id, page_no, offset);
  
  ...
}

// 生成undo page
dberr_t trx_undo_assign_undo(
    trx_t *trx,               /*!< in: transaction */
    trx_undo_ptr_t *undo_ptr, /*!< in: assign undo log from
                              referred rollback segment. */
    ulint type)               /*!< in: TRX_UNDO_INSERT or
                              TRX_UNDO_UPDATE */
{
  ...
  undo = trx_undo_reuse_cached(trx, rseg, type, trx->id, trx->xid, &mtr);
    
  ...
  err = trx_undo_create(trx, rseg, type, trx->id, trx->xid, &undo, &mtr);
}

static MY_ATTRIBUTE((warn_unused_result)) dberr_t
    trx_undo_create(trx_t *trx, trx_rseg_t *rseg, ulint type, trx_id_t trx_id,
                    const XID *xid, trx_undo_t **undo, mtr_t *mtr) {
  ...
  rseg_header = trx_rsegf_get(rseg->space_id, rseg->page_no, rseg->page_size, mtr);

  err = trx_undo_seg_create(rseg, rseg_header, type, &id, &undo_page, mtr);

  ...
  page_no = page_get_page_no(undo_page);

  offset = trx_undo_header_create(undo_page, trx_id, mtr);

  trx_undo_header_add_space_for_xid(undo_page, undo_page + offset, mtr);

  *undo = trx_undo_mem_create(rseg, id, type, trx_id, xid, page_no, offset);
  ...
}
```

从 MySQL 5.6 开始，MySQL 引入了独立的 undo 表空间，可以通过参数 `innodb_undo_tablespaces` 参数配置独立的 undo 表空间的个数，undo 可以不再通过 ibdata 进行管理（这也极大的减小了 ibdata 的大小）。从 MySQL 5.7 开始支持 truncate undo 表空间，进一步解决了 undo 空间膨胀后无法回收的问题。

在开启独立 undo 表空间的情况下，每个表空间内部包含了 128 个回滚段，读写事务在申请 undo 回滚段时，总是在不同的表空间交替进行分配。

当写操作（插入+更新）操作需要记录 undo 时，会通过已经分配的回滚段找到一个可用的 undo 。在前面的类图中可以看到，每个回滚段上对于不同类型的 undo 保存了两个列表：正在使用的列表和 cached 列表。在分配 undo page 时，会优先从 cached 列表中寻找，如果找不到的话再去分配一个新的 undo 。

前面也提到了，每个回滚段上都有一个 undo slots 数组，数组的长度为 1024，每个 slot 中保存的是一个 undo page 的地址，默认为空（FIL_NULL）。在分配一个新的 undo 时，首先需要找到一个为空的 slot，然后再去创建新的 page，并且将 slot 指向这个 page。结合代码分析一下：

```c++
|--> trx_undo_seg_create
|    |--> trx_rsegf_undo_find_free  // 寻找一个 free 的 slot
|    |
|    |--> fsp_reserve_free_extents
|    |--> fseg_create_general  // 创建一个新的 undo page
|    |--> fil_space_release_free_extents
|    |
|    |--> trx_undo_page_init  // 初始化
|    |--> flst_init  // 维护 TRX_UNDO_PAGE_LIST 列表
|    |--> flst_add_last
|    |
|    |--> trx_rsegf_set_nth_undo  // 修改 slot 指针
```

### 事务提交

事务提交过程中的undo处理过程可以简化为以下逻辑：

```c++
|--> ha_commit_trans
|    |--> trx_prepare_for_mysql
|    |    |--> trx_undo_set_state_at_prepare
|    |
|    |--> trx_commit_for_mysql
|    |    |--> trx_write_serialisation_history
|    |    |    |--> trx_serialisation_number_get
|    |    |    |--> trx_undo_set_state_at_finish
|    |    |    |--> trx_undo_update_clean
|    |    |
|    |    |--> trx_commit_in_memory
|    |    |    |--> trx_undo_insert_clean
```

其中，在trx_serialisation_number_get步骤中，会生成TrxUndoRsegs对象，插入到purge_sys->purge_queue中：

```c++
  if ((redo_rseg != NULL && redo_rseg->last_page_no == FIL_NULL) ||
      (temp_rseg != NULL && temp_rseg->last_page_no == FIL_NULL)) {
    TrxUndoRsegs elem(trx->no);

    if (redo_rseg != NULL && redo_rseg->last_page_no == FIL_NULL) {
      elem.push_back(redo_rseg);
    }

    if (temp_rseg != NULL && temp_rseg->last_page_no == FIL_NULL) {
      elem.push_back(temp_rseg);
    }

    mutex_enter(&purge_sys->pq_mutex);

    trx_sys_mutex_exit();

    purge_sys->purge_queue->push(elem);

    mutex_exit(&purge_sys->pq_mutex);
  }
```

在 trx_undo_set_state_at_finish 步骤时，会判断当前 undo 的状态：

```c++
page_t *trx_undo_set_state_at_finish(...) {
  ...
  undo_page = trx_undo_page_get(page_id_t(undo->space, undo->hdr_page_no),
                                undo->page_size, mtr);

  seg_hdr = undo_page + TRX_UNDO_SEG_HDR;
  page_hdr = undo_page + TRX_UNDO_PAGE_HDR;

  if (undo->size == 1 && mach_read_from_2(page_hdr + TRX_UNDO_PAGE_FREE) <
                             TRX_UNDO_PAGE_REUSE_LIMIT) {
    state = TRX_UNDO_CACHED;

  } else if (undo->type == TRX_UNDO_INSERT) {
    state = TRX_UNDO_TO_FREE;
  } else {
    state = TRX_UNDO_TO_PURGE;
  }

  undo->state = state;

  mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, state, MLOG_2BYTES, mtr);

  return (undo_page);
}
```

在trx_undo_update_clean步骤时，会将undo log加入到对应回滚段上的history列表：

```c++
void trx_undo_update_cleanup(...) {
  ...
  trx_purge_add_update_undo_to_history(
      trx, undo_ptr, undo_page, update_rseg_history_len, n_added_logs, mtr);
  
  ...
  if (undo->state == TRX_UNDO_CACHED) {
    UT_LIST_ADD_FIRST(rseg->update_undo_cached, undo);

    MONITOR_INC(MONITOR_NUM_UNDO_SLOT_CACHED);
  } else {
    ut_ad(undo->state == TRX_UNDO_TO_PURGE);

    trx_undo_mem_free(undo);
  }
}

void trx_purge_add_update_undo_to_history(...) {
  ...
  if (undo->state != TRX_UNDO_CACHED) {
    ...
    trx_rsegf_set_nth_undo(rseg_header, undo->id, FIL_NULL, mtr);

    ...
  }
  
  /* Add the log as the first in the history list */
  flst_add_first(rseg_header + TRX_RSEG_HISTORY,
                 undo_header + TRX_UNDO_HISTORY_NODE, mtr);
  
  ...
  if (rseg->last_page_no == FIL_NULL) {
    rseg->last_page_no = undo->hdr_page_no;
    rseg->last_offset = undo->hdr_offset;
    rseg->last_trx_no = trx->no;
    rseg->last_del_marks = undo->del_marks;
  }
}
```

### 后台purge

```c++
/* srv_start_purge_threads */

|--> srv_purge_coordinator_thread
|--> srv_worker_thread


/* srv_purge_coordinator_thread::trx_purge */

|--> trx_purge_attach_undo_recs // 收集指定数据的 undo pages
|    |--> trx_purge_fetch_next_rec // loop
|    |    |--> trx_purge_choose_next_log // 寻找下一个 log
|    |    |    |--> purge_sys->rseg_iter->set_next() // 更新 purge_sys->rseg 位置
|    |    |    |--> trx_purge_read_undo_rec // 更新 purge_sys->iter 信息
|    |    |    |    |--> trx_undo_get_first_rec
|    |    |
|    |    |--> trx_undo_build_roll_ptr
|    |    |--> trx_purge_get_next_rec // 寻找下一个 record
|    |    |    |--> trx_undo_page_get_next_rec // 从当前 page 寻找下一个 record
|    |    |    |--> trx_undo_get_next_rec
|    |    |    |    |--> trx_undo_page_get_next_rec
|    |    |    |    |--> trx_undo_get_next_rec_from_next_page // 从下一个 page 寻找，TRX_UNDO_PAGE_NODE
|    |    |    |--> trx_purge_rseg_get_next_history_log // 查找 TRX_UNDO_HISTORY_NODE, 找到下一个 hdr
|
|--> que_fork_scheduler_round_robin
|--> srv_que_task_enqueue_low
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

purge_sys 中维护了一个按照 trx_no 有序的小顶堆结构 purge_queue，在事务提交的过程中（commit阶段）会将事务对应的回滚段添加到 purge_queue 中。需要注意，并不是每次事务提交的时候都会将回滚段添加到 purge_queue 中，只有当 redo_rseg->last_page_no 为空时（即回滚段上之前不存在未 puge 的 undo），才会添加到 purge_queue 中。

若 redo_rseg->last_page_no 非空，则说明该回滚段上存在未 purg 的 undo，此时只会更新回滚段的 TRX_RSEG_HISTORY 列表，将新的 undo 添加到 TRX_RSEG_HISTORY 列表的头部。

coordinator 线程按照以下的逻辑收集 undo：

1. 从 purge_queue 里面弹出一个元素 TrxUndoRsegs，m_trx_undo_rsegs 指向弹出的元素；
2. 继续从 purge_queue 弹出元素，若弹出元素的 trx_no 和 m_trx_undo_rsegs 的 trx_no 相同，则直接进行合并，否则退出循环；
3. m_iter 指向 m_trx_undo_rsegs，开始遍历，purg_sys->rseg 指向待处理的回滚段；
4. 从待处理的回滚段中依次取 undo_record，若无法取到下一个 undo_record，则需要根据TRX_UNDO_HISTORY 的 next 指针，取下一个 trx 的 undo，并将 trx_rseg_t 重新 放入 purge_queue 中；
5. m_iter 正常遍历，当遍历到 m_trx_undo_rsegs 尾部时，重复 1；

## 参考文献

>http://mysql.taobao.org/monthly/2015/04/01/
>
>http://mysql.taobao.org/monthly/2018/03/01/
>
>http://mysql.taobao.org/monthly/2017/12/01/

## 附：undo格式

<img src="/images/innodb-undo-log-5.png" width="680px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>