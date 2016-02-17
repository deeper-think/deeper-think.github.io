---
layout: post
title: SR_IOV及虚拟机网卡直通
category: KVM虚拟化
tags: [SR_IOV，passthrough，KVM]
keywords: SR_IOV，passthrough，KVM
description: 
---

> 用正确的工具，做正确的事情

SR\_IOV，全称Single Root I/O Virtualization，定义了实现PCI-e设备在虚拟机之间高效共享的规范，通俗点讲也即对于支持SR_IOV的一块物理网卡（PF）可以可以虚拟出多个虚拟网卡（VF），每个虚拟网卡都有自己独立的PCI-e配置空间，同时又共享PF的一些功能如网卡控制器、网络接口等。

利用VT-D 的passthrough技术虚拟机直通VF，可以使得虚拟机网络IO的性能接近于物理环境。

因此SR\_IOV结合passthrough是一种可以很好的解决性能和可扩展性的IO虚拟化解决方案。

整体系统部署架构引用Ovirt上的一张图：

![PCI-e SR-IOV和Passthrough](http://7u2rbh.com1.z0.glb.clouddn.com/600px-Sr-iov.png)


## SR-IOV功能配置

Intel x85架构服务器 Intel I350网卡 SR-IOV功能配置过程：

- 首先通过修改bios确保已经开启硬件辅助虚拟化功能，如intel架构VT技术，修改内核启动项，添加内核启动参数（intel架构）：intel_iommu=on。

- 重新加载网卡驱动程序，并指定系统激活虚拟网口的数量：
	
	modprobe -r igb && modprobe max_vfs=4
	
	lspci| grep net

可以发现系统已经激活了4个vf虚拟网卡。


## kvm虚拟机直通VF

### 宿主机卸载VF

首先确定要卸载的pci设备的vendor & device和PCI设备编号：

	[root@localhost ~]# lspci -nn | grep net
	05:00.0 Ethernet controller [0200]: Intel Corporation I350 Gigabit Network Connection [8086:1521] (rev 01)
	05:00.1 Ethernet controller [0200]: Intel Corporation I350 Gigabit Network Connection [8086:1521] (rev 01)

假设要卸载第一块网卡：

	echo "8086 1521" > /sys/bus/pci/drivers/pci-stub/new_id
	echo 0000:05:00.0 > /sys/bus/pci/devices/0000:01:00.0/driver/unbind
	echo 0000:05:00.0 > /sys/bus/pci/drivers/pci-stub/bind

然后在查看当前系统pci设备信息，发现已经看不到该pci设备了。

也可以使用libvirt工具来卸载选定的pci设备，首先确定待卸载设备的相关PCI信息：

	[root@localhost ~]# virsh nodedev-list --tree
	computer
	  |
	.... 
	  |
	  +- pci_0000_00_1c_0
	  |   |
	  |   +- pci_0000_05_00_0
	  |   |   |
	  |   |   +- net_eth0_00_1e_67_45_7d_ff
	  |   |
	  |   +- pci_0000_05_00_1
	  |       |
	  |       +- net_eth1_00_1e_67_45_7e_00
	....

假设要卸载eth0,执行：

	virsh nodedev-dettach pci_0000_05_00_0

然后确认当前系统已经看不到该pci设备。


### kvm虚拟机直通VF 

使用libvirt启动虚拟机的话，libvirt配置文件：
	
	<devices> …
		<hostdev mode=’subsystem’ type=’pci’ managed=’yes’>
		<source>
        	<address domain=’0x000′ bus=’0x05′ slot=’0x00’ function=’0x0’/>
		</source>
	   	</hostdev>
	</devices>

使用qemu命令直接启动虚拟机：

	/usr/bin/qemu-kvm -name vdisk -enable-kvm -m 512 -smp 2 \
	-hda /mnt/nfs/vdisk.img \
	-monitor stdio \
	-vnc 0.0.0.0:0 \
	-device pci-assign,host=05:00.0



玩的开心 !!!
