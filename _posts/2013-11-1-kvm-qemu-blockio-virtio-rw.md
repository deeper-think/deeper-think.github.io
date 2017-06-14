---
layout: post
title: kvm block IO 03：qemu virtio 读写流程分析
category: KVM虚拟化
tags: [KVM，qemu]
keywords: KVM，qemu
description: 
---

> 用正确的工具，做正确的事情

## 总体读写流程概括

总体的读写流程概括，

> virtIO 读写过程主要分为前半部和后半部，前半部主要是提交并完成IO请求，后半部主要包括vring数据结构的更新以及一些notify的工作。前半部针对每一个IO请求创建一个协程，在协程内部提交IO，但进程并不会阻塞并等待IO的完成，如果IO没有完成则进程会暂时退出当前协程并继续后续的处理工作，直到IO完成会再次进入本次IO的协程，进而之前被调用的函数依次返回，表示本次IO的完成。前半部单纯完成IO的读写，IO读写完成后其他需要做的工作包括ving数据结构的更新以及notify的工作，都是在后半部完成。

## virtIO 读写流程前半部

进入协程之前：

	#0  blk_aio_prwv (blk=0x7fa84c782480, offset=9325244416, bytes=8192, qiov=0x7fa84dd488f0, co_entry=0x7fa84b2a0bbf <blk_aio_write_entry>, flags=0, 
    	cb=0x7fa84aefc3de <virtio_blk_rw_complete>, opaque=0x7fa84dd48890) at block/block-backend.c:996
	#1  0x00007fa84b2a0eee in blk_aio_pwritev (blk=0x7fa84c782480, offset=9325244416, qiov=0x7fa84dd488f0, flags=0, cb=0x7fa84aefc3de <virtio_blk_rw_complete>, opaque=0x7fa84dd48890)
    	at block/block-backend.c:1109
	#2  0x00007fa84aefccdb in submit_requests (blk=0x7fa84c782480, mrb=0x7fff73b42ee0, start=0, num_reqs=1, niov=-1) at /home/qemu-2.8.0/hw/block/virtio-blk.c:358
	#3  0x00007fa84aefcde9 in virtio_blk_submit_multireq (blk=0x7fa84c782480, mrb=0x7fff73b42ee0) at /home/qemu-2.8.0/hw/block/virtio-blk.c:391
	#4  0x00007fa84aefd82a in virtio_blk_handle_vq (s=0x7fa84c763cd0, vq=0x7fa84db91800) at /home/qemu-2.8.0/hw/block/virtio-blk.c:600
	#5  0x00007fa84aeff200 in virtio_blk_data_plane_handle_output (vdev=0x7fa84c763cd0, vq=0x7fa84db91800) at /home/qemu-2.8.0/hw/block/dataplane/virtio-blk.c:158
	#6  0x00007fa84af42584 in virtio_queue_notify_aio_vq (vq=0x7fa84db91800) at /home/qemu-2.8.0/hw/virtio/virtio.c:1243
	#7  0x00007fa84af44795 in virtio_queue_host_notifier_aio_read (n=0x7fa84db91860) at /home/qemu-2.8.0/hw/virtio/virtio.c:2046
	#8  0x00007fa84b25747a in aio_dispatch (ctx=0x7fa84c757cc0) at aio-posix.c:325
	#9  0x00007fa84b249696 in aio_ctx_dispatch (source=0x7fa84c757cc0, callback=0x0, user_data=0x0) at async.c:254
	#10 0x00007fa849f50d7a in g_main_context_dispatch () from /usr/lib64/libglib-2.0.so.0
	#11 0x00007fa84b2554ad in glib_pollfds_poll () at main-loop.c:215
	#12 0x00007fa84b255596 in os_host_main_loop_wait (timeout=708859108) at main-loop.c:260
	#13 0x00007fa84b255646 in main_loop_wait (nonblocking=0) at main-loop.c:508
	#14 0x00007fa84b0091e9 in main_loop () at vl.c:1967
	#15 0x00007fa84b01098a in main (argc=51, argv=0x7fff73b43628, envp=0x7fff73b437c8) at vl.c:4686

