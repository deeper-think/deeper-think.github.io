---
layout: post
title: linux常用配置
category: 工具
tags: [network, linux]
keywords: network, linux
description: 
---

> 我思故我在 -- 笛卡尔

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

enjoy the life !!!
