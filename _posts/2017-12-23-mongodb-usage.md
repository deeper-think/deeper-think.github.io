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
	

mongostat 数据采集：

	/usr/bin/mongostat 60 --host=127.0.0.1 --port=17088 -u UserAdmin -p passwd --authenticationDatabase admin --discover





玩的开心 !!!
