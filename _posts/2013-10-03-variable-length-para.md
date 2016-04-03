---
layout: post
title: c/c++中的变长参数:va_list的用法
category: 编程调试技巧
tags: [linux c, 变长参数]
keywords: linux c, 变长参数
description: 
---

> 用正确的工具，做正确的事情

分析qemu代码中很多函数出现va_list、va_start等的使用，以前只知道这个和变长参数的函数实现有关，因此在网上搜集了一下资料，相关知识点总结如下。变长参数的函数，比如c语言中经常使用的标准输出函数：

	int printf(char * format, ... );

第一个参数为格式化字符串，第二个参数就是变长参数声明，对于变长参数需要记住以下两点：

> 1.变长参数的定义和声明，用...代替参数类型。
  2.变长参数只能放在参数列表的最末尾。

使用变长参数编写计算一组数的平均值的函数 compute_mean:

	int compute_mean(int n, ...) {
	    int i, t, ret = 0;
	    va_list list; /* 定义变量列表指针 */
	
	    va_start(list, n); /* 绑定变量列表指针到变量列表 */
	    for(i = 0; i < n; i++) {
	        t = va_arg(list, int);/* 从变量列表中取出一个值 */
	        ret = ret + t;
	    }
	    va_end(list);/* 释放list指向的内存空间 */
	
	    return ret/n;
	}


含有变长参数的函数，函数处理变长参数的方法是：

> 1.va_list用于定义指向变量列表的指针list。
  2.va_start(list, n)将list绑定到变量列表。
  3.va_arg(list, int)从变量列表中去一个变量。
  4.va_end(list)释放变量列表空间。

另外va_list的使用有以下两点值得注意：

> 1.va_start一定要指定变量列表的长度。
  2.变量列表中的变量不一定是同一类型。


玩的开心 !!!
