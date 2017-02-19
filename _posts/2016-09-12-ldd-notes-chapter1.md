---
layout: post
title:读书笔记-linux设备驱动程序
category: 个人笔记
tags: [内核、驱动程序]
keywords:内核、驱动程序
description: 
---

> 用正确的工具，做正确的事情

## 02章：构造和运行模块

### 重要知识点记录和理解

#### 关于驱动版本依赖

在缺少modversions的情况下，我们的模块代码必须针对要链接的每个版本的内核重新编译。insmod装载内核过程，驱动与当前内核存在任何不兼容的问题，都会导致装载失败并输出： 

	Error inserting： '':-1 Invalid module format。

#### 关于内核符号表

insmod使用公共内核符号表来解析模块中使用但未定义的符号。公共内核符号表中包含了所有的全局内核函数和变量的地址，公共内核符号表是实现模块化驱动的基础。驱动程序也可以导出自己的全局项供其他模块使用：
	
	EXPORT_SYMBOL(name);
	EXPORT_SYMBOL_GPL(name);//导出的模块只能被GPL许可证下的模块使用

####　模块初始化

用于指定模块的初始化和清楚函数的宏：
	
	__init
	——initdata
	__exit
	__exitdata

使用上述宏声明的函数和变量只用于模块初始化阶段及模块清楚阶段，在模块其他的地方使用这些函数和变量会导出错误。因为标记为初始化的项目会在初始化结束后丢弃，而退出项目在内核未被配置为可卸载模块的情况下被丢弃。内核通过将相应的目标对象放置在可执行文件的特殊ELF段中而让这些标记起作用。

#### 关于模块参数

模块加载是可以传递参数给驱动模块:
	
	#include <linux/moduleparam.h> //使用的内核头文件
	module_param(whom, charp, S_IRUGO);//支持驱动加载时传递的参数必须使用该宏来声明，第一个参数为变量，第二个为变量类型，第三个参数为访问许可掩码
	
	insmod hello.ko whom="xiaoming" //加载驱动时传递参数给驱动


### 试验测试代码

模块代码：

	#include <linux/init.h>
	#include <linux/module.h>
	#include <linux/moduleparam.h>
	
	static char *whom="world";
	
	module_param(whom, charp, S_IRUGO);
	
	MODULE_LICENSE("Dual BSD/GPL");
	
	static int __init hello_init(void)
	{
        	printk(KERN_ALERT "Hello, %s\n", whom);
        	return 0;
	}
	
	static void __exit hello_exit(void)
	{
    	    printk(KERN_ALERT "Goodbye, curel %s\n", whom);
	}
	
	module_init(hello_init);
	module_exit(hello_exit);
	
	MODULE_AUTHOR("Tony");
	MODULE_DESCRIPTION("Hello world code for expriment,just for fun!!");
	MODULE_VERSION("1.0.0");
	
编译Makefile：
	
	ifneq ($(KERNELRELEASE),)
    		obj-m := hello.o
	else
        	KERNEL_DIR ?= /lib/modules/$(shell uname -r)/build
        	PWD := $(shell pwd)

	default:
        	make -C $(KERNEL_DIR) M=$(PWD) modules
		
	endif
	
	clean:
    	    rm -rf *.o *~ core .depend .*.cmd *.ko *.mod.c .tmp_versions *.order *.symvers

make -C $(KERNEL_DIR) M=$(PWD) modules： -C 选项指定的位置即内核源代码目录，其中保存有内核的顶层makefile文件。 M=选项让该makefile 在构造modules目标执勤啊返回到模块源代码目录。



玩的开心 !!!
