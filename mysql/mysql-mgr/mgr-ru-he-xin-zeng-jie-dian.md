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
// 登录 MySQL
```

&#x20;

## 备份主库



找到主节点进行备份（其实不是主也行，这里面向实施同学，要求统一）：

```
// 在操作系统（比如 linux 上执行）
// 注意：mysqldump 要使用 5.7 及以上版本
mysqldump -h {{hostname/ip}} -u root -p -q --single-transaction --set-gtid-purged=ON --all-databases > recover.sql
```





