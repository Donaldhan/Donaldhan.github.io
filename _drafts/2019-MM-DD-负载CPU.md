---
layout: page
title: my blog
subtitle: sub title
date: 2018-11-04 15:17:19
author: donaldhan
catalog: true
category: linux
categories:
    - linux
tags:
    - linux
---

# 引言



# 目录
* [](#)
    * [](#)
    * [](#)
* [总结](#总结)


CPU负载小于等于0.5算是一种理想状态


理想的状态是每个内核的负载为0.7左右，我比较赞同，0.7乘以内核数，得出服务器理想的CPU负载，比如我这台服务器，负载在3.0以下就可以。



CPU利用率在过去常常被我们这些外行认为是判断机器是否已经到了满负荷的一个标准，我看到长时间CPU使用率60-80%就认为机器有瓶颈出现。

在服务器其它方面配置合理的情况下，CPU数量和CPU核心数（即内核数）都会影响到CPU负载，因为任务最终是要分配到CPU核心去处理的。两块CPU要比一块CPU好，双核要比单核好。


load average:系统平均负载是CPU的Load，它所包含的信息不是CPU的使用率状况，而是在一段时间内CPU正在处理以及等待CPU处理的进程数之和的统计信息，也就是CPU使用队列的长度的统计信息。这个数字越小越好
CPU利用率:显示的是程序在运行期间实时占用的CPU百分比
CPU负载:显示的是一段时间内正在使用和等待使用CPU的平均任务数。CPU利用率高，并不意味着负载就一定大


linux系统CPU负载分析：https://shift-alt-ctrl.iteye.com/blog/2435140

Linux系统下CPU使用(load average)梳理：https://cloud.tencent.com/developer/article/1027288

Linux常用监控指标：https://book.open-falcon.org/zh_0_2/faq/linux-metrics.html


1、%us：用户态进程CPU占用率（时间）。

    2、%sy：系统内核CPU占用率，此值通常较低，但是有时候开启大量的系统进程，比如shell console进行日志输出、计算，也会导致很高的%sy。有些信息表示，系统级的swap操作也会导致%sy很高，本人未能实际验证。

    3、%wa：基本语义同iowait。

    4、%hi：硬中断，比如网卡、磁盘、键盘等触发的中断请求处理所消耗的CPU占比。

    5、%si：软中断，由应用进程程序触发的中断，对于一些涉及到高并发的网路IO、异步IO（NIO多路复用、AIO等）等应用、代理Server可能需要关注这些。软中断，我们没法避免和消除，也是很频繁和常见的，只需要注意一个事情，默认情况下（云平台优化的另说）si只发生在0号CPU上，所以0号CPU的%si很高（一般也不会特别高，因为si的处理通常都很快速，可能si的次数很大），我们可以通过“service irqbalance start” 开启中断平衡策略，此后si可以在多个CPU之间发生而不是集中在0号。（之所以在一个CPU上，主要是能耗考虑，其他CPU可以休眠节约能耗而不是频换唤醒来处理这种“很小”的中断请求）（vmstat 可以查看有关中断次数）。



```
donald@Donald_Draper:~$ top
top - 14:09:02 up 64 days, 23:39,  1 user,  load average: 0.19, 0.16, 0.16
Tasks: 101 total,   1 running, 100 sleeping,   0 stopped,   0 zombie
%Cpu(s):  4.2 us,  0.8 sy,  0.0 ni, 94.8 id,  0.1 wa,  0.0 hi,  0.0 si,  0.1 st
KiB Mem:   8191124 total,  7686748 used,   504376 free,    81044 buffers
KiB Swap:        0 total,        0 used,        0 free.  3644688 cached Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                                                                 
43624 root      20   0  327700  30052   6168 S  14.3  0.4 203:57.53 scribe_agent_v3                                                                                                                                                         
    3 root      20   0       0      0      0 S   3.7  0.0   2131:42 ksoftirqd/0                                                                                                                                                             
48697 ewallet   20   0  9.910g 3.430g  20704 S   2.7 43.9  74:19.86 java                                                                                                                                                                    
   13 root      20   0       0      0      0 S   2.3  0.0 532:13.96 ksoftirqd/1                                                                                                                                                             
   23 root      20   0       0      0      0 S   1.7  0.0 602:36.71 ksoftirqd/3                                                                                                                                                             
10649 root      20   0   29576   4548   4004 S   1.3  0.1  16:30.35 nss_agentd       
```

```
donald@Donald_Draper:~$ lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                4
On-line CPU(s) list:   0-3
Thread(s) per core:    1
Core(s) per socket:    1
Socket(s):             4
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 61
Model name:            Intel Core Processor (Broadwell)
Stepping:              2
CPU MHz:               2394.454
BogoMIPS:              4788.90
Hypervisor vendor:     KVM
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              4096K
NUMA node0 CPU(s):     0-3
```


```
donald@Donald_Draper:~$  iostat -txk 1
Linux 3.16.0-4-amd64 (hzabj-starcq-grexpect2.server.163.org)    06/13/2019      _x86_64_        (4 CPU)

06/13/2019 02:12:19 PM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           3.51    0.00    0.50    0.06    0.04   95.90

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00     0.06    0.00    5.36     0.07    19.97     7.47     0.01    2.38    1.34    2.38   0.56   0.30
vdb               0.00     0.00    0.00    0.00     0.00     0.00     7.85     0.00    0.99    0.99    1.00   0.88   0.00

06/13/2019 02:12:20 PM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           4.02    0.00    1.26    0.00    0.00   94.72

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
vdb               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

06/13/2019 02:12:21 PM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           4.02    0.00    0.50    0.25    0.00   95.23

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00     0.00    0.00   17.00     0.00    44.00     5.18     0.03    1.65    0.00    1.65   0.24   0.40
vdb               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

```

##

```java
```


###

```java
```


###

```java
```


## 总结
