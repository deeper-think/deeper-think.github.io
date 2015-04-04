---
layout: post
title: TCP连接的一些思考及验证
category: 技术
tags: [TCP连接]
keywords: TCP连接
description: 
---

> 用正确的工具，做正确的事情

写过服务器端程序的人应该都知道listen函数的第二个传入参数 LISTENQU，这个参数的含义是：

> 处于listen状态的TCP服务器端有一个连接缓冲队列，TCP服务器会将TCP三次握手完成之后的连接放在连接缓冲队列里面（该队列的长度和bind时的第三个参数有关系，具体量上的关系一直没有搞清楚，不同的实现不一样），等待服务器进程通过accept动作来取走已完成的连接。 那么如果服务器进程的执行过程被阻塞，accept的速度小于TCP连接完成的速度，那么连接缓冲队列会被填满。那么现在的问题是：

## 如果服务器端连接缓冲队列被填满，服务器端怎么处理新的连接请求呢？

为了搞清楚上面的问题，写了一个服务器端程序，bind后不执行accept动作，循环等待客户端连接的到达，LISTENQU为3，程序代码：
    int
    main(int argc, char **argv)
    {
        int                    listenfd, connfd;
        pid_t                childpid;
        socklen_t            clilen;
        struct sockaddr_in    cliaddr, servaddr;
		 
        listenfd = socket(AF_INET, SOCK_STREAM, 0);
    
        bzero(&servaddr, sizeof(servaddr));
        servaddr.sin_family      = AF_INET;
        servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
        servaddr.sin_port        = htons(SERV_PORT);
    
        bind(listenfd, (SA *) &servaddr, sizeof(servaddr));
    
        listen(listenfd, LISTENQ);
        printf("listen but not accept, buffer the connection in queue:\n");
        for ( ; ; ) {
            sleep(5);
            printf(".");
        }
        printf("\n");
        return 0;
    }

客户端连接请求的程序：

    int main(int argc, char **argv)
    {
        int                    sockfd;
        struct sockaddr_in    servaddr;
        unsigned int n = 0;
        if (argc != 2)
        {
            printf("%s\n","usage: tcpcli <IPaddress>");
            exit(0);
        }
    
        while(1) {
    
            sockfd = socket(AF_INET, SOCK_STREAM, 0); /* 创建连接sock */
    
            bzero(&servaddr, sizeof(servaddr)); /* 初始化服务器端套接口地址 */
            servaddr.sin_family = AF_INET;
            servaddr.sin_port = htons(8888);
            inet_pton(AF_INET, argv[1], &servaddr.sin_addr);
            printf("start the %dth connection  %d\n", n);
            if(0 != connect(sockfd, (SA *) &servaddr, sizeof(servaddr)))
            {
                printf("connect error\n"); /* 连接至服务器端 */
                return 0;
            }
            sleep(3);
            printf("done the %dth connection and the sockfd %d\n", n, sockfd);
            n++;
        }
    
        exit(0);
    }

执行结果：
    
    Every 1.0s: netstat -tan | grep 8888| grep ESTABLISHED                Wed Jan  8 02:12:24 2014
    
    tcp        0      0 192.168.206.129:8888        192.168.206.128:42977       ESTABLISHED
    tcp        0      0 192.168.206.129:8888        192.168.206.128:42975       ESTABLISHED
    tcp        0      0 192.168.206.129:8888        192.168.206.128:42976       ESTABLISHED
    tcp        0      0 192.168.206.129:8888        192.168.206.128:42974       ESTABLISHED

客户端netstat输出：

    Every 1.0s: netstat -atn | grep 8888 | grep ESTAB                                                   Tue Dec 24 20:33:00 2013
    
    tcp        0      0 192.168.206.128:43127       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43083       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43066       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43047       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43000       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43100       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43103       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43102       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43075       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43123       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43063       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43125       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43112       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43014       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43023       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43006       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43064       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43013       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43037       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:42989       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:42993       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43070       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43068       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43111       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43089       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43041       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43002       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43003       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43092       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43079       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43058       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43062       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:42998       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43085       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43124       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43080       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43067       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43069       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43071       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43019       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43024       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43004       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43097       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43105       192.168.206.129:8888        ESTABLISHED
    tcp        0      0 192.168.206.128:43130       192.168.206.129:8888        ESTABLISHED
    ...................................................

以及结合TCP抓包分析，看到的现象是：服务器端只缓冲了最开始的4个已完成的连接，之后的已完成的连接netstat看不到，猜测应该是直接在内核被丢弃，孙然服务器端的连接缓冲队列已经满了，但是对新到达的连接请求，TCP按照正常的方式与客户端完成三次握手，这导致了客户端nestat中不断有新的TCP连接建立，客户端执行到1024个连接左右，connect返回非0值，报错。

结论：

> 连接缓存队列满时，协议栈正常的进行TCP连接的建立，只不过是新建立的TCP连接直接被server端丢弃。

## 客户端向没有被accept的已完成的TCP连接发送数据时，会出现什么情况呢？

