---
title: perf 工具使用简介
date: 2020-07-18 20:07:30
tags:
- Linux
categories: 
- 其它
---

perf 工具是 Linux 平台下进行性能分析的一个非常常用的工具，这篇文章对 perf 工具的使用进行一个简单的介绍。

<!-- more -->

## perf 工具介绍

perf 是 Linux 平台下用来进行软件性能分析的工具。最初的时候叫做 Performance Counter，在 2.6.31 中第一次亮相，此后成为内核开发最为活跃的一个领域，在 2.6.32 中正式改名为 Performance Event，因为 perf 已不再仅仅作为 PMU 的抽象，而是能够处理所有的性能相关的事件。通过它，应用程序可以利用 PMU，tracepoint 和内核中的特殊计数器来进行性能统计。

perf 不但可以分析指定应用程序的性能问题（per thread），也可以用来分析内核的性能问题，当然也可以同时分析应用代码和内核，从而全面理解应用程序中的性能瓶颈。使用 perf 可以分析程序运行期间发生的硬件事件，比如 instructions retired ，processor clock cycles 等；也可以分析软件事件，比如 Page Fault 和进程切换。

## 命令

下面简单介绍下 perf 工具常见的几种使用模式，更加丰富的使用说明，可以参考：

> http://www.brendangregg.com/perf.html

### perf stat

执行一个命令并收集其运行过程中的各个数据，可以提供一个程序运行情况的总体概览，使用方式如下：

```shell
perf stat cmd
```

### perf top

实时显示当前系统的性能统计信息。该命令主要用来观察整个系统当前的状态，比如可以通过查看该命令的输出来查看当前系统最耗时的内核函数或某个用户进程，也可以用来实时查看某个进程的运行情况，使用方式如下：

```she
perf top

perf top -p $PID
```

### perf report

记录单个函数级别的统计信息，常用来分析某个进程的热点，perf record 不会将结果显示出来，而是将结果输出到文件中，可以用 perf report 来进行解析，使用方式如下：

```shell
perf record -F 99 -p PID -g -- sleep 10

perf report
```

