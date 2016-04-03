---
layout: post
title: 硬件虚拟化技术浅析（转）
category: KVM虚拟化
tags: [kvm, qemu, 虚拟化]
keywords: kvm, qemu, 虚拟化
description: 
---

> 用正确的工具，做正确的事情

## 1. 硬件虚拟化技术背景

硬件虚拟化技术通过虚拟化指令集、MMU(Memory Map Unit)以及IO来运行不加修改的操作系统。

传统的处理器通过选择不同的运行(Ring 特权)模式，来选择指令集的范围，内存的寻址方式，中断发生方式等操作。在原有的Ring特权等级的基础上，处理器的硬件虚拟化技术带来了一个新的运行模式：Guest模式[1]，来实现指令集的虚拟化。当切换到Guest模式时，处理器提供了先前完整的特权等级，让Guest操作系统可以不加修改的运行在物理的处理器上。Guest与Host模式的处理器上下文完全由硬件进行保存与切换。此时，虚拟机监视器(Virtual Machine Monitor)通过一个位于内存的数据结构(Intel称为VMCS, AMD称为VMCB)来控制Guest系统同Host系统的交互，以完成整个平台的虚拟化。

传统的操作系统通过硬件MMU完成虚拟地址到物理地址的映射。在虚拟化环境中，Guest的虚拟地址需要更多一层的转换，才能放到地址总线上：

    Guest虚拟地址 -> Guest物理地址 -> Host物理地址
                       ^               ^
                       |               |
                      MMU1            MMU2

其中MMU1可以由软件模拟(Shadow paging中的vTLB)或者硬件实现(Intel EPT、AMD NPT)。MMU2由硬件提供。　

系统的IO虚拟化技术，通常是VMM捕捉Guest的IO请求，通过软件模拟的传统设备将其请求传递给物理设备。一些新的支持虚拟化技术的设备，通过硬件技术(如Intel VT-d)，可以将其直接分配给Guest操作系统，避免软件开销。

[1]X86处理器的生产厂商有自己的称谓，比如英特尔将Guest模式称为non-root operation，与之相对的是root operation，本文称为host模式。

## 2.KVM的内部实现概述

KVM是Linux内核的一个模块，基于硬件虚拟化技术实现VMM的功能。该模块的工作主要是通过操作与处理器共享的数据结构来实现指令集以及MMU的虚拟化，捕捉Guest的IO指令(包括Port IO和mmap IO)以及实现中断虚拟化。至于IO设备的软件模拟，是通过用户程序QEMU来实现的。QEMU负责解释IO指令流，并将其请求换成系统调用或者库函数传给Host操作系统，让Host上的驱动去完成真正的IO操作。她们之间的关系如下图所示：

    +--------------+               +--------+    
    | Qemu         |               |        |   
    |              |               |        |               
    | +---+  +----+|               | Guest  |               
    | |vHD|  |vNIC||<-----+        |        |               
    | +---+  +----+|      |        |        |               
    +--------------+      |        +--------+               
         ^                |            ^                    
         | syscall        |IO stream   |                    
         | via FDs        |            |                      
    +----|----------------|------------|--------+             
    |    |                |            v        |             
    |    v                |       +----------+  |   
    |  +--------+         +------>|          |  |
    |  |drivers |<--+             |  kvm.ko  |  |
    |  +--------+   |             +----------+  |
    |    ^          |   Host kernel             |   
    +----|----------|---------------------------+
         v          v                           
    +--------+    +---+                    
    | HDD    |    |NIC|                    
    +--------+    +---+

从Host操作系统的角度来看，KVM Guest操作系统相当于一个进程运行在系统上，普通的命令如kill、top、taskset等可以作用于该Guest。该进程的用户虚拟空间就是Guest的物理空间，该进程的线程对应着Guest的处理器。

从Qemu的角度来看，KVM模块抽象出了三个对象，她们分别代表KVM自己，Guest的虚拟空间以(VM)及运行虚拟处理器(VCPU)。这三个对象分别对应着三个文件描述符，Qemu通过文件描述符用系统调用IOCTL来操作这三个对象，同KVM交互。此时，Qemu主要只模拟设备，她以前的CPU和MMU的模拟逻辑都被kvm.ko取代了。

