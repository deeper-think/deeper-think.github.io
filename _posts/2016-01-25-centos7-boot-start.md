---
layout: post
title: centos7开机启动问题
category: 技术
tags: [centos7，rc.local]
keywords: centos7，rc.local
description: 
---

> 用正确的工具，做正确的事情

centos6及之前版本的OS，开机启动执行的脚本可以通过直接写到/etc/rc.d/rc.local文件中来实现，然而测试发现centos7 os 写到该文件中的脚本开机并没有被执行，在redhat官方bugzilla中找到了问题的原因-- [bugzilla链接](https://bugzilla.redhat.com/show_bug.cgi?id=1178488)，重点信息：

	This is intentional, we want users to understatnd that rc.local does not and can't work in the same way as in rhel6.
	There is a explanatory comment in the rc.local.
	
	# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
	#
	# It is highly advisable to create own systemd services or udev rules
	# to run scripts during boot instead of using this file.
	#
	# In contrast to previous versions due to parallel execution during boot
	# this script will NOT be run after all other services.
	#
	# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
	# that this script will be executed during boot.

从官方回复看，rc.local中脚本开机时没有自动执行并不是系统bug，而是故意为之，新版本OS希望使用systemd services的方式来实现开机执行或者开机启动，看了 rc.local文件已经在弃用的路上了。

下面验证了通过systemd services来实现开机执行命令，首先是创建一个自己的systemd 服务，在
/usr/lib/systemd/system/ 目录创建服务配置文件bootstart.service：

	[Unit]
	Description=bootstart      //服务描述
	
	[Service]
	Type=forking               //服务后台执行
	ExecStart=/usr/local/bootstart/bin/bootstart.sh   //服务启动文件或者脚本
	PrivateTmp=true                                   //创建独立命名空间
	
	[Install]
	WantedBy=multi-user.target                      

其中/usr/local/bootstart/bin/bootstart.sh：
	
	#!/bin/bash
	
	echo "`date`:system start !!" >> /usr/local/bootstart/log/bootstart.log
	
然后设置服务开机启动：

		systemctl enable bootstart.service

重启系统查看日志文件，可以发现脚本中的命令被开机执行。





玩的开心 !!!
