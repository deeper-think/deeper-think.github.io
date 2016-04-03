---
layout: post
title: Multipath TCP（译）
category: 网络协议
tags: [Multipath, TCP]
keywords: Multipath,TCP
description:
---

> 用正确的工具，做正确的事情


## 1 背景介绍

当前互联网倚重两大协议：IP和TCP。IP协议提供不可靠的数据报服务，确保互联网上任何主机之间可以交换数据包，自1970年被发明至今IP协议不断增加新的特性如multicast、IPsec、QoS，以及目前最新的IP协议版本提供对IPv6的支持。

TCP协议工作在协议栈传输层，用来在IP协议之上提供可靠的字节流服务，自首次在实验室发明后TCP协议也经历不断的演化。然而TCP协议最初设计上的一个问题至今仍然一直困扰着用户，孙然TCP和IP协议被设计成独立的协议，但是他们之间并不完全独立，传统TCP协议中一个tcp连接被定义成一个五元组（源IP、源端口号、目的IP、目的端口号、协议标示），这也就在整个TCP连接保持过程中将一个tcp连接绑定到了连接两端：服务器端、客户端 的IP，这样的问题是当一台主机的IP地址发生变化时，比如从Ethernet 环境切到Wi-Fi时，其已经建立的TCP连接均不能保持，所有的tcp连接都需要重新建立。

研究人员已经提出了多种方案通过对网络层协议栈进行修改来解决上面的问题，比如 Mobile IP，HIP (Host Identity Protocol)，已经Shim6 (Site Multihoming by IPv6 Intermediation)等。针对网络层协议栈实现的方案存在的缺陷是他对协议栈上层隐藏了所有关于IP地址的变化，但是这种IP地址变化的信息是TCP协议 拥塞控制机制需要感知的。

研究人员也提出了其他的解决方案，这种解决方案将多IP地址暴露给传输层协议，该方案已经被用于STCP（Stream Control Transmission Protocol），以实现在一个TCP连接中支持多IP地址。
STCP使用多IP地址最初被用于解决传输故障切换场景，最新的STCP版本已经能够支持在一个TCP连接里面同时使用多条路径。不幸地是除了一些特殊的应用，比如电话网络信令系统等，STCP并没有被广泛的部署使用，其中主要一个问题是当前的防火墙设施、NAT网关等不支持对STCP数据包的处理，仅仅是将STCP数据包丢弃，另外一个问题是STCP提供了另外一套API给上层应用程序，这对上层应用的开发造成一定的困扰，这两台问题导致产生了经典的的“鸡和蛋”的问题：网络设施制造商生产的硬件不支持STCP，因为没有应用在使用STCP协议，应用开发者开发应用也不采用STCP，因为网络设施如防火墙等不支持对STCP数据包的处理。STCP开发者已经尝试通过在UDP协议基础上对STCP进行封装，以此实现对上层提供统一的API接口，来解决“鸡和蛋”的问题，但实际情况是STCP仍然没有被广泛的使用。

MPTCP的设计初衷即为解决上述的问题，更确切的将MPTCP的设计目标是:

- 能够在一个TCP连接中使用多个网络路径；


- TCP连接会话过程中只要其中一条路径可用，该TCP连接的数据包传输就应该被阻塞；

- mptcp的多路径对上层应用保持透明，mptcp和传统tcp一样，为上层提供统一的API接口；

- 启动mptcp后，tcp连接仍然可以采用单路径传输。

下面的章节详细介绍mptcp底层的基础架构准则，这些基本的准则对于在理想的网络环境理解和部署使用mtcp是足够的，很不幸的是正如后面的解释，由于目前中间件的流行，理想的网络环境并不是目前互联网的环境。

## 2 基本原理及架构

### 2.1 MPTCP 基本原理及架构

在MPTCP环境下一个TCP连接包括多条数据流也即“子流”，应用程序间任然通过调用传统的socket API接口进行TCP数据的传输，MPTCP负责对“子流”进行管理，每条子流真正负责实现对TCP数据的传输。从整体架构的角度考虑，MPTCP作为中间层处于socket API与一条或多条TCP 子流之间，如图1所示。MPTCP环境中连接的两端主机需要交换额外的信息，目前通过利用TCP协议头部的option字段来实现额外信息的交互以达到如下目标：

- 建立一个新的MPTCP连接；

- 添加一个TCP子流到一个MPTCP连接中；

- 在一个MPTCP中进行数据的传输。

![MPTCP整体架构图](http://7u2rbh.com1.z0.glb.clouddn.com/paasch1.png)

### 2.2 MPTCP连接建立

传统TCP类型，MPTCP连接建立通过三次握手来完成，同时握手数据包option字段携带额外的信息用于指明是否使用MPTCP，MPTCP连接建立过程如图2所示。

![MPTCP连接建立过程](http://7u2rbh.com1.z0.glb.clouddn.com/paasch2.png)

### 2.3 MPTCP创建并添加新的子TCP流

图2展示了一个MPTCP连接上建立第一个子TCP流的过程。在该MPTCP连接上使用其他的网络接口创建并添加 子TCP流的过程如图3所示，为了将每个子TCP流关联到相关的MTCP连接，每个MPTCP连接在两端主机上都需要有唯一的标示，传统TCP连接使用四元组来唯一标示一个TCP连接（源IP地址、源端口号、目的IP地址、目的端口号），然而互联网上广泛存在的NAT设备导致在MPTCP场景下TCP四元组并不能作为连接的唯一标示，

MPTCP连接两端的HOST需要对新创建添加的子TCP流有唯一的标示，传统TCP连接使用四元组来唯一标示一个TCP连接然而因为互联网上NAT设备的存在，在MPTCP的场景 传统TCP四元组并不能在连接的两端唯一标示一个子TCP流，MPTCP为每一个连接分配一个本地唯一的Token，当一个MPTCP连接创建新的子TCP流时，三次握手SYN数据包携带MP_JOIN 选型，该选项携带其所关联的MPTCP Token信息，如图3所示。

![MPTCP连接创建新的子流](http://7u2rbh.com1.z0.glb.clouddn.com/paasch3.png)

聪明的读者应该已经注意到 MPTCP连接建立SYN包MP\_CAPABLE选型并没有携带Token， 但是子TCP流建立时仍然需要携带其关联MPTCP连接的Token，这主要是为了避免MP_CAPABLE占满所有的TCP Option的字节空间，Token本身是根据MP\_CAPABLE选项Key值的Hash得到的。

### 2.4 MPTCP拥塞控制

从拥塞控制的角度考虑使用MPTCP会导致一些有趣的问题，一个MPTCP连接使用多个子TCP流进行数据的传输，每个子TCP流对应一条单独的传输路径，同一个时刻每条传输路径的拥塞状态各不相同，一个解决方案是针对每一条单独的子TCP流使用传统TCP协议的拥塞控制方案，这种解决方案容易实现但是会产生不公平的问题，MPTCP连接与传统的TCP连接相比将占用更多的传输路径上的交换资源，如图4所示。

![MPTCP使用多条流进行传输](http://7u2rbh.com1.z0.glb.clouddn.com/paasch4.png)


## 参考：　

1. [Multipath TCP-译自](http://queue.acm.org/detail.cfm?id=2591369)
