---
layout: post
title: qemu 核心机制分析：qemu 协程库及使用
category: KVM虚拟化
tags: [KVM，qemu]
keywords: KVM，qemu
description: 
---

> 用正确的工具，做正确的事情

qemu 协程库采用分层方式来实现，基本的分层结构以及相关源文件如下：

1. 最底层：最底层基于gun c 的ucontext 函数族，实现线程内部不同上下文的切换，相关文件ucontext.h（gun c 基础库）；
2. 中间层：中间层通过对ucontext库函数的封装，实现最简单的协程语义比如创建协程，删除协程，协程切换，相关文件coroutine_int.h, coroutine-ucontext.c（qemu）；
3. 接口层：qemu coroutine 接口层通过对中间层简单协程语义函数的封装，提供qemu其他模块使用协程的接口，相关文件coroutine.h，qemu-coroutine.c，qemu-coroutine-lock.c，qemu-coroutine-sleep.c，qemu-coroutine-io.c（qemu）；
4. 接口调用层：其他模块通过调用 qemu coroutine接口层函数来使用协程机制完成相关的工作，相关文件比如block-backend.c(qemu)。

下面详细分析每层的重要数据结构，函数功能以及函数实现。

## ucontext 函数族相关知识

相关重要数据结构及主要函数的功能在[ucontext函数簇使用总结](http://deeper-think.github.io/2013/06/30/ucontext-first-try.html)中有做详细的介绍。

## qemu基本的协程语义及实现
	
相关文件coroutine_int.h, coroutine-ucontext.c，基于ucontext

### 重要数据结构

	#define COROUTINE_STACK_SIZE (1 << 20)

COROUTINE_STACK_SIZE定义了makeucontext时的堆栈空间上限。

	typedef enum {
	     COROUTINE_YIELD = 1,
	     COROUTINE_TERMINATE = 2,
	     COROUTINE_ENTER = 3,
	} CoroutineAction;

CoroutineAction联合体定义了协程可以执行的“基本动作”:协程让出、协程结束、协程进入。
 
	struct Coroutine {
	    CoroutineEntry *entry;
	    void *entry_arg;
	    Coroutine *caller;
	    QSLIST_ENTRY(Coroutine) pool_next;
	    size_t locks_held;
	
	    /* Coroutines that should be woken up when we yield or terminate */
	    QSIMPLEQ_HEAD(, Coroutine) co_queue_wakeup;
	    QSIMPLEQ_ENTRY(Coroutine) co_queue_next;
	};

Coroutine定义了

## qemu协程接口函数功能及实现


## 协程在virtIO block中的使用








玩的开心 !!!
