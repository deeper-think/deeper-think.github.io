---
layout: post
title: 读书笔记:linux设备驱动程序-字符设备驱动程序
category: 个人笔记
tags: [内核]
keywords: 驱动
description: 
---

> 用正确的工具，做正确的事情

## 数据结构及重要函数实现

### 相关数据结构

scull驱动程序涉及到的相关数据结构以及数据结构之间的关系如下：

![scull数据结构](http://7u2rbh.com1.z0.glb.clouddn.com/scull.struct.png)

说明：

1. struct scull\_dev、struct\_qset 为驱动定义的数据结构，struct file、struct file\_operations、struct inode、struct cdev 为内核定义的数据结构。
2. struct file结构中的指针成员private\_data，在file open 系统调用过程中被赋值为该文件关联的设备结构，对于scull驱动也即会被赋值为scull\_dev结构对象的地址。
3. struct cdev为字符设备在内核中的表示，现实中存在多种类型的字符设备，不用的字符设备有不同的特性和功能，但是从内核的角度统一抽象为cdev， 不同的字符设备不同的特性以及功能由设备驱动程序来定义和实现，实现的细节对于内核来说是透明的。

### 驱动程序代码分析

linux系统上字符设备以文件的形式提供给用户空间，应用程序通过对文件的操作还实现对设备的访问，因此字符设备驱动需要实现两类的函数，一类是驱动的初始化和卸载的函数，另一类即实现file\_operations结构定义的相关文件操作接口函数。

#### scull\_init\_module分析

scull\_init\_module函数主要完成设备驱动的初始化工作，主要干了三件事情：1）设备号的申请和注册，2）设备数据结构scull_dev的创建和初始化，3）字符设备cdev的初始化和注册，关键函数调用关系如下图：

![scull_init_module函数调用](http://7u2rbh.com1.z0.glb.clouddn.com/scull_init_module.png)

详细代码实现及分析如下：

	615 int scull_init_module(void)
	616 {
	617         int result, i;
	618         dev_t dev = 0;
	619
	620 /*
	621  * Get a range of minor numbers to work with, asking for a dynamic
	622  * major unless directed otherwise at load time.
	623  */
	624         if (scull_major) {
	625                 dev = MKDEV(scull_major, scull_minor);
	626                 result = register_chrdev_region(dev, scull_nr_devs, "scull");
	627         } else {
	628                 result = alloc_chrdev_region(&dev, scull_minor, scull_nr_devs,
	629                                 "scull");
	630                 scull_major = MAJOR(dev);
	631         }
	632         if (result < 0) {
	633                 printk(KERN_WARNING "scull: can't get major %d\n", scull_major);
	634                 return result;
	635         }
	636
	637         /*
	638          * allocate the devices -- we can't have them static, as the number
	639          * can be specified at load time
	640          */
	641         scull_devices = kmalloc(scull_nr_devs * sizeof(struct scull_dev), GFP_KERNEL);
	642         if (!scull_devices) {
	643                 result = -ENOMEM;
	644                 goto fail;  /* Make this more graceful */
	645         }
	646         memset(scull_devices, 0, scull_nr_devs * sizeof(struct scull_dev));
	647
	648         /* Initialize each device. */
	649         for (i = 0; i < scull_nr_devs; i++) {
	650                 scull_devices[i].quantum = scull_quantum;
	651                 scull_devices[i].qset = scull_qset;
	652                 init_MUTEX(&scull_devices[i].sem);
	653                 scull_setup_cdev(&scull_devices[i], i);
	654         }
	655
	
	656         /* At this point call the init function for any friend device */
	657         dev = MKDEV(scull_major, scull_minor + scull_nr_devs);
	658         dev += scull_p_init(dev);
	659         dev += scull_access_init(dev);
	660
	661 #ifdef SCULL_DEBUG /* only when debugging */
	662         scull_create_proc();
	663 #endif
	664
	665         return 0; /* succeed */
	666
	667   fail:
	668         scull_cleanup_module();
	669         return result;
	670 }

620-631申请并注册主次设备号，如果驱动加载时没有传入住设备号则动态从内核申请并分配,如果驱动加载时传入了主设备号参数则注册主设备号到内核。641-654 创建字符设备数据结构scull\_dev，分配内存并且通过memset初始化内存区域，然后设置scull\_dev数据结构的成员变量quantum、qset，之后通过调用scull\_setup\_cdev注册设备到内核。

	





#### scull驱动文件操作接口函数








玩的开心 !!!



