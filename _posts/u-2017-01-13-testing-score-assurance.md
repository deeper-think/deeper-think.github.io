---
layout: post
title: 拨测指标优化方案
category: 工作笔记
tags: [拨测指标优化]
keywords: 拨测指标优化方案
description:
---


> 用正确的工具，做正确的事情

## 1 总体部署方案
![总体部署方案](http://7u2rbh.com1.z0.glb.clouddn.com/%E6%80%BB%E4%BD%93%E9%83%A8%E7%BD%B2%E6%96%B9%E6%A1%88.jpg)

总体方案说明：

1. 探针的流量经过局方交换机，局方交换机配置路由策略，将探针IP的全部流量（或者将探针IP的80/8080端口的流量，根据实际优化需求确定）策略到网宿互联交换机；
2. 网宿互联交换机上配置路由策略，将探针IP的全部流量绑定到一台frigate与squid合设的服务器；
3. 通过frigate及squid对探针的流量进行特殊处理，对探针拨测的指标进行优化。 

## 2 HTTP拨测指标保障方案

HTTP拨测指标的保障主要利用squid缓存功能以及先响应5字节的功能进行优化，可以对HTTP拨测的首字节时延、HTTP下载速率、90%元素的下载时延等的指标进行保障。

### 2.1 HTTP拨测指标优化总体方案及关键配置
总体的思路是探针的流量到frigate与squid合设服务器后，frigate将所有HTTP的流量识别出来疏导到本机squid服务，非HTTP的流量直接走本地source回源, frigate主配置文件配置：

    // routes(for AN only):
    //    PEER      -- route to frigate peer(BN)
    //    SOURCE    -- route to source(original server) directly
    // default: PEER cluster_label
	"default_route":"SOURCE", 
	....
	"routing":[
		"IP auto:tag_reset reset",
		"IP dg_reset reset"
                ],
	//routing规则可以清空

frigate http模块配置文件module_http.conf 配置：

	"nextproxies":[
                        // "<label> <balance> <peers=[ip:port ip:port ip:port]>"
                        "npSmall HASH 111.8.11.83:8101"
                ],

	"nextproxy_clusters":[
                        // "<label> <primary groups> <standby groups>"
                        //"np_clusterBig npBig:1",
                        "np_clusterSmall npSmall:1"
                ],

	"routing":[
				//"DOMAIN other_domains source",
        		"DOMAIN * NEXTPROXY np_clusterSmall"
              ],

上面的配置实现将本机所有的HTTP流量识别出来疏导到squid，其他所有的非HTTP流量疏导走本地出口回源。

通过配置squid强制缓存功能来优化HTTP拨测的首字节时延、HTTP下载速率、90%元素的下载时延等的指标，squid关键配置如下：
	
	//配置squid强制缓存24小时
	ck_bodylen off
	refresh_pattern -i  . 1440 100% 1440 override-expire ignore-all
	dyn_cache allow all

### 2.2 HTTP拨测指标优化常见问题及解决思路

以下是对HTTP拨测指标优化常见问题的总结和解决方法的记录。

#### 2.2.1 squid强制缓存后，探针所有的请求都被命中，探针拨测的首字节时延会非常低，不符合真实的网络情况

解决的方案是squid上对主页请求不进行缓存，让探针的主页请求在squid上不被命中，同时配置先响应5字节并且响应5字节时增加一个范围内的随机时延，这样在保障下载速率及90%元素下载时延的情况下可以将首字节时延也控制在预期的范围内，squid配置如下：
	
	//先响应5字节，控制不命中时首字节时延
	reply_5bytes_in_advance_access allow all
	reply_5bytes_in_advance_defer_time_max 120
	reply_5bytes_in_advance_defer_time_min 50
	
	//配置某些主页域名不缓存
	acl no_cache_url url_regex -i ^http://www.xunlei.com/$
	dyn_cache deny no_cache_url
	cache deny no_cache_url


#### 2.2.2 某些主页域名链接的一些二级元素，这些二级链接元素的下载时延非常大，影响主页域名90%元素的下载时延以及页面整体的下载速率

解决方案是squid上针对这些二级链接元素直接响应403，squid配置如下：
	
	//对google相关的元素请求，全部响应403
	acl rsp_403_regurl url_regex http://.*google.*
	http_access deny rsp_403_regurl

#### 2.2.3 squid强制缓存后，尤其是最主页域名强制缓存后，可能页面更新不及时，导致客户登录探针用浏览器访问页面时得到一些过期的页面

解决方案是在squid服务器上部署一个定时脚本任务，该脚本每天晚上凌晨的时候清空主页域名的缓存，通过wget模拟用户过squid请求主页，再次触发squid回源并缓存最新的主页面文件，脚本可以找李进锦提供。

#### 2.2.4 其他

待发现补。


## 3 ping/tcping拨测指标保障方案
	
在线路质量没法保障ping及tcping拨测 丢包率、时延以及连接成功率指标的情况下，通过frigate劫持并响应用户的ping包，或者tcping三次握手在frigate上完成，同时为了保障ping时延以及tcping时延的合理性以及丢包率的合理性，需要在frigate上配置随机时延以及随机丢包。

### 3.1 ping 拨测指标保障方案及关键配置
总体的思路是探针的流量到frigate服务器后，frigate将所有icmp的流量劫持，并用tc的方式限制，对frigate发出的icmp报文加入一定的随机延时值。frigate配置如下：

        "icmp":{
                // 0: disable, 1:kernel, 2:userspace
                "mode":2 ,
                "read_timeout":10,
                 ...
               }

 tc脚本如下：
           #!/bin/sh
           //延时作用在哪个网卡上，及对应的时延、抖动值
           DEV=bond1
           DELAY=30
           VARIATION=10
 
           // 在PREROUTING处设置匹配条件，可以匹配目的IP等，根据匹配条件给这条连接打上mark 1。这里的匹配条件可以根据实际情况做修改。
           iptables -t mangle -A PREROUTING -p icmp -s 120.240.0.114 -j CONNMARK --set-mark 1
           iptables -t mangle -A POSTROUTING -p icmp --icmp-type echo-request,echo-reply -j CONNMARK --restore-mark
 
           // 定义tc规则及优先级队列。
           tc qdisc del dev ${DEV} root 2>/dev/null
           tc qdisc add dev ${DEV} root handle 1: prio

           // 添加一个延时队列，进入这个队列的报文会被延时。如果使用distribution normal参数的话，指定的是正态分布，所以会有很小的概率会取到正态分布曲线边上的数值，导致出现延时很小或者延时很大的情况，如果不希望出现这种情况可以把distribution normal参数去掉。
           tc qdisc add dev ${DEV} parent 1:1 handle 10: netem delay `expr ${DELAY}`ms `expr ${VARIATION}`ms distribution normal
            
			// 设置过滤条件。handle 1 fw flowid 1:1表示打上mark 0x1的报文会进入1:1队列，从而被延时。
           tc filter add dev ${DEV} protocol ip parent 1:0 prio 1 handle 1 fw flowid 1:1

		   （丢包率、）

### 3.2 tcping 拨测指标保障方案及关键配置
思路是探针的流量到frigate服务器后，frigate将所有tcp的流量识别出来，对frigate发出的syn-ack报文加入一定的随机延时、抖动值，并在网卡设置一定丢包率回复客户端。该功能在预发布的frigate-1.7.5.0版本支持，frigate配置如下：

        "tcp":{
              "synack_enable":false,
              "synack_cache_timeout":0,
               ……
              },
         "tcping_delay":{
             //可以设置普通时间及高峰期，一一对应时延（抖动）值和丢包率。
                "busy_time": "19:00-23:00",
                "busy": [
                        // tag delay(ms) wave(ms) loss(%)
                        "tag1 50 20 0.6",
                        "tag2 60 10 0.6"
                ],
                "idle": [
                        // tag delay(ms) wave(ms) loss(%)
                        "tag1 30 10 0.5",
                        "tag2 40 10 0.4"
                ],
                "groups":[
                        // tag  src_ip  dst_ip
                        "tag1 0.0.0.0/0 2.2.2.2/24",
                        "tag1 172.1.1.1 0.0.0.0/24",
                        "tag2 172.1.1.1 2.2.2.2/24"
                ]
        },


对于目前frigate版本，是通过frigate劫持探针tcp流量并通过tc控制的方式实现tcping的优化，其配置如下：
frigate配置：（同上）

        "tcp":{
              "synack_enable":false,
              "synack_cache_timeout":0,
               ……
              },

tc脚本配置（通过tc脚本只能增加随机时延但是不能添加随机丢包）：

         #!/bin/sh
         DEV=bond1
         DEV2=bond0
         DELAY1=20
         VARIATION1=3
         DELAY2=30
         VARIATION2=5

		  //给某些类型的流量定义添加标签
         iptables -t mangle -A PREROUTING -p tcp -s 120.198.124.167 -j CONNMARK --set-mark 0x1
         iptables -t mangle -A PREROUTING -p tcp -s 120.240.14.234 -j CONNMARK --set-mark 0x1
		 iptables -t mangle -A PREROUTING -p tcp -s 120.236.240.228 -j CONNMARK --set-mark 0x2
         iptables -t mangle -A PREROUTING -p tcp -s 183.233.203.2 -j CONNMARK --set-mark 0x2
         iptables -t mangle -A POSTROUTING -p tcp --tcp-flags SYN,ACK,FIN,RST SYN,ACK -j CONNMARK --restore-mark
		 //定义TC优先级队里
         tc qdisc del dev ${DEV} root 2>/dev/null
         tc qdisc add dev ${DEV} root handle 1: prio
         
	     
		 tc qdisc add dev ${DEV} parent 1:1 handle 10: netem delay `expr ${DELAY1}`ms `expr ${VARIATION1}`ms distribution normal
         tc qdisc add dev ${DEV} parent 1:2 handle 20: netem delay `expr ${DELAY2}`ms `expr ${VARIATION2}`ms distribution normal
         tc qdisc add dev ${DEV} parent 1:3 handle 30: netem delay `expr ${DELAY3}`ms `expr ${VARIATION3}`ms 

	     // 设置过滤条件。handle 1 fw flowid 1:1表示打上mark 0x1的报文会进入1:1队列，从而被延时。
         tc filter add dev ${DEV} protocol ip parent 1:0 prio 1 handle 1 fw flowid 1:1
         tc filter add dev ${DEV} protocol ip parent 1:0 prio 1 handle 2 fw flowid 1:2
         tc filter add dev ${DEV} protocol ip parent 1:0 prio 1 handle 3 fw flowid 1:3


### 3.3 注意事项及相关问题和解决方案

#### 3.3.1 关于frigate重启导致tc脚本的相关iptables规则丢失的问题

frigate重启过程会清空系统iptables规则并且在启动的过程重新配置frigate使用的iptables规则，但是因为tc脚本对于frigate来说是未知的，tc脚本相关的iptables规则会被清空但是不会被重新配置，因此：


> tc 脚本被执行之后，还需要而外的措施来保障当系统iptables规则本服务器上其他应用给全部被清空后，重新执行tc脚本来配置tc相关的iptables规则。

当前比较可行的一个办法是通过crontab定时任务，周期性的检查tc脚本相关的iptables规则是否存在，如果不存在重新执行tc脚本来设置tc相关的iptables规则。



#### 3.3.2 其他

待发现补充。 


