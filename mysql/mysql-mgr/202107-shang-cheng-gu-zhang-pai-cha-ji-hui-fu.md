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

### 数据备份

对 master-01 和 slave-02 进行数据备份

```text
msyqldump -h 127.0.0.1 -u root -p -q --single-transaction --master-data=2 --all-databases > recover.sql
```

参数说明：

1. `-q:` --quick , Don't buffer query, dump directly to stdout.
2.  `​--single-transaction:` Creates a consistent snapshot by dumping all tables in a single transaction.
3. `--master-data=2:` 会注释掉 change master to， MGR 不需要指定具体位置，会自动寻找

### 将 master-01 设置 MGR 主库

首先设置 MGR 主库，确保指定实例作为集群的主库

```text

```

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

#### master-01



## ProxySQL 恢复

## 可用性测试



