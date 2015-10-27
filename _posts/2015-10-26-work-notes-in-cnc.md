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


	
Have a fun！！！