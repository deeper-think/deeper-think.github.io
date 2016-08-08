---
layout: post
title: 工作随记
category: 工作笔记
tags: [work-notes]
keywords: work-notes
description:
---

> 用正确的工具，做正确的事情

### 1 gdb常用调试命令

	backtrace（或bt）	\\查看各级函数调用及参数
	finish				\\连续运行到当前函数返回为止，然后停下来等待命令
	frame（或f） 		\\帧编号	选择栈帧
	info（或i） 			\\locals	查看当前栈帧局部变量的值
	list（或l）			\\列出源代码，接着上次的位置往下列，每次列10行
	list 行号			\\列出从第几行开始的源代码
	list 函数名			\\列出某个函数的源代码
	next（或n）			\\执行下一行语句
	print（或p）			\\打印表达式的值，通过表达式可以修改变量的值或者调用函数
	quit（或q）			\\退出gdb调试环境
	set var				\\修改变量的值
	start				\\开始执行程序，停在main函数第一行语句前面等待命令
	step（或s）			\\执行下一行语句，如果有函数调用则进入到函数中
	
### 2 source Insight常用快捷键总结

	Ctrl+= :Jump to definition
	Alt+/ :Look up reference
	F3 : search backward
	F4 : search forward
	F5: go to Line
	F7 :Look up symbols
	F8 :Look up local symbols
	F9 :Ident left
	F10 :Ident right
	Alt+, :Jump backword
	Alt+. : Jump forward
	Shift+F3 : search the word under cusor backward
	Shift+F4 : search the word under cusor forward
	F12 : incremental search
	Shift+Ctrl+f: search in project
	shift+F8 : hilight word

### 3 tcpdump 抓TCP三次握手相关数据包

	// 捕获syn数据包
	tcpdump -i <interface> "tcp[tcpflags] & (tcp-syn) != 0" 
	
	// 捕获ack数据包
	tcpdump -i <interface> "tcp[tcpflags] & (tcp-ack) != 0"

	// 捕获fin数据包
	tcpdump -i <interface> "tcp[tcpflags] & (tcp-fin) != 0"

	// 捕获TCP SYN或ACK包
	tcpdump -r <interface> "tcp[tcpflags] & (tcp-syn|tcp-ack) != 0"

	// 捕获RST包
	tcpdump -r <interface> "tcp[tcpflags] & (tcp-rst) != 0"

### 4 路由跟踪及链路状态监测

	mtr hostname or IP

### 5 常用脚本参数
	
echo 输出不换行：
	
	echo -n "hello world!!"

set 命令用法：

	[root@bogon xuc]# set $(/sbin/runlevel)
	[root@bogon xuc]# runlevel=$2
	[root@bogon xuc]# previous=$1
	[root@bogon xuc]# echo $runlevel $previous
	5 N
	[root@bogon xuc]# echo $(/sbin/runlevel)
	N 5

### 6 linux 开机自启动或自执行

/etc/rc.local文件中配置linux系统开机自动执行脚本：
	
	#!/bin/sh
	#
	# This script will be executed *after* all the other init scripts.
	# You can put your own initialization stuff in here if you don't
	# want to do the full Sys V style init stuff.

	touch /var/lock/subsys/local

	sh /usr/local/bin/record_system_boot
	
chkconfig 命令可以查看、修改、配置开机自启动服务：

	[root@bogon xuc]# chkconfig --list httpd
	httpd           0:off   1:off   2:off   3:off   4:off   5:off   6:off
	[root@bogon xuc]# chkconfig --level 23456 httpd  on
	[root@bogon xuc]# chkconfig --list httpd
	httpd           0:off   1:off   2:on    3:on    4:on    5:on    6:on

这样 系统启动级别为23456时会自动启动httpd 服务, 服务可执行文件存在于：
	
	 /etc/rc.d/init.d/

关于系统当前启动级别：

	[root@bogon xuc]# runlevel
	N 5