### 2.1 KVM的抽象对象

KVM同应用程序(Qemu)的交互接口为/dev/kvm，通过open以及ioctl系统调用可以获取并操作KVM抽象出来的三个对象，Guest的虚拟处理器(fd_vcpu[N]), Guest的地址空间(fd_vm), KVM本身(fd_kvm)。其中每一个Guest可以含有多个vcpu，每一个vcpu对应着Host系统上的一个线程。

Qemu启动Guest系统时，通过/dev/kvm获取fd_kvm和fd_vm，然后通过fd_vm将Guest的“物理空间”mmap到Qemu进程的虚拟空间，并根据配置信息创建vcpu[N]线程，返回fd_vcpu[N]。然后Qemu将操作fd_vcpu在其自己的进程空间mmap一块KVM的数据结构区域。该数据结构(下图中的shared)用于同kvm.ko交互，包含Guest的IO信息，如端口号，读写方向，内存地址等。Qemu通过这些信息，调用虚拟设备注册的回调函数来模拟设备的行为，并将Guest IO请求换成系统请求发送给Host系统。由于Guest的地址空间已经映射到Qemu的进程空间里面，Qemu的虚拟设备逻辑可以很方便的存取Guest地址空间里面的数据。三个对象之间的关系如下图所示：

    +----------+            |         +--------+
    | Qemu     | Host user  |         |        |
    |          |            |         |        |
    |          |            |         | Guest  |
    |  +------+|            |         | user   |
    |  |shared||            |         |        |
    |  +------+|            |         |        |
    |       ^  |            |         |        |
    +-------|--+            |         |        |
        |   |               |         |        |
     fds|   |               |         |        |
    ----|---|---------------|         |--------|
        |   |               |         |        |
        v   v   Host kernel |         | Guest  |
     +---------+            |         | kernel |
     |         |            |         |        |
     |  kvm.ko |----+       |         |        |
     |         |    |fd_kvm |         |        |
     +---------+    |       |         +--------+
                    v                     ^
                  +----+       fd_vm      |
                  |vmcs|----+--------------
      +------+    +----+    |          +------+
      | host |              |          | Guest|
      | mode |              |fd_vcpu   | mode |       
      +------+              |          +------+
          ^                 v             ^
          |             +-------+         |
          | vm exit     |  Phy  | vm entry|
          +-------------|  CPU  |---------+
                        +-------+


图中vm-exit代表处理器进入host模式，执行kvm和Qemu的逻辑。vm-entry代表处理器进入Guest模式，执行整个Guest系统的逻辑。如图所示，Qemu通过三个文件描述符同kvm.ko交互，然后kvm.ko通过vmcs这个数据结构同处理器交互，最终达到控制Guest系统的效果。其中fd_kvm主要用于Qemu同KVM本身的交互，比如获取KVM的版本号，创建地址空间、vcpu等。fd_vcpu主要用于控制处理器的模式切换，设置进入Guest mode前的处理器状态等等(内存寻址模式，段寄存器、控制寄存器、指令指针等)，同时Qemu需要通过fd_vcpu来mmap一块KVM的数据结构区域。fd_vm主要用于Qemu控制Guest的地址空间，向Guest注入虚拟中断等。

### 2.2 KVM的vcpu

如前文所述，KVM的vcpu对应着host系统上的一个线程。从Qemu的角度来看，她运行在一个loop中：

    for (;;) {
        kvm_run(vcpu);
        switch (shared_data->exit_reason) {
        ...
        case KVM_IO: 
            handle_io(vcpu);
            break;
        case KVM_MMIO:
            handle_mmio(vcpu);
            break;
        ...
        }
    }

