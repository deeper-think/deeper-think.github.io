---
layout: post
title: KVM虚拟化组件编译及调试环境搭建
category: 计算虚拟化
tags: [KVM，qemu，编译，调试]
keywords: KVM，qemu，编译，调试
description: 
---

> 用正确的工具，做正确的事情

学习研究计算虚拟化的原理及实现方式最好的途经莫过于走读开源方案的代码。为了更快更准确的理解代码，运行并跟踪核心代码的执行流程是不错的一种方式。本文总结kvm计算虚拟化方案核心组件qemu及kvm 源码的获取、编译及跟踪调试的的方法。

## qemu组件

### qemu源码下载
qemu是kvm虚拟化方案的核心组件，kvm模块提供了实现虚拟化的核心功能包括虚拟机cpu管理、内存管理等 ，并对外提供功能调用的接口，但kvm并不负责完整虚拟机的创建及外围硬件设备的模拟，这些功能由用户态组件qemu来实现。

qemu项目主页wiki地址为：
	
	http://wiki.qemu.org/Main_Page

qemu项目主页上提供qemu稳定发布版本包的下载及版本changelog说明。

qemu最新代码托管在git仓库上，可以通过git获取最新的qemu代码以及最新的代码提交信息：

	http://git.qemu.org/qemu.git

### qemu源码编译

编译qemu源码生成可执行文件的过程很简单：

	./configure && make -j && make install

以上编译出来的qemu可执行文件是没有带debug信息的，如果想编译支持debug版本的qemu，在configure过程需要通过参数来指定：

	./configure --enable-debug

此时生成的qemu可执行文件既可以支持debug。qemu编译完成后，除了生成实现x86虚拟机创建和管理的核心文件:qemu-system-x86_64以外，还同时后生成其他的一些平台虚拟机创建和管理的实例以及一些工具，比如：qemu-img等。

### qemu源码调试

kvm虚拟机创建及启动后，在hypervisor上以qemu进程的形式存在，主qemu进程下会挂有多个子线程，其中包括最重要的vcpu线程，那么调试虚拟机也即调试qemu进程。

最简单的方式是找到qemu进程的pid，然后直接gdb调试：

	[root qemu-2.5.0-rc3]# ps -ef |grep qemu
	qemu     147674      1  0 03:17 ?        00:00:17 /usr/local/bin/qemu-system-x86_64 -name centos_7_0 -S -machine pc-i440fx-2.5,accel=kvm,usb=off -m 10240 -realtime mlock=off -smp 4,sockets=4,cores=1,threads=1 -uuid 9d4064de-339d-4d73-a519-8e786e1f8d43 -no-user-config -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/domain-centos_7_0/monitor.sock,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=localtime -no-shutdown -boot strict=on -device piix3-usb-uhci,id=usb,bus=pci.0,addr=0x1.0x2 -drive file=/home/vms/centos_7_0.img,if=none,id=drive-virtio-disk0,format=raw -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1 -drive file=/home/isos/CentOS_7_0.iso,if=none,id=drive-ide0-0-1,readonly=on,format=raw -device ide-cd,bus=ide.0,unit=1,drive=drive-ide0-0-1,id=ide0-0-1 -netdev tap,fd=23,id=hostnet0,vhost=on,vhostfd=24 -device virtio-net-pci,netdev=hostnet0,id=net0,mac=00:17:3e:7d:a1:a9,bus=pci.0,addr=0x3 -chardev pty,id=charserial0 -device isa-serial,chardev=charserial0,id=serial0 -vnc 0.0.0.0:0 -k en-us -device cirrus-vga,id=video0,bus=pci.0,addr=0x2 -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x5 -msg timestamp=on
	root     199105  92774  0 04:36 pts/3    00:00:00 grep --color=auto qemu
	[root qemu-2.5.0-rc3]# gdb -p 147674

