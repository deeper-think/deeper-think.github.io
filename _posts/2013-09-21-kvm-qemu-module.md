---
layout: post
title: qemu 核心机制分析：module机制实现及分析
category: KVM虚拟化
tags: [KVM，qemu]
keywords: KVM，qemu
description: 
---

> 用正确的工具，做正确的事情

Qemu模块代码运行之前，首先需要完成一些初始化的操作，比如结构变量的创建以及结构变量注册到管理类型的全局队列等。Qemu各个模块代码都有专门实现用来完成模块初始化的函数，但是如何调用执行这些函数是一个问题？最简单粗暴的一种方式
，在Qemu main函数中直接调用各个模块的初始化函数，这样在Qemu main的开始部分会存在一大坨的函数调用，这样看起来非常的丑陋，程序设计是一门艺术，是艺术就要存在美感，呵哈，除了看起来比较丑陋之外还存在另外一个更大的问题，当每次Qemu中添加了新的模块代码时，都需要修改main函数以增加对模块初始化函数的调用，这有违程序设计的封装性和独立性。Qemu 利用gcc的构造函数（contructors）采用一种更加灵活优雅的方式解决的模块初始化的问题，核心思想：

>qemu main函数执行之前，各模块代码通过构造函数的执行将相关代码模块初始化函数的指针保存到链表数据结构，main函数中遍历一遍该链表结构，逐个调用各模块的初始化函数，几个完成各个模块的初始化工作。

下面详细分析和总结Qemu module机制的实现。

## 关于gcc构造函数和析构函数

下面是在gcc下 定义构造函数和析构函数格式：

	static void __attribute__((constructor)) start(void)
	static void __attribute__((destructor)) end(void)

start将在main函数被调用之前执行，end将在main函数退出后被调用执行。

## Qemu module机制分析和总结

### Qemu module机制的实现分析

Qemu module实现相关的两个源文件：
	
	include/qemu/module.h
	util/module.c

其中module.c文件定义了重要的数据结构和结构变量，如下：

	24 typedef struct ModuleEntry
	25 {
	26     void (*init)(void);
	27     QTAILQ_ENTRY(ModuleEntry) node;
	28     module_init_type type;
	29 } ModuleEntry;
	30
	31 typedef QTAILQ_HEAD(, ModuleEntry) ModuleTypeList;
	32
	33 static ModuleTypeList init_type_list[MODULE_INIT_MAX];
	34
	35 static ModuleTypeList dso_init_list;



1. 24~29行定义了ModuleEntry结构，该结构是对qemu模块初始化的抽象，函数指针成员init记录该模块初始化的函数地址，type成员表示该模块的类型，ModuleEntry结构变量通过一个队列结构来管理，node成员是维持队列结构的指针；
2. 31行定义了一个类型ModuleTypeList， 该类型可以用来定义尾队列；
3. 33行定义了一个全局静态尾队列数组init_type_list， 其中每一个数组元素为一个ModuleTypeList类型的元素，35行定义了一个单独的全局静态结构变量dso\_init\_list。

> 根据上面结构变量的定义可知，每个要进行初始化的模块都对应一个ModuleEntry类型的结构变量，一种类型的模块对应一个全局静态ModuleTypeList队列，ModuleEntry类型的结构变量会插入到ModuleTypeList类型的全局静态队列来管理和访问。

module.h文件通过以下宏定义以及函数声明，让其他qemu代码模块可以使用这种模块初始化机制，包括模块初始化函数的注册以及模块初始化函数的调用执行：

	34 /* This should not be used directly.  Use block_init etc. instead.  */
	35 #define module_init(function, type)                                         \
	36 static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
	37 {                                                                           \
	38     register_module_init(function, type);                                   \
	39 }
	40 #endif
	41
	42 typedef enum {
	43     MODULE_INIT_BLOCK,
	44     MODULE_INIT_OPTS,
	45     MODULE_INIT_QAPI,
	46     MODULE_INIT_QOM,
	47     MODULE_INIT_TRACE,
	48     MODULE_INIT_MAX
	49 } module_init_type;
	50
	51 #define block_init(function) module_init(function, MODULE_INIT_BLOCK)
	52 #define opts_init(function) module_init(function, MODULE_INIT_OPTS)
	53 #define qapi_init(function) module_init(function, MODULE_INIT_QAPI)
	54 #define type_init(function) module_init(function, MODULE_INIT_QOM)
	55 #define trace_init(function) module_init(function, MODULE_INIT_TRACE)
	56
	57 #define block_module_load_one(lib) module_load_one("block-", lib)
	58
	59 void register_module_init(void (*fn)(void), module_init_type type);
	60 void register_dso_module_init(void (*fn)(void), module_init_type type);
	61
	62 void module_call_init(module_init_type type);
	63 void module_load_one(const char *prefix, const char *lib_name);


1. 34~40行以及51~57行通过一系列宏定义了模块初始化注册的构造函数，模块注册构造函数最终通过调register\_module\_init来完成模块初始化函数的注册；
2. 41~50行定义了一个联合结构体类型，该联合类型包括了所有当前qemu 的模块类型；
3. 59行声明了真正完成模块初始化函数注册的函数，62行声明了实现调用模块初始化函数完成模块初始化的函数，module.c文件中实现了这两个函数。

### Qemu代码模块利用机制实现模块初始化

Raw-posix.c利用module机制实现main函数之前完成模块初始化函数的注册：

	block_init(bdrv_file_init);
	
注册模块初始化函数的调用堆栈：
	
	(gdb) bt
	#0  0x00007f14a1872504 in sleep () from /usr/lib64/libc.so.6
	#1  0x00007f14a3bd9ae8 in register_module_init (fn=0x7f14a3b10860 <bdrv_raw_init>, type=MODULE_INIT_BLOCK) at util/module.c:67
	#2  0x00007f14a3b10887 in do_qemu_init_bdrv_raw_init () at block/raw_bsd.c:490
	#3  0x00007f14a3f68c3d in __libc_csu_init ()
	#4  0x00007f14a17d5ac5 in __libc_start_main () from /usr/lib64/libc.so.6
	#5  0x00007f14a37329d9 in _start ()

调用模块初始化函数的调用堆栈：

	(gdb) bt
	#0  0x00007fac60379504 in sleep () from /usr/lib64/libc.so.6
	#1  0x00007fac626633e7 in bdrv_file_init () at block/raw-posix.c:2602
	#2  0x00007fac626e0bf0 in module_call_init (type=MODULE_INIT_BLOCK) at util/module.c:103
	#3  0x00007fac6260b570 in bdrv_init () at block.c:3234
	#4  0x00007fac6260b585 in bdrv_init_with_whitelist () at block.c:3240
	#5  0x00007fac623c6e58 in main (argc=12, argv=0x7fffede0ab48, envp=0x7fffede0abb0) at vl.c:3101	


玩的开心 !!!
