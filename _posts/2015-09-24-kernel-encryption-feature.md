---
layout: post
title: 工作笔记：传输加密功能
category: 个人笔记
tags: [内核、传输加密]
keywords: 内核、传输优化、传输加密
description:
---


> 对协议栈原有的IP报文进行加密（可选），并将加密后IP报文作为payload重新封装TCP/UDP头和IP头，从而把多条TCP/UDP流归并成一条TCP/UDP流进行传输。

## 1 传输加密功能相关文件

加密服务配置文件：

	/usr/local/stadmin/etc/stadmin.cfg

加密服务管理脚本：
	
	/etc/rc.d/init.d/stadmind

服务配置可执行文件及内核加密实现模块：

	/bin/c_stadmin
	[root@FM-31798 ~]# lsmod | grep st
	c_stcodec              16313  0
  
日志文件：

	/var/log/stadmind.log
	


## 2 传输加密服务配置

配置文件说明：

	[Mode]
	#route_mode=0: encyption mode; route_mode=1: routing mode
	route_mode=0

	[Role]
	#identity=client: A or AN node; identity=server: C or BN node; idetity=none: not set port
	identity=none//none表示不开启加密功能

	[EncryptMode]
	#encode_port: if the destination port is in the range of A-B, then the segment will be  encrypted
	encode_port=3800-3800    //对目的端口为3800的数据包做加密
	#conn_srcport:  source port setting in fack IP head
	conn_srcport=28288-28288 //伪造的TCP/UDP头的源端口号字段值
	#conn_dstport:  dest port setting in fack IP head
	conn_dstport=30000-30000 //伪造的TCP/UDP头的目的端口号字段值
	#conn_fackip:  fack ip setting in fack IP head
	conn_fackip= //伪造的IP头其源IP地址字段的值

查看服务当前状态：

	service stadmind status
  
	
	destination:
	encode_source_port:
	encode_dst_port:         //加密数据包的端口范围
         3800-3800
	source:
         28288-28288        //伪造tcp头 源端口号的值
	dst:
         30000-30000        //伪造tcp头 目的端口号的值
	passwd: 80 40 20 10  8  4  2  1  1  2  4  8 10 20 40 80
	status: stop            //服务状态，stop表示服务停止 
	src: local              //伪造ip头 的源IP地址，local表示为本机真实IP地址

注意：只有encode_dst_port不为空，并且status的值为running，表示内核加密功能正在工作。

## 3 开启加密后的tcpdump抓包问题

	echo 1 > /proc/sys/net/ipv4/tcpdump_sw

之后可以抓到真实的tcp连接通信数据包，否则只能抓到伪造的数据包，也即（如上面的配置）源端口为28288以及目的端口为30000的数据包。

例如在父节点上抓包，echo 0 > /proc/sys/net/ipv4/tcpdump_sw 后抓到的是加密数据包：

	15:07:02.502110 IP 101.100.190.44.28288 > 124.115.20.199.30000: . 187:227(40) ack 934574 win 2363
	15:07:02.502113 IP 101.100.190.44.28288 > 124.115.20.199.30000: . 187:227(40) ack 935974 win 2369
	15:07:02.502116 IP 101.100.190.44.28288 > 124.115.20.199.30000: . 187:227(40) ack 937374 win 2375
	15:07:02.502119 IP 101.100.190.44.28288 > 124.115.20.199.30000: . 187:227(40) ack 938774 win 2380
	15:07:02.502123 IP 101.100.190.44.28288 > 124.115.20.199.30000: . 187:227(40) ack 940174 win 2386

在父节点上抓包，echo 1 > /proc/sys/net/ipv4/tcpdump_sw 后抓到的是解密数据包：

	15:14:34.563531 IP 101.100.190.44.48951 > 124.115.20.199.3800: . ack 920075 win 2432
	15:14:34.563534 IP 101.100.190.44.48951 > 124.115.20.199.3800: . ack 922875 win 2432
	15:14:34.564058 IP 101.100.190.44.48951 > 124.115.20.199.3800: . ack 925675 win 2432
	15:14:34.564108 IP 101.100.190.44.48951 > 124.115.20.199.3800: . ack 928475 win 2432
	15:14:34.564116 IP 101.100.190.44.48951 > 124.115.20.199.3800: . ack 931275 win 2432

