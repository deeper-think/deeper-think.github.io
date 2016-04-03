---
layout: post
title: 关于MTU、MSS、TSO、GSO
category: 网络协议
tags: [协议栈, MSS, MTU, TSO/GSO]
keywords: 协议栈, MSS, MTU, TSO/GSO
description:
---

> 用正确的工具，做正确的事情


## 1 背景

最近抓包分析一个问题的时候，在分析数据包的过程中发现一个比较奇怪的现象，如下图：

![MSS问题](http://7u2rbh.com1.z0.glb.clouddn.com/MSSvsTSO.jpg)

上面的抓包数据TCP 三次握手时交换的MSS值是1460， 而接下来的数据包传输过程中大量的TCP数据包其payload超过了MSS值，根据之前从教科书上学到的关于MSS的理解是 TCP数据包的payload大小不能超过MSS值，找公司内核团队的人讨论原来是TSO的原因，瞬间感觉自己对传输协议栈了解的片面和无知，有必要对协议栈相关的几个概念加深理解，O(∩_∩)O~。

## 2 MTU、MSS、TSO、GSO 概念

为什么TCP协议会有MSS这个东西？这要从传输协议底层聊起，孙然网络上传输的是0/1 bit流，但是网络上的设备发送数据是以“网包”为单位的，介于硬件及驱动的特性对所发送和接收的“网包”大小有一定的限制，这里的“网包”实际上就是IP数据包也即IP头+TCP头+payload， 这个大小的限制在TCP/IP协议里也即MTU。

> MTU，全称 Maximum Transmission Unit ，在TCP/IP协议里 MTU表示网络设备能够通过的最大数据包大小，单位byte。

MTU可以理解为网络设备的特性，这里说的网络设备同时包括硬件及软件驱动，linux系统可以通过命令查看和修改网口的MTU值（linux验证mtu值孙然可以通过命令来修改，但是这个值不能无限大的修改，可能跟硬件特性有关）：

	[root@WSHTTP112 ~]# ifconfig eth0
	eth0      Link encap:Ethernet  HWaddr 00:E0:81:D5:4E:86
	          inet addr:111.23.5.72  Bcast:111.23.5.127  Mask:255.255.255.192
    	      inet6 addr: fe80::2e0:81ff:fed5:4e86/64 Scope:Link
    	      UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
    	      RX packets:613393259625 errors:88 dropped:2176690 overruns:0 frame:81
        	  TX packets:644594295745 errors:0 dropped:0 overruns:0 carrier:0
          	  collisions:0 txqueuelen:1000
          	  RX bytes:493563003086535 (448.8 TiB)  TX bytes:813097977132755 (739.5 TiB)
          	  Memory:faae0000-fab00000

	[root@WSHTTP112 ~]# echo "1600" > /sys/class/net/eth0/mtu
	[root@WSHTTP112 ~]# ifconfig eth0
	eth0      Link encap:Ethernet  HWaddr 00:E0:81:D5:4E:86
          	  inet addr:111.23.5.72  Bcast:111.23.5.127  Mask:255.255.255.192
          	  inet6 addr: fe80::2e0:81ff:fed5:4e86/64 Scope:Link
          	  UP BROADCAST RUNNING MULTICAST  MTU:1600  Metric:1
          	  RX packets:613393745037 errors:88 dropped:2176690 overruns:0 frame:81
          	  TX packets:644594698389 errors:0 dropped:0 overruns:0 carrier:0
          	  collisions:0 txqueuelen:1000
          	  RX bytes:493563516198082 (448.8 TiB)  TX bytes:813098460120287 (739.5 TiB)
          	  Memory:faae0000-fab00000

网络设备能够通过的数据包存在大小的限制也即不能超过MTU，但是上层网络应用程序一次提交给协议栈传输的数据量大小是没有限制的，因此必须要有一种方案避免最后形成的“网包”超过MTU，对于TCP/IP协议栈传统的解决方案是在传输层对应用层提交的数据进行分段，通过限制TCP数据包payload的大小不超过MSS来实现最终形成的“网包”大小不超过MTU，简单理解MSS=MTU-TCP\_header\_size-IP\_header\_size，MSS的官方定义：

> MSS,全称 Maximum Segment Size,是TCP协议定义的一个选项，MSS选项用于在TCP连接建立时，收发双方协商通信时每一个报文段所能承载的最大数据长度。

在传输层对数据进行分段在某种程度上降低了协议栈处理数据的性能和效率：

- 数据分段会消耗一定的系统资源包括CPU、内存等资源，在传输层分段需要分装更多的TCP数据包，会产生更多的校验和计算的负载；
- 数据包分段使得流经的下层协议栈的数据包增多，在传输层分段使得网络层、数据链路层需要处理更多的数据包；

TSO和GSO的提出就是为了解决上面的两个问题，这两种offload方案的核心思想是一样的，都是尽可能的推迟数据分段的时机来解决上面第2个问题，除此之外支持TSO的网络接口在硬件上实现了数据包分段、校验和计算及包头封装的功能，不但进一步将分段的时机推迟到硬件层还从某种程度上解决了数据分段给CPU带来的负载。TSO也可以看成一种硬件加速的方案，硬件加速的核实是专用计算交给专用硬件来完成，CPU只负责通用计算，比如GPU的出现等等。TSO和GSO可以简单描述为：

> TSO, 全程TCP Segment
Offload也即传输层不进行数据分段，而是利用网络接口的计算能力将数据分段、校验和计算及数据包头封装的工作交给网络接口的硬件模块来做，解决传输层数据分段导致的性能和效率的问题；

> GSO, 全程 Generic Segment
Offload, GSO不需要网络接口硬件特性的支持，GSO将数据分段的时机推迟到将“网包”交给网络接口前一时刻，解决传输层分段导致流经协议栈数据包量多的问题。

通过linux命令可以查看网络接口是否支持TSO和GSO特性：

	[root@12010753 ~]# ethtool -k eth0
	Offload parameters for eth0:
	Cannot get device udp large send offload settings: Operation not supported
	rx-checksumming: on
	tx-checksumming: on
	scatter-gather: on
	tcp segmentation offload: on
	udp fragmentation offload: off
	generic segmentation offload: off
	generic-receive-offload: on

通过linux命令，可以打开和关闭tso功能：

	ethtool -K eth0 tso on   
	ethtool -K eth0 tso off


## 3 TSO效果对比测试


## 参考

http://www.cnblogs.com/cobbliu/archive/2013/03/19/2970311.html


Have a fun！！！