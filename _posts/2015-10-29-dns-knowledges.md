---
layout: post
title: DNS相关技术总结
category: 系统及服务
tags: [DNS, bind]
keywords: DNS,bind
description:
---

> 用正确的工具，做正确的事情


## 1 域名系统

分布式系统

> 因特网规模很大，所以整个因特网只使用一个域名服务器是不可行的。因此，早在1983年因特网开始采用层次树状结构的命名方法，并使用分布式的域名系统DNS。并采用客户服务器方式。DNS使大多数名字都在本地解析(resolve)，仅有少量解析需要在因特网上通信，因此DNS系统的效率很高。由于DNS是分布式系统，即使单个计算机除了故障，也不会妨碍整个DNS系统的正常运行

域名结构

> 因特网在命名时采用的是层次树状结构的命名方法。任何一个连接在因特网上的主机或路由器，都有一个唯一的层次结构的名字，即域名(domain name)。这里，“域”(domain)是名字空间中一个可被管理的划分。

域名服务器

> 顶级域名服务器：负责管理在该顶级域名服务器注册的二级域名。权限域名服务器：负责一个“区”的域名服务器。本地域名服务器：本地服务器不属于下图的域名服务器的层次结构，但是它对域名系统非常重要。当一个主机发出DNS查询请求时，这个查询请求报文就发送给本地域名服务器。

域名解析

> 机向本地域名服务器的查询一般都是采用递归查询。本地域名服务器向根域名服务器的查询的迭代查询。

## 2 bind配置

bind主配置文件：

	options {
	    /*listen-on port 53 { 0.0.0.0; };*/
	    /*listen-on-v6 port 53 { ::1; };*/
	    directory   "/var/named";            //named相关文件的目录
	    dump-file   "/var/named/data/cache_dump.db";
	    statistics-file "/var/named/data/named_stats.txt";
	    memstatistics-file "/var/named/data/named_mem_stats.txt";
	    allow-query     { "any"; };         //服务网段
	    recursion yes;                      //是否允许递归查询
	
	    forwarders { 114.114.114.114;114.114.115.115; }; //forward 配置
	    forward only;
	
	    max-cache-ttl 60;                               //本地域名服务器最大缓存时间
	
	    dnssec-enable yes;
	    dnssec-validation yes;
	    dnssec-lookaside auto;
	
	    /* Path to ISC DLV key */
	    bindkeys-file "/etc/named.iscdlv.key";
	    managed-keys-directory "/var/named/dynamic";
	};
	
	/* 日志相关的配置 */
	logging {
	
    	channel default_debug {
    	    file "data/named.run";
    	    severity dynamic;
    	};
		
    	channel "query_log" {
    	    file "data/query.log" versions 10 size 50m;
    	    severity info;
    	    print-category yes;
    	    print-severity yes;
    	    print-time yes;
    	};
    	channel "warning_log" {
    	  	file "data/warning.log" versions 10 size 50m;
    	  	severity error;
    	  	print-category yes;
    	  	print-severity yes;
    	  	print-time yes;
    	};
		
		/* 将 queries分类的日志输出到上面定义的query_log channel */
    	category "queries" { 
    	    "query_log";
		};
		
	};

	/* zone 相关的配置 */
	zone "chinanetcenter.pub" IN {
	type master; //该域名服务器是主域名服务器，这个选项主要用在主备部署中
	file "chinanetcenter.pub.zone"; //解析域名cobb.com的zone文件内容，其路径由options中的directory指定
	allow-update { none; }; //定义了允许向主zone文件发送动态更新的匹配列表
	};

	zone "." IN {
    	type hint;
    	file "named.ca";
	};
	
	/* include 部分 */
	include "/etc/named.rfc1912.zones";
	include "/etc/named.root.key";

zone 模板配置文件：

	$TTL 86400
	$ORIGIN chinanetcenter.pub.
	@       IN  SOA ns1 root(	
	            2013031901      ;serial
    	        12h     ;refresh
        	    7200        ;retry
        	    604800      ;expire
        	    86400       ;mininum
        	    )
        	    NS  ns1.chinanetcenter.pub.
        	    MX  10  mail.chinanetcenter.pub.
	ns1     IN  A   192.168.111.134
	www     IN  A   192.168.111.134
	mail        IN  A   192.168.111.134
	ljx     IN  A   192.168.111.134
	ftp     IN  CNAME   ljx


## 参考

http://www.cnblogs.com/cobbliu/archive/2013/03/19/2970311.html


Have a fun！！！