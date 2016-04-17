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

## NAT网关配置及验证
	
NAT网关根据功能特性分为很多种不同的类型，常用的是1对多NAT也即NAPT。我们这里在Linux环境利用iptables nat表搭建一台1对多NAT网关服务器供其他内网服务器访问公网使用。 
	
### 验证环境

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

## 关于一些问题的思考

上面Linux环境利用iptables的snat表实现网关功能，通俗的说snat实现两点功能：

1. 内网服务器发出去的数据包，将数据包的内网源IP地址替换成网关公网IP地址； 
2. 从公网回来的发送给内网服务器的数据包，将数据包的目的IP从公网网关IP地址替换成正确的内网服务器IP地址，也即逆SNAT。

那么现在的问题是：

>假设有多台内网服务器或者个人PC在某段时间通过NAT网关访问同一网站，对于公网回来的数据包，NAT网关怎么实现将数据包的目的IP地址替换成正确的内网IP地址？

上面的问题比较容易解决，通俗点讲：

>正常情况下，两台服务器某段时间访问同一网站时，两台服务器上的客户端使用的临时端口号也相同的可能性不大，那么对于公网回源的数据包其源IP地址和源端口号不同，目的IP地址也相同即NAT网关的IP，但是其目的端口号不同，那么我们根据目的端口号将回来的数据包目的IP地址，替换成正确的内网服务器IP地址。

实际上上面说的方法linux系统通过网络层的连接跟踪来实现，centos7版本os可以通过文件：
	
	/proc/net/nf_conntrack

来查看当前的系统的连接跟踪情况，

	[root@localhost net]# cat nf_conntrack| grep 112.80.248.73
	ipv4     2 tcp      6 111 TIME_WAIT src=192.168.122.101 dst=112.80.248.73 sport=36491 dport=80 src=112.80.248.73 dst=172.16.1.186 sport=80 dport=36491 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
	ipv4     2 tcp      6 8 CLOSE src=172.16.1.186 dst=112.80.248.73 sport=38698 dport=80 src=112.80.248.73 dst=172.16.1.186 sport=80 dport=38698 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
	
从连接跟踪记录的信息，对于回来的数据包可以根据源IP地址、目的IP地址、源端口号、目的端口号唯一定位到一条记录，然后确定数据包的真实内网IP地址，从而实现正确的逆SNAT。

还有一个问题，假设在极端情况下：
	
>两台服务器某一段时间访问同一网站，并且两个客户端使用的临时端口号也相同，那么这个时候NAT网关如何处理？如果NAT网关对于上行的数据包仅做src ip 也即内网IP的替换，那么回来的数据包将无法区分应该NAT到那台内网服务器。

猜测解决上面问题的办法是：

>NAT网关不仅仅对内网发出去的数据包做源IP地址替换，同时必要的情况也会对源端口进行替换，以使得从公网回来的数据包，需要NAT到不同内网服务器的数据包其目的端口号不同。

为此我们使用两台内网服务器，并且调整内网服务器对临时端口号的使用，触发“某一段时间两台内网服务器使用相同的临时端口号访问同一个网站”的场景，然后通过nf_conntrack确认上面的猜测是正确的，如下图所示，

![临时端口号相同时的NAT](http://7u2rbh.com1.z0.glb.clouddn.com/portsame.png)



enjoy the life !!!
