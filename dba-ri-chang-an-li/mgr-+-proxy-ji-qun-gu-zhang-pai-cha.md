# MGR + Proxy 集群故障排查

有私有部署项目报DB连接失败。

当然啦，这只是表象，我们的数据库架构是 MGR 集群（3台 MySQL 实例） + n 个 proxy 组成。

可能是集群崩了，比如剩一个实例了，只能读不能写了。

也可能是 proxy 有问题了，没能成功转发请求。

也可能，是网络问题……

因为不在现场，所以只能请项目经理帮忙收集下信息：

## 查看 MGR 状态

每台实例上均执行以下操作：

### 查看实例的状态

#### 查看实例错误日志

less error.log 

#### 最后启动时间

登录进入 MySQL，执行 `\s` 命令：

```text
mysql> \s
```

#### 查看实例是否只读

```text
mysql> show global variables like '%read_only%';
```

### 查看 MGR 集群状态

#### 查看集群内的成员

```text
 select * from performance_schema.replication_group_members;
```

#### 查看 MGR 节点的连接状态

```text
select * from performance_schema.replication_connection_status\G
```

#### 查看 MGR 配置信息

```text
show global variables like '%group_replication%';
```

#### 查看当前实例是否为主节点

```text
select if((select @@server_uuid)=(select variable_value from performance_schema.global_status where variable_name='group_replication_primary_member'),1,0) as is_primary_mode,@@server_id
```

#### 查看节点在 proxy 中的配置

```text
select * from sys.gr_member_routing_candidate_status;
```

## 看 Proxy 状态

每个 proxy 上执行以下命令：（不排除是 proxy 故障，导致无法查询）

### 查看 proxy 错误日志

less error.log

### 查看 proxy 的最后启动时间

ps -ef \| grep proxysql

### 查看管理成员及其分组

```text
select * from mysql_replication_hostgroups;
```

```text
select * from mysql_servers;
```

### 查看连接日志

```text
select * from monitor.mysql_server_connect_log;
```

日志很可能过长，请分别按时间和出错内容进行查询

```text
select * from monitor.mysql_server_connect_log order by time_start_us desc limit 100;
```

```text
select * from monitor.mysql_server_connect_log where connect_error is not null order by time_start_us desc limit 100;
```

### 查看 ping log

如果连接日志有异常，查看 ping log 辅助查询

```text
 select * from monitor.mysql_server_ping_log order by time_start_us desc limit 100;
```

```text
 select * from monitor.mysql_server_ping_log where ping_error is not null order by time_start_us desc limit 100;
```

## 查看服务器状态

在每台服务器上执行以下命令：

### 查看服务是否 OOM

```text
dmesg
```

### 查看 docker 是否重启过

```text
docker ps | grep mysql
```

## 后记

项目经理说他执行了以下命令，使得访问恢复正常：

```text
load mysql servers to runtime; 
```

该命令只是把 proxy 的配置从磁盘加载到内存中，如果真是它决定了生死，那大概率就是 这个 proxy 的问题了。





