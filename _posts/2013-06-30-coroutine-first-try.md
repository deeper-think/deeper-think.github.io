---
layout: post
title: coroutine编程初探
category: 技术
tags: [coroutine]
keywords: coroutine
description: 
---

> 用正确的工具，做正确的事情

最近在看virtio-blk的代码，发现qemu端最终通过创建coroutine提交aio以及对aio结果结果的获取和处理，使用coroutine是一种编程模式。coroutine可以看成是一种微线程，用户态进程在执行过程中可以在多个coroutine之间快速切换，可以实现在单进程中任务“并发”，这里并发是指宏观上的，主要是通过当任务在执行过程中，因为需要等待事件或者资源而需要阻塞时并不阻塞进程，而是通过上下文切换，切换到其他可以执行的任务上下文中继续执行，通过减少单线程的阻塞，而实现了在单线程中多任务的“并发”执行。

对于经典的“生产值-消费者”模型，假设生产缓冲区为1，基于多线程的实现方式一般如下：

	#include <stdlib.h>
	#include <stdio.h>
	#include <pthread.h>
	
	int product = 0;
	pthread_cond_t buf_ready = PTHREAD_COND_INITIALIZER;
	pthread_cond_t prod_ready = PTHREAD_COND_INITIALIZER;
	pthread_mutex_t prod_lock = PTHREAD_MUTEX_INITIALIZER;
	
	void *producer(void *apue) {
	    printf("start producer!\n");
	    while(1) {
	        pthread_mutex_lock(&prod_lock);
	        if(1 == product)
	            pthread_cond_wait(&buf_ready, &prod_lock);
	        product = 1;
	        sleep(1);
	        printf("put a product!\n");
	        pthread_cond_signal(&prod_ready);
	        pthread_mutex_unlock(&prod_lock);
	    }
	}
	
	void *consumer(void *apue) {
	    printf("start consumer!\n");
	    while(1) {
	        pthread_mutex_lock(&prod_lock);
	        if(0 == product)
	            pthread_cond_wait(&prod_ready, &prod_lock);
	        product = 0;
	        sleep(1);
	        printf("get a product!\n");
	        pthread_cond_signal(&buf_ready);
	        pthread_mutex_unlock(&prod_lock);
	    }
	}
	
	int 
	main(void) {
	    pthread_t producer_tid, consumer_tid;
	    void *tret;
	    int err;
	    
	    pthread_create(&producer_tid, NULL, producer, NULL);
	    pthread_create(&consumer_tid, NULL, consumer, NULL);
	    pthread_join(producer_tid, &tret);
	    pthread_join(consumer_tid, &tret);
	    return 0;
	}
	

分析上面的实现可以发现，因为生产缓冲区的大小为1，孙然创建了两个线程，一个生产者线程，一个消费者线程，两个线程因为资源竞争，线程大部分cpu时间片都在阻塞，两个线程在串行交替的执行，当然在解决实际“生产值-消费者”问题时，生产缓冲区一般不会为1，但是即使在这种情况下，如果生产者和消费者线程的处理速度不平衡时，也会出现线程阻塞的问题。另在线程调度，线程间的同步，以及加锁解锁等操作，都需要额外的系统资源。因此对于一些实际问题，如果采用多个线程实现解决，但是实际上线程大部分时间片都处于阻塞状态的话，那么在这种情况下就需要考虑有没有必要采用多线程。


对于上面的“生产者-消费者”问题，如果不采用多线程，可以修改生产者或者消费者一方的实现，通过生产者调用消费者或者消费者调用生产者的方式实现：

	#include <stdlib.h>
	#include <stdio.h>
	
	int product = 0;
		
	void consumer(void) {
	    printf("start consumer!\n");
	    product = 0;
	    sleep(1);
	    printf("get a product!\n");
	}
	
	void producer(void) {
	    printf("start producer!\n");
	    while(1) {
	        product = 1;
	        sleep(1);
	        printf("put a product!\n");
	        consumer();
	    }
	}
	
	int 
	main(void) {
	    producer();
	    return 0;
	}
	

