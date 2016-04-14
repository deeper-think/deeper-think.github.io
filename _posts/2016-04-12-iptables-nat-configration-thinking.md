---
layout: post
title: 利用iptables搭建NAPT网关-配置及相关问题思考
category: 系统及服务
tags: [iptables,NAT]
keywords: iptables,NAT
description: 
---

> 用正确的工具，做正确的事情

NAT主要用来解决公网IP地址不足的问题，一般作为网关设备放置到内部网络的出口从而实现所有内网主机共享网关公网IP来上网。按照功能NAT分为很多中，不过目前工作中接触最多的是1对多NAT也即NAPT，这种NAT方式可以实现多个内网主机共享一个公网IP来访问网络。本文记录linux环境下利用iptables搭建NAT网关并验证其可用性，以及对一些问题的思考和分析。

## NAT网关配置
	
	NAT网关根据功能特性分为很多种不同的类型，常用的是1对多NAT也即NAPT。我们这里在Linux环境利用iptables nat表搭建一台1对多NAT网关服务器供其他内网服务器访问公网使用。 
	
### 实验网络

两台linux服务器用于搭建测试验证环境，使用两个子网172.16.1.0/24和192.168.122.0/24（172.16.1.0/24子网可以访问公网，逻辑上该网段的IP地址可以看成公网IP地址）。服务器A作为NAT网关服务器配置两个网口eth0和eth1，eth0 配置IP地址：172.16.1.186（可以看成公网IP），eth1配置私网IP：192.168.122.100。服务器B作为内网测试机通过NAT网关访问公网，服务器B只有一个网口eth0配置IP：192.168.122.100。

### NAT网关服务器配置
首先确保NAT网关服务器数据包转发模块开启：
	
	[root@localhost ~]# cat /proc/sys/net/ipv4/ip_forward
	0

0表示数据包抓包功能未开启，那么手动开启数据包转发功能：

	[root@localhost ~]# echo 1 > /proc/sys/net/ipv4/ip_forward
	[root@localhost ~]# cat /proc/sys/net/ipv4/ip_forward
	1

上面的修改是临时性的，系统重启后修改会丢失，如果想持久化修改该功能配置，可以通过修改/etc/sysctl.conf：

	net.ipv4.ip_forward = 1

然后使配置生效：

	sysctl -p /etc/sysctl.conf

然后 利用iptables 配置网关服务器NAT转发功能：

	iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE //IP MASQUERADE，IP伪装

上面利用iptables nat的IP伪装功能实现动态源IP NAT，这种方式适用于公网IP地址不固定的场景。对于公网IP地址固定的场景，也可以使用SNAT：

	iptables -t nat -A POSTROUTING -s 192.168.122.0/24 -o eth0 -j SNAT --to 172.16.1.186

### 内网服务器配置

将内网服务器的网关设定为NAT网关服务器的内网IP：

	TYPE=Ethernet
	BOOTPROTO=static
	NAME=eth0
	DEVICE=eth0
	ONBOOT=yes
	IPADDR=192.168.122.101
	NETMASK=255.255.255.0
	GATEWAY=192.168.122.100   //NAT网关服务器内网IP
	
确认服务器本地路由正确：

	[root@localhost ~]# route -n
	Kernel IP routing table
	Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
	0.0.0.0         192.168.122.100 0.0.0.0         UG    100    0        0 eth0
	192.168.122.0   0.0.0.0         255.255.255.0   U     100    0        0 eth0	

### 内网服务器访问公网验证

	[root@localhost ~]# wget www.baidu.com
	--2016-04-14 00:47:35--  http://www.baidu.com/
	Resolving www.baidu.com (www.baidu.com)... 112.80.248.74, 112.80.248.73
	Connecting to www.baidu.com (www.baidu.com)|112.80.248.74|:80... connected.
	HTTP request sent, awaiting response... 200 OK
	Length: unspecified [text/html]
	Saving to: ‘index.html.1’
	
    [ <=>                                                                                                            ] 98,313      --.-K/s   in 0.1s
	
	2016-04-14 00:47:35 (888 KB/s) - ‘index.html.1’ saved [98313]

	确认内网服务器可以通过NAT网关服务器正常访问公网。






















enjoy the life !!!
