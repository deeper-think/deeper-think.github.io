---
layout: post
title: linux常用配置
category: 工具
tags: [network, linux]
keywords: network, linux
description: 
---

> 用正确的工具，做正确的事情

### ifcfg常用配置项
    DEVICE=eth0
    BOOTPROTO=static
    TYPE=ether
    HWADDR=xx:xx:xx:xx:xx:xx (网卡mac地址，不用改)
    IPADDR=x.x.x.x(ip地址)
    NETMASK=x.x.x.x(子网掩码)
    BROADCAST=x.x.x.x.(广播地址)
    NETWORK=x.x.x.x(网络地址)
    GATEWAY=x.x.x.x(网关地址)
    ONBOOT=yes(开机自启动)
    DNS1=x.x.x.x(域名服务器地址)
    DNS2=x.x.x.x

### 修改网卡命名为eth0
首先，修改ifcfg文件名及配置项DEVICE的值，然后修改内核参数，添加启动参数 biosdevname=0，之后删除 /etc/udev/rules.d/70-persistent-net.rules。


### vim常用配置
    set expandtab //把tab转换成空格
    set shiftwidth=4 //When auto-indenting, indent by this much.【1】
    set softtabstop=4 //Make Vim treat <Tab> key as 4 spaces, but respect hard Tabs.
    set ignorecase //忽略大小写


### fstab文件、fdisk命令与e2label命令

fstab文件用于配置开机挂在的文件系统，fstab配置文件的格式：
	
	tmpfs                   /dev/shm                tmpfs   defaults        0 0
	devpts                  /dev/pts                devpts  gid=5,mode=620  0 0
	sysfs                   /sys                    sysfs   defaults        0 0
	proc                    /proc                   proc    defaults        0 0
	LABEL=/cache1           /cache1                 ext3    defaults        1 0

从左到右每一列的数据项的含义是：
分区名或者卷标：要挂载的文件系统的分区名(实体文件系统需要指定完整的分区路径)或卷标。
挂载点：文件系统的挂载点，系统上的一个目录。
文件系统类型：ext3、ext4等。
挂载类型：default或rw等。
是否备份：0表示否，1表示是。
是否开机检测：0表示否，1表示是。

fdisk命令用于物理磁盘探测，磁盘分区管理等。

e2label命令用于查看分区卷标或者设置分区卷标，我们可以通过分区的设备文件路径或者分区卷标来指定一个分区。



enjoy the life !!!
