---
layout: post
title: netmap安装及收发包测试
category: 技术
tags: [netmap]
keywords: netmap
description: 
---

> 我思故我在 -- 笛卡尔

netmap是一款高性能网络IO架构，主要通过两点实现快速的收发包：

> 1、绕过内核协议栈，简化收发包的流程。
  2、通过映射，实现用户态收发包的零拷贝。
  3、通过一次系统调用获取多个数据包，减少系统调用的负荷。

netmap已经合入到freebsd中，然而当前还没有合入到linux kernel中，因此在linux发型版上使用netmap，需要手动编译相关的模块，并做相应的替换。netmap的编译需要相应内核的源码树，这里注意的是可能netmap针对不同的内核版本可能存在兼容性的问题，在某些内核下编译netmap会报错，验证在kernel-2.6.37下编译netmap可以成功。编译过程：

## 编译内核安装内核并构建内核源码树

    [root@localhost home]# tar -xvf linux-2.6.37.tar.bz2
    [root@localhost home]# cd linux-2.6.37
    
    [root@localhost linux-2.6.37]# cp /boot/config-2.6.32-358.el6.x86_64 ./.config
    
    [root@localhost linux-2.6.37]# make menuconfig
    
    [root@localhost linux-2.6.37]# make all 
    
    [root@localhost linux-2.6.37]# make modules_install && make install

## 编译netmap模块及支持netmap的网卡驱动

    [root@localhost home]# cd netmap/
    [root@localhost netmap]# pwd
    /home/netmap
    
    [root@localhost netmap]# cd LINUX/
    [root@localhost LINUX]# make KSRC=/home/linux-2.6.37

上面的命令执行成功后，会生成netmap内核模块，已经支持netmap的响应网卡驱动：

    [root@localhost LINUX]# find . -name "*.ko"
    ./ixgbe/ixgbe.ko
    ./e1000e/e1000e.ko
    ./virtio_net.ko
    ./r8169.ko
    ./igb/igb.ko
    ./e1000/e1000.ko
    ./netmap_lin.ko

另外执行：

    make KSRC=/home/linux-2.6.37 apps

可以编译一些netmap测试程序，执行以上命令成功将会在/home/netmap/examples目录下生成一些可执行程序：

    [root@localhost examples]# ./
    bridge       testlock     test_select
    pkt-gen      testmmap     vale-ctl

到此netmap编译成功。

## netmap收发包测试

使用测试程序pkt-gen，测试netmap收发包，我这边使用的是82545EM网卡以及e1000驱动为例(vm虚拟机)。首先，更换网卡驱动并加载netmap模块：

    [root@localhost netmap]# lsmod | grep e1000
    e1000                 143791  0
    
    [root@localhost netmap]# mv /lib/modules/2.6.37/kernel/drivers/net/e1000/e1000.ko /lib/modules/2.6.37/kernel/drivers/net/e1000/e1000.ko-bak ###避免模块自动加载
    
    [root@localhost netmap]# rmmod e1000
    [root@localhost netmap]# insmod /home/netmap/LINUX/e1000/e1000.ko ###自动加载netmap_lin模块

成功执行完上述命令，可以在dmesg中看到如下日志信息：

    824.462575 [2606] netmap_init               run mknod /dev/netmap c 10 59 # error 0
    netmap: loaded module
    191.789427 [1049] netmap_mem_global_config  reconfiguring

然后，执行以下命令可以直接从网卡收发包：

    # send about 500 million packets of 60 bytes each.
    # wait 5s before starting, so the link can go up
    pkt-gen -i eth0 -f tx -n 500111222 -l 60 -w 5
    # you should see about 14.88 Mpps
      
    pkt-gen -i eth0 -f rx # act as a receiver


玩的开心 !!!
