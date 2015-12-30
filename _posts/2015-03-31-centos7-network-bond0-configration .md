---
layout: post
title: centos7网络及bond0配置 
category: 技术
tags: [centos7，网络配置，bond0配置]
keywords: centos7，网络配置，bond0配置
description: 
---

> 用正确的工具，做正确的事情

### 配置网络

编辑网口配置文件ifcfg-eth0（以eth0为例）：

	TYPE=Ethernet
	BOOTPROTO=none
	DEFROUTE=yes
	PEERDNS=yes
	PEERROUTES=yes
	IPV4_FAILURE_FATAL=no
	NAME=eth0                                 //根据实际情况修改
	UUID=586bbe6a-bc15-4290-9307-47b5da2eaf51 //根据实际情况修改，或者不配置
	DEVICE=eth0                               //根据实际情况修改
	ONBOOT=yes
	IPADDR=111.8.9.130                        //根据实际情况修改
	NETMASK=255.255.255.192                   //根据实际情况修改
	GATEWAY=111.8.9.190                       //根据实际情况修改

重启网络，使配置文件生效：

	nohup systemctl restart NetworkManager &           
 
注意:

> centos7 必须使用 systemctl来重启服务，并且必须使用NetworkManager来使网络配置生效，而非7以前的系统使用network服务。 另外必须要通过nohup命令在后台执行网络重启命令，否则很有可能网络关闭后，ssh连接断开，网路启动的过程不会被执行。



检查NetworkManager服务重启后是否正常：

	systemctl status NetworkManager

通过ifconfig命令确认网口配置是否已经生效，通过ping确认网络已经连通正常。

### bond0配置

编辑配置ifcfg-bond0配置文件：

	vi ifcfg-bond0

	DEVICE=bond0
	NAME=bond0
	TYPE=Bond
	BONDING_MASTER=yes
	IPADDR=172.16.1.135                    //根据实际情况配置
	NETMASK=255.255.255.0	               //根据实际情况配置
	GATEWAY=172.16.1.254                   //根据实际情况配置
	ONBOOT=yes
	BOOTPROTO=none
	NM_CONTROLLED=yes
	BONDING_OPTS="mode=0 miimon=100"

编辑配置关联bond0的网口配置文件：

	vi ifcfg-eth0
	DEVICE=eth0
	NAME=bond0-slave
	TYPE=Ethernet
	BOOTPROTO=none
	NM_CONTROLLED=yes
	ONBOOT=yes
	MASTER=bond0
	SLAVE=yes
	
	vi ifcfg-eth1
	DEVICE=eth1
	NAME=bond0-slave
	TYPE=Ethernet
	BOOTPROTO=none
	NM_CONTROLLED=yes
	ONBOOT=yes
	MASTER=bond0
	SLAVE=yes


重启网络使网络配置生效：

	nohup systemctl restart NetworkManager &

检查NetworkManager服务重启后是否正常：

	systemctl status NetworkManager

通过ifconfig命令确认网口配置是否已经生效，通过ping确认网络已经连通正常。


玩的开心 !!!
