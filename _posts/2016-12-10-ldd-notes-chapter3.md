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

620-631申请并注册主次设备号，如果驱动加载时没有传入住设备号则动态从内核申请并分配,如果驱动加载时传入了主设备号参数则注册主设备号到内核。641-654 创建字符设备数据结构scull\_dev，分配内存并且通过memset初始化内存区域，然后设置scull\_dev数据结构的成员变量quantum、qset，之后通过调用scull\_setup\_cdev注册设备到内核。 scull_setup_cdev函数的实现如下：

	601 static void scull_setup_cdev(struct scull_dev *dev, int index)
	602 {
	603         int err, devno = MKDEV(scull_major, scull_minor + index);	
	604
	605         cdev_init(&dev->cdev, &scull_fops);
	606         dev->cdev.owner = THIS_MODULE;
	607         dev->cdev.ops = &scull_fops;
	608         err = cdev_add (&dev->cdev, devno, 1);
	609         /* Fail gracefully if need be */
	610         if (err)
	611                 printk(KERN_NOTICE "Error %d adding scull%d", err, index);
	612 }

605行调用内核函数cdev\_init初始化scull\_dev 成员变量 cdev， 在scull\_init\_module动态分配内存的形式创建了scull\_dev结构，但是其成员变量cdev 并没有被初始化，通过cdev\_init初始化cdev，将scull\_fops与 cdev 做了关联，因此607行代码其实是不需要的。608行调用cdev\_add将字符设备注册到内核，调用cdev\_add时传入了设备号devno参数，将cdev与设备号关联。

> cdev为字符设备在内核中的表示，cdev 比较重要的两个成员变量为file\_operations ops和dev\_t dev, file\_operations定义该字符设备的访问操作接口，并最终由驱动来实现。dev\_t保存具体的设备号，用户空间通过设备号来为设备创建设备文件的时候（创建inode），通过设备号将用户空间的设备文件（以及内核的inode）与内核的cdev对象关联起来。

#### 文件操作接口函数

	238 int scull_open(struct inode *inode, struct file *filp)
	239 {
	240         struct scull_dev *dev; /* device information */
	241
	242         dev = container_of(inode->i_cdev, struct scull_dev, cdev);
	243         filp->private_data = dev; /* for other methods */
	244
	245         /* now trim to 0 the length of the device if open was write-only */
	246         if ( (filp->f_flags & O_ACCMODE) == O_WRONLY) {
	247                 if (down_interruptible(&dev->sem))
	248                         return -ERESTARTSYS;
	249                 scull_trim(dev); /* ignore errors */
	250                 up(&dev->sem);
	251         }
	252         return 0;          /* success */
	253 }


scull\_open的传入参数是struct inode和struct file结构对象的指针，struct inode和struct file均为内核层创建管理以及使用的数据结构（猜测struct inode结构是应用程序mknode创建设备文件时内核创建的结构，struct file结构是应用程序打开设备文件时内核层在内存中创建的结构），242~243行 根据struct inode的成员变量i_cdev得到包含该变量的struct scull\_dev结构对象地址，并且将该结构对象地址赋值给struct file结构的 private\_data字段，对private\_data进行赋值的意义在于？之后应用层系统调用通过设备文件来访问和管理设备时只需要通过 内核的struct file结构就可以了。
	
> 通过filp->private_data将内核层抽象的struct file对象与具体的设备关联起来。

scull\_read的实现如下：

	/*
	289  * Data management: read and write
	290  */	
	291
	292 ssize_t scull_read(struct file *filp, char __user *buf, size_t count,
	293                 loff_t *f_pos)
	294 {
	295         struct scull_dev *dev = filp->private_data;
	296         struct scull_qset *dptr;        /* the first listitem */
	297         int quantum = dev->quantum, qset = dev->qset;
	298         int itemsize = quantum * qset; /* how many bytes in the listitem */
	299         int item, s_pos, q_pos, rest;
	300         ssize_t retval = 0;
	301
	302         if (down_interruptible(&dev->sem))
	303                 return -ERESTARTSYS;
	304         if (*f_pos >= dev->size)
	305                 goto out;
	306         if (*f_pos + count > dev->size)
	307                 count = dev->size - *f_pos;
	308
	309         /* find listitem, qset index, and offset in the quantum */
	310         item = (long)*f_pos / itemsize;
	311         rest = (long)*f_pos % itemsize;
	312         s_pos = rest / quantum; q_pos = rest % quantum;
	313
	314         /* follow the list up to the right position (defined elsewhere) */
	315         dptr = scull_follow(dev, item);
	316
	317         if (dptr == NULL || !dptr->data || ! dptr->data[s_pos])
	318                 goto out; /* don't fill holes */
	319
	320         /* read only up to the end of this quantum */
	321         if (count > quantum - q_pos)
	322                 count = quantum - q_pos;
	323
	324         if (copy_to_user(buf, dptr->data[s_pos] + q_pos, count)) {
	325                 retval = -EFAULT;
	326                 goto out;
	327         }
	328         *f_pos += count;
	329         retval = count;
	330
	331   out:
	332         up(&dev->sem);
	333         return retval;
	334 }

