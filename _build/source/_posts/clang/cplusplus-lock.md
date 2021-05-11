---
title: MySQL 中几种锁的使用
date: 2020-07-17 22:49:43
tags:
- 锁
categories: 
- C/C++

---

MySQL 中常用的几种锁包括：mysql_rwlock_t、mysql_mutex_t、mysql_cond_t，本文结合 MySQL 源码对各种锁的使用做一个简单的总结。

<!-- more -->

## mysql_rwlock_t

### 数据结构

```c++
/**
  Type of an instrumented rwlock.
  @c mysql_rwlock_t is a drop-in replacement for @c pthread_rwlock_t.
  @sa mysql_rwlock_init
  @sa mysql_rwlock_rdlock
  @sa mysql_rwlock_tryrdlock
  @sa mysql_rwlock_wrlock
  @sa mysql_rwlock_trywrlock
  @sa mysql_rwlock_unlock
  @sa mysql_rwlock_destroy
*/
typedef struct st_mysql_rwlock mysql_rwlock_t;

struct st_mysql_rwlock
{
  /** The real rwlock */
  native_rw_lock_t m_rwlock;
  /**
    The instrumentation hook.
    Note that this hook is not conditionally defined,
    for binary compatibility of the @c mysql_rwlock_t interface.
  */
  struct PSI_rwlock *m_psi;
};

typedef pthread_rwlock_t native_rw_lock_t;
```

### 使用方式

```c++
PSI_rwlock_key key_rwlock_LOCK_xxx;
mysql_rwlock_t m_lock;

mysql_rwlock_init(key_rwlock_LOCK_xxx, &m_lock);

mysql_rwlock_rdlock(&m_lock);
mysql_rwlock_wrlock(&m_lock);
mysql_rwlock_trywrlock(&m_lock);
mysql_rwlock_unlock(&m_lock);

mysql_rwlock_destroy(&m_lock);
```

### 说明

mysql_rwlock_t 基本上可以看作是对 pthread_rwlock_t 的一个封装，pthread_rwlock_t 是读写锁，同时可以有多个线程获得读锁，同时只允许有一个线程获得写锁。其他线程在等待锁的时候同样会进入睡眠。读写锁在互斥锁的基础上，允许多个线程读，在某些场景下能提高性能。

## mysql_mutex_t

### 数据结构

```c++
/**
  Type of an instrumented mutex.
  @c mysql_mutex_t is a drop-in replacement for @c my_mutex_t.
  @sa mysql_mutex_assert_owner
  @sa mysql_mutex_assert_not_owner
  @sa mysql_mutex_init
  @sa mysql_mutex_lock
  @sa mysql_mutex_unlock
  @sa mysql_mutex_destroy
*/
typedef struct st_mysql_mutex mysql_mutex_t;

struct st_mysql_mutex
{
  /** The real mutex. */
  my_mutex_t m_mutex;
  /**
    The instrumentation hook.
    Note that this hook is not conditionally defined,
    for binary compatibility of the @c mysql_mutex_t interface.
  */
  struct PSI_mutex *m_psi;
};

typedef native_mutex_t my_mutex_t;

typedef pthread_mutex_t native_mutex_t;
```

### 使用方式

```c++
PSI_mutex_key key_LOCK_xxx;
mysql_mutex_t m_lock;

mysql_mutex_init(key_LOCK_xxx, &m_lock, NULL);

mysql_mutex_lock(&m_lock);
mysql_mutex_unlock(&m_lock);

mysql_mutex_destroy(&m_lock);
```

### 使用技巧

1. Thread 如果已经获得了 mutex， 那么如果再次 lock 的话，会产生死锁，所以 MySQL 内部大量的 mutex_owner 判断逻辑（ mysql_mutex_assert_owner(&m_mutex) ）；
2. 与1类似，可以在利用一个计数值来统计当前正在等到锁的线程个数，锁 owner 可以根据此计数值来判断是否需要释放锁。

### 说明

mysql_mutex_t 基本上可以看作是对 pthread_mutex_t 的一个封装，pthread_mutex_t 是互斥锁，同一瞬间只能有一个线程能够获取锁，其他线程在等待获取锁的时候会进入休眠状态。因此 pthread_mutex_t 消耗的 CPU 资源很小，但是性能不高，因为会引起线程切换。

## mysql_cond_t

### 数据结构

```c++
/**
  Type of an instrumented condition.
  @c mysql_cond_t is a drop-in replacement for @c pthread_cond_t.
  @sa mysql_cond_init
  @sa mysql_cond_wait
  @sa mysql_cond_timedwait
  @sa mysql_cond_signal
  @sa mysql_cond_broadcast
  @sa mysql_cond_destroy
*/
typedef struct st_mysql_cond mysql_cond_t;

struct st_mysql_cond
{
  /** The real condition */
  pthread_cond_t m_cond;
  /**
    The instrumentation hook.
    Note that this hook is not conditionally defined,
    for binary compatibility of the @c mysql_cond_t interface.
  */
  struct PSI_cond *m_psi;
};
```

### 使用方式

```c++
PSI_cond_key key_COND_xxx;
mysql_cond_t m_cond;

mysql_cond_init(key_COND_xxx, &m_cond);

// mysql_cond_t 需要与 mysql_mutex_t 配合使用
struct timespec abs_timeout;
mysql_cond_timedwait(&m_cond, &m_mutex, &abs_timeout);

mysql_cond_signal(&m_cond);
mysql_cond_broadcast(&m_cond);

mysql_cond_destroy(&m_cond);
```

## 补充

除了 pthread_rwlock_t 和 pthread_mutex_t 之外，还有一个常用的锁 pthread_spinlock_t。pthread_spinlock_t 是自旋锁，同一瞬间也只能有一个线程能够获取锁，不同的是，其他线程在等待获取锁的过程中并不进入睡眠状态，而是在 CPU 上进入自旋等待。自旋锁的性能很高，但是只适合对很小的代码段加锁（或短期持有的锁），自旋锁对 CPU 的占用相对较高。