---
layout: post
title: 关于MTU、MSS、TSO、GSO
category: 技术
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

既然网络设备能够通过的最大数据包大小受MTU值的限制，进而对于tcp协议，一个tcp分段其payload的大小有必要限制，否则该tcp分段经过协议栈最终封装成“网包”后会超过 MTU 而无法通过网络设备，tcp分段payload的最大值，也即MSS：

> MSS,全称 Maximum Segment Size,是TCP协议定义的一个选项，MSS选项用于在TCP连接建立时，收发双方协商通信时每一个报文段所能承载的最大数据长度。

基于TCP协议数据传输的上层应用一次可以将大块的数据提交给协议栈，MSS的作用是TCP协议在封装TCP报文前根据MSS的值对上层数据进行分段，以保证每个分片最后形成的“网包”不会超过MTU。

## 3 TSO效果对比测试


## 参考

http://www.cnblogs.com/cobbliu/archive/2013/03/19/2970311.html


Have a fun！！！