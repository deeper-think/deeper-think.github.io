---
layout: post
title: 工作笔记（网宿）
category: 工作
tags: [work-notes]
keywords: work-notes
description:
---

> 用正确的工具，做正确的事情


## 1 2015年10月26日

### 1.1 tcpdump 抓TCP三次握手相关数据包

	// 捕获syn数据包
	tcpdump -i <interface> "tcp[tcpflags] & (tcp-syn) != 0" 
	
	// 捕获ack数据包
	tcpdump -i <interface> "tcp[tcpflags] & (tcp-ack) != 0"

	// 捕获fin数据包
	tcpdump -i <interface> "tcp[tcpflags] & (tcp-fin) != 0"

	// 捕获TCP SYN或ACK包
	tcpdump -r <interface> "tcp[tcpflags] & (tcp-syn|tcp-ack) != 0"

### 1.2 路由跟踪及链路状态监测

	mtr hostname or IP



	
Have a fun！！！