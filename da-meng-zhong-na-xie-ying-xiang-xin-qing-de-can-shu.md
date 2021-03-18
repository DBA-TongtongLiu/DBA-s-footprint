# 达梦中那些影响心情的参数

## 插入报错 SQLAllocHandle {HY000}

### 问题场景

应用使用 ORM 开启事务，批量插入语句，起初正常，一定条数之后开始报错：

```text
SQLAllocHandle: {HY000} Stmt handle to the limit the number of statements or system of memory
```

### 解决方案

首先查看下报错中提到的两个相关参数 `statements` 和 `memory` 的当前值：

```text
SQL> select name,type,value,description  from v$parameter where name in ('memory_pool', 'max_session_statement');
+------------------------+--------------+--------------+--------------------------------------------------------+
| name                   | TYPE         | VALUE        | description                                            |
+------------------------+--------------+--------------+--------------------------------------------------------+
| MEMORY_POOL            | IN FILE      | 200          | Memory Pool Size In Megabyte                           |
| MAX_SESSION_STATEMENT  | SYS          | 100          | Maximum number of statement handles for each session   |
+------------------------+--------------+--------------+--------------------------------------------------------+
SQLRowCount returns 2
2 rows fetched
```

可以看到 `memory pool` 只有 200M，`session statement` 也只有 100 条，都太小了，我们扩大这两个参数。

### 达梦中的参数类型（type）说明

> `IN FILE`：静态参数。只能修改 ini 文件，修改后重启DB才能生效，为系统级参数，生效后会影响所有的会话。  
>  `SYS` 和 `SESSION`：ini 文件和内存同时可修改，修改后即时生效。其中，SYS为系统级参数，修改后会影响所有的会话；SESSION 为会话级参数，服务器运行过程中被修改时，之前创建的会话不受影响，只有新创建的会话使用新的参数值。  
>  `READ ONLY`：DB在运行过程中，手动参数不能被修改，静态和动态参数可以修改。

可以看出 `memory pool` 必须通过 `ini 文件` 修改，而要让 `session statement` 永久生效（不要在重启后就又变回去）也要在 `ini 文件` 中修改，所以我们找到文件所在路径，备份后修改这两个参数为自己需要的值：

```text
root@Kylin:/opt/dmdbms/data/tinaliu# cat /opt/dmdbms/data/tinaliu/dm.ini | grep -E "MEMORY_POOL|MAX_SESSION_STATEMENT"
        MEMORY_POOL                     = 4096                   #Memory Pool Size In Megabyte
        MAX_SESSION_STATEMENT           = 5000                   #Maximum number of statement handles for each session
```

顺便说一句，如果只修改 `session statement` 是可以使用语句直接操作的：

```text
SQL> alter system set 'max_session_statement' = 5000;
SQLRowCount returns 0
```

测试发现：本次卡住我们的，正是参数 `max_session_statement`。那为什么我们的程序，要由这么多的句柄支撑呢？

我们改小句柄数\(100\)，分别对以下语句进行测试： 

1. 普通插入：insert      👌 

2. 查询 `last_insert_id`：select scope\_identity\(\)      🙅（第101条报错） 

3. 普通查询：select \* from test.tbl\_test;      🙅 （第101条报错）

经过简单的测试，我们可以初步判断：**句柄数`max_session_statement`是用于限制查询结果集的。**

那么这个限制在什么维度生效？

我们进行以下测试： 

1. 同一个session中，执行200条查询语句，每条单独提交    👌 

2. 同一个事务中，执行200条查询语句，统一提交     🙅（第101条报错）

经过测试，我们可以得出结论：句柄数`max_session_statement`是用于限制在**一个事务中**查询结果集个数的。

我们将此参数修改为 5000。毕竟，再大些的事务，就一定要拆分了。

