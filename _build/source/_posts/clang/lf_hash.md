---
title: MySQL LF_HASH
date: 2020-07-17 23:07:18
tags:
- LF_HASH
categories: 
- C/C++

---

MySQL 中有很多实现非常好的数据结构，本文要介绍的 LF_HASH 就是其中一个。

<!-- more -->

## 要解决的问题

LF_HASH 主要解决的是以下两个问题：

1. HASH 结构并发修改的问题，特别是查询、插入、删除并发操作时，如何保证当前查询到的元素不会被其它线程删除？
2. 扩容问题，如何保证在数据元素增多时，有效的进行扩容，减少数据的移动工作？

第一个问题，解决办法是引入 pin 的概念，通过 pin 来记录正在被使用的数据元素，防止被其它线程删除（和 Hazard Pointer 的异同点？）

第二个问题，解决办法是先取 key 的 hash 值，然后对 hash 的二进制进行 reverse，将高位相同的元素放入同一个 bucket。扩容时，只需要在单个 bucket 内插入一个新的锚点，不需要进行任何的数据移动。

> https://kernelmaker.github.io/MySQL_lf_allocator
>
> https://baotiao.github.io/2019/09/15/mysql-lf_hash/

## 不解决的问题

LF_HASH 只保证 HASH 结构的线程安全，并不保证单个元素不被修改，单个数据元素还是要通过单独的 lock 来进行保护。

## 使用指南

LF_HASH 的使用和普通 HASH 稍有区别，主要包括以下几点：

### 初始化 LF_HASH

```c++
/*
  Initializes lf_hash, the arguments are compatible with hash_init

  @note element_size sets both the size of allocated memory block for
  lf_alloc and a size of memcpy'ed block size in lf_hash_insert. Typically
  they are the same, indeed. But LF_HASH::element_size can be decreased
  after lf_hash_init, and then lf_alloc will allocate larger block that
  lf_hash_insert will copy over. It is desireable if part of the element
  is expensive to initialize - for example if there is a mutex or
  DYNAMIC_ARRAY. In this case they should be initialize in the
  LF_ALLOCATOR::constructor, and lf_hash_insert should not overwrite them.
  See wt_init() for example.
  As an alternative to using the above trick with decreasing
  LF_HASH::element_size one can provide an "initialize" hook that will finish
  initialization of object provided by LF_ALLOCATOR and set element key from
  object passed as parameter to lf_hash_insert instead of doing simple memcpy.
*/
void lf_hash_init2(LF_HASH *hash, uint element_size, uint flags,
                   uint key_offset, uint key_length,
                   hash_get_key_function get_key, CHARSET_INFO *charset,
                   lf_hash_func *hash_function, lf_allocator_func *ctor,
                   lf_allocator_func *dtor, lf_hash_init_func *init) {
  lf_alloc_init2(&hash->alloc, sizeof(LF_SLIST) + element_size,
                 offsetof(LF_SLIST, key), ctor, dtor);
  lf_dynarray_init(&hash->array, sizeof(LF_SLIST *));
  hash->size = 1;
  hash->count = 0;
  hash->element_size = element_size;
  hash->flags = flags;
  hash->charset = charset ? charset : &my_charset_bin;
  hash->key_offset = key_offset;
  hash->key_length = key_length;
  hash->get_key = get_key;
  hash->hash_function = hash_function ? hash_function : cset_hash_sort_adapter;
  hash->initialize = init;
  DBUG_ASSERT(get_key ? !key_offset && !key_length : key_length);
}

void lf_hash_destroy(LF_HASH *hash) {
  LF_SLIST *el, **head = (LF_SLIST **)lf_dynarray_value(&hash->array, 0);

  if (unlikely(!head)) {
    return;
  }
  el = *head;

  while (el) {
    LF_SLIST *next = el->link;
    if (el->hashnr & 1) {
      lf_alloc_direct_free(&hash->alloc, el); /* normal node */
    } else {
      my_free(el); /* dummy node */
    }
    el = (LF_SLIST *)next;
  }
  lf_alloc_destroy(&hash->alloc);
  lf_dynarray_destroy(&hash->array);
}
```

初始化时需要定义 key 的获取方式，构造&析构方式，插入方法等。其中构造&析构方式，插入方法可以缺省；

### 查询 LF_HASH

```c++
lf_hash_get_pins(&m_hash);

lf_hash_search(&m_hash, pins, key, length);

lf_hash_search_unpin(pins);

lf_hash_put_pins(pins);
```

注意，查询的时候必须用 pins&unpin 进行保护，不论查询的结果如何。为什么要这样处理呢？这里涉及到一个对“无锁”概念的理解问题。无锁 HASH 中的无锁到底指的是什么？

无锁指的是：HASH 中的数据是安全的，特别指删除过程是安全的，查询的线程不会被删除的线程影响。如果做到安全呢？pin 在这个过程中起到了关键作用，LF_HASH 在删除时，不是立即删除，而是放入到内部的待删除队列，此时删除操作返回成功，再次查询也无法查询到结果。但是，已经 pin 住的查询依旧有效，数据的访问也正常，直到 unpin 操作完成，删除操作才会真正执行。

### 插入 LF_HASH

```c++
lf_hash_get_pins(&m_hash);

lf_hash_insert(&m_hash, pins, item);

lf_hash_put_pins(pins); 
```

###  删除 LF_HASH

```c++
lf_hash_get_pins(&m_hash);

lf_hash_delete(&m_hash, pins, key, length);

lf_hash_put_pins(pins);
```

## 注意事项

LF_HASH 的使用过程中需要注意：由于 LF_HASH 的删除操作“同步返回，异步删除”的特点，HASH 中管理的对象最好利用一个状态位进行标记，删除前先维护状态位的值。