修改客户端代码，客户端先发起100次连接，然后依次向连接fd中写2个字节的数据，tcpdump抓包观察服务器端TCP的反应。客户端程序修改如下：

    int main(int argc, char **argv)
    {
        int                    sockfds[100];
        struct sockaddr_in    servaddr;
        unsigned int n = 0;
        if (argc != 2)
        {
            printf("%s\n","usage: tcpcli <IPaddress>");
            exit(0);
        }
    
        for(n = 1; n <= 100; n++){
            sockfds[n-1] = socket(AF_INET, SOCK_STREAM, 0); /* 创建连接sock */
            bzero(&servaddr, sizeof(servaddr)); /* 初始化服务器端套接口地址 */
            servaddr.sin_family = AF_INET;
            servaddr.sin_port = htons(8888);
            inet_pton(AF_INET, argv[1], &servaddr.sin_addr);
            printf("start the %dth connection  %d\n", n);
            if(0 != connect(sockfds[n-1], (SA *) &servaddr, sizeof(servaddr)))
            {
                printf("connect error\n"); /* 连接至服务器端 */
                return 0;
            }
            sleep(2);
            printf("done the %dth connection and the sockfd %d\n", n, sockfds[n-1]);
        }
        
        for(n =1; n <= 100; n++){
            if(-1 == write(sockfds[n-1],"c\0",2)) {
                printf("write error for sockfd %d\n", sockfds[n-1]);
            }
        }
        return 0;
    }


结论：

> 存在于tcp连接缓冲区里面的TCP连接：服务器端正常接收客户端的数据，并发送ACK确认。奇怪了，此时的tcp连接在服务器端仍然绑定在监听端口，也就是说多个连接共用同一个端口，服务器端如何区分数据属于哪一个TCP连接的？TCP连接是四元组，服务器端绑定在同一个端口的TCP连接是通过(源IP、源端口号)来区分数据。不存在于tcp连接缓冲区里面的TCP连接：在服务器端看来已经感知不到的TCP连接，服务器端直接发送RST给客户端TCP。

## 客户端向没有被accept的TCP连接里面写数据后，服务器端accept TCP连接然后读socket，是否可以读取到正确的数据呢？

为了得到上面问题的答案，修改了客户端代码，先和服务器端建立100个TCP连接，然后依次向每个TCP连接里面写入对应的编号，代码如下：

    int main(int argc, char **argv)
    {
        int                    sockfds[100];
        struct sockaddr_in    servaddr;
        unsigned int n = 0;
        char tmp[10];
    
        memset(tmp, 10, 0);
        if (argc != 2)
        {
            printf("%s\n","usage: tcpcli <IPaddress>");
            exit(0);
        }
    
        for(n = 1; n <= 100; n++){
            sockfds[n-1] = socket(AF_INET, SOCK_STREAM, 0); /* 创建连接sock */
            bzero(&servaddr, sizeof(servaddr)); /* 初始化服务器端套接口地址 */
            servaddr.sin_family = AF_INET;
            servaddr.sin_port = htons(8888);
            inet_pton(AF_INET, argv[1], &servaddr.sin_addr);
            printf("start the %dth connection  %d\n", n);
            if(0 != connect(sockfds[n-1], (SA *) &servaddr, sizeof(servaddr)))
            {
                printf("connect error\n"); /* 连接至服务器端 */
                return 0;
            }
            sleep(1);
            printf("done the %dth connection and the sockfd %d\n", n, sockfds[n-1]);
        }
    
        for(n =1; n <= 100; n++){
            snprintf(tmp, sizeof(tmp), "%d\0", n);
            if(-1 == write(sockfds[n-1], tmp, 2)) {
                printf("write error for sockfd %d\n", sockfds[n-1]);
            }
        }
        return 0;
    }

修改服务器端代码，服务器端监听端口200秒之后，开始accept TCP连接，并且读取TCP连接中的数据，代码如下：

    int
    main(int argc, char **argv)
    {
        int                    listenfd, connfd;
        pid_t                childpid;
        socklen_t            clilen;
        int n = 0;
        char        line[1024];
        struct sockaddr_in    cliaddr, servaddr;
    
        memset(line, 0, sizeof(line));
        listenfd = socket(AF_INET, SOCK_STREAM, 0);
    
        bzero(&servaddr, sizeof(servaddr));
        servaddr.sin_family      = AF_INET;
        servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
        servaddr.sin_port        = htons(SERV_PORT);
    
        bind(listenfd, (SA *) &servaddr, sizeof(servaddr));
    
        listen(listenfd, LISTENQ);
        printf("listen but not accept, buffer the connection in queue:\n");
        for (n = 0; n < 100 ; n++) {
            sleep(2);
            write(1,".",1);
        }
    
        printf("\nbegin accept the connection:\n");
        n = 1;
        while(1){
            clilen = sizeof(cliaddr);
            connfd = accept(listenfd, (SA *) &cliaddr, &clilen);
            printf("accepted the %dth connetion\n", n);
            n = n + 1;
            read(connfd, line, 1024);
            printf("datas in connfd is :%s\n", line);
        }
        printf("\n");
        return 0;
    }

运行客户端和服务器端程序，最后服务器端的输出如下：

    accepted the 1th connetion
    datas in connfd is :1
    accepted the 2th connetion
    datas in connfd is :2
    accepted the 3th connetion
    datas in connfd is :3
    accepted the 4th connetion
    datas in connfd is :4

结论：

> tcp服务器端，已建立但没有被accept的tcp连接，存在于tcp服务器端的连接缓冲队列里面，客户端写到这些连接的数据可以被服务器端accept该连接后，正常的接受。


玩的开心 !!!
