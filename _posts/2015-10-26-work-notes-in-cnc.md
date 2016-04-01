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

	// 捕获RST包
	tcpdump -r <interface> "tcp[tcpflags] & (tcp-rst) != 0"

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


### 4 linux history 命令显示时间

	HISTTIMEFORMAT="%Y-%m-%d %H:%M:%S "
	

### 5 yum epel源地址

	http://dl.fedoraproject.org/pub/epel/beta/7/x86_64/epel-release-7-0.2.noarch.rpm
	
	http://dl.fedoraproject.org/pub/epel/
	

### 6 centos7 相关配置文件及命令

grub配置文件（静态配置文件，用于生成启动grub2文件）：

	/etc/default/grub

grub2文件（软连接，实际上连接到/boot/grub2/grub.cfg）：

	/etc/grub2.cfg

重新生成grub2文件：

	grub2-mkconfig -o /boot/grub2/grub.cfg

### 7 tar压缩文件命令

	tar -zcvf test.tar.gz ./test

压缩本地目录下的test文件，并生成压缩文件test.tar.gz。


### 8 centos发行版rpm包源码包

	http://vault.centos.org/6.6/updates/Source/SPackages/

	
### 9 squid 配置403响应

	acl rsp_403_domains reqdomain dlsw.baidu.com
	http_access deny rsp_403_domains

注意squid访问控制规则在配置文件中的先后顺序决定了规则生效的优先级，如果一个HTTP请求已经被前面的规则匹配，则会按照前面的策略实现访问控制，后面的规则则不会生效。

### 10 squid 未命中资源先响应部分请求头

	acl rsp_5bytes_domains url_regex ^http:.*
	reply_5bytes_in_advance_access allow rsp_5bytes_domains

### 10 squid清除已经缓存的内容
	/usr/local/squid/bin/squidclient -p 80 -m PURGE "url"

Have a fun！！！