295行从传入参数struct file结构指针的private\_data字段获取struct scull\_dev变量指针， private\_data是在 scull\_open函数中被赋值的。297~298行获取设备量子集的长度以及量子的长度，并根据量集的长度以及量子的长度计算出每个量子集可以存储的字节数也即 itemsize。struct scull\_dev和struct scull_qset结构的定义如下：

	/*
 	* Representation of scull quantum sets.
 	*/
	struct scull_qset {	
        void **data;
        struct scull_qset *next;
	};
	
	struct scull_dev {
        struct scull_qset *data;  /* Pointer to first quantum set */
        int quantum;              /* the current quantum size */
        int qset;                 /* the current array size */
        unsigned long size;       /* amount of data stored here */
        unsigned int access_key;  /* used by sculluid and scullpriv */
        struct semaphore sem;     /* mutual exclusion semaphore     */
        struct cdev cdev;         /* Char device structure              */
	};



1. 根据struct scull\_qset和struct scull\_dev的结构定义， scull\_dev的数据是一个由量子集scull\_qset组成的链表，量子集的数据是一个指针数组，数组中的每个指针指向一个量子，一个量子是一片连续的内存区域;
2. 304~307行，如果f\_pos超过了设备数据集合长度，则直接返回，如果f\_ops+cout超过了设备数据集合的长度，则读取的数据量设置为从当前位置到数据集合的末尾之间的长度；
3. 310到312行，310计算读起始位置落在那个量子集item，311行计算读起始位置在该量子集中的偏移rest，312在rest值的基础上计算读起始位置落在那个量子（s\_pos）中以及在该量子中的偏移（q\_pos）；
4. 315~318行通过函数scull\_follow将工作指针dptr定位到读起始位置所在的量子以及一些异常情况的处理；
5. 320~322行判断要读取的总长度是否会超过当前量子的尾部，如果超过的话则只读取到当前量子的末尾，324~327行调用copy_to_user 拷贝要读取的数据到buff中，328~329行根据本次实际读取的长度移动文件指针并反馈实际读取的长度；
6. scull\_read的实现是并不一定一次将数据全部读完，如果cout超过了当前量子的末尾，则只读当前量子中的数据，也即一次只读一个量子，读取cout中指定的数据量，由应用层（库函数）通过多次系统调用，多次调用scull\_read来实现。
	
scull\_write的实现如下：

	336 ssize_t scull_write(struct file *filp, const char __user *buf, size_t count,
	337                 loff_t *f_pos)
	338 {
	339         struct scull_dev *dev = filp->private_data;
	340         struct scull_qset *dptr;
	341         int quantum = dev->quantum, qset = dev->qset;
	342         int itemsize = quantum * qset;
	343         int item, s_pos, q_pos, rest;
	344         ssize_t retval = -ENOMEM; /* value used in "goto out" statements */
	345
	346         if (down_interruptible(&dev->sem))
	347                 return -ERESTARTSYS;
	348
	349         /* find listitem, qset index and offset in the quantum */
	350         item = (long)*f_pos / itemsize;
	351         rest = (long)*f_pos % itemsize;
	352         s_pos = rest / quantum; q_pos = rest % quantum;
	353
	354         /* follow the list up to the right position */
	355         dptr = scull_follow(dev, item);
	356         if (dptr == NULL)
	357                 goto out;
	358         if (!dptr->data) {
	359                 dptr->data = kmalloc(qset * sizeof(char *), GFP_KERNEL);
	360                 if (!dptr->data)
	361                         goto out;
	362                 memset(dptr->data, 0, qset * sizeof(char *));
	363         }
	364         if (!dptr->data[s_pos]) {
	365                 dptr->data[s_pos] = kmalloc(quantum, GFP_KERNEL);
	366                 if (!dptr->data[s_pos])
	367                         goto out;
	368         }
	369         /* write only up to the end of this quantum */
	370         if (count > quantum - q_pos)
	371                 count = quantum - q_pos;
	372
	373         if (copy_from_user(dptr->data[s_pos]+q_pos, buf, count)) {
	374                 retval = -EFAULT;
	375                 goto out;
	376         }
	377         *f_pos += count;
	378         retval = count;
	379
	380         /* update the size */
	381         if (dev->size < *f_pos)
	382                 dev->size = *f_pos;
	383
	384   out:
	385         up(&dev->sem);
	386         return retval;
	387 }




1. 338~357行同scull\_write类似，计算写起始位置所在的量子集，写起始位置所在的量子以及在量子中的偏移，并定位dptr指针到写起始位置所在的量子集；
2. 358~363行如果写起始位置所在的量子集，其内存还没有分配则申请分配内存；
3. 364~368行如果写起始位置所在的量子，其内存还没有分配则申请分配内存；
4. 369~378行写buff数据到相应的量子，但是一次写的数据量不超过当前量子的末尾， 更新f_pos并返回本次写的长度；
5. 380~382行根据当前f_pos的位置，更新当前设备文件的大小。


玩的开心 !!!



