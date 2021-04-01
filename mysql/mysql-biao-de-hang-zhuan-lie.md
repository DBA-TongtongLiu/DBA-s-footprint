# MySQL 表的行转列

行转列，列转行这种需求，在 DBA 的从业生涯中过分常见。今天，就结合小伙伴儿的实际需求，跟大家分享一个行转列的实际案例。

现存在表A:table\_org，表结构简略抽象如下：

```text
+------------------+---------------------+------+-----+-------------------+-----------------------------+
| Field            | Type                | Null | Key | Default           | Extra                       |
+------------------+---------------------+------+-----+-------------------+-----------------------------+
| id               | int(10) unsigned    | NO   | PRI | NULL              | auto_increment              |
| uri              | varchar(4096)       | NO   |     |                   |                             |
| ...              | ...                 | ...  |     | ...               |                             |
| insert_time      | datetime            | NO   |     | CURRENT_TIMESTAMP |                             |
| update_time      | datetime            | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+------------------+---------------------+------+-----+-------------------+-----------------------------+
```

现在存在的问题是，由于设计问题，uri 字段中可能存在多个值，比如：

```text
/daily/list,/daily/detail,/daily/info,....
```

经查询，最多可能同时存在15个 uri，但是每行 uri 个数不固定，可能是 0 也有可能是 2，3，5... 查询方法：

```text
mysql> select distinct(length(uri) - length((replace(uri, ',', ''))))+1 from tbl_org;
+---------------------------------------------------+
| (length(uri) - length((replace(uri, ',', ''))))+1 |
+---------------------------------------------------+
|                                                 1 |
|                                                 4 |
|                                                 3 |
|                                                 2 |
|                                                14 |
|                                                 7 |
|                                                13 |
|                                                 5 |
|                                                 6 |
|                                                10 |
|                                                 8 |
|                                                 9 |
+---------------------------------------------------+
13 rows in set (0.03 sec)
```

说明：_我们将',' 替换为 ''，损失的长度即为',' 的个数，uri 的个数= ',' 的个数+1_

**行转列的一个常用方法就是：`join`**

我们在 test 库中创建一个 id 为 1-15 的表：\(也可以更大点儿，比如：1-100\)

```text
mysql> select * from test.tina_num;
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
|  4 |
|  5 |
|  6 |
|  7 |
|  8 |
|  9 |
| 10 |
| 11 |
| 12 |
| 13 |
| 14 |
| 15 |
+----+
15 rows in set (0.03 sec)
```

  
 使用语句：

```text
select  substring_index(substring_index(uri,',', t.id),',',-1)  from tbl_org d join tina_num t on 1=1 where t.id <= (length(uri)-length(replace(uri,',','')) + 1);
```

即可得到结果：

```text
+--------------------------------------------------------+
| substring_index(substring_index(uri,',', t.id),',',-1) |
+--------------------------------------------------------+
| /daily/list                                            |
| /daily/detail                                          |
| /daily/info                                            |
| ...                                                    |
+--------------------------------------------------------+
```

说明：

1.获得改行该字段中 uri 的个数, 我们将此记为：row\_num

```text
(length(uri)-length(replace(uri,',','')) + 1)
```

2.有 row\_num 个 uri，就应该转化为 row\_num 行，就要和 tina\_num 表中的 row\_num 行做 `join`：

```text
from tbl_org d join tina_num t on 1=1 where t.id <= row_num
```

组合起来就是：

```text
from tbl_org d join tina_num t on 1=1 where t.id <= (length(uri)-length(replace(uri,',','')) + 1)
```

3.使用 substring\_index 将 uri 切段儿：

```text
substring_index(uri,',', t.id)
```

MySQL官方文档中对`substring_index`的定义为：返回指定个数切片之前的所有字符

_Return a substring from a string before the specified number of occurrences of the delimiter_

简单示例：

```text
mysql> select substring_index('A,B', ',', 1);
+--------------------------------+
| substring_index('A,B', ',', 1) |
+--------------------------------+
| A                              |
+--------------------------------+
1 row in set (0.02 sec)

mysql> select substring_index('A,B', ',', 2);
+--------------------------------+
| substring_index('A,B', ',', 2) |
+--------------------------------+
| A,B                            |
+--------------------------------+
1 row in set (0.02 sec)
```

所以逐次获得1，2，...n 个 uri 的语句就是：

```text
select  substring_index(substring_index(uri,',', t.id),',',-1)
```

4.合起来的整条语句：

```text
select  substring_index(substring_index(uri,',', t.id),',',-1)  from tbl_org d join tina_num t on 1=1 where t.id <= (length(uri)-length(replace(uri,',','')) + 1);
```



