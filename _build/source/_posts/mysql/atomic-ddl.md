---
title: MySQL Atomic DDL
date: 2020-12-05 20:03:12
tags:
- Feature
categories: 
- MySQL

---

Atomic DDL 是 MySQL 8.0 的重要特征之一，关于 Atomic DDL 的介绍文档，流传比较广泛的就是参考文献中提到的两篇文章，文章中对 Atomic DDL 的实现思想和基本过程进行了介绍，建议大家阅读一下。本文尝试对完整的建表/删表过程进行分析。

<!-- more -->

## 建表过程

```c++
|--> mysql_create_table
|    |--> open_tables
|    |    |--> open_and_process_table
|    |    |    |--> open_table  // 打开表
|    |    |    |    |--> check_if_table_exists
|    |    |    |    |    |--> dd::table_exists
|    |    |    |    |    |    |--> client->acquire // dd::Abstract_table
|    |    |    |    |
|    |    |    |    |--> tc->get_table  //  从 Table_cache_manager 查找
|    |    |    |    |
|    |    |    |    |--> get_table_share_with_discover  // 构建 TABLE_SHARE
|    |    |    |    |    |--> get_table_share
|    |    |    |    |    |    |--> table_def_cache->find  // 从 Table_definition_cache 查找
|    |    |    |    |    |    |--> process_found_table_share
|    |    |    |    |    |    |
|    |    |    |    |    |    |--> alloc_table_share
|    |    |    |    |    |    |--> table_def_cache->emplace  // 维护 Table_definition_cache
|    |    |    |    |    |    |
|    |    |    |    |    |    |--> thd->dd_client()->acquire
|    |    |    |    |    |    |--> open_table_def
|    |    |    |    |
|    |    |    |    |--> open_table_from_share  // 通过 TABLE_SHARE 打开 TABLE
|    |    |    |    |    |--> ha_open
|    |    |    |    |    |    |--> ha_innobase::open // 打开一个 innodb 表
|    |    |    |    |    |    |    |--> lookup_table_handler
|    |    |    |    |    |    |    |--> dict_table_check_if_in_cache_low
|    |    |    |    |    |    |    |--> dd_open_table
|    |    |    |    |    |    |    |    |--> dd_open_table_one
|    |    |    |    |    |    |    |
|    |    |    |    |    |    |    |--> row_create_prebuilt
|    |    |    |    |    |    |    |--> info(...)
|    |    |    |    |    |    |    |    |--> info_low
|    |    |    |    |    |    |    |    |    |--> update_thd
|    |    |    |    |    |    |    |    |    |    |--> row_update_prebuilt_trx
|    |    |    |    |
|    |    |    |    |--> tc->add_used_table  // 添加到 Table_cache_manager
|    |    |    |    |
|    |    |    |    |--> thd->set_open_tables
|    |    |    |    |--> table->init
|    |
|    |--> mysql_create_table_no_lock
|    |    |--> create_table_impl
|    |    |    |--> check_engine
|    |    |    |--> set_table_default_charset
|    |    |    |--> get_new_handler
|    |    |    |
|    |    |    |--> check_if_table_exists
|    |    |    |
|    |    |    |--> mysql_prepare_create_table
|    |    |    |
|    |    |    |--> rea_create_base_table
|    |    |    |    |--> dd::create_table
|    |    |    |    |    |--> dict->get_dd_table
|    |    |    |    |    |--> create_dd_user_table
|    |    |    |    |    |--> fill_dd_table_from_create_info
|    |    |    |    |    |    |--> fill_dd_columns_from_create_fields
|    |    |    |    |    |    |--> fill_dd_indexes_from_keyinfo
|    |    |    |    |    |    |--> file->get_extra_columns_and_keys
|    |    |    |    |
|    |    |    |    |--> thd->dd_client()->store // dd::Table
|    |    |    |    |--> thd->dd_client()->acquire_for_modification
|    |    |    |    |
|    |    |    |    |--> ha_create_table
|    |
|    |--> trans_commit_stmt
|    |--> trans_commit_implicit
|    |    |--> thd->dd_client()->commit_modified_objects
|    |    |    |--> remove_uncommitted_objects // dd::Abstract_table + dd::Tablespace
|    |    |    |    |--> invalidate
|    |
|    |--> trans_rollback_stmt
|    |--> trans_rollback
|    |    |--> thd->dd_client()->rollback_modified_objects
|    |    |    |--> remove_uncommitted_objects
|    |
|    |--> post_ddl
```



