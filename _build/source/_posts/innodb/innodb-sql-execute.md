---
title: InnoDB 中 SQL 执行过程
date: 2020-12-05 21:06:52
categories: 
- InnoDB

---

前面 MySQL 中的 SQL 执行过程中，已经简单介绍了 1 条 SQL 在 MySQL Server 层的执行路径，本文继续介绍 1 条 SQL 在 InnoDB 层的执行路径。

<!-- more -->

## 读操作

对于读操作，InnnoDB 的主要处理逻辑在 row_search_for_mysql (row_search_mvcc) 方法中，基本的步骤整理如下（基于 5.6 版本）：

```c++
|--> row_search_mvcc
|    // 1. check prefetch cache
|    |--> row_sel_dequeue_cached_row_for_mysql
|    
|    // 2. check AHI
|    |--> row_sel_try_search_shortcut_for_mysql
|    |    |--> btr_pcur_open_with_no_init
|    |    |    |--> btr_cur_search_to_nth_level
|    |    |    |    |--> btr_search_guess_on_hash
|    |    |    |    |--> ...
|    
|    // 3. open/restore index
|    |--> sel_restore_position_for_mysql
|    |    |--> btr_pcur_restore_position
|    |
|    |--> btr_pcur_open_with_no_init
|    |    |--> btr_cur_search_to_nth_level
|    |    |    |--> btr_search_guess_on_hash
|    |    |    |--> buf_page_get_gen
|    |    |    |--> page_cur_search_with_match
|    |    |    |    |--> page_dir_get_nth_slot
|    |    |    |    |--> page_dir_slot_get_rec
|    |    |    |    |--> cmp_dtuple_rec_with_match
|    |
|    |--> btr_pcur_open_at_index_side
|    |    |--> btr_cur_open_at_index_side
|    |    |    |--> buf_page_get_gen
|    
|    // 4. match record (mvcc check)
|    |--> lock_clust_rec_cons_read_sees
|    |    |--> view->changes_visible
|    |--> row_sel_build_prev_vers_for_mysql
|    |    |--> row_vers_build_for_consistent_read // loop
|    |    |    |--> trx_undo_prev_version_build
|    |    |    |    |--> trx_undo_get_undo_rec
|    |    |    |    |--> trx_undo_update_rec_get_update
|    |    |    |    |--> row_upd_rec_in_place
|    |    |    |--> /* modifications_visible */
|    |
|    |--> lock_sec_rec_cons_read_sees
|    |--> row_sel_get_clust_rec_for_mysql
|    |    |--> lock_clust_rec_cons_read_sees
|    |    |--> row_sel_build_prev_vers_for_mysql
|    |
|    |--> row_sel_store_mysql_rec
|    
|    // 5. move cousor to next index record
```

其中：

1. btr_search_guess_on_hash 方法会从 AHI 中进行查询

2. buf_page_get_gen 方法会从指定的 (space, offset) 获取1个page，并放入 BP 中，基本步骤如下：

```c++
/* buf_page_get_gen */

|--> buf_page_hash_get_low // 1. 查看 page 是否在 page_hash 中
|
|--> buf_page_read // 2. 从文件中读取 page
|    |--> buf_page_read_low
|    |    |--> buf_page_init_for_read
|    |    |    |--> buf_LRU_get_free_block // 获取1个新的block
|    |    |    |--> buf_page_hash_get_low // 再次检查 page_hash 中是否存在
|    |    |    |--> fil_tablespace_deleted_or_being_deleted_in_mem // 检查tablespace是否删除
|    |    |    |--> buf_page_init
|    |    |    |    |--> buf_block_init_low
|    |    |    |    |--> buf_page_init_low
|    |    |    |    |--> HASH_INSERT
|    |    |    |--> buf_page_set_io_fix
|    |    |    |--> buf_LRU_add_block
|    |    |
|    |    |--> _fil_io
|    |    |
|    |    |--> buf_page_io_complete
|    |    |    |--> ibuf_merge_or_delete_for_page
|
|--> buf_block_fix
|
|--> buf_page_make_young_if_needed
|
|--> mtr_memo_push / mtr_add_page
|
|--> buf_read_ahead_linear
```

以上只是 InnoDB 读操作的一个基本流程，具体内容待后续继续分析。

## 写操作

基于 8.0 版本

### 更新操作

