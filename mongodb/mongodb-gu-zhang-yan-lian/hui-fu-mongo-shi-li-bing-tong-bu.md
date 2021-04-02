# 恢复 mongo 实例并同步

## 安装相同版本 MongoDB

参考：[安装 MongoDB](https://liu-tongtong.gitbook.io/dba/mongodb/mongodb-fu-ben-ji-da-jian#an-zhuang-mongodb)

## 备份数据

本次恢复的实例是阿里云上的实例，所以下载即可：

```text
wget -c '<数据备份文件外网下载地址>' -O hins_data.tar.gz
```

这里要注意一下原始格式。有些是 `.tar.gz` 有些是 `_qp.xb`，二者处理方式不同

## 恢复数据

### 解压数据

```text
tar xvf hins_data.tar.gz
```

查看解压出来的数据所在位置：

```text
[root@baseimage mongodb]# pwd
/home/mongodb
```

### 配置文件

在指定位置创建配置文件：

```text
touch /root/mongo/mongod.conf
```

将如下内容写入配置文件：

```text
systemLog:
    destination: file
    path: /root/mongo/mongod.log
    logAppend: true
security:
    authorization: enabled
storage:
    dbPath: /home/mongodb  # 上文解压出来的文件路径
    directoryPerDB: true
net:
    port: 27017
    unixDomainSocket:
        enabled: false
processManagement:
    fork: true  # 后台启动
    pidFilePath: /root/mongo/mongod.pid
```

大家可以看到没有指定引擎，是因为相同版本的 Mongo 使用的默认引擎是一致的，如果没有修改，无需额外指定。

我们使用的版本是 4.0，默认引擎是 `wiredTiger`。 查看方法：

```text
> db.serverStatus().storageEngine
{
	"name" : "wiredTiger",
	"supportsCommittedReads" : true,
	"supportsSnapshotReadConcern" : true,
	"readOnly" : false,
	"persistent" : true
}
```

## 启动实例

```text
mongod -f /root/mongo/mongod.conf
```

检查日志有误错误（日志路径在配置文件中）：

```text
tail -f /root/mongo/mongod.log
```

### 登录实例

```text
mongo --host 127.0.0.1 -u root -p --authenticationDatabase admin
```

由于这是恢复出来的实例，所以密码是原实例设置的密码。

## 开启同步

## 检查数据

推荐 Hash 校验:

```text
# 对当前库下的每张表做 Hash 校验
db.collection_env.runCommand({dbHash:1})

# 对指定集合做校验
db.runCommand ( { dbHash: 1, collections: [ <collection1>, ... ] } )
```

如果数据在源源不断写入，那么执行时间不统一是会影响到校验结果的。

在流量不大的情况下，可以使用 `command + d` ，再 `command + shift + i` 开启两个同步窗口：`Keyboard input will be sent to multiple sessions`  然后使用dbHash命令进行校验。

