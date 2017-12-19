## mongodb shell
### 连接数据库

	for i in {17089..17095};do mongo 127.0.0.1:$i/admin -u WCSUserAdmin -p VXIHF6I1XQr4 --eval "rs.status()" |egrep 'name|stateStr';done

	mongo 127.0.0.1:17095/admin -u WCSUserAdmin -p VXIHF6I1XQr4

### 更改副本权重

mongo 交互命令下：
	cfg = rs.conf()
	cfg.members[0].priority = 1
	cfg.members[1].priority = 10
	##根据情况调整各个成员的优先级
	rs.reconfig(cfg)