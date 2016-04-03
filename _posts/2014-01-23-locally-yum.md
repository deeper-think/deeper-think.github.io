---
layout: post
title: CentOS利用ISO镜像搭建本地yum源
category: 系统及服务
tags: [本地源,yum]
keywords: yum
description:
---

> 用正确的工具，做正确的事情

### 1、挂载iso镜像到/mnt目录

	 mount -o loop /dev/cdrom /mnt/

### 2、创建yum配置文件 /etc/yum.repos.d/CentOS-Local.repo

	[Local]
	name=CentOS-5.8.local
	baseurl=file:///mnt
	gpgcheck=0
	enabled=1
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-5


### 3、更新yum缓冲

	yum clean all

### 4、安装gcc，确认本地源搭建成功

	yum install gcc


玩得开心！！！
