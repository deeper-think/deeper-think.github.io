---
layout: post
title: qemu 核心机制之二：module机制实现及分析
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

![尾队列指针结构](http://7u2rbh.com1.z0.glb.clouddn.com/tailqueue.png)


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

## 利用tail queue创建和使用队列

示例代码如下：

	#include<stdio.h>
	#include<stdlib.h>
	#include<string.h>
	#include "queue.h"
	
	struct AddrBook {
	    int total_acount;
    	const char * tag;
    	QTAILQ_HEAD(AddrBookHead, Contact) head;
	};
	
	struct Contact {
    	const char *name;
    	const char *telnum;
    	QTAILQ_ENTRY(Contact) next;
	};
	
	typedef struct AddrBook AddrBook;
	typedef struct Contact Contact;
	
	static AddrBook myAddrBook = {
    	.total_acount = 0,
    	.tag = "MyAddressBook",
    	.head = QTAILQ_HEAD_INITIALIZER(myAddrBook.head),
	};
	
	Contact* contact_create(const char* name, const char* telnum);
	
	void main()
	{
	
    	Contact* contact = NULL;
    	printf("Insert head: <name1,15960270001>\n");
    	contact = contact_create("name1","15960270001");
    	QTAILQ_INSERT_HEAD(&myAddrBook.head,contact,next);
	
    	printf("Insert head: <name2,15960270002>\n");
    	contact = contact_create("name2","15960270002");
    	QTAILQ_INSERT_HEAD(&myAddrBook.head,contact,next);
	
    	printf("Insert head: <name3,15960270003>\n");
    	contact = contact_create("name3","15960270003");
    	QTAILQ_INSERT_HEAD(&myAddrBook.head,contact,next);
	
    	printf("Insert head: <name4,15960270004>\n");
    	contact = contact_create("name4","15960270004");
    	QTAILQ_INSERT_HEAD(&myAddrBook.head,contact,next);
	
    	printf("Traversing the queue foreach elements: \n");
    	QTAILQ_FOREACH(contact, &myAddrBook.head, next)
    	    printf("%s,%s\n",contact->name,contact->telnum);
	
    	printf("Traversing the queue foreach elements by reverse: \n");
    	QTAILQ_FOREACH_REVERSE(contact, &myAddrBook.head, AddrBookHead, next)
    	    printf("%s,%s\n",contact->name,contact->telnum);
	
    	printf("Insert tail: <name5,15960270005>\n");
    	contact = contact_create("name5","15960270005");
    	QTAILQ_INSERT_TAIL(&myAddrBook.head,contact,next);

		printf("Insert tail: <name6,15960270006>\n");
    	contact = contact_create("name6","15960270006");
    	QTAILQ_INSERT_TAIL(&myAddrBook.head,contact,next);
	
    	printf("Traversing the queue foreach elements: \n");
    	QTAILQ_FOREACH(contact, &myAddrBook.head, next)
        printf("%s,%s\n",contact->name,contact->telnum);
	
    	printf("Insert <name7,15960270007> after <name3, 15960270003> \n");
    	Contact* listelm = NULL;
    	QTAILQ_FOREACH(contact, &myAddrBook.head, next)
    	{
    	    listelm = contact;
    	    if(0==strcmp(listelm->name,"name3")) break;
    	}
    	contact = contact_create("name7","15960270007");
    	QTAILQ_INSERT_AFTER(&myAddrBook.head, listelm, contact, next);
	
    	printf("Traversing the queue foreach elements: \n");
    	QTAILQ_FOREACH(contact, &myAddrBook.head, next)
    	    printf("%s,%s\n",contact->name,contact->telnum);
	
    	printf("Insert <name8,1596027000> before <name1, 15960270001> \n");
    	QTAILQ_FOREACH(contact, &myAddrBook.head, next)
    	{
    	    listelm = contact;
    	    if(0==strcmp(listelm->name,"name1")) break;
    	}
    	contact = contact_create("name8","15960270008");
    	QTAILQ_INSERT_BEFORE(listelm, contact, next);	
		
		printf("Traversing the queue foreach elements: \n");
    	QTAILQ_FOREACH(contact, &myAddrBook.head, next)
        	printf("%s,%s\n",contact->name,contact->telnum);
	
    	printf("Remove <name2, 1596027000> from queue\n");
    	QTAILQ_FOREACH(contact, &myAddrBook.head, next)
    	{
        	listelm = contact;
        	if(0==strcmp(listelm->name,"name2")) break;
    	}
    	QTAILQ_REMOVE(&myAddrBook.head, listelm, next);
	
    	printf("Traversing the queue foreach elements: \n");
    	QTAILQ_FOREACH(contact, &myAddrBook.head, next)
        	printf("%s,%s\n",contact->name,contact->telnum);
	}
	
	Contact* contact_create(const char* name, const char* telnum)
	{
    	Contact* contact = malloc(sizeof(*contact));
    	contact->name = name;
    	contact->telnum = telnum;
    	return contact;
	}


编译执行输出如下：

	-bash-4.2# ./a.out
	Insert head: <name1,15960270001>
	Insert head: <name2,15960270002>
	Insert head: <name3,15960270003>
	Insert head: <name4,15960270004>
	Traversing the queue foreach elements:
	name4,15960270004
	name3,15960270003
	name2,15960270002
	name1,15960270001
	Traversing the queue foreach elements by reverse:
	name1,15960270001
	name2,15960270002
	name3,15960270003
	name4,15960270004
	Insert tail: <name5,15960270005>
	Insert tail: <name6,15960270006>
	Traversing the queue foreach elements:
	name4,15960270004
	name3,15960270003
	name2,15960270002
	name1,15960270001
	name5,15960270005
	name6,15960270006
	Insert <name7,15960270007> after <name3, 15960270003>
	Traversing the queue foreach elements:
	name4,15960270004
	name3,15960270003
	name7,15960270007
	name2,15960270002
	name1,15960270001
	name5,15960270005
	name6,15960270006
	Insert <name8,1596027000> before <name1, 15960270001>
	Traversing the queue foreach elements:
	name4,15960270004
	name3,15960270003
	name7,15960270007
	name2,15960270002
	name8,15960270008
	name1,15960270001
	name5,15960270005
	name6,15960270006
	Remove <name2, 1596027000> from queue
	Traversing the queue foreach elements:
	name4,15960270004
	name3,15960270003
	name7,15960270007
	name8,15960270008
	name1,15960270001
	name5,15960270005
	name6,15960270006

注意QTAILQ_REMOVE只是将元素从队列中删除，但是该元素本身还是存在与内存中，并没有在内存中被释放。



玩的开心 !!!