修改系统启动级别，/etc/inittab:
	
	\# Default runlevel. The runlevels used are:
	\#   0 - halt (Do NOT set initdefault to this)
	\#   1 - Single user mode
	\#   2 - Multiuser, without NFS (The same as 3, if you do not have networking)
	\#   3 - Full multiuser mode
	\#   4 - unused
	\#   5 - X11
	\#   6 - reboot (Do NOT set initdefault to this)
	\#

	id:5:initdefault:


### 7 linux history 命令显示时间

	HISTTIMEFORMAT="%Y-%m-%d %H:%M:%S "
	

### 8 yum epel源地址

	http://dl.fedoraproject.org/pub/epel/beta/7/x86_64/epel-release-7-0.2.noarch.rpm
	
	http://dl.fedoraproject.org/pub/epel/
	

### 9 centos7 相关配置文件及命令

grub配置文件（静态配置文件，用于生成启动grub2文件）：

	/etc/default/grub

grub2文件（软连接，实际上连接到/boot/grub2/grub.cfg）：

	/etc/grub2.cfg

重新生成grub2文件：

	grub2-mkconfig -o /boot/grub2/grub.cfg

### 10 tar压缩文件命令

	tar -zcvf test.tar.gz ./test

压缩本地目录下的test文件，并生成压缩文件test.tar.gz。


### 11 centos发行版rpm包源码包

	http://vault.centos.org/6.6/updates/Source/SPackages/

	
### 12 squid 配置403响应

	acl rsp_403_domains reqdomain dlsw.baidu.com
	http_access deny rsp_403_domains

注意squid访问控制规则在配置文件中的先后顺序决定了规则生效的优先级，如果一个HTTP请求已经被前面的规则匹配，则会按照前面的策略实现访问控制，后面的规则则不会生效。

### 13 squid 未命中资源先响应部分请求头

	acl rsp_5bytes_domains url_regex ^http:.*
	reply_5bytes_in_advance_access allow rsp_5bytes_domains

### 14 squid清除已经缓存的内容
	/usr/local/squid/bin/squidclient -p 80 -m PURGE "url"
	
### 15 squid配置HTTP优化相关

	mem_hot_onoff off  //on的话为首次命中不缓存，为提高命中率该配置应该为off
	
	maximum_original_buffer_size 300 KB//对响应的内容针对某些关键字进行提花
	read_ahead_gap 300 KB
	keyword_replace_for_HTTP10_onoff on
	
	page_replace_content_fulltext on
	keyword_replace_access allow all
	keyword_replace_log /usr/local/squid/var/logs/keyword_replace.log
	page_rewrite_content_type text/plain text/xml text/html application/json text/javascript xml
	keyword_list https
	replace_string_list http
	
	redirect_forward 301 302 //分频配置，源站响应302不直接响应给用户而是再次请求得到200响应再返回给用户。

### 16 frigate相关配置
	
将来自指定IP地址的HTTP流量引导到HA\/CS，其他流量走本地回源（针对探针优化），frigate.conf 配置：
	
	"source_groups":[
                "sg_tuning 172.31.253.13/32"
                // "<label> [ip/mask ip/mask ip/mask]>"
     ],
	
	"routing":[
        //<BY=IP> <filter=destination_group/source_group> <TO=PEER/SOURCE/RESET> <IGNORE:COMPASS;PAYLOAD>
        "IP sg_tuning source",
	],

module_http.conf 配置：

	"nextproxies":[
                        // "<label> <balance> <peers=[ip:port ip:port ip:port]>"
                        "npTz HASH 192.168.20.45:8101",
                ],
	
	"nextproxy_clusters":[
                        // "<label> <primary groups> <standby groups>"
                        "np_clusterTz npTz:1",
	],

	"source_groups":[
                "sg_tz 120.198.57.148/32"
                ],

	"routing":[
                //liantong
	"IP sg_tz NEXTPROXY np_clusterTz",
	],
	




	
		

Have a fun！！！