该线程同Guest的vcpu紧密相连。如果我们把线程的执行看作Guest vcpu的一部分，那么从Host的角度来看，该vcpu在三种不同的上下文中运行：Host user/Host kernel/Guest，将运行于一个更大的循环当中。该vcpu的运行逻辑如下图：

          Host user   |  Host kernel  | Guest mode   |
                      |               |              |
                      |               |              |
     +->kvm_run(vcpu)-------+         |              | 
     |                |     v         |              |
     |                | +->vm entry----------+       |     
     |                | |             |      v       |
     |                | |             |   Execute    |
     |                | |             |   Natively   |
     |                | |             |      |       |
     |                | |  vm exit<----------+       |  
     |                | |    |        |              |  
     |                | |    |        |              |
     |            Yes | |    v        |              |   
     |     +----------------I/O ?     |              |    
     |     |          | |    | No     |              |
     |     |          | |    |        |              |      
     |     |          | |    v        |              |
     |     v      Yes | |  Signal     |              |
     +--Handle IO<---------Pending?   |              |
                      | |    | No     |              |
                      | +----+        |              |

实际上，在host上通过ps命令看到的关于vcpu这个线程的运行时间正是这三种上下文的总和。

### 2.3 KVM的IO虚拟化

#### 2.3.1 IO的虚拟化

传统系统中，设备都直接或间接的挂在PCI总线上。PCI设备通过PCI配置空间以及设备地址空间接收操作系统的驱动请求和命令，通过中断机制通知反馈操作系统。配置空间和设备地址空间都将映射到处理器Port空间或者操作系统内存空间中，所以设备的软件模拟需要VMM将相关的Guest PIO和MMIO请求截获，通过硬件虚拟化提供的机制将其传送给软件。模拟软件处理完后再通过VMM提供的虚拟中断机制反馈Guest。如下图所示：

            +-----------------------------------+
            | +--------------+                  |   
            | | PCI config   |     +----------+ |
            | +--------------+<--->| driver   | | 
            | +--------------+<--->|          | | 
            | | Device memory|     +----------+ |
            | +--------------+           ^      |   
            |       ^                    |      |   
            +-------|--------------------|------+  
                    |                    | vINTR via VMM    
    PIO/MMIO via VMM|         +----------+           
                    v         |                       
             +------------------------+    
             | +--------+  +--------+ |    
             | |  PCI   |  | Device | |    
             | | config |  | memory | |  Virtual Device    
             | +--------+  +--------+ |              
             +------------------------+    
                          |   
                          v   
                    +------------+
                    |host driver |
                    +------------+

虚拟设备的软件逻辑放在用户层也可以放在内核中。完全的虚拟设备模拟，可以处理在Guest中不加修改的驱动请求。通常这将消耗大量的处理器cycle去模拟设备。如果可以修改或者重写Guest的驱动代码，那么虚拟设备和驱动之间的IO接口可以根据虚拟化的特性重新定义为更高层更加高效的接口，如下图所示：

        +----------------+
        |                |
        | +-----------+  |
        | |para-driver|  |
        | +-----------+  |
        +-------^--------+
                |         
                | new I/O interface via VMM
                v
            +---------+                      
            |Virtual  |                       
            |device   |                       
            +---------+                       
                |                             
                v
           +------------+
           |host driver |
           +------------+

KVM的virtio正是通过这种方式提供了高速IO通道。

除了软件模拟，现有的硬件虚拟化技术还可以将一些支持虚拟化技术的新兴硬件直接分配给Guest。除了需要支持虚拟化技术的硬件(可以发起remmappable的MSI中断请求)，设备的直接分配一般还需要主板上的芯片以及CPU支持，比如英特尔的VT-d技术。支持虚拟化技术的硬件平台主要做两件事，一个是DMA Remapping，将DMA请求中的Guest的物理地址映射到Host的物理地址，另一个是中断Remapping，将能remappable的中断请求根据由VMM设置，位于内存的IRT(Interrupt Remapping Table)发送到指定的vcpu上。

