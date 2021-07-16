# 202107 尚诚故障排查及恢复

## 故障原因

### 处理方式

解散 MGR 集群，使业务全部连接单点（原 master）进读写操作。当前业使用正常。

预计低峰期对 MGR 集群进行恢复。

## MGR 集群恢复

### 前置检查

检查链接：

```text
select * from information_schema.processlist where command <> 'Sleep';
```

检查流量：

```text
tail -f master.log
```

检查单主模式是否开启：

```text
show global variables like 'group_replication_single_primary_mode';
```

检查 server\_id 是否冲突：

```text
show global variables like 'server_id';
```

检查实例当前的地址：

```text
show global variables like 'group_replication_local_address';
```

检查 MGR 集群当前的配置：

```text
show global variables like 'group_replication_group_seeds';
```

### 数据备份

对 master-01 和 slave-02 进行数据备份

```text
msyqldump -h 127.0.0.1 -u root -p -q --single-transaction --master-data=2 --all-databases > recover.sql
```

参数说明：

1. `-q:` --quick , Don't buffer query, dump directly to stdout.
2.  `​--single-transaction:` Creates a consistent snapshot by dumping all tables in a single transaction.
3. `--master-data=2:` 会注释掉 change master to， MGR 不需要指定具体位置，会自动寻找

### slave-03 恢复数据

将 `gtid_executed` , `gtid_purged` 变量置空

```text
reset master；
```

导入备份的数据（master）：

```text
source recovery.sql;
```

生效导入

```text
flush privileges;
```

### 恢复 MGR 集群

#### 将 master-01 设置为 MGR 主库

```text
change master to master_user='repluser',master_password='123456' for channel 'group_replication_recovery';

set global group_replication_bootstrap_group=on;

start group_replication;

set global group_replication_bootstrap_group=off;
```

#### 首先修复 slave-03

```text
set group_replication_local_address=':24901'

set global group_replication_allow_local_disjoint_gtids_join=on;

start group_replication;
```

#### 检查

执行完此操作后，在主库进行可用性测试，观从库同步是否正常。

#### 恢复 slave-02

将 slave-02 的数据永久保留，按照 slave-03 的操作步骤进行恢复操作。

验证 slave-02 的可用性

## ProxySQL 恢复

### 添加主从服务器列表

```text
insert into mysql_servers(hostgroup_id,hostname,port,weight,comment) values(10,'192.168.1.144',3306,1,'master'),(10,'192.168.1.145',3306,1,'slave1'),(10,'192.168.1.146',3306,3,'slave2');
```

### 加载和保存

```text
 load mysql servers to runtime;
 
 save mysql servers to disk;
  
 select * from mysql_servers
```

### 可用性测试

确保建表、删表、增删改查没有问题

### 错误日志

查看 proxy 和 mysql 的错误日志，观察是否有报错

## 可用性测试

```text
create database test;

create table test(id int);

insert into test.test values (1);

select * from test.test;

update test.test id = 2 where id = 1;

delete from test.test;

drop table test.test;

drop database test.test;
```