通过修改consumer的实现，使consumer每次消费一个product后就退出，producter依然循环生产product，但每生产一个product就调用consumer去消费，但通过这种方式实现的“生产者和消费者”模型，已经与原本的“生产者和消费者”模型有区别，通过这种方式实现的“生产者-消费者”模型，实际上是通过每次创建一个消费者去消费一个产品然后退出，如果每次消费者消费产品之前需要做大量的初始化的工作，那么这种实现方式就存在效率问题，这种实现方式已经不是原本的“生产者-消费者”问题了。


使用协程的方式，创建生产者协程和消费者协程，通过在生产者和消费者协程间切换，来在单线程内实现producer生产product和consumer消费product的“并发”执行（使用了setucontext函数族，该函数族的介绍见下文）：
	
	#include <stdlib.h>
	#include <stdio.h>
	#include <ucontext.h>
	int product = 0;
	ucontext_t main_context, producter_context, consumer_context;
	
	void producter(void) {
	    printf("start producer, and do initialization!\n");
	    swapcontext(&producter_context, &main_context);
	    while(1) {
	        product = 1;
	        sleep(1);
	        printf("put a product!\n");
	        swapcontext(&producter_context, &consumer_context);
	    }
	}
	
	void consumer(void) {
	    printf("start consumer, and do initialization!\n");
	    swapcontext(&consumer_context, &main_context);
	    while(1) {
	        product = 0;
	        sleep(1);
	        printf("get a product!\n");
	        swapcontext(&consumer_context, &producter_context);
	    }
	}
	
	void init_context(void) {
	    char producter_stack[SIGSTKSZ];
	    char consumer_stack[SIGSTKSZ];
	
	    getcontext(&producter_context);
	    producter_context.uc_link          = &main_context;
	    producter_context.uc_stack.ss_sp   = producter_stack;
	    producter_context.uc_stack.ss_size = sizeof(producter_stack);
	    makecontext(&producter_context, (void (*)(void)) producter, 0);
	    
	    getcontext(&consumer_context);
	    consumer_context.uc_link          = &main_context;
	    consumer_context.uc_stack.ss_sp   = consumer_stack;
	    consumer_context.uc_stack.ss_size = sizeof(consumer_stack);
	    makecontext(&consumer_context, (void (*)(void)) consumer, 0);
	    return;
	}
	
	int 
	main(void) {
	    static int is_finished = 0;
	    init_context();
	    if (!is_finished) {
	        is_finished = 1;
	        while(1) {
	            printf("main1\n");
	            swapcontext(&main_context, &producter_context);
	            printf("main2\n");
	            swapcontext(&main_context, &consumer_context);
	            printf("main3\n");
	        }
	    }
	}
	

