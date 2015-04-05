---
layout: post
title: tcpdump常用命令
category: 工具
tags: [tcpdump]
keywords: tcpdump
description:
---

> 用正确的工具，做正确的事情

## 监视指定协议的数据包

打印TCP会话中的的开始和结束数据包, 并且数据包的源或目的不是本地网络上的主机.(nt: localnet, 实际使用时要真正替换成本地网络的名字))

    tcpdump 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0 and not src and dst net localnet'

## 使用tcpdump抓取HTTP包

    tcpdump  -XvvennSs 0 -s 0 -i eth0 tcp[20:2]=0x4745 or tcp[20:2]=0x4854

0x4745 为"GET"前两个字母"GE",0x4854 为"HTTP"前两个字母"HT"。-s 参数用于指定抓取数据包的长度，默认抓取数据包长度事68字节， -s 0 表示抓取完整数据包。

## tcpdump解析cap文件

    tcpdump -q -n -e -t -r *cap



玩得开心！！！
