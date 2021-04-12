---
description: 如果你也被慢查询、CPU 打满等问题困扰，读它！
---

# MongoDB 慢查询

## 查看实例上的慢查询

### 查询语句说明

假设我们想查询实例上所有执行时间超过 `1s` 的慢语句。

```text
db.currentOp(
  {
    "active": true,
    "secs_running": {
        "$gt": 1
    }
  }
)
```

如果想要查看指定 db 上的慢查询：

```text
db.currentOp(
  {
    "active": true,
    "secs_running": {
        "$gt": 1
    },
    "ns": /^siber\./
  }
)
```

### 查询结果分析

截取其中一条语句进行分析：

```text
{
			"host" : "a**2.cloud.nu29:3**9",
			"desc" : "conn6603723",
			"connectionId" : 6603723,
			"client" : "172.16.10.108:58243",
			"appName" : "MongoDB Shell",
			"clientMetadata" : {
				"application" : {
					"name" : "MongoDB Shell"
				},
				"driver" : {
					"name" : "MongoDB Internal Client",
					"version" : "4.2.0"
				},
				"os" : {
					"type" : "Darwin",
					"name" : "Mac OS X",
					"architecture" : "x86_64",
					"version" : "20.3.0"
				}
			},
			"active" : true,
			"currentOpTime" : "2021-04-12T14:36:51.243+0800",
			"opid" : 1210807065,
			"lsid" : {
				"id" : UUID("4494c815-cc94-42bf-88cf-45924faef935"),
				"uid" : BinData(0,"3t2w+sv/3h3pGF2C4Y8XgsDXkmAm1lgwz8idwd27XZ0=")
			},
			"secs_running" : NumberLong(2),
			"microsecs_running" : NumberLong(2946118),
			"op" : "command",
			"ns" : "siber.collection_log_case",
			"command" : {
				"count" : "collection_log_case",
				"query" : {
					"methodname" : "manage_user.ManageUserService.CreateUser"
				},
				"lsid" : {
					"id" : UUID("4494c815-cc94-42bf-88cf-45924faef935")
				},
				"$clusterTime" : {
					"clusterTime" : Timestamp(1618209403, 2),
					"signature" : {
						"hash" : BinData(0,"4LE8FiDFhAU84meQOSn/vVRaoTg="),
						"keyId" : NumberLong("6909073628604661763")
					}
				},
				"$db" : "siber"
			},
			"planSummary" : "COLLSCAN",
			"numYields" : 3449,
			"locks" : {
				"Global" : "r",
				"Database" : "r",
				"Collection" : "r"
			},
			"waitingForLock" : false,
			"lockStats" : {
				"Global" : {
					"acquireCount" : {
						"r" : NumberLong(3450)
					}
				},
				"Database" : {
					"acquireCount" : {
						"r" : NumberLong(3450)
					}
				},
				"Collection" : {
					"acquireCount" : {
						"r" : NumberLong(3450)
					}
				}
			}
		},
```

所有输出的这些字段，都可以用于查询条件，常用的有：