详细的gdb调试过程及常用的调试命令如下：

	(gdb) b virtio.c:virtqueue_flush            \\设置一个debug断点
	Breakpoint 1 at 0x7f1f81e909b6: file /home/qemu-2.5.0-rc3/hw/virtio/virtio.c, line 296.
	(gdb) i b                                   \\打印当前已设置的断点
	Num     Type           Disp Enb Address            What
	1       breakpoint     keep y   0x00007f1f81e909b6 in virtqueue_flush at /home/qemu-2.5.0-rc3/hw/virtio/virtio.c:296
	(gdb) c                                     \\程序继续向下执行
	Continuing.
	[Thread 0x7f1cd10c1700 (LWP 199269) exited]
	
	[New Thread 0x7f1cd10c1700 (LWP 199283)]
	
	Breakpoint 1, virtqueue_flush (vq=0x7f1f850151b0, count=1) at /home/qemu-2.5.0-rc3/hw/virtio/virtio.c:296
	296         trace_virtqueue_flush(vq, count);
	(gdb)
	Continuing.
	
	Breakpoint 1, virtqueue_flush (vq=0x7f1f850151b0, count=1) at /home/qemu-2.5.0-rc3/hw/virtio/virtio.c:296
	296         trace_virtqueue_flush(vq, count);
	(gdb) list                                   \\打印当前执行 的程序上下文 
	291     void virtqueue_flush(VirtQueue *vq, unsigned int count)
	292     {
	293         uint16_t old, new;
	294         /* Make sure buffer is written before we update index. */
	295         smp_wmb();
	296         trace_virtqueue_flush(vq, count);
	297         old = vring_used_idx(vq);
	298         new = old + count;
	299         vring_used_idx_set(vq, new);
	300         vq->inuse -= count;
	(gdb) n                                       \\程序向下执行一条语句
	[Thread 0x7f1cd10c1700 (LWP 199283) exited]
	297         old = vring_used_idx(vq);
	(gdb) print old                               \\打印old变量的当前值
	$1 = 32543
	(gdb) n
	298         new = old + count;
	(gdb) n
	299         vring_used_idx_set(vq, new);
	(gdb) s                                       \\向下执行一条语句或者进入函数
	vring_used_idx_set (vq=0x7f1f850151b0, val=8249) at /home/qemu-2.5.0-rc3/hw/virtio/virtio.c:191
	191         pa = vq->vring.used + offsetof(VRingUsed, idx);
	(gdb) bt                                      \\打印当前堆栈的信息
	\#0  0x00007f1f81e90579 in vring_used_idx_set (vq=0x7f1f850151b0, val=8249) at /home/qemu-2.5.0-rc3/hw/virtio/virtio.c:191
	\#1  0x00007f1f81e909f8 in virtqueue_flush (vq=0x7f1f850151b0, count=1) at /home/qemu-2.5.0-rc3/hw/virtio/virtio.c:299	
	\#2  0x00007f1f81e90a80 in virtqueue_push (vq=0x7f1f850151b0, elem=0x7f1f84e31050, len=1) at /home/qemu-2.5.0-rc3/hw/virtio/virtio.c:309
	\#3  0x00007f1f81e5ab6b in virtio_blk_complete_request (req=0x7f1f84e31040, status=0 '\000') at /home/qemu-2.5.0-rc3/hw/block/virtio-blk.c:58
	\#4  0x00007f1f81e5abb6 in virtio_blk_req_complete (req=0x7f1f84e31040, status=0 '\000') at /home/qemu-2.5.0-rc3/hw/block/virtio-blk.c:64
	\#5  0x00007f1f81e5ad9f in virtio_blk_rw_complete (opaque=0x7f1f84e31040, ret=0) at /home/qemu-2.5.0-rc3/hw/block/virtio-blk.c:122
	\#6  0x00007f1f8216b696 in bdrv_co_complete (acb=0x7f1f84bda320) at block/io.c:2114
	\#7  0x00007f1f8216b812 in bdrv_co_do_rw (opaque=0x7f1f84bda320) at block/io.c:2153
	\#8  0x00007f1f821c8867 in coroutine_trampoline (i0=-2067939040, i1=32543) at util/coroutine-ucontext.c:80
	\#9  0x00007f1f7cf78110 in __start_context () at /lib64/libc.so.6
	\#10 0x00007fffb46a3ca0 in  ()
	\#11 0x0000000000000000 in  ()




## kvm模块

### kvm源码下载

kvm是一个内核模块，并且被合并入linux kernel 2.6.20以后的内核版本，因此kvm源代码的获取有两种方式：通过下载内核源码来获取kvm源代码或者下载当前kvm项目最新开发版本的代码。

最新的kernel源码可以kernel项目主页上下载到：

	https://www.kernel.org/

单独下载当前kvm 开发版本的代码：

	https://git.kernel.org/cgit/virt/kvm/kvm.git/


### kvm源码编译




玩的开心 !!!