blk\_aio\_prwv函数创建协程并进入协程，并通过上下文切换进入协程入口函数blk\_aio\_write\_entry，blk\_aio\_prwv函数的详细实现：

	988 static BlockAIOCB *blk_aio_prwv(BlockBackend *blk, int64_t offset, int bytes,
	989                                 QEMUIOVector *qiov, CoroutineEntry co_entry,
	990                                 BdrvRequestFlags flags,
	991                                 BlockCompletionFunc *cb, void *opaque)
	992 {
	993     BlkAioEmAIOCB *acb;
	994     Coroutine *co;
	995 
	996     bdrv_inc_in_flight(blk_bs(blk));
	997     acb = blk_aio_get(&blk_aio_em_aiocb_info, blk, cb, opaque);
	998     acb->rwco = (BlkRwCo) {
	999         .blk    = blk,
	1000         .offset = offset,
	1001         .qiov   = qiov,
	1002         .flags  = flags,
	1003         .ret    = NOT_DONE,
	1004     };
	1005     acb->bytes = bytes;
	1006     acb->has_returned = false;
	1007 
	1008     co = qemu_coroutine_create(co_entry, acb);
	1009     qemu_coroutine_enter(co);
	1010 
	1011     acb->has_returned = true;
	1012     if (acb->rwco.ret != NOT_DONE) {
	1013         aio_bh_schedule_oneshot(blk_get_aio_context(blk),
	1014                                 blk_aio_complete_bh, acb);
	1015     }
	1016 
	1017     return &acb->common;
	1018 }

在协程中，通过上下文切换调用协程入口函数：
	
	(gdb) bt
	#0  raw_co_prw (bs=0x7fa84c788af0, offset=9325252608, bytes=4096, qiov=0x7fa84dad5ea0, type=2) at block/raw-posix.c:1253
	#1  0x00007fa84b2a7c49 in raw_co_pwritev (bs=0x7fa84c788af0, offset=9325252608, bytes=4096, qiov=0x7fa84dad5ea0, flags=0) at block/raw-posix.c:1291
	#2  0x00007fa84b2b1252 in bdrv_driver_pwritev (bs=0x7fa84c788af0, offset=9325252608, bytes=4096, qiov=0x7fa84dad5ea0, flags=0) at block/io.c:875
	#3  0x00007fa84b2b2800 in bdrv_aligned_pwritev (bs=0x7fa84c788af0, req=0x7fa5a1c9cba0, offset=9325252608, bytes=4096, align=512, qiov=0x7fa84dad5ea0, flags=0) at block/io.c:1360
	#4  0x00007fa84b2b34b0 in bdrv_co_pwritev (child=0x7fa84c78e190, offset=9325252608, bytes=4096, qiov=0x7fa84dad5ea0, flags=0) at block/io.c:1610
	#5  0x00007fa84b25d1a7 in raw_co_pwritev (bs=0x7fa84c782650, offset=9325252608, bytes=4096, qiov=0x7fa84dad5ea0, flags=0) at block/raw_bsd.c:243
	#6  0x00007fa84b2b1252 in bdrv_driver_pwritev (bs=0x7fa84c782650, offset=9325252608, bytes=4096, qiov=0x7fa84dad5ea0, flags=0) at block/io.c:875
	#7  0x00007fa84b2b2800 in bdrv_aligned_pwritev (bs=0x7fa84c782650, req=0x7fa5a1c9ceb0, offset=9325252608, bytes=4096, align=1, qiov=0x7fa84dad5ea0, flags=0) at block/io.c:1360
	#8  0x00007fa84b2b34b0 in bdrv_co_pwritev (child=0x7fa84c78e1e0, offset=9325252608, bytes=4096, qiov=0x7fa84dad5ea0, flags=0) at block/io.c:1610
	#9  0x00007fa84b2a04fe in blk_co_pwritev (blk=0x7fa84c782480, offset=9325252608, bytes=4096, qiov=0x7fa84dad5ea0, flags=0) at block/block-backend.c:849
	#10 0x00007fa84b2a0c57 in blk_aio_write_entry (opaque=0x7fa84cf10ca0) at block/block-backend.c:1037
	#11 0x00007fa84b33dba1 in coroutine_trampoline (i0=1297928528, i1=32680) at util/coroutine-ucontext.c:79
	#12 0x00007fa848f47cf0 in ?? () from /usr/lib64/libc.so.6
	#13 0x00007fff73b425b0 in ?? ()
	#14 0x0000000000000000 in ?? ()

