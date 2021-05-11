---
title: tcpdump 工具使用简介
date: 2020-07-18 20:38:37
tags:
- 网络
- 工具
categories: 
- Linux

---

tcpdump 工具是 Linux 平台下进行网络抓包的工具，tcpdump 抓包 + wireshark 分析也是分析网络问题的黄金搭档。本文对 tcpdump 的使用进行一个简单总结。

<!-- more -->

## tcpdump 工具介绍

tcpdump 是一款强大的网络抓包工具，它使用 libpcap 库来抓取网络数据包，最初设计用于观察 TCP/IP 性能问题，但是后来逐步发展成为一个完整的网络分析工具。

## 命令

下面简单介绍下 tcpdump 工具常见的几种使用模式，更加丰富的使用说明，可以参考：

> https://www.tcpdump.org/manpages/tcpdump.1.html
>
> https://github.com/mylxsw/growing-up/blob/master/doc/tcpdump简明教程.md

### 简易命令

```shell
tcpdump -tttt -s0 -X -vv tcp port 8080 -w xxx.cap
```

### 使用选项

- **-i any** 监听所有的网卡接口，用来查看是否有网络流量
- **-i eth0** 只监听 eth0 网卡接口
- **-D** 显示可用的接口列表
- **-n** 不要解析主机名
- **-nn** 不要解析主机名或者端口名
- **-q** 显示更少的输出（更加 quiet）
- **-t** 输出可读的时间戳
- **-tttt** 输出最大程度可读的时间戳
- **-X** 以 hex 和 ASCII 两种形式显示包的内容
- **-XX** 与 **-X** 类似，增加以太网header的显示
- **-v, -vv, -vvv** 显示更加多的包信息
- **-c** 只读取 x 个包，然后停止
- **-w file**  把捕获的包数据写入到文件中
- **-C size**  使用 **-w** 写入文件时，限制文件的最大大小，超出时新开一个文件，单位 MB
- **-s** 指定每一个包捕获的长度，单位是 byte，使用 -s0 可以捕获整个包的内容
- **-S** 输出绝对的序列号
- **-e** 获取以太网 header
- **-E** 使用提供的秘钥解密 IPSEC 流量

### 表达式

可以使用表达式过滤指定类型的流量，有三种主要的表达式类型：

- 类型（type）选项包含：host，net，port
- 方向（dir）选项包含：src，dst
- 协议（proto）选项包含：tcp，udp，icmp，ah 等