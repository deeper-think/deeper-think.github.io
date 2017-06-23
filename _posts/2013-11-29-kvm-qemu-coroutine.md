---
layout: post
title: qemu 核心机制分析：coroutine机制分析
category: KVM虚拟化
tags: [KVM，qemu]
keywords: KVM，qemu
description: 
---

> 用正确的工具，做正确的事情

qemu coroutine机制总体上分层概括为：

1. 最底层：最底层基于gun c 的ucontext 函数族，实现线程内部不同上下文的切换，相关文件ucontext.h（gun c 基础库）；
2. 中间层：中间层通过对ucontext库函数的封装，实现最简单的协程语义比如创建协程，删除协程，协程切换，相关文件coroutine_int.h, coroutine-ucontext.c（qemu）；
3. 接口层：qemu coroutine 接口层通过对中间层简单协程语义函数的封装，提供qemu其他模块使用协程的接口，相关文件coroutine.h，qemu-coroutine.c，qemu-coroutine-lock.c，qemu-coroutine-sleep.c，qemu-coroutine-io.c（qemu）；
4. 接口调用层：其他模块通过调用 qemu coroutine接口层函数来使用协程机制完成相关的工作，相关文件比如block-backend.c(qemu)。

下面详细分析每层的重要数据结构，函数功能以及函数实现。

## ucontext 函数族相关知识


## qemu基本的协程语义及实现


## qemu协程接口函数功能及实现


## 协程在virtIO block中的使用








玩的开心 !!!