PC平台上，通常北桥(或者类似结构的root-complex)连接着CPU、内存以及外设。用于DMA Remapping和中断Remapping的硬件逻辑位于北桥中。如下所示：

          +-------------+
          |cpu0, cpu1...|              
          +-------------+    
                ^    
                |        <-- System Bus  
                |                |   
                v                v   
       +---------------------+  
       |  North Bridge       |   
       |                     |       +--------+
       |    +--------+       |<----->| Memory |
       |    |  vt-d  |       |       +--------+
       |    +--------+       |   
       +---------------------+   
             ^            ^          
             |            |   
             v            v      
        +--------+    +--------+
        | PCI-e  |    | South  |<-----> PCI legacy devices...
        | device |    | Bridge |
        +--------+    +--------+

目前，只有支持MSI的PCI/PCI-e设备才能直接分配给Guest。其中PCI-e设备可以直接与北桥相连或者桥连，然后单独分配给一个Guest。在一个桥后的所有的桥连PCI设备只能作为一个整体分配给一个Guest。KVM在硬件虚拟化的平台上支持PCI-e/PCI设备的直接分配。

#### 2.3.2 VirtIO

VirtIO为Guest和Qemu提供了高速的IO通道。Guest的磁盘和网络都是通过VirtIO来实现数据传输的。由于Guest的地址空间mmap到Qemu的进程空间中，VirtIO以共享内存的数据传输方式以及半虚拟化(para-virtualized)接口为Guest提供了高效的硬盘以及网络IO性能。其中，KVM为VirtIO设备与Guest的VirtIO驱动提供消息通知机制，如下图所示：

         +---------------+
         |  Qemu         |
         |    +--------+ |        +-------------------+
         |    | VirtIO | |        | +---------+       |
         |    | Device | |        | | VirtIO  | Guest |
         |    +--------+ |        | | Driver  |       |  
         +------|--^-----+        | +---------+       |  
                |  |              +---|---^-----------+  
          irqfd |  |              PIO |   |               
          fd_vm |  |ioeventfd         |   |vInterrupt             
       ---------|--|------------------|---|------------
                v  |                  v   |
            +----------+         +--------------+ Host
            | eventfd  |<------->|  KVM.ko      | kernel
            | core     |         |              |
            +----------+         +--------------+

如图所示，Guest VirtIO驱动通过访问port空间向Qemu的VirtIO设备发送IO发起消息。而设备通过读写irqfd或者IOCTL fd_vm通知Guest驱动IO完成情况。irqfd和ioeventfd是KVM为用户程序基于内核eventfd机制提供的通知机制，以实现异步的IO处理(这样发起IO请求的vcpu将不会阻塞)。之所以使用PIO而不是MMIO，是因为：KVM处理PIO的速度快于MMIO。

## 3 KVM-IO可能优化地方

### 3.1 Virt-IO的硬盘优化

从图1中可以看到，Guest的IO请求需要经过Qemu处理后通过系统调用才会转换成Host的IO请求发送给Host的驱动。虽然共享内存以及半虚拟化接口的通信协议减轻了IO虚拟化的开销，但是Qemu与内核之间的系统模式切换带来的开销是避免不了的。

目前Linux内核社区中的vhost就是将用户态的Virt-IO网络设备放在了内核中，避免系统模式切换以及简化算法逻辑最终达到IO减少延迟以及增大吞吐量的目的。如下图所示：

                                 +-------------------+
                                 | +---------+       |
                                 | | VirtIO  | Guest |
                                 | | Driver  |       |  
                                 | +-----+---+       |  
                                 +---|---^-----------+  
                                 PIO |   |               
                                     |   | vInterrupt             
       ------------------------------|---|--------------
                                     v   |
            +----------+         +--------------+  Host
            | Vhost    |<------->|  KVM.ko      |  kernel
            | net      |         |              |
            +----^-----+         +--------------+
                 |               
                 |               
             +---v----+          
             | NIC    |          
             | Driver |          
             +--------+

目前KVM的磁盘虚拟化还是在用户层通过Qemu模拟设备。我们可以通过vhost框架将磁盘的设备模拟放到内核中达到优化的效果。

### 3.2 普通设备的直接分配(Direct Assign)

