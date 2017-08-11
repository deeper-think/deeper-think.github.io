---
layout: post
title: centos7 系统启动及模块加载相关问题分析总结
category: 系统及服务
tags: [centos7,module]
keywords: centos7,module
description: 
---

> 用正确的工具，做正确的事情

最近公司运维反馈项目上一些服务器的网卡丢包，优化和解决网卡丢包的过程涉及到igb驱动升级以及修改网卡队列个数的一些操作，但在执行这些操作的过程碰到一些比较困惑的问题，存在困惑的原因归根到底在于对centos7系统启动过程了解不够清晰，下面简单总结centos7 启动的大致过程，以及igb驱动升级以及修改网卡队列个数这些操作遇到的问题的分析和解决。

## centos7 系统启动流程梳理

linux系统启动是一个复杂的过程，这里只是大致梳理其启动过程，另外不同发行版不同版本号的linux系统启动过程存在很大的差异，这里是对centos 7系统启动过程的大致梳理， 主要是以下四个阶段：



### BIOS 阶段

这个阶段硬件上电，BIOS启动并开始硬件自检，之后BIOS根据配置的启动顺序开始从相应的介质载入BOOT LOAD程序，如果配置的是优先从本地硬盘启动，那么BIOS从本地硬盘的主引导扇区载入 BOOT LOAD程序到内存，然后将控制权交给BOOT LOAD程序。

> 开机按F2可以进入BIOS配置界面，在BIOS配置界面可以设置系统的启动顺序，比如从本地硬盘启动，或者从CDROM启动，或者PXE启动。

### Boot Load阶段

centos Boot Load程序实体为grub2程序，开机过程进入Boot Load阶段的标志是出现选择要启动的内核的界面，之后 grub2程序会载入boot目录下的两个及其重要的文件：

> 内核镜像vmlinuz-`uname -r` 和 initramfs-`uname -r`.img, vmlinuz-`uname -r` 也即内核镜像，就是内核编译时直接编译进内核的的那部分代码的二进制。 initramfs-`uname -r`.img，initial RAM File System ，是一个内存文件系统，该内存文件系统存放了内核驱动以及系统启动相关的一些文件。

grub2载入上述两个系统启动相关的镜像文件并且将 grub中指定的kernel command line 传递给kernel 后，接下来开始kernel启动的过程，系统启动的控制权交给kernel。

> centos7系统 /etc/default/grub 文件的 GRUB_CMDLINE_LINUX 配置项用来配置需要传递给内核的参数， 修改该配置项之后需要通过命令grub2-mkconfig > /boot/grub2/grub.cfg 重新生成grub2的配置文件。

### kernel阶段

kernel的启动需要完成系统内存、CPU的初始化，比如内存的分页机制在这个阶段完成初始化，CPU从实模式切换到虚模式，系统对内存的读写开始利用虚拟内存的机制。但是一些外围设备的内核驱动并没有在这个阶段加载，当然应用层的服务器也没有在这个阶段启动，kernel完成系统核心模块的加载和初始化之后会拉起系统第1个进程（1号进程） systemd服务：
	
	-bash-4.2# ps -ef | grep system
	root          1      0  0 Jul31 ?        00:01:34 /usr/lib/systemd/systemd --switched-root --system --deserialize 21

kernel 将系统启动的控制权交给 systemd服务。

###systemd阶段

kernel阶段已经完成了kernel核心模块的加载和初始化，这个时候核心系统已经就绪，比如内存子系统，CPU子系统已经可以正常工作，CPU也从实模式切换到虚模式，那么systemd服务需要完成剩下的两类工作：

1. 外围设备相关驱动的加载，实现外围设备可以正常工作，但是这个过程加载的驱动不是；
2. 完成系统用户态服务的启动。

这个阶段完成之后，整个系统启动过程就结束了，系统可以正常工作和使用了。


## 相关问题的总结

### 配置igb网卡RSS队列数

首先确认igb驱动是否支撑RSS参数，如下：

	[root@localhost ~]# modinfo igb
	filename:       /lib/modules/3.10.0-514.26.2.el7.x86_64/updates/drivers/net/ethernet/intel/igb.ko
	version:        5.3.5.7
	license:        GPL
	description:    Intel(R) Gigabit Ethernet Network Driver
	author:         Intel Corporation, <e1000-devel@lists.sourceforge.net>
	rhelversion:    7.3
	srcversion:     EF38ED61A1F74215DA95019
	......
	alias:          pci:v00008086d00001F41sv*sd*bc*sc*i*
	alias:          pci:v00008086d00001F40sv*sd*bc*sc*i*
	depends:        i2c-core,ptp,dca,i2c-algo-bit
	vermagic:       3.10.0-514.26.2.el7.x86_64 SMP mod_unload modversions 
	parm:           InterruptThrottleRate:Maximum interrupts per second, per vector, (max 100000), default 3=adaptive (array of int)
	parm:           IntMode:Change Interrupt Mode (0=Legacy, 1=MSI, 2=MSI-X), default 2 (array of int)
	parm:           Node:set the starting node to allocate memory on, default -1 (array of int)
	.........
	parm:           RSS:Number of Receive-Side Scaling Descriptor Queues (0-8), default 1, 0=number of cpus (array of int)
	.........
	parm:           LRO:Large Receive Offload (0,1), default 0=off (array of int)
	parm:           debug:Debug level (0=none, ..., 16=all) (int)

如果parm部分没有出现RSS这个参数，说明当前igb驱动没有提供RSS接口，这个时候不可以传递这个参数给igb驱动，否则会导致驱动加载失败。然后通过kernel command line 传递启动参数给igb驱动：

	GRUB_CMDLINE_LINUX="rd.lvm.lv=centos/root rd.lvm.lv=centos/swap crashkernel=auto rhgb quiet console=ttyS0 igb.RSS=8"
	
	grub2-mkconfig > /boot/grub2/grub.cfg

重启系统即可。

### igb网卡驱动升级相关的问题

centos7通过编译源码来升级igb驱动，make && make install 之后重启系统后发现系统加载的还是原来的igb驱动，主要是因为centos7开机启动机制发生了一系列的变化导致， 正确的方式是：

1. 编译igb源码，执行make && make install；
2. depmod -a ，重新生成 /usr/lib/modules/`uname -r`/modules.dep;
3. 重新生成initramfs-`uname -r`.img 文件，

	mv /boot/initramfs-3.10.0-514.26.2.el7.x86_64.img /boot/initramfs-3.10.0-514.26.2.el7.x86_64.img.bak
	
	dracut -f -v

	lsinitrd /boot/initramfs-3.10.0-514.26.2.el7.x86_64.img | grep igb //确认是否已经更新为新的驱动

4. 重启系统


玩的开心 !!!
