---
layout: post
title: 关于dereferencing pointer to incomplete type及typedef的理解
category: 编程调试技巧
tags: [编译错误]
keywords: 编译错误
description: 
---

> 用正确的工具，做正确的事情

## 关于dereferencing pointer to incomplete type错误

最近走读分析qemu 后端block io实现代码，想在代码中添加打印调试信息，如下（block.c文件bdrv\_open\_common函数中添加1002,1003行，打印opts队里中所有元素的name和str）：

	981 static int bdrv_open_common(BlockDriverState *bs, BdrvChild *file,
	982                             QDict *options, Error **errp)
	983 {
	984     printf("Enter in bdrv_open_common:\n");
	985     int ret, open_flags;
	986     const char *filename;
	987     const char *driver_name = NULL;
	988     const char *node_name = NULL;
	989     const char *discard;
	990     const char *detect_zeroes;
	991     QemuOpts *opts;
	992     QemuOpt *opt = NULL;
	993     BlockDriver *drv;
	994     Error *local_err = NULL;
	995
	996     assert(bs->file == NULL);
	997     assert(options != NULL && bs->options != options);	
	998
	999     opts = qemu_opts_create(&bdrv_runtime_opts, NULL, 0, &error_abort);
	1000     qemu_opts_absorb_qdict(opts, options, &local_err);
	1001
	1002     QTAILQ_FOREACH(opt, (&(opts->head)), next)
	1003         printf("name:%s, value_str:%s\n",opt->name,opt->str);


重新编译qemu源码时报“dereferencing pointer to incomplete type”相关错误，具体报错信息如下：

	In file included from /home/qemu-2.8.0/include/qemu/option.h:29:0,
                 from /home/qemu-2.8.0/include/qemu-common.h:19,
                 from ./trace/generated-tracers.h:6,
                 from /home/qemu-2.8.0/include/trace.h:4,
                 from block.c:25:
	block.c: In function ‘bdrv_open_common’:
	block.c:1002:32: error: dereferencing pointer to incomplete type
     	QTAILQ_FOREACH(opt, (&(opts->head)), next)
	                                ^
	/home/qemu-2.8.0/include/qemu/queue.h:414:24: note: in definition of macro ‘QTAILQ_FOREACH’
         for ((var) = ((head)->tqh_first);                               \
                        ^
	/home/qemu-2.8.0/include/qemu/queue.h:416:31: error: dereferencing pointer to incomplete type
	                 (var) = ((var)->field.tqe_next))
                               ^
	block.c:1002:5: note: in expansion of macro ‘QTAILQ_FOREACH’
    	 QTAILQ_FOREACH(opt, (&(opts->head)), next)
    	 ^
	block.c:1003:45: error: dereferencing pointer to incomplete type
    	     printf("name:%s, value_str:%s\n",opt->name,opt->str);
	                                             ^
	block.c:1003:55: error: dereferencing pointer to incomplete type
    	     printf("name:%s, value_str:%s\n",opt->name,opt->str);
    	                                                   ^
	make: *** [block.o] Error 1

从网上搜索到的信息，大多数解释报错的原因：

> 你的指针指向了一个不完全的类型，这个类型一般是结构体或者联合体，也就是说你只给出了类型的声明，但是没有给出类型的定义，如果你想通过该类型的指针直接访问该类型的成员时就会报这个错误。解决的办法，只要找到出错对应的变量的结构体或联合的定义的头文件，并把这些头文件包含进来即可解决此问题。

因此对于qemu编译报错，只要在block.c文件中include 定义了QemuOpt以及QemuOpts结构体的文件即可：

	#include "include/qemu/option_int.h"
	

## 关于typedef的理解

对于c语言 typedef 关键字的作用，

> 大一时候的C语言老师讲的typedef 可以用来声明定义新的类型，这样你写程序要定义结构体变量的时候，就可以少写一个struct 了。

在还没有接触面向对象编程以及对于程序的“封装性”和模块“独立性”没有任何概念的年代，我对上面老师讲的typedef的作用深信不疑，哈哈哈。当然上面老师讲的，确实是typedef能够实现的一个作用，但是C语言对于typedef的使用不仅限于此。

typedef可以用来封装结构体类型成一个新的变量，在C语言中实现面向对象编程的一些思想。下面C实现的一个小的工程，该工程包括3个源文件：structA.c、structA.h、main.c， 具体实现如下。

structA.c的实现：

	#include <stdlib.h>
	#include <stdio.h>
	#include "structA.h"
	
	struct structA {
	    int a;
    	int b;
	};
	
	structA * create_structA(void)
	{
    	structA *ret = malloc(sizeof(structA));
    	return ret;
	}
	
	void print_structA(structA *s)
	{
    	printf("a=%d, b=%d\n", s->a, s->b);
	}
	
	void set_structA(structA *s, int a, int b)
	{
    	s->a = a;
    	s->b = b;
	}


structA.c 定义了一个结构体对象以及该对象上的一些方法。

structA.h 的实现如下：

	typedef struct structA structA;
	
	structA * create_structA(void);
	void print_structA(structA *s);
	void set_structA(structA *s, int a, int b);

structA.h对structA.c中定义的结构体对象进行了封装，封装成一个新的类型，并声明了该类型相关的可用方法。

main.c 的实现如下：
	
	#include <stdlib.h>
	#include <stdio.h>
	
	#include "structA.h"
	
	int main(void) {
    	structA *s1 = NULL;
    	s1 = create_structA();
    	set_structA(s1, 10, 100);
    	print_structA(s1);
	}

main.c中使用structA.h定义的类型来定义变量，并且使用该类型的方法对该变量执行相关的操作，编译执行该工程，如下：

	-bash-4.2# gcc main.c structA.c
	-bash-4.2# ./a.out
	a=10, b=100

注意不可以在main.c中直接访问 structA的成员，因为对于main来说 structA的实现是不可见的，我们修改main函数如下：

	int main(void) {
    	structA *s1 = NULL;
    	s1 = create_structA();
    	set_structA(s1, 10, 100);
    	s1 -> a = 50; //增加一行，访问structA的成员。
    	print_structA(s1);
	}

然后编译报错：

	-bash-4.2# gcc main.c structA.c
	main.c: In function ‘main’:
	main.c:10:8: error: dereferencing pointer to incomplete type
    	s1 -> a = 50;
        	^

该报错与上面qemu编译报错是一样的，解决的办法可以将struct structA结构的定义放到structA.h里面，但是这样就破坏了structA类型的封装性，最好的办法是：

> 在structA.h中声明新的接口，在structA.c里面实现该接口，在main里面通过接口来执行上面对structA类型变量的操作。同样对于上面qemu编译报错，这种方案也是更好的一种办法。


玩的开心 !!!
