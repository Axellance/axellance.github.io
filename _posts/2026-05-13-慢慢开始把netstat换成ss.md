---
title: 慢慢开始把netstat换成ss
author: Axellance
date: 2026-05-13
layout: post
category: 运维
tags: [运维]
mermaid: true
---

## 一、ss优势

想想当时刚学会用 `netstat` 的时候，一手命令开始查进程连接、找端口，当时觉得这个工具已经无法被替代了，但是紧跟着业务场景的丰富，它也处于被淘汰的边缘了。

在一般的轻量级应用服务器上，使用 `netstat` 还是很犀利的，但是在高并发服务器上，服务多、连接多导致 `netstat` 慢到卡顿。其官方就推出了一款属于 iproute2 工具集的 `ss` ，作为 `netstat` 的替代品。

`ss` （Socket Statistics）可以做到直接读取内核netlink信息，毫秒级输出，支持精细化过滤、状态统计、进程关联，已经成为了现代linux运维必备的网络排查利器。

| 特性     | ss                            | netstat                     |
| -------- | ----------------------------- | --------------------------- |
| 执行速度 | 高并发秒出                    | 遍历/proc，连接多会导致卡顿 |
| 数据来源 | 内核netlink直接读取           | 解析/proc/net/tcp等文件     |
| 过滤能力 | 支持/ip/端口/进程组合过滤     | 依赖grep/awk二次筛选        |
| 信息维度 | 定时器、tcp内部信息、内存统计 | 仅基础连接信息              |
| 默认安装 | CentOS7+/Ubuntu16+预装        | 新版本系统已逐步移除        |

首先介绍一下Linux中的两类用户，一种是超级用户（root），另外一种是普通用户。在管理一台服务器时，我们通常不会使用root去执行操作，因为root用户具有很大的权限，可能会执行一些危险的命令，导致服务器崩溃。然而，在运维过程中我们可能需要执行一些系统管理任务，我们通常会使用sudo来执行这些命令。

线上服务器排查网络，优先用 `ss`，后续新的linux版本会逐步淘汰 `netstat`

## 二、ss核心参数

基础格式：

```shell
ss [选项] [过滤条件]
```

- -t/-u: 仅显示TCP/UDP连接
- -l: 只看监听中端口（服务是否启动）
- -a: 显示所有套接字（监听+已建立+关闭中）
- -n: 禁用域名/服务名解析，纯数字输出，最快
- -p: 显示关联进程 PID / 进程名（需root权限）
- -s: 输出连接统计摘要（总连接、各状态数量）
- -o: 显示TCP定时器（排查超时、长连接）
- -i: 显示TCP内部信息（拥塞、重传、滑动窗口）
- -m: 显示socket内存使用
- -4/-6: 仅IPv4 / IPv6 连接
- -x: 显示UNIX

## 三、ss高级用法

`ss` 原生支持状态、IP、端口、逻辑组合过滤，不用管道也能精准排查，这是最核心的高级能力。

### 1. 按TCP状态过滤

支持所有TCP状态：established、syn-sent、syn-recv、fin-wait-1、fin-wait-2、time-wait、close-wait、last-ack、listening。

```shell
#查看所有已建立的tcp连接
ss -t state established
#查看time-wite连接（排查端口耗尽）
ss -t state time-wait
#查看close-wait连接（服务未正常关闭连接，必查）
ss -t state close-wait
#查看tcp监听端口
ss -t state listening
```

### 2. 按端口过滤

运算符：`==` / `!=` / `gt`(>) / `lt` (<) / `ge` (>=) / `le` (<=)

```shell
#目标端口为8080/443的tcp连接
ss -lt dport == :80
ss -lt dport == :443
#源端口为22的连接（本机对外ssh）
ss -lt sport == :22
#源端口 > 1024 的临时端口连接
ss -lt sport gt :1024
#组合：80 端口已建立连接
ss -t state established '( dport == :80 or sport == :80 )'
```

### 3. 按ip过滤（指定客户端/服务端ip）

```shell
#目标IP为192.168.2.10的所有连接
ss dst 192.168.2.10
#源IP为 10.0.0.0/24 网段的连接
ss src 10.0.0.0/24
#组合：特定ip访问 3306 端口
ss -tn dst 192.168.2.105 dport == :3306
```

先学到这，后面再写一些通过自动化脚本。