raw\_co\_prw函数的详细实现如下：

	1250 static int coroutine_fn raw_co_prw(BlockDriverState *bs, uint64_t offset,
	1251                                    uint64_t bytes, QEMUIOVector *qiov, int type)
	1252 {
	1253     BDRVRawState *s = bs->opaque;
	1254 
	1255     if (fd_open(bs) < 0)
	1256         return -EIO;
	1257 
	1258     /*
	1259      * Check if the underlying device requires requests to be aligned,
	1260      * and if the request we are trying to submit is aligned or not.
	1261      * If this is the case tell the low-level driver that it needs
	1262      * to copy the buffer.
	1263      */
	1264     if (s->needs_alignment) {
	1265         if (!bdrv_qiov_is_aligned(bs, qiov)) {
	1266             type |= QEMU_AIO_MISALIGNED;
	1267 #ifdef CONFIG_LINUX_AIO
	1268         } else if (s->use_linux_aio) {
	1269             LinuxAioState *aio = aio_get_linux_aio(bdrv_get_aio_context(bs));
	1270             assert(qiov->size == bytes);
	1271             return laio_co_submit(bs, aio, s->fd, offset, qiov, type);
	1272 #endif
	1273         }
	1274     }
	1275 
	1276     return paio_submit_co(bs, s->fd, offset, qiov, bytes, type);
	1277 }

在raw\_co\_prw函数内，将根据虚拟机的配置以及系统是否支撑linux\_aio来决定最后的读写方式是通过linux 异步aio还是通过线程池的方式，如果是异步aio的话，进入laio\_co\_submit，如果是线程池的话 进入paio\_submit\_co，不论是那种方式都不会因为等待io的完成而阻塞，我这里的环境和创建虚拟机的配置使用的是linux\_aio的方式，因此这里重点分析laio\_co\_submit的实现，具体实现如下：

	392 int coroutine_fn laio_co_submit(BlockDriverState *bs, LinuxAioState *s, int fd,
	393                                 uint64_t offset, QEMUIOVector *qiov, int type)
	394 {
	395     int ret;
	396     struct qemu_laiocb laiocb = {
	397         .co         = qemu_coroutine_self(),
	398         .nbytes     = qiov->size,
	399         .ctx        = s,
	400         .ret        = -EINPROGRESS,
	401         .is_read    = (type == QEMU_AIO_READ),
	402         .qiov       = qiov,
	403     };
	404 
	405     ret = laio_do_submit(fd, &laiocb, offset, type);
	406     if (ret < 0) {
	407         return ret;
	408     }
	409 
	410     if (laiocb.ret == -EINPROGRESS) {
	411         qemu_coroutine_yield();
	412     }
	413     return laiocb.ret;
	414 }

396~405行创建laiocb结构变量并将QEMUIOVector qiov封装到该结构变量中，然后调用laio\_do\_submit来提交IO，laio\_do\_submit函数的返回并不一定该IO已经完成，有可能只是将laiocb插入到LinuxAioState结构变量的io\_q.pending队列里面；410~412行，如果IO还没有完成，则先退出协程，也即退回到blk\_aio\_prwv函数的1011行。从上面的过程可以看出，

> qemu每次提交IO都会创建一个协程，协程内完成IO的提交，但是协程内并不会阻塞来等待IO的完成，如果IO没有完成，处于EINPROGRESS状态，该协程会先从本协程内退出，等该IO完成后会再次进入该协程。




## virtio 读写流程后半部



	



玩的开心 !!!
