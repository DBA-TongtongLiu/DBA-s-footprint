---
description: 本文适用于 MGR 整体崩坏，且主库独立提供一段时间服务的情况。区别于新增节点
---

# MGR 集群只剩一台节点怎么办

## 故障原因

### 故障现象

MGR 集群从库全部断开

### 处理方式

解散 MGR 集群，使业务全部连接单点（原 master）进读写操作。当前业使用正常。

预计低峰期对 MGR 集群进行恢复。

注意：掉一台机器，和重新搭建 MGR 的情况不一样。如果是新增节点，或者尝试恢复一台机器，请参考文档`《MGR 如何新增节点？》`

## MGR 集群恢复

将承担读写的那台实设置为 MGR 主库，使用 mysqldump 扩容两个新的从库，连接到该主库上。

### 前置检查

检查链接：

```
select * from information_schema.processlist where command <> 'Sleep';
```

检查流量：

```
tail -f master.log
```

检查单主模式是否开启：

```
show global variables like 'group_replication_single_primary_mode';
```

检查 server\_id 是否冲突：

```
show global variables like 'server_id';
```

检查实例当前的地址：

```
show global variables like 'group_replication_local_address';
```

检查 MGR 集群当前的配置：

```
show global variables like 'group_replication_group_seeds';
```

### 将 master-01 设置为 MGR 主库

```
reset master;

change master to master_user='repluser',master_password='123456' for channel 'group_replication_recovery';

set global group_replication_bootstrap_group=on;

start group_replication;

set global group_replication_bootstrap_group=off;
```

### 数据备份

对 master-01 和 slave-02 进行数据备份

```
mysqldump -h 127.0.0.1 -u root -p -q --single-transaction --set-gtid-purged=ON --all-databases > recover.sql
```

参数说明：

1. `-q:` --quick , Don't buffer query, dump directly to stdout.
2. &#x20;`​--single-transaction:` Creates a consistent snapshot by dumping all tables in a single transaction.
3. `--master-data=2:` 会注释掉 change master to， MGR 不需要指定具体位置，会自动寻找

### 修复 slave-03

将 `gtid_executed` , `gtid_purged` 变量置空

```
reset master；
```

导入备份的数据（master）：

```
set sql_log_bin = 0;
source recovery.sql;
```

生效导入

```
flush privileges;
```

加入 MGR 集群：

```
// 如果这一步报错：invalid hostname or ip，则使用真实 ip，例如：set group_replication_local_address='10.10.10.22:24901'
set group_replication_local_address=':24901'

set global group_replication_allow_local_disjoint_gtids_join=on;

change master to master_user='{{user}}',master_password='{{pwd}}' for channel 'group_replication_recovery';

start group_replication;
```

执行完此操作后，在主库进行可用性测试，观从库同步是否正常。

### 恢复 slave-02

将 slave-02 的数据永久保留，按照 slave-03 的操作步骤进行恢复操作。

验证 slave-02 的可用性

## ProxySQL 恢复

### 添加主从服务器列表

```
insert into mysql_servers(hostgroup_id,hostname,port,weight,comment) values(10,'192.168.1.144',3306,1,'master'),(10,'192.168.1.145',3306,1,'slave1'),(10,'192.168.1.146',3306,3,'slave2');
```

### 加载和保存

```
 load mysql servers to runtime;
 
 save mysql servers to disk;
  
 select * from mysql_servers
```

### 检查用户、规则等是否存在

```
select * from mysql_users;

select * from mysql_replication_hostgroups;
```

如果不存在，添加:

```
 insert into mysql_users(username,password,default_hostgroup) values('proxysql','123456',10);

insert into mysql_replication_hostgroups values (10,20,'read_only','proxysql');
```

### 可用性测试

确保建表、删表、增删改查没有问题

### 错误日志

查看 proxy 和 mysql 的错误日志，观察是否有报错

## 可用性测试

```
create database test;

create table test(id int);

insert into test.test values (1);

select * from test.test;

update test.test id = 2 where id = 1;

delete from test.test;

drop table test.test;

drop database test.test;
```

