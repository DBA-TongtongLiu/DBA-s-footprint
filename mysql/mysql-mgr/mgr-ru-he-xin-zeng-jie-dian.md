---
description: 本文适用于 MGR 集群正常，需要新增一台节点的情形。与 MGR 整体崩坏区别处理。
---

# MGR 如何新增节点？

## 检查 MGR 集群状态



检查 MGR 集群成员及运行状态：

```
// 登录 MySQL 执行
select * from performance_schema.replication_group_members;
```



检查当前登录环境是否是主库：

```
// 登录 MySQL 执行
select if((select @@server_uuid)=(select variable_value from performance_schema.global_status where variable_name='group_replication_primary_member'),1,0) as is_primary_mode,@@server_id;
```

&#x20;

## 备份主库



找到主节点进行备份（其实不是主也行，这里面向实施同学，要求统一）：

```
// 在操作系统（比如 linux 上执行）
// 注意：mysqldump 要使用 5.7 及以上版本
mysqldump -h {{hostname/ip}} -u root -p -q --single-transaction --set-gtid-purged=ON --all-databases > recover.sql
```



## 从库恢复



```
// 登录 MySQL 执行
reset master;
```

```
// 登录 MySQL 执行：不记 binlog，不生成 GTID
set sql_log_bin = 0;
```

```
// 登录 MySQL 执行：把导出的 sql 文件导入
source ***.sql
```

```
// 登录 MySQL 执行：使 mysql.user 等权限表中的用户生效
flush privileges;
```

```
// 登录 MySQL 执行：这个不执行也可以
set sql_log_bin = 1;
```

```
// 登录 MySQL 执行
change master to master_user='{{user}}',master_password='{{pwd}}' for channel 'group_replication_recovery';
```

```
// 登录 MySQL 执行
start group_replication;
```
