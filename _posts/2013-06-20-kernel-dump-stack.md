---
layout: post
title: 内核调试之dump_stack
category: 编程调试技巧
tags: [kernel, debug]
keywords: kernel, debug
description: 
---

> 用正确的工具，做正确的事情

内核功能实现中大量使用了回调函数，回调函数的使用增加了程序实现的灵活性和可扩展性，但同时也增加了理解代码的难度。为了更好的理解代码的功能实现方式，在程序执行中跟踪代码的执行流程和函数的调用关系是非常有帮助的一种方式。

用户态程序有各种debug方式，如c实现的用户态程序，可以非常方便的通过gdb在调试跟踪代码的执行及函数调用关系，内核及内核模块的调试工具较少，比较常用的方式就是增加日志打印，除此之外还一种比较有效的方式利用 dump_stack函数打印内核函数调用栈。

下面是最简单的hello world内核模块的实现及Makefile，已经在hello world内核模块实现代码中增加了函数调用及dump_stack的使用：

	#include <linux/module.h>//与module相关的信息
	
	#include <linux/kernel.h>
	#include <linux/init.h>      //与init相关的函数
	
	void func_test1(void);
	void func_test2(void);
	
	static int __init hellokernel_init(void)
	{
	        printk(KERN_INFO "Hello kernel!\n");
    	    func_test1();
    	    return 0;
	}
	
	static void __exit hellokernel_exit(void)
	{
    	    printk(KERN_INFO "Exit kernel!\n");
	}
	
	void func_test1(void)
	{
    	printk(KERN_INFO "Enter func_test1!\n");
    	func_test2();
	}
	
	void func_test2(void)
	{
    	printk(KERN_INFO "Enter func_test2!\n");
    	dump_stack();
	}

Makefile文件：
	
	obj-m := helloworld.o
	
	PWD       := $(shell pwd)

	all:
	        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
	
	clean:
	        rm -rf *.o *~ core .*.cmd *.mod.c ./tmp_version

将编译出的内核模块插入内核，然后dmesg可以看到相关的dump_stack信息：
	
	[root@Centos7-1 hello_world_driver]# dmesg
	[253880.127057] Hello kernel!
	[253880.127063] Enter func_test1!
	[253880.127064] Enter func_test2!
	[253880.127069] CPU: 19 PID: 216989 Comm: insmod Tainted: G           OE  ------------   3.10.0-327.3.1.el7.x86_64 #1
	[253880.127072] Hardware name: Intel Corporation S2600JF/S2600JF, BIOS SE5C600.86B.02.04.0003.102320141138 10/23/2014
	[253880.127076]  ffffffff81951020 00000000d5cd27ec ffff8807e8e7bd38 ffffffff8163516c
	[253880.127083]  ffff8807e8e7bd48 ffffffffa058202a ffff8807e8e7bd58 ffffffffa0033017
	[253880.127088]  ffff8807e8e7bd88 ffffffff810020e8 ffffffffa0584018 ffffffffa0584050
	[253880.127093] Call Trace:
	[253880.127106]  [<ffffffff8163516c>] dump_stack+0x19/0x1b
	[253880.127111]  [<ffffffffa058202a>] func_test1+0x2a/0x30 [helloworld]
	[253880.127116]  [<ffffffffa0033017>] hellokernel_init+0x17/0x1000 [helloworld]
	[253880.127123]  [<ffffffff810020e8>] do_one_initcall+0xb8/0x230
	[253880.127131]  [<ffffffff810ed4ae>] load_module+0x134e/0x1b50
	[253880.127139]  [<ffffffff81316810>] ? ddebug_proc_write+0xf0/0xf0
	[253880.127157]  [<ffffffff810e9743>] ? copy_module_from_fd.isra.42+0x53/0x150
	[253880.127162]  [<ffffffff810ede66>] SyS_finit_module+0xa6/0xd0
	[253880.127170]  [<ffffffff816458c9>] system_call_fastpath+0x16/0x1b




玩的开心 !!!
