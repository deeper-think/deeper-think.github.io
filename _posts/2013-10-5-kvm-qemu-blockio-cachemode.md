---
layout: post
title:  kvm-qemu virtio分析：cache、io模式分析
category: KVM虚拟化
tags: [KVM，qemu]
keywords: KVM，qemu
description: 
---

> 用正确的工具，做正确的事情

kvm虚拟机内部块设备的IO最后都会落到hypervisor层对文件或者设备的读写， hypervisor层对文件或者设备的读写支持不同的 cache、io模式。如果是通过libvirt xml文件定义并启动虚拟机，通过cache选项来指定不同的cache模式：

	<disk type='file' device='disk'>
	    <driver name='qemu' type='raw' cache='directsync' io='native'/>//此处
		<source file='/home/vms/centos_6_8_ldd.img'/>
		<target dev='hda' bus='virtio'/>
	</disk>

如果是直接通过qemu命令来启动虚拟机，通过cache、io选项来指定不同的模式：

	-drive file=/home/vms/centos_6_8_ldd.img,format=raw,if=none,id=drive-virtio-disk0,cache=directsync,aio=native -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1

libvirt官方指导文档对qemu cache、io支持的模式以及每种模式的意义做了解释：

> The optional cache attribute controls the cache mechanism, possible values are "default", "none", "writethrough", "writeback", "directsync" (like "writethrough", but it bypasses the host page cache) and "unsafe" (host may cache all disk io, and sync requests from guest are ignored). Since 0.6.0, "directsync" since 0.9.5, "unsafe" since 0.9.7

>The optional io attribute controls specific policies on I/O; qemu guests support "threads" and "native". Since 0.8.8

从字面上里面存在一些模糊不清的地方，下面从代码的角度进行分析上面不同模式的区别。

## cache 模式分析

qemu后端打开block设备的镜像文件：

	(gdb) bt
	#0  qemu_open (name=0x7f72f93760e0 "/home/vms/centos_6_8_ldd.img", flags=16386) at util/osdep.c:206
	#1  0x00007f72f7190226 in raw_open_common (bs=0x7f72f9370af0, options=0x7f72f9375020, bdrv_flags=24738, open_flags=0, errp=0x7ffe770f2810)
    at block/raw-posix.c:443
	#2  0x00007f72f7190536 in raw_open (bs=0x7f72f9370af0, options=0x7f72f9375020, flags=24738, errp=0x7ffe770f2810) at block/raw-posix.c:538
	#3  0x00007f72f7136a57 in bdrv_open_common (bs=0x7f72f9370af0, file=0x0, options=0x7f72f9375020, errp=0x7ffe770f28d8) at block.c:1117
	#4  0x00007f72f71386de in bdrv_open_inherit (filename=0x7f72f9368e50 "/home/vms/centos_6_8_ldd.img", reference=0x0, options=0x7f72f9375020, flags=41090,
    parent=0x7f72f936a650, child_role=0x7f72f7a6a140 <child_file>, errp=0x7ffe770f2a28) at block.c:1846
	#5  0x00007f72f7137d9a in bdrv_open_child (filename=0x7f72f9368e50 "/home/vms/centos_6_8_ldd.img", options=0x7f72f936e8e0,
    bdref_key=0x7f72f766783a "file", parent=0x7f72f936a650, child_role=0x7f72f7a6a140 <child_file>, allow_none=true, errp=0x7ffe770f2a28) at block.c:1601
	#6  0x00007f72f713855c in bdrv_open_inherit (filename=0x7f72f9368e50 "/home/vms/centos_6_8_ldd.img", reference=0x0, options=0x7f72f936e8e0, flags=8322,
    parent=0x0, child_role=0x0, errp=0x7ffe770f2d18) at block.c:1807
	#7  0x00007f72f7138aef in bdrv_open (filename=0x7f72f9368e50 "/home/vms/centos_6_8_ldd.img", reference=0x0, options=0x7f72f9367d90, flags=128,
    errp=0x7ffe770f2d18) at block.c:1937
	#8  0x00007f72f71890c2 in blk_new_open (filename=0x7f72f9368e50 "/home/vms/centos_6_8_ldd.img", reference=0x0, options=0x7f72f9367d90, flags=128,
    errp=0x7ffe770f2d18) at block/block-backend.c:160
	#9  0x00007f72f6ed8b42 in blockdev_init (file=0x7f72f9368e50 "/home/vms/centos_6_8_ldd.img", bs_opts=0x7f72f9367d90, errp=0x7ffe770f2d18)
    at blockdev.c:582
	#10 0x00007f72f6ed9c72 in drive_new (all_opts=0x7f72f92eac60, block_default_type=IF_IDE) at blockdev.c:1080
	#11 0x00007f72f6ef1c7d in drive_init_func (opaque=0x7f72f9308e80, opts=0x7f72f92eac60, errp=0x0) at vl.c:1187
	#12 0x00007f72f7222db2 in qemu_opts_foreach (list=0x7f72f7a8f2a0 <qemu_drive_opts>, func=0x7f72f6ef1c4d <drive_init_func>, opaque=0x7f72f9308e80,
    errp=0x0) at util/qemu-option.c:1116
	#13 0x00007f72f6efa30f in main (argc=51, argv=0x7ffe770f3228, envp=0x7ffe770f33c8) at vl.c:4482


	#0  raw_open (bs=0x7efcaa5d5650, options=0x7efcaa5d98e0, flags=8354, errp=0x7fffe04a2790) at block/raw_bsd.c:414
	#1  0x00007efca80c3ad2 in bdrv_open_common (bs=0x7efcaa5d5650, file=0x7efcaa5e1190, options=0x7efcaa5d98e0, errp=0x7fffe04a2858) at block.c:1126
	#2  0x00007efca80c56de in bdrv_open_inherit (filename=0x7efcaa5d3e50 "/home/vms/centos_6_8_ldd.img", reference=0x0, options=0x7efcaa5d98e0, flags=8322,
    parent=0x0, child_role=0x0, errp=0x7fffe04a2b48) at block.c:1846
	#3  0x00007efca80c5aef in bdrv_open (filename=0x7efcaa5d3e50 "/home/vms/centos_6_8_ldd.img", reference=0x0, options=0x7efcaa5d2d90, flags=128,
    errp=0x7fffe04a2b48) at block.c:1937
	#4  0x00007efca81160c2 in blk_new_open (filename=0x7efcaa5d3e50 "/home/vms/centos_6_8_ldd.img", reference=0x0, options=0x7efcaa5d2d90, flags=128,
    errp=0x7fffe04a2b48) at block/block-backend.c:160
	#5  0x00007efca7e65b42 in blockdev_init (file=0x7efcaa5d3e50 "/home/vms/centos_6_8_ldd.img", bs_opts=0x7efcaa5d2d90, errp=0x7fffe04a2b48)
    at blockdev.c:582
	#6  0x00007efca7e66c72 in drive_new (all_opts=0x7efcaa555c60, block_default_type=IF_IDE) at blockdev.c:1080
	#7  0x00007efca7e7ec7d in drive_init_func (opaque=0x7efcaa573e80, opts=0x7efcaa555c60, errp=0x0) at vl.c:1187
	#8  0x00007efca81afdb2 in qemu_opts_foreach (list=0x7efca8a1c2a0 <qemu_drive_opts>, func=0x7efca7e7ec4d <drive_init_func>, opaque=0x7efcaa573e80,
    errp=0x0) at util/qemu-option.c:1116
	#9  0x00007efca7e8730f in main (argc=51, argv=0x7fffe04a3058, envp=0x7fffe04a31f8) at vl.c:4482




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

