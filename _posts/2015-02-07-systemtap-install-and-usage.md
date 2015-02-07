---
layout: post
title: systemtap 安装及常用脚本
category: 工具
tags: [systemtap]
keywords: systemtap
description: 
---

> 我思故我在 -- 笛卡尔

systemtap基于kprobes，可以非常方便的进行内核跟踪、性能分析，是系统程序员、架构师必备的工具。下面是systemtap在centos环境的使用总结，同样适用于redhat环境，如果想在其他linux发行版使用systemtap，可能环境安装过程相对来说更加麻烦一些。

## systemtap环境安装
systemtap本身需要安装systemtap-runtime、systemtap，centos发行版yum源有这两个rpm，直接yum安装即可。 另外systemtap的使用还需要内核环境的支持，需要安装与内核匹配的三个rpm包：kernel-devel、kernel-debuginfo、kernel-debuginfo-common，kernel-devel可以直接通过yum安装源，非常不幸的是centos yum安装源里面找不到kernel-debuginfo、kernel-debuginfo-common这两个rpm包，不过可以从 -- [centos debuginfo](http://debuginfo.centos.org/)中找到这两个rpm包，直接下载下来手动安装即可，但是一定要注意这两个rpm包的版本号必须和内核版本号完全一致，包括小版本号，否则执行 systemtap脚本时非常有可能报错。

## systemtap脚本基础
stp脚本基本由两种元素组成：探针、句柄，当我们跟踪的事件发生时，将触发我们定义的探针，并执行为这个探针定义的句柄，探针的定义以probe 关键字开头，探针的句柄包括在一对“{” 、“}”中。stp脚本基本上由基本块：probe event {statements}组成。和shell脚本一样，stp脚本开头通过语句 ”#!/usr/bin/stap“ 指定解释器的路径。

### systemtap事件
systemtap事件分为同步事件和异步事件，执行到内核指定位置代码时触发同步事件，systemtap可以探测的同步事件包括：

1. 系统调用 syscall.system_call: 系统调用入口和exit处：syscall.system_call和syscall.system_call.return，比如对于close系统调用：syscall.close和syscall.close.return;
2. vfs.file_operation: vfs.file_operation和vfs.file_operation.return;
3. 内核函数 kernel.function("function"): kernel.function(“sys_open”)和kernel.function(“sys_open”).return,并且支持通配符,比如跟踪某个内核文件的所有函数：probe kernel.function("*@net/socket.c") { };
4. kernel.trace("tracepoint")：2.6.30及newer为内核中的特定事件定义了instrumentation，入kernel.trace(“kfree_skb”)代表内核中每次网络buffer被释放掉时的event。
5. 跟踪某个模块的函数module("module").function("function")：probe module("ext3").function("*") { }。

可以定义探针的异步事件包括：

1. begin：SystemTap session开始时触发，当SystemTap脚本开始运行时触发；
2. end：SystemTap session终止时触发；
3. timer事件：probe timer.s(4) { printf("hello world/n") }。

### systemtap句柄
句柄内部的语句和C语言语句类似，这里主要总结常用的systemtap 内置函数：

1. printf ("format string/n", arguments)，%s：字符串，%d数字，以 , 隔开；
2. tid()：当前线程ID；
3. uid()：当前用户ID；
4. cpu()：当前CPU号；
5. gettimeofday_s()：自从Epoch开始的秒数；
6. ctime()将从Unix Epoch开始的秒数转换成date；
7. pp()：描述当前被处理的探针点的字符串；
8. thread_indent(1)：进程相关信息；
9. name：标记系统调用的名字，仅用于syscall.system_call中；
10. target()：与stap script -x process ID or stap script -c command联合使用，如果想在脚本中获得进程ID或命令可以如此做。

## systemtap常见用法总结

### 创建探测模块并在本机或目标机运行
    stap -r `uname -r` -e 'probe vfs.read {exit()}' -m simple
    staprun simple.ko

### 查看探测点可用变量
    stap -L 'kernel.function("vfs_read")'

### 跟踪每一次pread系统调用的时延
    #!/usr/bin/stap
    
    probe begin
    {
        log("begin to probe")
    }

    probe syscall.pread.return
    {
        time = gettimeofday_us() - @entry(gettimeofday_us())
        printf ("%d\n", time)
    }
    
    probe timer.ms(60000) # after 10 minutes
    {
        exit ()
    }
    
    probe end
    {
        log("end to probe")
    }






enjoy the life !!!