```c++
|--> create_impl
|    |--> create_table
|    |    |--> create_table_def
|    |    |    |--> dict_mem_table_create
|    |    |    |--> row_create_table_for_mysql
|    |    |    |    |--> dict_build_table_def
|    |    |    |    |    |--> dict_build_table_space_for_table
|    |    |    |    |    |    |--> log_ddl->write_delete_space_log
|    |    |    |    |    |    |--> fil_ibd_create
|    |    |    |    |
|    |    |    |    |--> dict_table_add_system_columns
|    |    |    |    |
|    |    |    |    |--> dict_table_add_to_cache
|    |    |    |    |--> log_ddl->write_remove_cache_log
|    |    |
|    |    |--> create_index
|    |    |    |--> dict_mem_index_create
|    |    |    |--> row_create_index_for_mysql
|    |    |    |    |--> dict_build_index_def
|    |    |    |    |
|    |    |    |    |--> dict_index_add_to_cache_w_vcol
|    |    |    |    |    |--> dict_index_build_internal_clust
|    |    |    |    |    |--> dict_index_build_internal_non_clust
|    |    |    |    |
|    |    |    |    |--> dict_create_index_tree_in_mem
|    |    |    |    |    |--> btr_create
|    |    |    |    |    |--> log_ddl->write_free_tree_log
|    |
|    |--> create_table_update_global_dd
|    |    |--> dd_create_implicit_tablespace
|    |    |    |--> dd_create_tablespace
|    |    |    |    |--> dd_client->store // dd::Tablespace
|    |
|    |--> create_table_update_dict
```



```c++
/* Dictionary_client 操作 */

|--> acquire
|    |--> acquire_uncommitted
|    |
|    |--> m_registry_committed.get
|    |
|    |--> Shared_dictionary_cache::instance()->get
|    |    |--> m_map<T>()->get
|    |    |
|    |    |--> get_uncached
|    |    |    |--> Storage_adapter::get
|    |    |    |    |--> instance()->core_get
|    |    |    |    |    |--> m_core_registry.get
|    |    |    |    |
|    |    |    |    |--> trx.otx.get_table // 从系统表读取
|    |    |
|    |    |--> m_map<T>()->put
|    |
|    |--> m_registry_committed.put



|--> store
|    |--> m_registry_committed.get // 检查
|    |
|    |--> Storage_adapter::store // 保存
|    |    |--> instance()->core_store
|    |    |    |--> m_core_registry.put
|    |    |
|    |    |--> object->impl()->store // 保存到系统表
|    |    |--> sdi::store
|    |
|    |--> register_uncommitted_object
|    |    |--> m_registry_committed.get
|    |    |--> m_registry_dropped.get
|    |    |--> m_registry_uncommitted.put
```

## 删表过程

```c++
|--> mysql_rm_table
|    |--> mysql_rm_table_no_locks
|    |    |--> drop_base_table
|    |    |    |--> ha_delete_table
|    |    |    |
|    |    |    |--> dd::drop_table
|    |    |
|    |    |--> post_ddl
```



```c++
|--> delete_impl
|    |--> row_drop_table_for_mysql
|    |    |--> row_drop_table_from_cache
|    |    |--> log_ddl->write_drop_log
|    |    |--> log_ddl->write_delete_space_log
|    |
|    |--> dd_drop_tablespace
```

## 参考文献

> http://mysql.taobao.org/monthly/2018/03/02/
>
> http://mysql.taobao.org/monthly/2018/07/02/