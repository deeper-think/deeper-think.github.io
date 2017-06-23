---
layout: post
title: ucontext函数簇使用总结
category: 编程调试技巧
tags: [coroutine]
keywords: coroutine
description: 
---

> 用正确的工具，做正确的事情

最近在读qemu virtio-blk的代码实现时发现qemu基于ucontext封装了自己的协程库，并在virtio-blk的代码流程中使用了该库来避免阻塞以及线程内"并发"。协程这个东西在很多语言中比如python提供了语言级的支持，之前或多或少有些了解，但是并不是很熟悉去原理及内部的实现机制，但既然都是基于类似于ucontext族等的底层技术来实现，那么有必要好好研究研究ucontext 族相关的数据结构以及函数功能及其实现。

## ucontext族重要数据结构

ucontext族最终的数据结构 struct ucontext，该结构体包括的成员变量在不同的平台架构会有所不同，但是都会包括下面几个关键成员：

	/* 执行上下文结构体 */
	typedef struct ucontext {
	struct ucontext *uc_link;    //从当前上下文返回时，将切换到的上下文环境.
	sigset_t         uc_sigmask; //保存上下文中阻塞的信号.
	stack_t          uc_stack;   //上下文使用的栈空间.
	mcontext_t       uc_mcontext;//保存当前上下文的执行状态，如寄存器状态，cpu计数，etc.
	....
	} ucontext_t;


## ucontext族关键函数的功能及实现

	int   getcontext(ucontext_t *ucp);

官方说明：

1. The function getcontext() initializes the structure pointed at by ucp to the currently active context.
2.  When successful, getcontext() returns 0.On error, returns -1 and set errno appropriately.

	int   setcontext(const ucontext_t *ucp);

官方说明：

1. The function setcontext() restores the user context pointed at by ucp.  A successful call does not return.  The context should have been obtained by a call of getcontext(), or makecontext(3), or passed as third argument to a signal handler.
2. If the context was obtained by a call of getcontext(), program execution continues as if this call just returned.
3. If the context was obtained by a call of makecontext(3), program execution continues by a call to the function func specified as the second argument of that call to makecontext(3).  When the function func returns, we continue with the uc_link member of the structure ucp specified as the first argument of that call to makecontext(3). When this member is NULL, the thread exits.
4. If the context was obtained by a call to a signal handler, then old standard text says that "program execution continues with the program.
5. instruction following the instruction interrupted by the signal".However, this sentence was removed in SUSv2, and the present verdict is "the result is unspecified".
6. When successful,setcontext() does not return.  On error, return -1 and set errno appropriately.
	
	void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...);
	
官方说明：

1. The makecontext() function modifies the context pointed to by ucp (which was obtained from a call to getcontext(3)).Before invoking makecontext(), the caller must allocate a new stack for this context and assign its address to ucp->uc_stack, and define a successor context and assign its address to ucp->uc_link.
2. When this context is later activated (using setcontext(3) or swapcontext()) the function func is called, and passed the series of integer (int) arguments that follow argc; the caller must specify the number of these arguments in argc.  When this function returns, the successor context is activated.  If the successor context pointer is NULL, the thread exits.

	  int swapcontext(ucontext_t *oucp, const ucontext_t *ucp); 

官方说明：
1. The swapcontext() function saves the current context in the structure pointed to by oucp, and then activates the context pointed to by ucp.
2. When successful, swapcontext() does not return.  (But we may return later, in case oucp is activated, in which case it looks like swapcontext() returns 0.)  On error, swapcontext() returns -1 and sets errno appropriately.

> getcontext与swapcontext函数会保存"执行上下文"到ucp数据结构中，所谓的“执行上下文”即当前程序运行相关寄存件当前值的集合，其中包括堆栈寄存器SP，但是需要强调的是不包括整个程序堆栈空间。因此 当程序的执行再次回到ucp所指的上下文时，保存在程序堆栈空间里面的局部变量的值有可能已经发生了变化。


## ucontext族使用实例

### 利用ucontext 实现生产者消费者模型

利用swapcontext在生产者和消费者协程间切换，来在单线程内实现producer生产product和consumer消费product的“并发”执行（使用了setucontext函数族，该函数族的介绍见下文）：
	
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
	
### 另外一个示例

	#include <stdio.h>
	#include <stdlib.h>
	#include <ucontext.h>
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
	
## 参考：　

1. [Coroutines in C](http://www.chiark.greenend.org.uk/~sgtatham/coroutines.html)
2. [Lua coroutine 不一样的多线程编程思路](http://timyang.net/lua/lua-coroutine/)
3. [setcontext-wiki](http://en.wikipedia.org/wiki/Setcontext)


玩的开心 !!!
