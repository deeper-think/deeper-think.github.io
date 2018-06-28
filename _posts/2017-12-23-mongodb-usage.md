---
layout: post
title: mongodb使用总结
category: mongdb数据库
tags: [mongodb]
keywords: mongodb
description: 
---

> 用正确的工具，做正确的事情

## mongodb 常用命令

查所有shard的副本集状态：
	
	for i in {17088..17098};do mongo 127.0.0.1:$i/admin -u UserAdmin -p UserAdminPASSWD --eval "rs.status()" |egrep --color 'PRIMARY|name|stateStr|WARNING|set';done


连接到某个分片副本：

	mongo 127.0.0.1:17095/admin -u UserAdmin -p UserAdminPASSWD

调节副本集各成员的状态：

	cfg = rs.conf()
	cfg.members[0].priority = 1
	rs.reconfig(cfg)

通过数据拷贝来恢复副本集成员：

	db.shutdownServer() //关闭待恢复的副本集
	db.fsyncLock() //复制源写请求锁定
	scp -P57891 -r root@10.23.167.191:/cache3/mongod_data ./ //从复制源拷贝数据
	chown -R mongod.mongod /cache3/mongod_data/ //修改复制的数据的属性
	db.fsyncUnlock() //复制源解锁

副本集增加和删除成员：
	
	rs.add("10.23.167.74:17091")
	rs.remove("10.23.167.74:17091")

调整修改balance窗口：

	use config
	db.settings.update({ _id : "balancer" }, { $set : { activeWindow : { start : "6:00", stop : "14:00" } } }, true )

	db.settings.update({ _id : "balancer" }, { $unset : { activeWindow : true } }) //不设置窗口

mongodb账号权限管理：

	db.grantRolesToUser( "WCSUserAdmin" , [ { role: "root", db: "admin" } ])

mongodb创建用户：

	db.createUser({"user": "userAdmin","pwd": "userAdmin","customData": { "user": "userAdmin","pwd": "userAdmin" },"roles": [ { role: "userAdminAnyDatabase", db: "admin" },{ role: "userAdmin", db: "admin" }] })


## mongodb权限认证

与mongodb权限认证有关的三个常用配置项：

	security.authorization
	security.keyFile
	setParameter.enableLocalhostAuthBypass

### security.authorization配置项

security.authorization 默认值disabled。 默认情况下，mongodb没有操作权限验证，在这种情况下采用如下匿名的方式连接登录mongodb后也可以做任何的操作，包括：创建用户，创建数据库等：

	[root@localhost home]# mongo 127.0.0.1:17088/admin
	MongoDB shell version v3.4.12
	connecting to: mongodb://127.0.0.1:17088/admin
	MongoDB server version: 3.4.12
	Server has startup warnings: 
	2018-05-23T22:43:44.603+0800 I CONTROL  [initandlisten] 
	2018-05-23T22:43:44.603+0800 I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.
	2018-05-23T22:43:44.603+0800 I CONTROL  [initandlisten] **          Read and write access to data and configuration is unrestricted.
	2018-05-23T22:43:44.603+0800 I CONTROL  [initandlisten] 
	2018-05-23T22:43:44.603+0800 I CONTROL  [initandlisten] 
	2018-05-23T22:43:44.603+0800 I CONTROL  [initandlisten] ** WARNING: soft rlimits too low. rlimits set to 1024 processes, 64000 files. Number of processes should be at least 32000 : 0.5 times number of files.
	> show dbs
	admin     0.000GB
	fileinfo  0.000GB
	local     0.000GB


security.authorization 设置为true，开启权限验证。

### security.keyFile配置项

设置security.keyFile的同时也就将security.authorization 设置为true，也即开启权限验证。 security.keyFile用于mongos与mongod以及mongod cs和shard之间的同步和通信。

### setParameter.enableLocalhostAuthBypass配置项

在3.4版本验证，没有发现该配置项有什么作用。


## mongostat 数据采集

mongostat采集当前mongodb的状态数据：

	/usr/bin/mongostat 60 --host=127.0.0.1 --port=17088 -u WCSMongoStatUser -p passwd --authenticationDatabase admin --discover

创建用于mongostat数据采集的账号：

	db.createUser({"user": "WCSMongoStatUser","pwd": "passwd","customData": { "user": "WCSMongoStatUser","pwd": "passwd" },"roles": [ { role: "clusterMonitor", db: "admin" }] })




玩的开心 !!!
