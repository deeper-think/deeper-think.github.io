---
layout: post
title: 虚拟机串口配置及使用
category: KVM虚拟化
tags: [KVM，console]
keywords: KVM，console
description: 
---

> 用正确的工具，做正确的事情

在虚拟机网络故障或者网络没有配置前，通过串口登录虚拟机是一种较为便利的方式。本文总结kvm虚拟机console配置的方法及使用方式。

## 虚拟机串口设备配置

采用libvirt来启动虚拟机，虚拟机libvirt配置文件中配置console口的配置如下：

	<devices>
		<console type='pty'>
		<target type='serial' port='0'/>
		</console>
    </devices>

## 虚拟机内部启用串口配置

### 配置securetty，允许串口登录

	echo “ttyS0” >> /etc/securetty

### 修改内核启动参数，将内核输出到 ttyS0

编辑/etc/default/grub文件，GRUB_CMDLINE_LINUX中增加 ：

	console=ttyS0

重新生成centos7 内核启动grub文件：

	grub2-mkconfig -o /boot/grub2/grub.cfg

### 修改inittab文件，内核启动时创建ttyS0

修改/etc/inittab文件，添加：

	S0:12345:respawn:/sbin/agetty ttyS0 115200

以使得内核启动时创建ttyS0。

## 通过console登录虚拟机

通过libvirt 客户端命令可以直接通过串口登录虚拟机：

	virsh console centos_7_0

退出串口登录使用组合件：

	ctrl+]



玩的开心 !!!
