---
layout: post
title: NPTL线程私有数据（Thread-Specific-Data）
category: 编程调试技巧
tags: [编程技巧]
keywords: 编程技巧
description: 
---

> 用正确的工具，做正确的事情

最近分析阅读qemu代码，qemu实现中使用了Thread-Specific-Data，简称TSD。多线程编程时，当我们需要创建每个线程私有的全局或者静态变量时，其中一种办法是在变量前面加__thread关键字，另外一种办法就是创建线程私有数据即TSD。相对来说第一种办法跟简单明了，第二种办法不太好理解，并且第二种办法除了用来创建线程私有的全局或者静态变量，可能更多的被用来实现线程退出时线程资源的处理和释放，下面总结TSD相关的数据结构和函数以及大致的实现方法。

## TSD数据结构

	static pthread_key_t exit_key;

用户通过关键字pthread\_key\_t来"创建"一个pthread\_key类型的对象 exit\_key， glibc中 pthread\_key\_t类型的实现：

	/* Keys for thread-specific data */
	typedef unsigned int pthread_key_t;

因此exit_key实际上只是一个普通的"静态无符号整形"变量， 静态变量存放在数据段，NPTL中各个线程共享数据段，因此exit\_key本身不是线程私有的。

## TSD相关功能函数

### pthread\_key\_create()

	#include <pthread.h>	
	int pthread_key_create(pthread_key_t *key, void (*destructor)(void*)); 

