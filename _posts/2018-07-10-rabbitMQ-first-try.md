---
layout: post
title: RabbitMQ 初试
category: 系统及服务
tags: RabbitMQ
keywords: RabbitMQ
description:
---

> 用正确的工具，做正确的事情

最近了解学习微服务架构，很多地方提高利用消息队列进行服务之间解耦，另外看各个公有云厂家的平台上都有提供消息队列服务，感觉消息队列服务是目前搭建业务系统常用的一种服务组件。
因此近期有时间对AMQP（高级消息队列协议）以及RabbitMQ（AMQP的一种实现）做了简单的了解，下面是对一些AMQP基本概念以及RabbitMQ简单部署和测试的总结。

### AMQP协议基本概念

AMQP协议定义的实体以及实体之间的关系：

![AMQP协议架构](http://7u2rbh.com1.z0.glb.clouddn.com/AMQP.png)


#### Message

消息，由消息头和消息体组成，消息头由各种可选项属性组成，常用的属性包括routing-key（exchange根据binding做消息转发时使用）、priority（指定消息的优先级）、delivery-mode等，消息体也即消息携带的内容，对AMQP来讲是不透明的。

#### Publisher

消息的生产者。

#### Exchange

交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列中。有4种类型的exchange，包括direct、fanout、topic、headers。

#### Binding

消息队列与交换器之间的绑定，同时也是交换器将消息路由到“绑定的该交换器队列”的路由规则。

#### Queue

消息队列，保存消息生产者发送的消息，消息暂存在消息队列里面等待消息consumer将消息取走。

#### Connection和Channel

Connection 表示一个TCP连接（假设AMQP基于tcp协议实现），channel是多路复用连接中的一条独立的双向数据流通道。由于网络连接的建立和释放代价比较高，因此消息生产者或者消费者都是通过channel来发送和接受消息。

#### Consumer

消息的消费者。

#### broker

表示消息队列服务器实体，包括exchange和queue以及之间的binding。


### rabbitMQ 部署以及测试

yum安装rabbitmq-server并启动rabbit-server：

	yum install rabbitmq-server
	
	/sbin/rabbitmq-server -detached
	Warning: PID file not written; -detached was passed

	[root@localhost xuc]# /sbin/rabbitmqctl status
	Status of node rabbit@localhost ...
	[{pid,90713},
	{running_applications,[{rabbit,"RabbitMQ","3.3.5"},
                        {os_mon,"CPO  CXC 138 46","2.2.14"},
                        {mnesia,"MNESIA  CXC 138 12","4.11"},
                        {xmerl,"XML parser","1.3.6"},
                        {sasl,"SASL  CXC 138 11","2.3.4"},
                        {stdlib,"ERTS  CXC 138 10","1.19.4"},
                        {kernel,"ERTS  CXC 138 10","2.16.4"}]},
	{os,{unix,linux}},
	{erlang_version,"Erlang R16B03-1 (erts-5.10.4) [source] [64-bit] [smp:24:24] [async-threads:30] [hipe] [kernel-poll:true]\n"},
	{memory,[{total,125000440},
          {connection_procs,2800},
          {queue_procs,12856},
          {plugins,0},
          {other_proc,14158160},
          {mnesia,61216},
          {mgmt_db,0},
          {msg_index,34600},
          {other_ets,767392},
          {binary,14984},
          {code,16707114},
          {atom,602729},
          {other_system,92638589}]},
	{alarms,[]},
	{listeners,[{clustering,25672,"::"},{amqp,5672,"::"}]},
	{vm_memory_high_watermark,0.4},
	{vm_memory_limit,13383031193},
	{disk_free_limit,50000000},
	{disk_free,563123699712},
	{file_descriptors,[{total_limit,1025400},
                    {total_used,3},
                    {sockets_limit,922858},
                    {sockets_used,1}]},
	{processes,[{limit,1048576},{used,149}]},
	{run_queue,0},
	{uptime,50631}]
	...done.

安装rabbitMQ python客户端 pika：pip install pika

	pip install pika

运行“hello，workld”测试rabbitMQ 消息队列服务，消息生产者：
	
	#!/usr/bin/env python
	import pika
	
	connection = pika.BlockingConnection(pika.ConnectionParameters(
			host='localhost'))
	channel = connection.channel()
	
	
	channel.queue_declare(queue='hello')
	
	channel.basic_publish(exchange='',
						routing_key='hello',
						body='Fuck mmy life!')
	print(" [x] Sent 'Hello World!'")
	connection.close()

消息消费者测试代码：

	#!/usr/bin/env python
	import pika
	
	connection = pika.BlockingConnection(pika.ConnectionParameters(
			host='localhost'))
	channel = connection.channel()
	
	
	channel.queue_declare(queue='hello')
	
	def callback(ch, method, properties, body):
		print(" [x] Received %r" % body)
	
	channel.basic_consume(callback,
						queue='hello',
						no_ack=True)
	
	print(' [*] Waiting for messages. To exit press CTRL+C')
	channel.start_consuming()
	
测试验证RabbitMQ服务：

	[root@localhost xuc]# python sent.py 
	[x] Sent 'Hello World!'
	[root@localhost xuc]# python sent.py 
	[x] Sent 'Hello World!'
	[root@localhost xuc]# python sent.py 
	[x] Sent 'Hello World!'
	[root@localhost xuc]# python sent.py 
	[x] Sent 'Hello World!'
	[root@localhost xuc]# python sent.py 
	[x] Sent 'Hello World!'
	
	[root@localhost xuc]# /sbin/rabbitmqctl list_queues
	Listing queues ...
	hello	5
	...done
		
	[root@localhost xuc]# python receive.py 
	[*] Waiting for messages. To exit press CTRL+C
	[x] Received 'Hello,world!'
	[x] Received 'Hello,world!'
	[x] Received 'Hello,world!'
	[x] Received 'Hello,world!'
	[x] Received 'Hello,world!'
	
常用rabbitmqctl命令：

	/sbin/rabbitmqctl stop   ##stop本地节点
	/sbin/rabbitmqctl -n rabbit@server.example.com stop ##指定stop 远程节点
	/sbin/rabbitmqctl stop_app  ##stop节点app
	/sbin/rabbitmqctl start_app  ##start节点app
	/sbin/rabbitmqctl reset  ##重置节点，队列会被清空
	/sbin/rabbitmqctl list_queues  ##list当前队列
	/sbin/rabbitmqctl list_exchanges  ##list所有交换器
	/sbin/rabbitmqctl list_exchanges name type durable auto_delete  ##按照指定格式list所有交换器
	/sbin/rabbitmqctl list_bindings	##list所有绑定。
	
### RabbitMQ高可用性-集群模式

在本机创建三个rabbit节点并组成集群模式，步骤如下：
	
	### 创建三个rabbitMQ节点
	export RABBITMQ_NODENAME=test_rabbit_1 RABBITMQ_NODE_PORT=5672&&/sbin/rabbitmq-server -detached
	export RABBITMQ_NODENAME=test_rabbit_2 RABBITMQ_NODE_PORT=5673&&/sbin/rabbitmq-server -detached
	export RABBITMQ_NODENAME=test_rabbit_3 RABBITMQ_NODE_PORT=5674&&/sbin/rabbitmq-server -detached

	### 将test_rabbit_2加入集群 
	/sbin/rabbitmqctl -n test_rabbit_2 stop_app
	/sbin/rabbitmqctl -n test_rabbit_2 reset
	/sbin/rabbitmqctl -n test_rabbit_2 join_cluster test_rabbit_1@localhost
	
	### 将test_rabbit_3加入集群
	/sbin/rabbitmqctl -n test_rabbit_3 stop_app
	/sbin/rabbitmqctl -n test_rabbit_3 reset
	/sbin/rabbitmqctl -n test_rabbit_3 join_cluster test_rabbit_1@localhost
	
	### 查看集群状态
	[root@localhost home]# /sbin/rabbitmqctl -n test_rabbit_1 cluster_status
	Cluster status of node test_rabbit_1@localhost ...
	[{nodes,[{disc,[test_rabbit_1@localhost,test_rabbit_2@localhost,
                test_rabbit_3@localhost]}]},
	{running_nodes,[test_rabbit_3@localhost,test_rabbit_2@localhost,
                 test_rabbit_1@localhost]},
	{cluster_name,<<"test_rabbit_2@localhost">>},   ###为什么是test_rabbit_2，很奇怪？
		{partitions,[]}]
	...done.


关于RabbitMQ高可用性的问题涉及到更多细节的东西，比如RabbitMQ镜像集群等，高可用和高可靠性是生产环境部署必须要考虑的事情，这里暂时略过。


### RabbitMQ管理界面

启用管理插件：

	rabbitmq-plugins enable rabbitmq_management

通过浏览器访问web管理界面，默认登陆账号和密码为guest/guest：

	http://IP:15672/

web管理界面如下：

![AMQP协议架构](http://7u2rbh.com1.z0.glb.clouddn.com/rabbitMQ-web.png)

	
	
玩得开心！！！
