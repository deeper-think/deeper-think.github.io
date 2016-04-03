---
layout: post
title: flashcache混合存储环境搭建
category: 系统及服务
tags: [混合存储, flashcache]
keywords: flashcache
description: 
---

> 我思故我在 -- 笛卡尔

公司线上的一些服务，为了降低IO的延迟，当前硬件上采用全SSD存储的配置，成本比较高，希望能够采用混合存储的模式(SSD+SATA)来降低成本，为此测试facebook的flashcache混合存储和全SSD，在真实线上业务IO下IO的性能差别。

flashcache的功能需要通过编译源码的方式得到KO文件，将KO插入内核来得到，在CentOS6.X上面编译flashcahce，只要安装了内核源码树(kernel-devel包)，一切都很正常，妥妥的很顺利。然而线上服务器使用的5.X的CentOS，在已经安装kernel-devel rpm的情况下，编译的过程中直接报错：/usr/src/redhat/SPECS目录不存在之类的，分析MakeFile文件，发现如果是redhat-5.X的发行版（centos也是属于redhat，/etc/redhat-release），那么到/usr/src/redhat/BUILD下找内核源码树来用，SHIT！以前都是使用6.X的发行版，源码树一般在/usr/src/kernels目录下，MakeFile通过引用/lib/modules/2.6.18-308.el5/build获取，第一次碰到5.X的发行版这么奇葩。

废话不多说，中间的过程很曲折，最后总结解决问题的方法步骤很简单，这里写出来分享，也避免以后还需要搭建flashcache环境会犯同样的错误。

## 安装RPM编译环境：

    yum install rpm-build

上面的命令会在/usr/src下产生一个redhat目录，所有的子目录都为空。

## 安装内核源码：

    rpm -i kernel-2.6.18-274.el5.src.rpm

这会在/usr/src/redhat/SOURCES下安装内核源码包，以及在/usr/src/redhat/SPECS产生内核的SPEC文件。修改内核的SPEC文件，将版本号改成目标内核对应的版本号。

## 创建内核源码树：

    rpmbuild -bp --target=$(uname -m) kernel.spec

上面的命令在/usr/src/redhat/BUILD目录下生成内核源码树，在执行上面命令之前需要先安装一些rpm包：

    yum install rpm-build redhat-rpm-config unifdef
　　
执行上面的步骤之后，就可以正常编译flashcache源码并生成flashcache.ko文件，执行：

    insmod /lib/modules/2.6.18-308.el5/extra/flashcache/flashcache.ko
　　
执行命令查看内核模块：

    lsmod | grep flashcache

结果如下：

    [root@localhost flashcache-stable_v3.1.1]# lsmod | grep flashcache
    flashcache            124352  0
    dm_mod                102417  12 flashcache,dm_multipath,dm_raid45,dm_snapshot,dm_zero,dm_mirror,dm_log

表明flashcahce编译安装成功。


玩的开心 !!!
