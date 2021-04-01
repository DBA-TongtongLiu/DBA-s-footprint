# MongoDB 复制集搭建

## 安装 MongoDB

### 安装包

1. 从网站 [https://www.mongodb.org/dl/linux/x86\_64](https://www.mongodb.org/dl/linux/x86_64) 选择合适的安装包下载
2. 放到服务器上（或本地），解压
3. 将安装包整体 `mv` 到 `/usr/local/mongodb` 路径下

### 环境变量

配置环境变量：

```text
vi /etc/profile 
```

在末尾增加：

```text
export MONGODB_HOME=/usr/local/mongodb  
export PATH=$PATH:$MONGODB_HOME/bin
```

加载环境变量：

```text
source /etc/profile
```

查看是否加载成功：

```text
which mongod
```

## 创建三个 MongoDB 实例

### 创建相关目录

```text
mkdir -p /data/db{1,2,3}
```

### 创建配置文件

```text
vim /data/db1/mongod.conf
```

```text
systemLog:
  destination: file
  path: /data/db1/mongod.log
  logAppend: true
storage:
  dbPath: /data/db1    # 数据目录
net:
  bindIp: 0.0.0.0
  port: 28017   # 端口
replication:
  replSetName: rs0
processManagement:
  fork: true
```

在三个目录下分别创建配置文件，自行修改配置文件中的：

1. systemLog.path
2. storage.dbPath
3. net.port

### 启动实例

```text
mongod -f /data/db1/mongod.conf
mongod -f /data/db2/mongod.conf
mongod -f /data/db3/mongod.conf
```

### 查看启动是否成功

```text
ps -ef | grep mongod
```

## 搭建复制集

### 初始化复制集

选择任意实例进入：

```text
mongo --port 28017
```

进行配置

```text
rs.initiate({
    _id: "rs0",
    members: [{
        _id: 0,
        host: "localhost:28017"
    },{
        _id: 1,
        host: "localhost:28018"
    },{
        _id: 2,
        host: "localhost:28019"
    }]
})
```

### 查看复制集状态

```text
rs.status()
```

输出示例：

```text
{
	"set" : "rs0",
	"date" : ISODate("2021-04-01T09:09:30.052Z"),
	"myState" : 1,
	"term" : NumberLong(1),
	"syncingTo" : "",
	"syncSourceHost" : "",
	"syncSourceId" : -1,
	"heartbeatIntervalMillis" : NumberLong(2000),
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1617268168, 1),
			"t" : NumberLong(1)
		},
		"readConcernMajorityOpTime" : {
			"ts" : Timestamp(1617268168, 1),
			"t" : NumberLong(1)
		},
		"appliedOpTime" : {
			"ts" : Timestamp(1617268168, 1),
			"t" : NumberLong(1)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1617268168, 1),
			"t" : NumberLong(1)
		}
	},
	"lastStableCheckpointTimestamp" : Timestamp(1617268118, 1),
	"electionCandidateMetrics" : {
		"lastElectionReason" : "electionTimeout",
		"lastElectionDate" : ISODate("2021-04-01T08:55:38.158Z"),
		"electionTerm" : NumberLong(1),
		"lastCommittedOpTimeAtElection" : {
			"ts" : Timestamp(0, 0),
			"t" : NumberLong(-1)
		},
		"lastSeenOpTimeAtElection" : {
			"ts" : Timestamp(1617267327, 1),
			"t" : NumberLong(-1)
		},
		"numVotesNeeded" : 2,
		"priorityAtElection" : 1,
		"electionTimeoutMillis" : NumberLong(10000),
		"numCatchUpOps" : NumberLong(0),
		"newTermStartDate" : ISODate("2021-04-01T08:55:38.162Z"),
		"wMajorityWriteAvailabilityDate" : ISODate("2021-04-01T08:55:38.337Z")
	},
	"members" : [
		{
			"_id" : 0,
			"name" : "localhost:28017",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 874,
			"optime" : {
				"ts" : Timestamp(1617268168, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2021-04-01T09:09:28Z"),
			"syncingTo" : "",
			"syncSourceHost" : "",
			"syncSourceId" : -1,
			"infoMessage" : "",
			"electionTime" : Timestamp(1617267338, 1),
			"electionDate" : ISODate("2021-04-01T08:55:38Z"),
			"configVersion" : 1,
			"self" : true,
			"lastHeartbeatMessage" : ""
		},
		{
			"_id" : 1,
			"name" : "localhost:28018",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 842,
			"optime" : {
				"ts" : Timestamp(1617268158, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1617268158, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2021-04-01T09:09:18Z"),
			"optimeDurableDate" : ISODate("2021-04-01T09:09:18Z"),
			"lastHeartbeat" : ISODate("2021-04-01T09:09:28.179Z"),
			"lastHeartbeatRecv" : ISODate("2021-04-01T09:09:28.835Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncingTo" : "localhost:28017",
			"syncSourceHost" : "localhost:28017",
			"syncSourceId" : 0,
			"infoMessage" : "",
			"configVersion" : 1
		},
		{
			"_id" : 2,
			"name" : "localhost:28019",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 842,
			"optime" : {
				"ts" : Timestamp(1617268158, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1617268158, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2021-04-01T09:09:18Z"),
			"optimeDurableDate" : ISODate("2021-04-01T09:09:18Z"),
			"lastHeartbeat" : ISODate("2021-04-01T09:09:28.179Z"),
			"lastHeartbeatRecv" : ISODate("2021-04-01T09:09:28.834Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncingTo" : "localhost:28017",
			"syncSourceHost" : "localhost:28017",
			"syncSourceId" : 0,
			"infoMessage" : "",
			"configVersion" : 1
		}
	],
	"ok" : 1,
	"operationTime" : Timestamp(1617268168, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1617268168, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	}
}
```

### 调整复制集配置

如有明确需要，可以调整。不然，跳过即可。

```text
var conf = rs.conf()

// 将0号节点的优先级调整为10
conf.members[0].priority = 10;

// 将1号节点调整为hidden节点
conf.members[1].hidden = true;

// hidden节点必须配置{priority: 0}
conf.members[1].priority = 0;
```

应用调整：

```text
rs.reconfig(conf);
```

