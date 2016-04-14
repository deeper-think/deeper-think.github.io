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

## NAT服务器配置
	
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


























## 问题现象
最近公司运维反馈有几个线上项目有用户投诉，说用户播放搜狐视频时被公司的缓存系统引导到缓存服务器后视频就无法播放了。查看缓存服务器的log并没有发现请求失败的日志，更奇怪的是log并没有来自用户的HTTP Get请求日志。到测试环境去测试发现很容易复现问题，客户端wireshark抓包分析客户端请求的数据流。

#### 用户请求引导阶段
播放搜狐视频过程中 flash player不断向源站发起视频资源的HTTP Get请求，HTTP Get请求被公司的缓存系统捕获到并发送302重定向数据包来引导用户请求到缓存系统，如下所示：

![用户发起视频资源的HTTP Get请求并被302重定向引导](http://7u2rbh.com1.z0.glb.clouddn.com/302redirect.jpg)

从上图可以确认用户请求引导阶段是正常的。

#### 用户请缓存系统发起请求阶段
接着分析用户向缓存服务器发起的请求及缓存服务器的响应，发现用户收到302重定向后没有直接向缓存服务器的80端口再次发起视频资源的HTTP Get请求，而是尝试向缓存服务器的843端口建立TCP连接，但是缓存服务器没有开启843端口，因此多次syn请求都被服务器rst掉了，如下所示：

![用户向缓存服务器843端口发起syn后被缓存服务器rst掉](http://7u2rbh.com1.z0.glb.clouddn.com/syn-rst.png)

再分析接下来的数据流，发现客户端flash player 再也没有发起视频资源的请求，至此大致发现问题所在。

## 相关原理
谷歌搜索[flash ，843]关键字，发现是flash 10以后引入的一个新的特性：

> flash 10以后引入了“跨域策略文件”，主要用于视频服务器对本机资源的访问权限进行控制。 flash跨域请求访问视频资源时首先会向视频服务器发起“跨域策略文件”请求，如果请求失败，则表示该视频服务器禁止任何第三方域的flash跨域请求。

## 解决方法
缓存服务器或者前端负载均衡服务器必须要监听843端口，当有客户端发起policy-file-request请求时，响应正确的cross-domain-policy文件给客户端flash player。用C语言写了一个简单的TCP服务器来实现该功能。-- [github 项目地址](https://github.com/deeper-think/flash-policy-serv)

enjoy the life !!!
