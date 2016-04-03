---
layout: post
title: linux 配置网卡bonding
category: 系统及服务
tags: [bonding]
keywords: bonding
description:
---

> 用正确的工具，做正确的事情

### 绑定网卡的配置文件及bond设备的配置文件

网卡em2配置文件，其他网卡类似：

	DEVICE=em2
	BOOTPROTO=none
	ONBOOT=yes
	TYPE=Ethernet
	MASTER=bond0
	SLAVE=yes
	IPV6INIT=no
	USERCTL=no

bond0配置文件，bond0用于桥接：

	DEVICE=bond0
	ONBOOT=yes
	BRIDGE=br0
	BOOTPROTO=none
	TYPE=Ethernet
	IPV6INIT=no
	USERCTL=no

### 配置bonding模块自动加载

	vi /etc/modprobe.d/bonding.conf
	
	###追加
	alias bond0 bonding
	options bonding mode=0 miimon=200


第一次要手动加载模块：

	modprobe bonding

### 重启网络并确认bonding成功

	/etc/init.d/network restart
	
	cat /proc/net/bonding/bond0
	
	##输出
	thernet Channel Bonding Driver: v3.6.0 (September 26, 2009)
	
	Bonding Mode: load balancing (round-robin)
	MII Status: up
	MII Polling Interval (ms): 0
	Up Delay (ms): 0
	Down Delay (ms): 0
	
	Slave Interface: em2
	MII Status: up
	Speed: 1000 Mbps
	Duplex: full
	Link Failure Count: 0
	Permanent HW addr: 90:b1:1c:50:35:bd
	Slave queue ID: 0
		
	ifconfig | grep HWaddr
	
	###输出
	bond0     Link encap:Ethernet  HWaddr 90:B1:1C:50:35:BD  
	br0       Link encap:Ethernet  HWaddr 90:B1:1C:50:35:BD  
	em1       Link encap:Ethernet  HWaddr 90:B1:1C:50:35:BC  
	em2       Link encap:Ethernet  HWaddr 90:B1:1C:50:35:BD  
	em3       Link encap:Ethernet  HWaddr 90:B1:1C:50:35:BD  
	em4       Link encap:Ethernet  HWaddr 90:B1:1C:50:35:BD  
	p4p1      Link encap:Ethernet  HWaddr 90:B1:1C:50:35:BD  
	p4p2      Link encap:Ethernet  HWaddr 90:B1:1C:50:35:BD
	
### 配置系统启动自动绑定

	vi /etc/rc.d/rc.local
	
	###追加
	ifenslave bond0 em2 em3 xxxx	
	

完成！！ 玩得开心！！！

### 相关文档

1. [介绍了bonding的原理以及各种bonding模式](http://www.cnblogs.com/wuxulei/p/3270256.html)
