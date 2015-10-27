---
layout: post
title: 工作笔记（网宿）
category: 工作
tags: [work-notes]
keywords: work-notes
description:
---

> 用正确的工具，做正确的事情

### 1 tcpdump 抓TCP三次握手相关数据包

	// 捕获syn数据包
	tcpdump -i <interface> "tcp[tcpflags] & (tcp-syn) != 0" 
	
	// 捕获ack数据包
	tcpdump -i <interface> "tcp[tcpflags] & (tcp-ack) != 0"

	// 捕获fin数据包
	tcpdump -i <interface> "tcp[tcpflags] & (tcp-fin) != 0"

	// 捕获TCP SYN或ACK包
	tcpdump -r <interface> "tcp[tcpflags] & (tcp-syn|tcp-ack) != 0"

### 2 路由跟踪及链路状态监测

	mtr hostname or IP

### 3 常用脚本参数
	
echo 输出不换行：
	
	echo -n "hello world!!"

set 命令用法：

	[root@bogon xuc]# set $(/sbin/runlevel)
	[root@bogon xuc]# runlevel=$2
	[root@bogon xuc]# previous=$1
	[root@bogon xuc]# echo $runlevel $previous
	5 N
	[root@bogon xuc]# echo $(/sbin/runlevel)
	N 5

### 4 linux 开机自启动或自执行

/etc/rc.local文件中配置linux系统开机自动执行脚本：
	
	#!/bin/sh
	#
	# This script will be executed *after* all the other init scripts.
	# You can put your own initialization stuff in here if you don't
	# want to do the full Sys V style init stuff.

	touch /var/lock/subsys/local

	sh /usr/local/bin/record_system_boot
	
chkconfig 命令可以查看、修改、配置开机自启动服务：

	[root@bogon xuc]# chkconfig --list httpd
	httpd           0:off   1:off   2:off   3:off   4:off   5:off   6:off
	[root@bogon xuc]# chkconfig --level 23456 httpd  on
	[root@bogon xuc]# chkconfig --list httpd
	httpd           0:off   1:off   2:on    3:on    4:on    5:on    6:on

这样 系统启动级别为23456时会自动启动httpd 服务, 服务可执行文件存在于：
	
	 /etc/rc.d/init.d/

关于系统当前启动级别：

	[root@bogon xuc]# runlevel
	N 5

修改系统启动级别，/etc/inittab:
	
	\# Default runlevel. The runlevels used are:
	\#   0 - halt (Do NOT set initdefault to this)
	\#   1 - Single user mode
	\#   2 - Multiuser, without NFS (The same as 3, if you do not have networking)
	\#   3 - Full multiuser mode
	\#   4 - unused
	\#   5 - X11
	\#   6 - reboot (Do NOT set initdefault to this)
	\#

	id:5:initdefault:


Have a fun！！！