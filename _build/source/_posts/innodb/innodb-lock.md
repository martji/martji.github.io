---
title: InnoDB 锁机制
date: 2020-12-05 21:11:33
categories: 
- InnoDB

---

本文主要对 InnoDB 层的锁机制进行说明。

<!-- more -->

## 行级锁的基本概念

### 数据结构

<img src="/images/innodb-lock-1.png" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

注释：

1. lock_t 是一个内存结构，内存从 trx_lock_t 上进行分配，也可以理解为在 trx 上进行分配
2. lock_sys_t 是一个全局的数据结构，通过 hash_table_t 保存了所有分配的 lock_t
3. 锁等待借助于 srv_slot_t 中的 os_event_t 来实现，lock_sys_t 中也保存了所有使用的 srv_slot_t
4. trx_lock_t 的 blocking_trx 中保存了 trx 间的锁等待关系，标记当前 trx 在等待另外一个 trx 分配的锁
5. trx_lock_t 的 wait_thr 字段标记了等待当前锁的线程

### 行级锁类型

```bash
// 1. LOCK_REC_NOT_GAP : 记录锁

// 2. LOCK_GAP : 区间锁

// 3. LOCK_ORDINARY(Next-Key Lock) : 包含记录本身及记录之前的 GAP
```

### 源码分析

加行锁的入口包含 sel_set_rec_lock 和 row_ins_set_rec_lock，基本调用路径如下：

```c++
|--> sel_set_rec_lock
|    |--> btr_pcur_get_block
|    |
|    |--> lock_clust_rec_read_check_and_lock
|    |    |--> page_rec_get_heap_no
|    |    |--> lock_rec_convert_impl_to_expl  // 隐式锁升级
|    |    |    |--> lock_clust_rec_some_has_impl
|    |    |    |--> trx_rw_is_active
|    |    |    |--> lock_rec_convert_impl_to_expl_for_trx
|    |    |    |    |--> lock_rec_add_to_queue
|    |    |    |    |    |--> rec_lock.create
|    |    |    |    |    |    |--> lock_alloc
|    |    |    |    |    |    |--> lock_add // 维护 hash + trx->lock->trx_locks 列表
|    |    |
|    |    |--> lock_rec_lock
|    |    |    |--> lock_rec_lock_fast
|    |    |    |    |--> lock_rec_get_first_on_page
|    |    |    |
|    |    |    |--> lock_rec_lock_slow
|    |    |    |    |--> lock_rec_has_expl
|    |    |    |    |--> lock_rec_other_has_conflicting  // 锁冲突检查
|    |    |    |    |    |--> lock_rec_has_to_wait  // Lock_iter::for_each(rec_id, ...) 按照 rec_id 遍历 hash, 检查是否需要等待锁
|    |    |    |    |
|    |    |    |    |--> rec_lock.add_to_waitq  // 创建锁等待关系
|    |    |    |    |    |--> create
|    |    |    |    |    |--> lock_create_wait_for_edge
|    |    |    |    |    |--> set_wait_state
|    |    |    |    |    |    |--> que_thr_stop
|    |    |    |    |
|    |    |    |    |--> lock_rec_add_to_queue
|    |
|    |--> lock_sec_rec_read_check_and_lock
|    |    |--> page_rec_get_heap_no
|    |    |--> lock_rec_convert_impl_to_expl
|    |    |--> lock_rec_lock



|--> row_ins_set_rec_lock
|    |--> lock_clust_rec_read_check_and_lock
|    |
|    |--> lock_sec_rec_read_check_and_lock



|--> lock_clust_rec_modify_check_and_lock
|    |--> lock_rec_convert_impl_to_expl
|    |
|    |--> lock_rec_lock
```

锁等待判断：

```c++
|--> lock_rec_has_to_wait
```

锁等待处理入口：

```c++
|--> lock_wait_suspend_thread
|    |--> trx_lock_wait_timeout_get
|    |
|    |--> lock_wait_table_reserve_slot  // slot 绑定到 thr
|    |--> os_event_wait(slot->event)
|    |--> lock_wait_table_release_slot
```

锁释放入口：

```c++
|--> trx_commit_in_memory
|    |--> trx_release_impl_and_expl_locks
|    |    |--> lock_trx_release_locks
|    |    |    |--> lock_release
|    |    |    |    |--> lock_rec_dequeue_from_page
|    |    |    |    |    |--> lock_rec_discard
|    |    |    |    |    |--> lock_rec_grant // 遍历 page，查看是否有需要唤醒的锁等待
|    |    |    |    |    |    |--> lock_grant_or_update_wait_for_edge_if_waiting
|    |    |    |    |    |    |    |--> lock_grant_or_update_wait_for_edge
|    |    |    |    |    |    |    |    |--> lock_has_to_wait_in_queue // 检查是否需要继续等待
|    |    |    |    |    |    |    |    |--> lock_grant // 解锁，唤醒 trx->lock->wait_thr
|    |    |    |    |    |    |    |    |    |--> lock_reset_wait_and_release_thread_if_suspended
|    |    |    |    |    |    |    |    |    |    |--> lock_reset_lock_and_trx_wait
|    |    |    |    |    |    |    |    |    |    |--> lock_wait_release_thread_if_suspended
|    |    |    |    |    |    |    |    |    |    |    |--> que_thr_end_lock_wait
|    |    |    |    |    |    |    |    |    |    |    |--> os_event_set(thr->slot->event)
|    |    |    |    |    |    |    |    |--> lock_update_wait_for_edge 
|    |    |    |    |    |    |
|    |    |    |    |    |    |--> lock_grant_cats
|    |    |    |    |
|    |    |    |    |--> lock_table_dequeue
```

## 参考文献

> InnoDB 事务锁系统简介：http://mysql.taobao.org/monthly/2016/01/01/
>
> InnoDB 锁子系统浅析：http://mysql.taobao.org/monthly/2017/12/02/
>
> InnoDB 行锁分析：http://mysql.taobao.org/monthly/2018/05/04/
>
> https://time.geekbang.org/column/article/75659