```c++
|--> row_update_for_mysql
|    |--> row_update_for_mysql_using_upd_graph
|    |    |--> row_upd_step
|    |    |    |--> row_upd
|    |    |    |    |--> row_upd_clust_step // 更新主键
|    |    |    |    |    |--> btr_pcur_restore_position // 恢复cursor，找到待更新的页，记录到mtr
|    |    |    |    |    |    |--> btr_cur_optimistic_latch_leaves
|    |    |    |    |    |    |--> dict_index_build_data_tuple
|    |    |    |    |    |    |--> open_no_init
|    |    |    |    |    |    |    |--> btr_cur_search_to_nth_level
|    |    |    |    |    |
|    |    |    |    |    |--> row_upd_del_mark_clust_rec // 标记删除
|    |    |    |    |    |    |--> btr_cur_del_mark_set_clust_rec
|    |    |    |    |    |    |    |--> trx_undo_report_row_operation
|    |    |    |    |    |    |    |--> btr_rec_set_deleted_flag // 修改记录
|    |    |    |    |    |    |    |--> row_upd_rec_sys_fields
|    |    |    |    |    |  
|    |    |    |    |    |--> row_upd_clust_rec // 原地更新
|    |    |    |    |    |    |--> btr_cur_update_in_place
|    |    |    |    |    |    |    |--> btr_cur_upd_lock_and_undo
|    |    |    |    |    |    |    |    |--> trx_undo_report_row_operation
|    |    |    |    |    |    |    |
|    |    |    |    |    |    |    |--> row_upd_rec_sys_fields
|    |    |    |    |    |    |    |--> row_upd_rec_in_place // 修改记录
|    |    |    |    |    |    |    |--> btr_cur_update_in_place_log
|    |    |    |    |    |    |    |    |--> row_upd_write_sys_vals_to_log
|    |    |    |    |    |    |
|    |    |    |    |    |    |--> btr_cur_optimistic_update
|    |    |    |    |    |    |    |--> btr_cur_update_in_place
|    |    |    |    |    |    |    |
|    |    |    |    |    |    |    |--> page_cur_delete_rec
|    |    |    |    |    |    |    |--> row_upd_index_entry_sys_field
|    |    |    |    |    |    |    |--> btr_cur_insert_if_possible
|    |    |    |    |    |    |
|    |    |    |    |    |    |--> btr_cur_pessimistic_update
|    |    |    |    |    |
|    |    |    |    |    |--> row_upd_clust_rec_by_insert // 删除 + 插入
|    |    |    |    |    |    |--> btr_cur_del_mark_set_clust_rec
|    |    |    |    |    |    |    |--> trx_undo_report_row_operation
|    |    |    |    |    |    |--> row_ins_clust_index_entry
|    |    |    |    |    |    |    |--> row_ins_clust_index_entry_low
|    |    |    |    |    |    |    |    |--> btr_cur_optimistic_insert
|    |    |    |    |    |    |    |    |    |--> btr_cur_ins_lock_and_undo
|    |    |    |    |    |    |    |    |    |    |--> trx_undo_report_row_operation
|    |    |    |    |
|    |    |    |    |--> row_upd_sec_step // 更新二级索引
|    |    |    |    |    |--> row_upd_del_multi_sec_index_entry
|    |    |    |    |    |--> row_upd_multi_sec_index_entry
|    |    |    |    |    |--> row_upd_sec_index_entry
|    |    |    |    |    |    |-->row_upd_sec_index_entry_low // 删除 + 插入
|    |    |    |    |    |    |    |--> row_build_index_entry
|    |    |    |    |    |    |    |--> row_search_index_entry
|    |    |    |    |    |    |    |--> btr_cur_del_mark_set_sec_rec
|    |    |    |    |    |    |    |--> row_build_index_entry
|    |    |    |    |    |    |    |--> row_ins_sec_index_entry
```

### 插入操作

```c++
|--> row_insert_for_mysql
|    |--> row_insert_for_mysql_using_ins_graph
|    |    |--> row_get_prebuilt_insert_row
|    |    |    |--> ins_node_set_new_row
|    |    |    |    |--> row_ins_alloc_sys_fields
|    |    |
|    |    |--> row_mysql_convert_row_to_innobase
|    |    |
|    |    |--> row_ins_step
|    |    |    |--> row_ins
|    |    |    |    |--> row_ins_index_entry_step
|    |    |    |    |    |--> row_ins_index_entry
|    |    |    |    |    |    |--> row_ins_clust_index_entry // 插入主键
|    |    |    |    |    |    |    |--> row_ins_clust_index_entry_low
|    |    |    |    |    |    |    |    |--> row_ins_clust_index_entry_by_modify // 可以复用
|    |    |    |    |    |    |    |    |--> btr_cur_optimistic_insert // 乐观插入
|    |    |    |    |    |    |    |    |    |--> btr_cur_ins_lock_and_undo
|    |    |    |    |    |    |    |    |    |    |--> trx_undo_report_row_operation
|    |    |    |    |    |    |    |    |    |    |--> row_upd_index_entry_sys_field
|    |    |    |    |    |    |    |    |    |--> page_cur_tuple_insert // 插入记录
|    |    |    |    |    |    |    |    |    |    |--> page_cur_insert_rec_low
|    |    |    |    |    |    |    |    |--> btr_cur_pessimistic_insert // 悲观插入
|    |    |    |    |    |    |
|    |    |    |    |    |    |--> row_ins_sec_index_multi_value_entry
|    |    |    |    |    |    |
|    |    |    |    |    |    |--> row_ins_sec_index_entry // 插入二级索引
|    |    |    |    |    |    |    |--> row_ins_sec_index_entry_low
|    |    |    |    |    |    |    |    |--> btr_cur_search_to_nth_level // 查找插入的位置
|    |    |    |    |    |    |    |    |--> btr_cur_optimistic_insert
```

其中：

trx_undo_report_row_operation 会进行 undo 的记录和数据的修改，基本步骤如下：

```c++
/* trx_undo_report_row_operation */

|--> trx_undo_assign_undo // 1. 分配 undo 空间

|--> buf_page_get_gen  // 2. 获取 undo page

|--> trx_undo_page_report_insert
|--> trx_undo_page_report_modify
```