如果编译使用最新版本的kvm模块，需要升级内核到对应的最新版本，如果不想编译并升级整个内核，可以下载当前使用linux OS 发行版自带内核的源码包，然后单独编译kvm 模块并动态插入到当前运行的内核中即可。

对于centos 发行版其源码包的下载链接：

	http://vault.centos.org/6.6/updates/Source/SPackages/

### kvm源码编译及安装

这里选择下载当前运行OS自带内核对应的源码包，并单独编译kvm模块的方式。

首先是要先获取当前运行内核对应的源码，查看当前OS内核版本，并到centos官网下载对应的内核源码rpm包：

	[root@Centos7-1 home]# uname -r
	3.10.0-327.3.1.el7.x86_64

安装内核源码rpm包：

	[root@Centos7-1 home]# rpm -ivh kernel-3.10.0-327.3.1.el7.src.rpm
	Updating / installing...
   	1:kernel-3.10.0-327.3.1.el7        ################################# [100%]

然后/root/rpmbuild/SOURCES目录下会生成对应的内核源码压缩包：

	/root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7.tar.xz

解压该压缩包就可以得到当前运行内核的源码：

	tar -xvf linux-3.10.0-327.3.1.el7.tar.xz

接下来进入内核源码kvm模块的目录并清理编译环境：

	cd /root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7/arch/x86/kvm
	
	make clean CONFIG_KVM=m CONFIG_INTEL_KVM=m -C /root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7/ M=/root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7/arch/x86/kvm           //编译环境清理

单独编译kvm模块：

	make CONFIG_KVM=m CONFIG_INTEL_KVM=m -C /root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7/ M=/root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7/arch/x86/kvm          //编译模块   


编译报错：

	make: Entering directory `/root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7'
	
  	ERROR: Kernel configuration is invalid.
    	     include/generated/autoconf.h or include/config/auto.conf are missing.
   		     Run 'make oldconfig && make prepare' on kernel src to fix it.
	
	
  	WARNING: Symbol version dump /root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7/Module.symvers is missing; modules will have no dependencies and modversions.

  	LD      /root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7/arch/x86/kvm/built-in.o
  	CC [M]  /root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7/arch/x86/kvm/../../../virt/kvm/kvm_main.o
	In file included from <command-line>:0:0:
	/root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7/include/linux/kconfig.h:4:32: fatal error: generated/autoconf.h: No such file or directory
 	#include <generated/autoconf.h>
	                                ^
	compilation terminated.
	make[1]: *** [/root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7/arch/x86/kvm/../../../virt/kvm/kvm_main.o] Error 1
	make: *** [_module_/root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7/arch/x86/kvm] Error 2
	make: Leaving directory `/root/rpmbuild/SOURCES/linux-3.10.0-327.3.1.el7'

报错原因是没有构建内核源码树，解决的办法当然是先构建内核源码树，进入内核源码根目录并执行：
	
	make oldconfig && make -j

之后重新编译kvm模块就可以成功了。编译完成后，可以看到kvm模块源码目录下生成了几个ko文件：

	[root@Centos7-1 kvm]# ll | grep ko
	-rw-r--r-- 1 root root 1075656 Jan 19 22:08 kvm-amd.ko
	-rw-r--r-- 1 root root 1818303 Jan 19 22:08 kvm-intel.ko
	-rw-r--r-- 1 root root 9563429 Jan 19 22:08 kvm.ko

用编译出的ko模块替换之前的模块（模块路径：/usr/lib/modules/），然后重新生成依赖：

	depmod -a

然后重新加载自编译模块到内核即可：

	modprobe kvm-intel

通过上面的方法，可以在kvm源码中加入调试信息或者利用dump_stack打印函数调用堆栈，以此来调试kvm。


玩的开心 !!!