如前文所述，目前只有特殊的PCI设备才能直接分配给相应的Guest，即VMM-bypass，避免额外的软件开销。我们可以在KVM中软实现DMA以及中断的remapping功能，然后将现有的普通设备直接分配给Guest。如下图所示：

                   +----------------+
                   |  Guest         |
                   |  +---------+   |
         +-------->|  | Driver  |   |
         |         |  +---------+   |
         |         +------------^---+
       D |              |       |    
       M |      DMA Req.|       | vINTR
       A |              |       |
         |      +-------|-------|----------+   
       O |      |       v KVM   |          |  
       p |      |   +------------------+   |  
       e |      |   | DMA remmapping   |   |              
       r |      |   |                  |   |  
       a |      |   | INTR remmapping  |   |              
       t |      |   +-----------^------+   |  
       i |      +-------|-------|----------+  
       o |              |       | INTR        
       n |              v       |             
         |              +---------+           
         +------------->| Deivce  |           
                        +---------+

这将大大减少Guest驱动同物理设备之间的路径(省去了KVM的涉入)，去掉了虚拟设备的模拟逻辑，不过IO性能的提高是以增加KVM的逻辑复杂度的代价换来的。此时，IO的性能瓶颈从Qemu/KVM转移到物理设备，但是IO的稳定性、安全性将会更加依赖于KVM的remapping逻辑实现。

### 3.3 普通设备的复用

在普通设备的直接分配的基础上，我们甚至可以在多个Guest之间复用设备，好比m个进程跑在n个处理器上一样(n < m)。比如将一个硬盘分成多个区，每一个分区作为一个块设备直接分配给Guest;或者直接将n个网卡分配给m个Guest(n < m)。其中磁盘的复用，只需在KVM中添加分区管理的逻辑，而网卡的复用则要复杂一些：KVM需要为设备提供多个设备上下文(每一个设备上下文对应着一个Guest)，同时还需要提供算法逻辑对设备上下文进行切换和调度。如下图所示：

                            |                  KVM   |
                            |  Device context        |
                            |  queue                 |
               +------+     |     +-+                |
               |Guest |---------->| |                |
               -------+     |     +-+                |
                            |      |                 |
               +------+     |     +-+                |
               |Guest |---------->| |   +----------+ |
               +------+     |     +-+   | Device   | |
                            |      |    | Scheduler| |
               +------+     |     +-+   +----------+ |
               |Guest |---------->| |-----+          |
               +------+     |     +-+     |          |
                            |          +--v--------+ |
                            | Current--->+--+  DM  | |     +-----+
                            | Context  | +--+------------->| NIC |
                            |          +-----------+ |     +-----+
                            |                        |

其中，Device Modle(DM)实现前文提到的remapping逻辑，Device Scheduler用于选择和切换设备上下文实现物理设备的复用。在普通设备直接分配的基础上，通过对现有普通设备的复用，将会带来廉价、灵活、高效的IO性能。与之相对的是，目前已经有支持SR-IOV的网卡，从硬件上实现复用功能，支持多个(静态，即最大数目是内置的)虚拟的PCI网卡设备，价格昂贵，且受到一个网口总带宽有限的限制(软件复用技术，可以复用多个网卡，进而提高系统总的带宽)。

## 参考：

1[代码] qemu-kvm. git://git.kernel.org/pub/scm/virt/kvm/qemu-kvm.git

2[代码] Linux-2.6{/virt/kvm/*, arch/x86/kvm/*, drivers/virtio/*, drivers/block/virtio_blk.c, drivers/vhost/*}

3[手册] Intel® Virtualization Technology for Directed I/O

4[手册] Intel® 64 and IA-32 Architectures Software Developer’s Manual 3B.

5[论文] Towards Virtual Passthrough I/O on Commodity Devices. 2008.

6[论文] kvm: the Linux Virtual Machine Monitor. 2007.

7[论文] virtio: Towards a De-Facto Standard For Virtual I/O Devices. 2008

8[论文] High Performance Network Virtualization with SR-IOV. 2010.

9[论文] QEMU, a Fast and Portable Dynamic Translator. 2005.

## 转自：
[淘宝核心系统团队博客](http://rdc.taobao.com/blog/cs/?p=1425)

玩的开心 !!!