摘自[社区](https://linux.die.net/man/3/pthread_key_create) 的函数说明：

> The pthread\_key\_create() function shall create a thread-specific data key visible to all threads in the process.Although the same key value may be used by different threads, the values bound to the key by pthread\_setspecific() are maintained on a per-thread basis and persist for the life of the calling thread.Upon key creation, the value NULL shall be associated with the new key in all active threads. Upon thread creation, the value NULL shall be associated with all defined keys in the new thread.An optional destructor function may be associated with each key value. At thread exit, if a key value has a non-NULL destructor pointer, and the thread has a non-NULL value associated with that key, the value of the key is set to NULL, and then the function pointed to is called with the previously associated value as its sole argument.
> 
> If successful, the pthread\_key\_create() function shall store the newly created key value at *key and shall return zero. Otherwise, an error number shall be returned to indicate the error. 

glibc中该函数的具体实现如下：

	//文件：nptl/vars.c
	41 /* Table of the key information.  */
	42 struct pthread_key_struct __pthread_keys[PTHREAD_KEYS_MAX]
	43   __attribute__ ((nocommon));
	44 hidden_data_def (__pthread_keys)

	//文件：nptl/pthread_key_create.c
	24 int
	25 __pthread_key_create (key, destr)
	26      pthread_key_t *key;
	27      void (*destr) (void *);
	28 {
	29   /* Find a slot in __pthread_kyes which is unused.  */
	30   for (size_t cnt = 0; cnt < PTHREAD_KEYS_MAX; ++cnt)
	31     {
	32       uintptr_t seq = __pthread_keys[cnt].seq;
	33 
	34       if (KEY_UNUSED (seq) && KEY_USABLE (seq)
	35       /* We found an unused slot.  Try to allocate it.  */
	36       && ! atomic_compare_and_exchange_bool_acq (&__pthread_keys[cnt].seq,
	37                              seq + 1, seq))
	38     {
	39       /* Remember the destructor.  */
	40       __pthread_keys[cnt].destr = destr;
	41 
	42       /* Return the key to the caller.  */
	43       *key = cnt;
	44 
	45       /* The call succeeded.  */
	46       return 0;
	47     }
	48     }
	49 
	50   return EAGAIN;
	51 }

从实现的角度理解pthread\_key\_create()，即遍历全局对象数组\_\_pthread\_keys以找到其中没有被使用的一项，然后将该数组项的下标cnt保存到*key，将destr函数保存到__pthread_keys[cnt].destr。全局对象数组\_\_pthread\_keys也是线程共享的。

### pthread\_setspecific（）

	#include <pthread.h>
	int pthread_setspecific(pthread_key_t key, const void *value); 
	
[社区](https://linux.die.net/man/3/pthread_setspecific)的函数说明：	

> The pthread\_setspecific() function shall associate a thread-specific value with a key obtained via a previous call to pthread\_key\_create(). Different threads may bind different values to the same key. These values are typically pointers to blocks of dynamically allocated memory that have been reserved for use by the calling thread.The effect of calling pthread\_getspecific() or pthread\_setspecific() with a key value not obtained from pthread\_key\_create() or after key has been deleted with pthread\_key\_delete() is undefined.

>If successful, the pthread\_setspecific() function shall return zero; otherwise, an error number shall be returned to indicate the error.

该函数glibc中具体的实现如下：

	24 int
	25 __pthread_setspecific (key, value)
	26      pthread_key_t key;
	27      const void *value;
	28 {
	29   struct pthread *self;
	30   unsigned int idx1st;
	31   unsigned int idx2nd;
	32   struct pthread_key_data *level2;
	33   unsigned int seq;
	34 
	35   self = THREAD_SELF;
	36 
	37   /* Special case access to the first 2nd-level block.  This is the
	38      usual case.  */
	39   if (__builtin_expect (key < PTHREAD_KEY_2NDLEVEL_SIZE, 1))
	40     {
	41       /* Verify the key is sane.  */
	42       if (KEY_UNUSED ((seq = __pthread_keys[key].seq)))
	43     /* Not valid.  */
	44     return EINVAL;
	45 
	46       level2 = &self->specific_1stblock[key];
	.....
	87   /* Store the data and the sequence number so that we can recognize
	88      stale data.  */
	89   level2->seq = seq;
	90   level2->data = (void *) value;
	91 
	92   return 0;
	93 }

从实现的角度看pthread\_setspecific()函数，46行根据key从specific\_1stblock数组中找到key对应的pthread\_key\_data对象 level2， 然后90行将value的值保存到level2->data。specific\_1stblock是结构对象pthread的成员，pthread的具体实现：
	
	//源文件：nptl/descr.h
	123 /* Thread descriptor data structure.  */
	124 struct pthread
	125 {
	126   union
	127   {
	128 #if !TLS_DTV_AT_TP
	129     /* This overlaps the TCB as used for TLS without threads (see tls.h).  */
	130     tcbhead_t header;
	131 #else
	.....
	294   /* We allocate one block of references here.  This should be enough
	295      to avoid allocating any memory dynamically for most applications.  */
	296   struct pthread_key_data
	297   {
	298     /* Sequence number.  We use uintptr_t to not require padding on
	299        32- and 64-bit machines.  On 64-bit machines it helps to avoid
	300        wrapping, too.  */
	301     uintptr_t seq;
	302 
	303     /* Data pointer.  */
	304     void *data;
	305   } specific_1stblock[PTHREAD_KEY_2NDLEVEL_SIZE];
	306 
	307   /* Two-level array for the thread-specific data.  */
	308   struct pthread_key_data *specific[PTHREAD_KEY_1STLEVEL_SIZE];
	309 
	310   /* Flag which is set when specific data is set.  */
	311   bool specific_used;
	......
	388   (sizeof (struct pthread) - offsetof (struct pthread, end_padding))
	389 } __attribute ((aligned (TCB_ALIGNMENT)));

296~308行定义了跟TSD相关的对象，而pthread是在allocate\_stack（）函数中通过mmap的方式创建，并且pthread是线程私有的，进而利用此实现线程私有数据。

### pthread\_getspecific()

	#include <pthread.h>
	
	void *pthread_getspecific(pthread_key_t key);

[社区](https://linux.die.net/man/3/pthread\_setspecific)的函数说明：

>The pthread\_getspecific() function shall return the value currently bound to the specified key on behalf of the calling thread.The pthread\_getspecific() function shall return the thread-specific data value associated with the given key. If no thread-specific data value is associated with key, then the value NULL shall be returned.

pthread\_getspecific的具体实现以key作为下标从pthread对象的成员specific\_1stblock数组中得到pthread\_key\_data 对象指针，进入获取与key绑定在一起的data。


玩的开心~！