<table>
  <thead>
    <tr>
      <th style="text-align:left">&#x5B57;&#x6BB5;&#x540D;</th>
      <th style="text-align:left">&#x8BF4;&#x660E;</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">client</td>
      <td style="text-align:left">
        <p>&#x8BE5;&#x8BED;&#x53E5;&#x6765;&#x81EA;&#x4E8E;&#x54EA;&#x4E2A;&#x5BA2;&#x6237;&#x7AEF;</p>
        <p>&#x53EF;&#x7528;&#x4E8E;&#x67E5;&#x8BE2;&#x6307;&#x5B9A;&#x5BA2;&#x6237;&#x7AEF;&#x7684;&#x8BF7;&#x6C42;</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">opid</td>
      <td style="text-align:left">
        <p>&#x8BE5;&#x64CD;&#x4F5C;&#x7684;&#x552F;&#x4E00;&#x6807;&#x8BC6;</p>
        <p>db.killOp()&#x65F6;&#x4F7F;&#x7528;&#x6B64;&#x5B57;&#x6BB5;&#x7684;&#x503C;</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">active</td>
      <td style="text-align:left">
        <p>&#x64CD;&#x4F5C;&#x662F;&#x5426;&#x5DF2;&#x542F;&#x52A8;</p>
        <p>&#x7A7A;&#x95F2;&#x8FDE;&#x63A5;&#x6216;&#x8005;&#x5185;&#x90E8;&#x7EBF;&#x7A0B;&#x8BE5;&#x5B57;&#x6BB5;&#x4E3A;false</p>
        <p>&#x67D0;&#x4E9B;&#x5DF2;&#x7ECF;&#x8BA9;&#x6B65;&#x7684;&#x64CD;&#x4F5C;&#xFF0C;&#x72B6;&#x6001;&#x4ECD;&#x53EF;&#x80FD;&#x4E3A;
          true</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">secs_running</td>
      <td style="text-align:left">
        <p>&#x64CD;&#x4F5C;&#x5DF2;&#x5F00;&#x59CB;&#x591A;&#x957F;&#x65F6;&#x95F4;&#xFF08;&#x79D2;&#xFF09;</p>
        <p>&#x5F53; active &#x4E3A; true &#x65F6;&#xFF0C;&#x8BE5;&#x5B57;&#x6BB5;&#x6709;&#x503C;</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">microsecs_running</td>
      <td style="text-align:left">
        <p>&#x64CD;&#x4F5C;&#x5DF2;&#x5F00;&#x59CB;&#x591A;&#x957F;&#x65F6;&#x95F4;&#xFF08;&#x5FAE;&#x79D2;&#xFF09;</p>
        <p>&#x5F53; active &#x4E3A; true &#x65F6;&#xFF0C;&#x8BE5;&#x5B57;&#x6BB5;&#x6709;&#x503C;</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">op</td>
      <td style="text-align:left">&#x64CD;&#x4F5C;&#x7C7B;&#x578B;&#xFF0C;query&#x3001;insert&#x3001;command&#x2026;&#x2026;</td>
    </tr>
    <tr>
      <td style="text-align:left">ns</td>
      <td style="text-align:left">namespace&#xFF0C; db.collection &#x7684;&#x5F62;&#x5F0F;</td>
    </tr>
    <tr>
      <td style="text-align:left">command</td>
      <td style="text-align:left">&#x8BED;&#x53E5;&#x5177;&#x4F53;&#x7684;&#x6267;&#x884C;&#x4FE1;&#x606F;</td>
    </tr>
    <tr>
      <td style="text-align:left">planSummary</td>
      <td style="text-align:left">&#x6267;&#x884C;&#x8BA1;&#x5212;&#xFF0C;&#x53EF;&#x7528;&#x4E8E;&#x6162;&#x8BED;&#x53E5;&#x5206;&#x6790;</td>
    </tr>
    <tr>
      <td style="text-align:left">locks</td>
      <td style="text-align:left">
        <p>&#x64CD;&#x4F5C;&#x5BF9;&#x54EA;&#x4E9B;&#x6A21;&#x5757;&#x52A0;&#x4EC0;&#x4E48;&#x7C7B;&#x578B;&#x7684;&#x9501;</p>
        <p>&#x6A21;&#x5757;&#xFF1A;</p>
        <p>Global</p>
        <p>MMAPV1Journal</p>
        <p>Database</p>
        <p>Collection</p>
        <p>Metadata</p>
        <p>oplog</p>
        <p>&#x9501;&#x7C7B;&#x578B;&#xFF1A;</p>
        <p>R-&#x8BFB;&#x9501; W-&#x5199;&#x9501;</p>
        <p>r-&#x8BFB;&#x610F;&#x5411;&#x9501; w-&#x5199;&#x610F;&#x5411;&#x9501;</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">waitingForLock</td>
      <td style="text-align:left">
        <p>true&#xFF1A;&#x6B63;&#x5728;&#x7B49;&#x9501;</p>
        <p>false&#xFF1A;&#x5DF2;&#x7ECF;&#x83B7;&#x5F97;&#x9501;</p>
      </td>
    </tr>
  </tbody>
</table>



