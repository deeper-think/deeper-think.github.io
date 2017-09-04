---
layout: post
title: 线程局部存储(Thread-local storage)
category: 编程调试技巧
tags: [编程技巧]
keywords: 编程技巧
description: 
---

> 用正确的工具，做正确的事情

## __thread关键字的功能

摘自[gcc 官方解释](https://gcc.gnu.org/onlinedocs/gcc-3.4.1/gcc/Thread-Local.html)的一段：

> Thread-local storage (TLS) is a mechanism by which variables are allocated such that there is one instance of the variable per extant thread. The run-time model GCC uses to implement this originates in the IA-64 processor-specific ABI, but has since been migrated to other processors as well. It requires significant support from the linker (ld), dynamic linker (ld.so), and system libraries (libc.so and libpthread.so), so it is not available everywhere.

## 一个验证的小程序

	#include <stdlib.h>  
	#include <stdio.h>  
	#include <pthread.h>  
	
	static int var1= 15; 
	static __thread int var2 = 15; 
	
	static void* worker1(void* arg);  
	static void* worker2(void* arg);  
	
	int main(){  
    	pthread_t pid1,pid2;  
	
    	static __thread  int temp=10;//修饰函数内的static变量  
	
    	pthread_create(&pid1,NULL,worker1,NULL);  
    	pthread_create(&pid2,NULL,worker2,NULL);  
    	pthread_join(pid1,NULL);  
    	pthread_join(pid2,NULL);  
	
    	printf("main var1 addr :%p\n", &var1);
    	printf("main var2 addr :%p\n", &var2);
    	printf("temp : %d\n", temp);
    	return 0;  
	}  	
	
	static void* worker1(void* arg){  
    	printf("worker1 var1 :%d\n", ++var1);
    	printf("worker1 var1 addr :%p\n", &var1);
    	printf("worker1 var2 :%d\n", ++var2);
    	printf("worker1 var2 addr :%p\n", &var2);
	}  
	
	static void* worker2(void* arg){  
    	sleep(1);  
    	printf("worker2 var1 :%d\n", var1); 
    	printf("worker2 var1 addr :%p\n", &var1);
		
    	printf("worker2 var2 :%d\n", var2);
    	printf("worker2 var2 addr :%p\n", &var2);
	}

输出：

	worker1 var1 :16
	worker1 var1 addr :0x60104c
	worker1 var2 :16
	worker1 var2 addr :0x7f24df1f06f8
	worker2 var1 :16
	worker2 var1 addr :0x60104c
	worker2 var2 :15
	worker2 var2 addr :0x7f24de9ef6f8
	main var1 addr :0x60104c
	main var2 addr :0x7f24df9d1738

从输出可以看到var1作为全局变量被多个线程共享，而var2则在不同的线程都有单独一个实例。

玩的开心 !!!
