---
title: cgroups 使用简介
date: 2020-07-18 21:03:55
tags:
- Linux
categories: 
- 其它
---

cgroups 是 Linux 内核提供的一种可以限制、记录、隔离进程组所使用的物理资源的机制，本文简单介绍下 cgroups 的使用情况。

<!-- more -->

## cgroups 原理

cgroups 的实现本质上是给系统进程挂上钩子（hooks），在 task 运行的过程中涉及到某个资源时就会触发钩子上所附带的 subsystem 进行检测，最终根据资源类别的不同使用对应的技术进行资源限制和优先级分配。cgroups 的目的是将任意进程进行分组化管理。

> https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/index

### 基本概念

cgroups 主要由 task，cgroup，subsystem 以及 hierarchy 组成

- task（任务）：在 cgroups 中，任务就是系统的一个进程；
- cgroup （控制族群）：控制族群是一组按照某种标准划分的进程，cgroups 中的资源控制都是以控制族群为单位实现的。一个进程可以加入某个控制族群，也可以迁移至另一个控制族群；
- hierarchy（层级）：控制族群可以组织成层级的形式，即一颗控制族群树,其上的子节点继承父节点的特定属性（cgroup 结构树）；
- subsystem（子系统）：通常是一个资源控制器，例如 CPU 子系统可以控制 CPU 时间分配，一个子系统必须附加到控制族群树上才能起作用，该树上的所有控制族群都受到该子系统的控制；

### 子系统

subsystem 实际上就是 cgroups 的资源控制系统，每种 subsystem 独立地控制一种资源，常见的子系统如下：

- cpu：使用调度程序控制 task 对 CPU 的使用；
- cpuset：可以为 cgroup 中的 task 分配独立的 CPU（此处针对多处理器系统）和内存；
- memory：可以设定 cgroup 中 task 对内存使用量的限定；
- blkio：可以为块设备设定输入/输出限制，比如物理驱动设备（包括磁盘、固态硬盘、USB等）；

## cgroup 使用

```shell
# 关键信息
1. cat /proc/cgroups 查看当前 cgroup 运行情况
2. mount | grep cgroup 查看 cgroup 挂载情况
3. cgroup 内的节点保持继承关系，子目录自动继承父目录的规则
4. 删除 cgroup 下的目录时，不能直接 rm -rf ，需要使用 rmdir
5. 通过规则目录下的 cgroup.procs 与进程进行绑定，限定进程的资源
```

### CPU 隔离

CPU 隔离有两种模式：共享模式和独享模式，其中共享模式：

1. 通过 cpu 子系统进行限制，/cgroup/cpu/xxx
2. 主要由以下文件进行配置：

```
-rw-r--r-- 1 root root 0 Sep 23 14:54 cpu.cfs_period_us
-rw-r--r-- 1 root root 0 Sep 23 14:54 cpu.cfs_quota_us
```

独享模式：

1. 通过 cpuset 子系统进行限制，/cgroup/cpuset/xxx
2. 主要由以下文件进行配置：

```
-rw-r--r-- 1 root root 0 Sep 23 14:45 cpuset.cpus
```

注：

1. cfs_period_us 用来配置时间周期长度，cfs_quota_us 用来配置当前 cgroup 在设置的周期长度内所能使用的 CPU 时间数。cfs_quota_us/cfs_period_us 可以当做是对应的核数限制；
2. cpuset.cpus 需要设置成全部核，cpuset.mems 需要设置为 0；
3. 通过 cgroup.procs 与进程进行绑定；
4. cpuset.cpus 里直接绑定 cpu 核，cpu 核的信息可以通过 /proc/cpuinfo 进行查看；

### Memory 隔离

Memory 的隔离：

1. 通过 memory 子系统进行限制，/cgroup/memory/xxx
2. 主要由以下文件进行配置：

```
-rw-r--r-- 1 root root 0 Sep 23 14:54 memory.limit_in_bytes
```

注：memory.oom_control文件用来配置当内存超出限制后的是否需要kill申请内存的进程。

### IOPS 限制

IOPS 的隔离：

1. 通过 blkio 子系统进行限制，/cgroup/blkio/xxx
2. 主要由以下文件进行配置：

```
-rw-r--r-- 1 root root 0 Nov 19 16:41 blkio.throttle.read_iops_device
-rw-r--r-- 1 root root 0 Nov 19 16:41 blkio.throttle.write_iops_device
```

注：与内存限制不同的是，IOPS 的限制其实是对于块设备的 IO 限制，所以配置文件中会指明对应的设备 ID。

### 参考文献

> https://staight.github.io/2019/03/07/linux资源管理器-cgroups理解
>
> https://segmentfault.com/a/1190000006917884
>
> https://segmentfault.com/a/1190000007241437
>
> https://segmentfault.com/a/1190000008125359
>
> https://segmentfault.com/a/1190000008323952