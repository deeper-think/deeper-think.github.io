---
layout: post
title: qemu尾队列分析及应用
category: KVM虚拟化
tags: [KVM，qemu]
keywords: KVM，qemu
description: 
---

> 用正确的工具，做正确的事情

qemu中有很多采用“队列”结构来组织数据结构，比如QemuOpts等。qemu通过宏定义实现了多种结构的队列，实现的文件：
	
	include/qemu/queue.h

下面将详细分析qemu中尾队列的实现，同时通过一个demo程序来展示如何在自己的应用程序中直接使用其来实现自己的尾队列数据结构。

## Tail queue的实现分析

尾队列的结构如下，图1是队列为空时的结构，图2是包括三个元素的队列结构。





### tail queue的定义

下面部分为定义和创建Tail queue的宏。

	/*
 	* Tail queue definitions.
 	*/
	#define Q_TAILQ_HEAD(name, type, qual)                                  \
	struct name {                                                           \
        qual type *tqh_first;           /* first element */             \
        qual type *qual *tqh_last;      /* addr of last next element */ \
	}
	#define QTAILQ_HEAD(name, type)  Q_TAILQ_HEAD(name, struct type,)
	
	#define QTAILQ_HEAD_INITIALIZER(head)                                   \
	        { NULL, &(head).tqh_first }
	
	#define Q_TAILQ_ENTRY(type, qual)                                       \
	struct {                                                                \
        qual type *tqe_next;            /* next element */              \
        qual type *qual *tqe_prev;      /* address of previous next element */\
	}
	#define QTAILQ_ENTRY(type)       Q_TAILQ_ENTRY(struct type,)

宏QTAILQ\_HEAD(name, type)定义一个结构体类型，结构体类型的名称为name， 结构体类型包括两个指针成员，指针的类型为type，注意第二个指针为“指向指针的指针”，可以直接用该结构体类型来定义结构体变量，比如：
	
	QTAILQ\_HEAD(QemuOptHead, QemuOpt) head

对其宏展开为：

	struct QemuOptHead {                                                           
        QemuOpt *tqh_first;  
        QemuOpt **tqh_last;
	} head

宏QTAILQ\_HEAD\_INITIALIZER(head)定义初始化成员变量head的值，初始化的结果是head.tqh\_first赋值为null，head.tqh\_last 为&(head).tqh\_first 变量的地址，也即head.tqh\_last初始化时指向(head).tqh\_first。

QTAILQ\_ENTRY(type) 定义了一个结构体类型，该结构体类型包括两个成员变量，tqe\_next为指向队列下一个元素的指针，tqe\_prev为指向“前一个元素的tqe\_next”。


### tail queue的操作函数

	/*
	 * Tail queue functions.
 	*/
	#define QTAILQ_INIT(head) do {                                          \
	        (head)->tqh_first = NULL;                                       \
	        (head)->tqh_last = &(head)->tqh_first;                          \
	} while (/*CONSTCOND*/0)
	
宏QTAILQ\_INIT(head)用于初始化Tail queue的队列头。

	#define QTAILQ_INSERT_HEAD(head, elm, field) do {                       \
			/*
			 如果队列不为空，因为是在头部插入，因此需要调整队列头部后向指针以及队列当前第一个元素的前向指针指向，因为是在头部插入，因此不需要调整队列头部前向指针的指向。需要修改需要对当前队列的第一个元素的“前向指针”进行赋值，前向指针应该指向待插入元素的后向指针。
			*/
	        if (((elm)->field.tqe_next = (head)->tqh_first) != NULL)        \
	                (head)->tqh_first->field.tqe_prev =                     \
	                    &(elm)->field.tqe_next;                             \
			/*
			  如果队列为空
			*/
	        else                                                            \
	                (head)->tqh_last = &(elm)->field.tqe_next;              \
	        (head)->tqh_first = (elm);                                      \
	        (elm)->field.tqe_prev = &(head)->tqh_first;                     \
	} while (/*CONSTCOND*/0)
	
宏QTAILQ\_INSERT\_HEAD(head, elm, field)用于在tail queue的队列头插入elm元素， head为队列头“指针”，elm为待插入元素， filed 为elm元素的一个QTAILQ\_ENTRY（type）类型的成员变量，该变量保存了用于维持队列结构的指针。

	#define QTAILQ_INSERT_TAIL(head, elm, field) do {                       \
	        (elm)->field.tqe_next = NULL;                                   \
	        (elm)->field.tqe_prev = (head)->tqh_last;                       \
	        *(head)->tqh_last = (elm);                                      \
	        (head)->tqh_last = &(elm)->field.tqe_next;                      \
	} while (/*CONSTCOND*/0)
	