c语言并没有对协程提供语言级别的支持，但是setcontext函数族提供在用户态进程空间执行上下文切换的能力，一些高级语言提供了对coroutine语言级别的支持，如lua。通过使用setcontext函数族，可以封装自己的c语言协程库，比如开源项目libtask就是基于setcontext实现的一套协程库，qemu的coroutine也是通过setcontext函数族实现的，以下是整理翻译维基对setcontext函数族的介绍以及提供的示例代码。

	#include <stdio.h>
	#include <stdlib.h>
	#include <ucontext.h>
	
	/* 执行上下文结构体 */
	/* typedef struct ucontext {
	 *         struct ucontext *uc_link;    //从当前上下文返回时，将切换到的上下文环境.
	 *         sigset_t         uc_sigmask; //保存上下文中阻塞的信号.
	 *         stack_t          uc_stack;   //上下文使用的栈空间.
	 *         mcontext_t       uc_mcontext;//保存当前上下文的执行状态，如寄存器状态，cpu计数，etc.
	 *         ....
	 *		   } ucontext_t; */
		
	/* setcontext 家族中的函数集合,切换到ucp所指定的上下文中 
	 * int   setcontext(const ucontext_t *ucp) */
	 
	/* 保存当前上下文到 ucp中
	 * int   getcontext(ucontext_t *ucp) */
	
	/* 在ucp 环境中，创建一个可选的控制线程，之前ucp环境必须被初始化，并且ucp.uc_stack必须指向已分配的一段栈空间。
	 * 当通过setcontext或swapcontext切换到ucp上下文时，控制流从func开始执行。
	 * void  makecontext(ucontext_t *ucp, void *func(), int argc, ...) */
	
	/* 切换执行控制权到 ucp指定的环境，保存当前执行环境到oucp       
	 * int   swapcontext(ucontext_t *oucp, ucontext_t *ucp) */
	
	void loop(
	    ucontext_t *loop_context,
	    ucontext_t *other_context,
	    int *i_from_iterator)
	{
	    int i;
	
	    for (i = 0; i < 10; i++) {
	        /* 将循环计数写到迭代器中 */
	        *i_from_iterator = i;
	
	        /* 保存当前执行上下文到 loop_context,
	         * 并切换到other_context 执行上下文中 */
	        swapcontext(loop_context, other_context);
	    }
	
	    /* loop_context上下文执行流将结束，
	     * 并且执行上下文将自动切换到(&loop_context->uc_link)上下文
	     * 因此不用显式调用 setcontext(&loop_context->uc_link) */
	}
	int main(void)
	{
	    /* 定义3个contexts
	     * (1) main_context1: 保存main函数中的执行上下文，loop 执行流完成时，切换到的执行上下文
	     * (2) main_context2: 保存main函数中的执行上下文，loop 执行流执行过程中，通过swapcontext切换到main中的执行上下文
	     * (3) loop_context:  保存loop函数中的执行上下文，main函数中通过swapcontext 从main_context2切换到 loop_context */
	    ucontext_t main_context1, main_context2, loop_context;
	
	    char iterator_stack[SIGSTKSZ];
	    /* iterator_loop 结束标志 */
	    volatile int iterator_finished;
	    /* 迭代器 */
	    volatile int i_from_iterator;
	
	    /* makecontext loop_context 之前必须要通过 getcontext 并初始化loop_context */
	    getcontext(&loop_context);
	    loop_context.uc_link          = &main_context1;
	    loop_context.uc_stack.ss_sp   = iterator_stack;
	    loop_context.uc_stack.ss_size = sizeof(iterator_stack);
	
	    /* 填充loop_context执行流 */
	    makecontext(&loop_context, (void (*)(void)) loop, 
	        3, &loop_context, &main_context2, &i_from_iterator);
	
	    /* 清除迭代器 finished 标志 */
	    iterator_finished = 0;
	    
	    /* 保存当前context。当loop上下文控制流执行完成时，执行上下文将切换到该处 */ 
	    getcontext(&main_context1);
	
	    if (!iterator_finished) {
	        /* 设置iterator_finished，当iterator上下文执行流结束时，
	         * 上下文切换到uc_link指定的上下文，也即main_context1.
	         * 此时if条件判断失败，迭代操作不会被重启*/
	        iterator_finished = 1;
	
	        while(1) {
	            /* 保存当前执行上下文到 main_context2，切换到当前执行上下文到 loop_context */
	            swapcontext(&main_context2, &loop_context);
	            printf("%d\n", i_from_iterator);
	        }
	    }
	
	    return 0;
	}
	

上面主要说了两点：第一、协程这种编程技巧能够解决多线程不能很好解决的问题，除此之外协程可以代替callback，降低代码的复杂度。第二、setcontext函数族的使用方法。接下来有时间分析一下，qemu协程库和libtask的实现，整理一套coroutine库，在以后解决实际问题时，可以使用coroutine这种编程技巧。

### 参考：　

1. [Coroutines in C](http://www.chiark.greenend.org.uk/~sgtatham/coroutines.html)
2. [Lua coroutine 不一样的多线程编程思路](http://timyang.net/lua/lua-coroutine/)
3. [setcontext-wiki](http://en.wikipedia.org/wiki/Setcontext)


玩的开心 !!!
