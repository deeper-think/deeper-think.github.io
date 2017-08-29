---
layout: post
title: qemu 核心机制分析：qemu 协程库及使用
category: KVM虚拟化
tags: [KVM，qemu]
keywords: KVM，qemu
description: 
---

> 用正确的工具，做正确的事情

qemu Coroutine机制的实现可以分为两个层次，最底层利用UContext函数簇以及setjmp/longjmp库函数实现基本的协程语义，包括创建一个新的Coroutine、进入一个已创建的Coroutine以及删除Coroutine，上层在基本协程语义的基础上实现了Coroutine池、Coroutine锁的功能。

## UContext函数簇以及setjmp/longjmp库函数

UContext相关重要数据结构及主要函数的功能在[ucontext函数簇使用总结](http://deeper-think.github.io/2013/06/30/ucontext-first-try.html)中有做详细的介绍。

	#include <setjmp.h>
	int setjmp(jmp_buf env);

功能说明：setjmp 函数的功能是将函数在此处的上下文保存在 jmp\_buf 结构体中，以供 longjmp 从此结构体中恢复。该函数一次调用返回两次，其中直接从函数调用返回一次，另外通过longjmp跳转回之前上下文时返回一次，如果直接从函数调用返回，则返回值为0 ，如果从longjmp跳转返回，则返回值为longjmp的第二个参数。

	void longjmp(jmp_buf env, int val);

功能说明：longjmp 函数的功能是从 jmp\_buf 结构体中恢复由 setjmp 函数保存的上下文，该函数不返回，而是从 setjmp 函数中返回。如果val的值为0， 则从setjmp返回的时候返回值为1，如果val的值不为0则从setjmp返回的时候返回值为val。



## 基本Coroutine语义

基本Coroutine语义实现了创建一个新的Coroutine、进入一个已经创建的Coroutine以及删除一个Coroutine，qemu源代码里面相关的文件为：coroutine_int.h, coroutine-ucontext.c。

### 重要数据结构
	
	/* 联合体定义了协程上下文可能发生的动作，包括让出、结束以及进入。 */
	33 typedef enum {
	34     COROUTINE_YIELD = 1,
	35     COROUTINE_TERMINATE = 2,
	36     COROUTINE_ENTER = 3,
	37 } CoroutineAction;    
	38 
	
	/* 跟Coroutine的使用有关的结构体，跟Coroutine的实现比如堆栈没有关系，
	 * 该结构的对象会嵌在CoroutineUContext结构，作为该结构的一个成员变量。
	 */
	39 struct Coroutine {
	40     CoroutineEntry *entry;     //协程入口函数,严格点讲该函数并不是进入协程的函数，而是通过longjmp跳转进入协程之后，通过回调开始执行该函数。
	41     void *entry_arg;           //协程入口函数的参数
	42     Coroutine *caller;		  //父协程
	43     QSLIST_ENTRY(Coroutine) pool_next;  //协程资源池的结构指针
	44     size_t locks_held;                  //协程锁
	45 
	46     /* Coroutines that should be woken up when we yield or terminate */
	47     QSIMPLEQ_HEAD(, Coroutine) co_queue_wakeup;   //该协程让出或者结束时需要唤醒的协程队列
	48     QSIMPLEQ_ENTRY(Coroutine) co_queue_next;
	49 };

	/* 协程上下文，跟具体的协程实现相关的结构体，包括堆栈空间，但是该结构体不会直接暴露给协程的使用者 */
	34 typedef struct {
	35     Coroutine base;    
	36     void *stack;          //协程堆栈空间
	37     size_t stack_size;    //协程堆栈空间的大小
	38     sigjmp_buf env;       //协程入口位置，通过longjmp(env,var)跳转进入该协程上下文
	39 
	40 #ifdef CONFIG_VALGRIND_H
	41     unsigned int valgrind_stack_id;
	42 #endif
	43 
	44 } CoroutineUContext;
	45 

	46 /**
	47  * 两个全局静态变量，线程私有的，引入协程之后一个线程会同时存在多个执行上下文，current指针用于记录当前线程所处的执行上下文，leader表示线程原来的上下文。
	48  */
	49 static __thread CoroutineUContext leader;
	50 static __thread Coroutine *current;
	51

 	   /* 
	52  * 非常巧妙的用法，利用联合体进行灵活的类型转换，详细解释：
	53  * va_args to makecontext() must be type 'int', so passing
	54  * the pointer we need may require several int args. This
	55  * union is a quick hack to let us do that
	56  */
	57 union cc_arg {
	58     void *p;
	59     int i[2];
	60 };

上面是跟qemu 协程基本语义实现相关的数据结构，接下来分析三个协程创建，切换以及删除相关的函数。

### 创建协程：qemu\_coroutine\_new（）
   
	/*
	 * 创建一个新的协程，分为两步：
	 * 1、创建CoroutineUContext结构对象，并创建并初始化协程堆栈空间，也即协程上下文初始化；
	 * 2、 swapcontext进入协程上下文，创建并记录该协程上下文入口点，然后longjmp离开该协程上下文，其中记录协程上下文入口点并且离开该协程上下文通过函数coroutine_trampoline实现。
	 */
	85 Coroutine *qemu_coroutine_new(void)
	86 {
	87     CoroutineUContext *co;
	88     ucontext_t old_uc, uc;//临时变量，主要目的便于利用UContext函数簇实现co->stack及协程上下文的初始化。
	89     sigjmp_buf old_env;
	90     union cc_arg arg = {0};
	91 
	92     /* The ucontext functions preserve signal masks which incurs a
	93      * system call overhead.  sigsetjmp(buf, 0)/siglongjmp() does not
	94      * preserve signal masks but only works on the current stack.
	95      * Since we need a way to create and switch to a new stack, use
	96      * the ucontext functions for that but sigsetjmp()/siglongjmp() for
	97      * everything else.
	98      */
	99     if (getcontext(&uc) == -1) {
	100         abort();
	101     }   
	102     
	103     co = g_malloc0(sizeof(*co));
	104     co->stack_size = COROUTINE_STACK_SIZE;
	105     co->stack = qemu_alloc_stack(&co->stack_size);//创建协程堆栈空间
	106     co->base.entry_arg = &old_env; /* stash away our jmp_buf */  
	107     
	108     uc.uc_link = &old_uc;
	109     uc.uc_stack.ss_sp = co->stack;//uc与co指向同一个堆栈空间，利用uc函数簇来初始化协程堆栈空间及上下文。
	110     uc.uc_stack.ss_size = co->stack_size;
	111     uc.uc_stack.ss_flags = 0;
	112 
	113 #ifdef CONFIG_VALGRIND_H
	114     co->valgrind_stack_id =
	115         VALGRIND_STACK_REGISTER(co->stack, co->stack + co->stack_size);
	116 #endif
	117 
	118     arg.p = co;
	119 
	120     makecontext(&uc, (void (*)(void))coroutine_trampoline,
	121                 2, arg.i[0], arg.i[1]);
	122 
	123     /* swapcontext() in, siglongjmp() back out */
	124     if (!sigsetjmp(old_env, 0)) {
	125         swapcontext(&old_uc, &uc);//利用UContext函数簇进入协程上下文，协程入口开始为coroutine_trampoline
	126 
	127     }
	128     return &co->base;
	129 }

87~122行均是创建CoroutineUContext结构对象，分配协程堆栈空间并利用UContext函数簇完成协程上下文的初始化，124~125记录当前上下文并利用swapcontext进入协程上下文，协程上下文的入口函数为coroutine_trampoline，在coroutine_trampoline里面创建并记录该协程的入口位置，便于以后进入该协程执行具体的任务，然后跳转出该协程，coroutine_trampoline的实现如下：

	/*
	 * trampoline:动词，在蹦床上弹跳，很形象的函数名。
	 */
	62 static void coroutine_trampoline(int i0, int i1)
	63 {
	64     union cc_arg arg;
	65     CoroutineUContext *self;
	66     Coroutine *co;
	67     
	68     arg.i[0] = i0;
	69     arg.i[1] = i1;
	70     self = arg.p;
	71     co = &self->base;
	72     
	73     /* Initialize longjmp environment and switch back the caller */
	74     if (!sigsetjmp(self->env, 0)) {                 //记录协程入口位置
	75         siglongjmp(*(sigjmp_buf *)co->entry_arg, 1);//跳转返回原上下文
	76     }   
	77     
	78     while (true) {
	79         co->entry(co->entry_arg);
	80         qemu_coroutine_switch(co, co->caller, COROUTINE_TERMINATE);
	81     }   
	82 }

### 进入协程：qemu\_coroutine\_switch（）

	165 CoroutineAction __attribute__((noinline))
	166 qemu_coroutine_switch(Coroutine *from_, Coroutine *to_,
	167                       CoroutineAction action)
	168 {
	169     CoroutineUContext *from = DO_UPCAST(CoroutineUContext, base, from_);
	170     CoroutineUContext *to = DO_UPCAST(CoroutineUContext, base, to_);
	171     int ret;
	172 
	173     current = to_;
	174 
	175     ret = sigsetjmp(from->env, 0);
	176     if (ret == 0) {
	177         siglongjmp(to->env, action);
	178     }
	179     return ret;
	180 }

qemu\_coroutine\_switch()函数从from\_协程上下文切换到to_协程上下文，关键代码是175~178行，首先保存当前上下文到from->env，以便于以后从to协程上下文返回，然后通过siglongjmp进入目标to协程上下文。

### 删除协程：qemu\_coroutine\_delete()

	150 void qemu_coroutine_delete(Coroutine *co_)
	151 {
	152     CoroutineUContext *co = DO_UPCAST(CoroutineUContext, base, co_);
	153 
	154 #ifdef CONFIG_VALGRIND_H
	155     valgrind_stack_deregister(co);
	156 #endif
	157 
	158     qemu_free_stack(co->stack, co->stack_size);
	159     g_free(co);
	160 }
qemu\_coroutine\_delete（）函数删除co_所指向的协程，主要是两个协程数据结构的释放CoroutineUContext结构对象以及该结构对象的成员stack。

## Coroutine池及Coroutine锁

利用UContext函数簇以及setjmp/longjmp库函数，基本的协程语义实现了协程的创建、切换以及删除的基本功能，在协程基本功能的基础上，上面一层进一步实现了协程资源池以及协程锁的功能，相关qemu源代码文件coroutine.h，qemu-coroutine.c，qemu-coroutine-lock.c，qemu-coroutine-sleep.c，qemu-coroutine-io.c，下面总结Coroutine池以及Coroutine锁实现相关的数据结构以及功能函数

### 重要数据结构

	/**
	 * CoQueues are a mechanism to queue coroutines in order to continue executing
	 * them later. They provide the fundamental primitives on which coroutine locks
	 * are built.
	 */
	typedef struct CoQueue {
	    QSIMPLEQ_HEAD(, Coroutine) entries;
	} CoQueue;

	/**
	 * Provides a mutex that can be used to synchronise coroutines
	 */
	typedef struct CoMutex {
	    bool locked;
	    Coroutine *holder;
	    CoQueue queue;
	} CoMutex;

	typedef struct CoRwlock {
    	bool writer;
    	int reader;
    	CoQueue queue;
	} CoRwlock;
	
CoQueue与CoMutex/CoRwlock是与Coroutine锁相关的数据结构, 提供两种类型的锁：互斥锁CoMutex和读写锁CoRwlock， 这两种类型的锁都有一个CoQueue类型的成员变量queue，大致的使用方法是Coroutine尝试加锁不成功，会将自己放到锁对象的queue里面，暂时退出该协程上下文，等锁就绪后再通过遍历queue来唤醒相关的协程，避免因为等待锁而出现阻塞。


	static QSLIST_HEAD(, Coroutine) release_pool = QSLIST_HEAD_INITIALIZER(pool);
	static unsigned int release_pool_size;
	static __thread QSLIST_HEAD(, Coroutine) alloc_pool = QSLIST_HEAD_INITIALIZER(pool);
	static __thread unsigned int alloc_pool_size;
	static __thread Notifier coroutine_pool_cleanup_notifier;

上面的全局静态变量主要用于实现Coroutine资源池，上层创建Coroutine时会首先尝试从Coroutine池中alloc\_pool或release\_pool中取，如果取不到才会调用qemu\_coroutine\_new（）来创建一个新的，同样上层需要释放一个Coroutine时也不是直接释放该Coroutine对象，而是先将对象放到资源池里面。

### Coroutine操作相关的函数

#### 创建协程qemu\_coroutine\_create（）

	45 Coroutine *qemu_coroutine_create(CoroutineEntry *entry, void *opaque)
	46 {
	47     Coroutine *co = NULL;
	48 
	49     if (CONFIG_COROUTINE_POOL) { //如果有启动资源池，那么协程的创建首先尝试从协程池里面获取
	50         co = QSLIST_FIRST(&alloc_pool); //尝试从alloc_pool里面取，如果从alloc_pool取不到则，
	51         if (!co) {
	52             if (release_pool_size > POOL_BATCH_SIZE) {
	53                 /* Slow path; a good place to register the destructor, too.  */
	54                 if (!coroutine_pool_cleanup_notifier.notify) {
	55                     coroutine_pool_cleanup_notifier.notify = coroutine_pool_cleanup;
	56                     qemu_thread_atexit_add(&coroutine_pool_cleanup_notifier);
	57                 }
	58 
	59                 /* This is not exact; there could be a little skew between
	60                  * release_pool_size and the actual size of release_pool.  But
	61                  * it is just a heuristic, it does not need to be perfect.
	62                  */
					   /* 将release_pool池的资源挪到alloc_pool,然后再次从alloc_pool里面取 */
	63                 alloc_pool_size = atomic_xchg(&release_pool_size, 0);
	64                 QSLIST_MOVE_ATOMIC(&alloc_pool, &release_pool);
	65                 co = QSLIST_FIRST(&alloc_pool);
	66             }
	67         }
	68         if (co) { //如果最终从alloc_pool里面获取到资源，则将该资源从池里面删除，同时修改alloc_pool资源数量 
	69             QSLIST_REMOVE_HEAD(&alloc_pool, pool_next);
	70             alloc_pool_size--;
	71         }
	72     }
	73      
	74     if (!co) {   //如果没有从资源池里面获取到资源，则调用qemu_coroutine_new创建一个新的Coroutine
	75         co = qemu_coroutine_new();
	76     }	
	77 
	78     co->entry = entry;  //将入口函数以及入口函数参数的地址保存到Coroutine结构对象里面,并初始化wakeup队列
	79     co->entry_arg = opaque;
	80     QSIMPLEQ_INIT(&co->co_queue_wakeup);
	81     return co;
	82 }

上面是对qemu\_coroutine\_create（）函数的分析，其中有两个疑问：

1. 52~57行，看起来像是一种异步通知的机制,具体实现有待进一步分析；
2. 这里Coroutine为了实现"池"的概念，创建了两个资源池alloc_pool以及release_pool，为什么创建两个pool，目的何在？难道是为了实现无锁队列。

#### 删除协程coroutine\_delete()

	84 static void coroutine_delete(Coroutine *co)
	85 {
	86     co->caller = NULL; //父协程置空
	87 
	88     if (CONFIG_COROUTINE_POOL) {
	89         if (release_pool_size < POOL_BATCH_SIZE * 2) {  //删除协程时，待删除的协程优先放入release_pool里面
	90             QSLIST_INSERT_HEAD_ATOMIC(&release_pool, co, pool_next);
	91             atomic_inc(&release_pool_size);
	92             return;
	93         }
	94         if (alloc_pool_size < POOL_BATCH_SIZE) {//判断 alloc_pool是否已经满，尝试将待删除的协程alloc_pool
	95             QSLIST_INSERT_HEAD(&alloc_pool, co, pool_next);
	96             alloc_pool_size++;
	97             return;
	98         }
	99     }
	100 
	101     qemu_coroutine_delete(co);  //如果资源池内资源的数量已经达到上限，然后才直接删除释放该协程对象
	102 }

#### 进入协程qemu\_coroutine\_enter（）

	104 void qemu_coroutine_enter(Coroutine *co)
	105 {
	106     Coroutine *self = qemu_coroutine_self();   //获取当前协程
	107     CoroutineAction ret;
	108 
	109     trace_qemu_coroutine_enter(self, co, co->entry_arg);
	110 
	111     if (co->caller) {
	112         fprintf(stderr, "Co-routine re-entered recursively\n");
	113         abort();
	114     }
	115 
	116     co->caller = self;       //保存当前上下文，作为待进入协程的"父"
	117     ret = qemu_coroutine_switch(self, co, COROUTINE_ENTER);  //切换进入co协程上下文。
	118 
	119     qemu_co_queue_run_restart(co); //依次唤醒co->co_queue_wakeup里面的协程，进入其上下文开始执行
	120 
	121     switch (ret) {
	122     case COROUTINE_YIELD: //co上下文执行过程中碰到需要等待的事件，因此主动从co上下文让出
	123         return;
	124     case COROUTINE_TERMINATE: //协程已经执行完成，因此可以删除协程
	125         assert(!co->locks_held);
	126         trace_qemu_coroutine_terminate(co);	
	127         coroutine_delete(co);
	128         return;
	129     default:
	130         abort();
	131     }
	132 }

117行，qemu\_coroutine\_switch函数内部通过longjmp进入co上下文，该函数返回表明已经从co协程上下文返回，ret表示通过何种方式从co上下文返回，是主动让出还是执行完成。


#### 让出协程qemu\_coroutine\_yield()

	134 void coroutine_fn qemu_coroutine_yield(void)
	135 {
	136     Coroutine *self = qemu_coroutine_self();
	137     Coroutine *to = self->caller;
	138 
	139     trace_qemu_coroutine_yield(self, to);
	140 
	141     if (!to) {
	142         fprintf(stderr, "Co-routine is yielding to no one\n");
	143         abort();
	144     }
	145 
	146     self->caller = NULL;
	147     qemu_coroutine_switch(self, to, COROUTINE_YIELD);
	148 }

该函数的实现逻辑比较简单，函数的声明使用了coroutine_fn标记，表明该函数只能在Coroutine上下文内被调用执行。

### Coroutine锁相关的操作函数

Coroutine实现了"互斥"锁和"读写"锁, 互斥锁的lock_init、lock以及unlock的逻辑比较简单，下面只对“读写”锁的实现进行简单分析，互斥锁的实现类似。

#### "读写"锁初始化

	156 void qemu_co_rwlock_init(CoRwlock *lock)
	157 {
	158     memset(lock, 0, sizeof(*lock));
	159     qemu_co_queue_init(&lock->queue);
	160 }

lock指向的CoRwlock结构对象内存置0，CoQueue成员变量初始化。

#### “读写”锁lock操作
		
		//获取“读”锁
	162 void qemu_co_rwlock_rdlock(CoRwlock *lock)
	163 {
	164     Coroutine *self = qemu_coroutine_self();
	165 
	166     while (lock->writer) { //如果已加“写”锁，则等待
	167         qemu_co_queue_wait(&lock->queue); //将当前Coroutine插入lock queue队列，并让出该协程上下文
	168     }
	169     lock->reader++;       //否则获取到"读"锁
	170     self->locks_held++;
	171 }

		//获取“写”锁
	192 void qemu_co_rwlock_wrlock(CoRwlock *lock)
	193 {
	194     Coroutine *self = qemu_coroutine_self();
	195 
	196     while (lock->writer || lock->reader) {//如果已加“写”锁或者"读"锁，则等待
	197         qemu_co_queue_wait(&lock->queue); //将当前Coroutine插入lock queue队列，并让出该协程上下文
	198     }
	199     lock->writer = true;                 //否则获取到“写锁”
	200     self->locks_held++;
	201 }

代码逻辑比较简单，基本上严格按照读写锁的原理来实现，如果已加“写”锁，则此时“读”/“写”锁均会失败，如果以加“读”锁，则申请“读”锁成功，申请“写”锁失败。


#### “读写”锁unlock操作
		
		//释放“读写”锁
	173 void qemu_co_rwlock_unlock(CoRwlock *lock)
	174 {
	175     Coroutine *self = qemu_coroutine_self();
	176 
	177     assert(qemu_in_coroutine());
	178     if (lock->writer) {    //如果释放的是"写"锁
	179         lock->writer = false;
	180         qemu_co_queue_restart_all(&lock->queue);//那么需要唤醒queue里面所有的Coroutine
	181     } else {                
	182         lock->reader--;
	183         assert(lock->reader >= 0);
	184         /* Wakeup only one waiting writer */
	185         if (!lock->reader) {                   //如果释放的是“读”锁,并且目前已经没有对象持有"读"锁，这时queue里面可能存在因为请求“写”锁而被让出的Coroutine，因此尝试唤醒一个Coroutine。
	186             qemu_co_queue_next(&lock->queue);
	187         }
	188     }
	189     self->locks_held--;
	190 }

这里"读"锁和"写"锁的unlock没有在函数上区分，而是在一个函数里面实现的，主要是因为根据当前锁的状态以及"读写"锁的特性可以推断出要unlock的是"读"锁还是"写"锁，另外186行是qemu\_co\_queue\_next而不是qemu\_co\_queue\_restart\_all,因为这里要唤醒的是因为请求“写”锁的Coroutine，而“写”锁只能被一个Coroutine持有。

最后总结一些qemu协程的实现首先是在UContext函数簇以及setjmp/longjmp的基础上实现了简单的协程创建、进入以及删除的功能，然后在此基础上实现了高级协程的功能包括Coroutine资源池以及Coroutine锁，其他功能模块直接调用高级协程功能函数来使用协程。


玩的开心 !!!