宏QTAILQ\_INSERT\_TAIL(head, elm, field)用于在tail queue的队列尾插入elm，filed 为elm元素中的一个QTAILQ\_ENTRY（type）类型的成员变量。


	#define QTAILQ_INSERT_AFTER(head, listelm, elm, field) do {             \
	        if (((elm)->field.tqe_next = (listelm)->field.tqe_next) != NULL)\
	                (elm)->field.tqe_next->field.tqe_prev =                 \
	                    &(elm)->field.tqe_next;                             \
	        else                                                            \
	                (head)->tqh_last = &(elm)->field.tqe_next;              \
	        (listelm)->field.tqe_next = (elm);                              \
	        (elm)->field.tqe_prev = &(listelm)->field.tqe_next;             \
	} while (/*CONSTCOND*/0)


宏QTAILQ\_INSERT\_AFTER(head, listelm, elm, field)用于在listelem之后插入elm元素， field为QTAILQ\_ENTRY（type）类型的成员变量，head为队列头指针。

	#define QTAILQ_INSERT_BEFORE(listelm, elm, field) do {                  \
	        (elm)->field.tqe_prev = (listelm)->field.tqe_prev;              \
	        (elm)->field.tqe_next = (listelm);                              \
	        *(listelm)->field.tqe_prev = (elm);                             \
	        (listelm)->field.tqe_prev = &(elm)->field.tqe_next;             \
	} while (/*CONSTCOND*/0)
	
宏QTAILQ\_INSERT\_BEFORE(listelm, elm, field)用于在listelm之前插入elm元素，与上面宏类似。


	#define QTAILQ_REMOVE(head, elm, field) do {                            \
	        if (((elm)->field.tqe_next) != NULL)                            \
	                (elm)->field.tqe_next->field.tqe_prev =                 \
	                    (elm)->field.tqe_prev;                              \
	        else                                                            \
	                (head)->tqh_last = (elm)->field.tqe_prev;               \
	        *(elm)->field.tqe_prev = (elm)->field.tqe_next;                 \
	        (elm)->field.tqe_prev = NULL;                                   \
	} while (/*CONSTCOND*/0)

宏QTAILQ\_REMOVE(head, elm, field)删除队列elm所指的元素。

	
	#define QTAILQ_FOREACH(var, head, field)                                \
	        for ((var) = ((head)->tqh_first);                               \
	                (var);                                                  \
	                (var) = ((var)->field.tqe_next))
	
宏QTAILQ\_FOREACH(var, head, field) 遍历队列， 遍历过程中遍历到的每个元素的指针保存在var。



	#define QTAILQ_FOREACH_SAFE(var, head, field, next_var)                 \
	        for ((var) = ((head)->tqh_first);                               \
	                (var) && ((next_var) = ((var)->field.tqe_next), 1);     \
	                (var) = (next_var))

宏QTAILQ\_FOREACH\_SAFE(var, head, field, next\_var)与QTAILQ\_FOREACH的区别在于考虑到并发的场景，在“遍历过程中元素被删除”而导致遍历异常中断的问题。
	

	#define QTAILQ_FOREACH_REVERSE(var, head, headname, field)              \
	        for ((var) = (*(((struct headname *)((head)->tqh_last))->tqh_last));    \
	                (var);                                                  \
	                (var) = (*(((struct headname *)((var)->field.tqe_prev))->tqh_last)))
	
宏QTAILQ\_FOREACH\_REVERSE用于反方向遍历队列，与宏QTAILQ\_FOREACH相比多了一个宏参数headname，headname为定义head的类型名称。

### tail queue的操作函数

	#define QTAILQ_EMPTY(head)               ((head)->tqh_first == NULL)

宏QTAILQ\_EMPTY用于将队列头的tqh\_first成员置为NULL。

	#define QTAILQ_FIRST(head)               ((head)->tqh_first)

宏QTAILQ\_FIRST(head)为指向队列第一个元素的指针。

	#define QTAILQ_NEXT(elm, field)          ((elm)->field.tqe_next)

宏QTAILQ\_NEXT(elm, field)为指向elm后面元素的指针。


	#define QTAILQ_IN_USE(elm, field)        ((elm)->field.tqe_prev != NULL)

宏QTAILQ\_IN\_USE(elm, field) 为一个逻辑判断， 用于判断elm是否在队列中。

	#define QTAILQ_LAST(head, headname) \
        (*(((struct headname *)((head)->tqh_last))->tqh_last))

宏QTAILQ\_LAST(head, headname)为指向队列最后一个元素的指针。

	#define QTAILQ_PREV(elm, headname, field) \
        (*(((struct headname *)((elm)->field.tqe_prev))->tqh_last))

宏QTAILQ\_PREV(elm, headname, field)为指向elm前面一个元素的指针。





玩的开心 !!!
