# 生成达梦的数据初始化脚本

## 背景介绍

为适应**国产化适配**的需求，我们选择使用达梦（`DM8` ）数据库替代 MySQL 做为国产化版本产品方案的存储。

我们最终的导数方案：

1. 将表结构文件导到`MySQL`中做中转
2. 使用达梦自带的 `DTS` 工具将表结构和数据导入至达梦中
3. 在达梦中对表结构进行修改。
4. 使用脚本对表结构和数据进行校验
5. 使用达梦自带的 `DTS` 将表结构和数据导出为SQL脚本

**为何选择这样的一个方案呢？**

这是因为 MySQL 和 DM 语法不同且各自都有一定的自身限制。 DTS 工具虽带有一定的字段长度映射功能，但多次测试发现其并未生效，若逐一对字段进行修改，又过分浪费人力。 尤其在测试阶段，频繁的导入导出操作，更是得不偿失。

我将在接下来的每个步骤中解释选择这个导数方案的原因和必要性。   


## MySQL 和 DM 的语法差别

### 为表和字段添加注释的方法不同

\[MySQL\] 直接写在表、字段定义的后面，例如:

```text
CREATE TABLE `tbl_test` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  ...

  `auto_create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '记录创建时间',
  `auto_update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '记录更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='测试用表'
```

\[DM\] 使用单独语句添加:

```text
COMMENT ON TABLE "TEST"."TBL_TEST" IS '测试用表';
...

COMMENT ON COLUMN "TEST"."TBL_TEST"."ID" IS '自增主键';
```

### 特殊表名、列名的声明

有的时候，设计不够规范，表名、列名可能与数据库的保留字重名。

\[MySQL\] 通常使用反引号，比如：

```text
mysql> CREATE TABLE `desc` ( `desc` varchar(256) );
Query OK, 0 rows affected (0.02 sec)

mysql> desc `desc`;
+-------+--------------+------+-----+---------+-------+
| Field | Type         | Null | Key | Default | Extra |
+-------+--------------+------+-----+---------+-------+
| desc  | varchar(256) | NO   |     |         |       |
+-------+--------------+------+-----+---------+-------+
1 row in set (0.02 sec)
```

\[DM\] 通常使用双引号（用反引号直接报错），比如：

```text
CREATE TABLE "desc" ( "desc" varchar(256) );
```

### 将 varchar 类型修改为 text 类型的语法

\[MySQL\] 直接使用 alter 命令修改：

```text
mysql> alter table tbl_test modify `name` text;
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

\[DM8\] 无法通过命令将已定义的表中字段由 varchar 修改为 text，会报错：`-6160：数据类型的变更无效`。

**必须要 drop 表后重建才行**，同类的还有 blob，clob 等类型。   


### varchar 类型长度含义不同

\[MySQL\] 假设现在表 tbl\_test 有 varchar\(128\) 的字段 name, 我们是真正的可以把 128 个英文字符、汉字字符、emoji 字符插入到该字段中的。 如下：

```text
-- 创建测试表
CREATE TABLE `tbl_test` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(128) COLLATE utf8mb4_bin DEFAULT NULL
) ENGINE=InnoDB


-- 插入测试数据，分别为英文字符、汉字字符、emoji 字符
-- 注意 client 、connection 、column 字符集都要求是 utf8mb4

mysql> insert into tbl_test values (1,repeat('a', 128)), (2,repeat('中',128)), (3,repeat('😄',128));
Query OK, 3 rows affected (0.03 sec)
Records: 3  Duplicates: 0  Warnings: 0


-- 查看字符长度和字节长度
mysql> select  id, char_length(name), length(name) from tbl_test;
+------+-------------------+--------------+
| id   | char_length(name) | length(name) |
+------+-------------------+--------------+
|    1 |               128 |          128 |
|    2 |               128 |          384 |
|    3 |               128 |          512 |
+------+-------------------+--------------+
3 rows in set (0.02 sec)
```

可以看到：在 utf8mb4 字符集下，英文占用1个字节，一般汉字占3个字节，emoji表情占4个字节。 **MySQL 的 varchar 长度显然是以字符长度为准的**

\[DM\] 达梦中则不然

```text
-- 创建测试表
CREATE TABLE tbl_test ( id int DEFAULT NULL,  name varchar(128))

-- 插入英文字符
SQL> insert into tbl_test values (1,repeat('a', 128))
SQLRowCount returns 1

-- 插入中文字符（报错）
insert into tbl_test values (2,repeat('中', 128))
执行失败：列[NAME]长度超出定义

-- 测试发现，中文字符修改为  <= 128/3 的长度可插入
insert into tbl_test values (2,repeat('中', 42))

-- 查看字符长度和字节长度
SQL> SELECT ID,LENGTH(name),CHAR_LENGTH(name),OCTET_LENGTH(name) FROM tbl_test;
+------------+-------------+------------------+-------------------+
| ID         | LENGTH(name)| CHAR_LENGTH(name)| OCTET_LENGTH(name)|
+------------+-------------+------------------+-------------------+
| 1          | 128         | 128              | 128               |
| 2          | 42          | 42               | 126               |
+------------+-------------+------------------+-------------------+
SQLRowCount returns 2
2 rows fetched
```

可以看出： 

1. **达梦中 varchar 的长度计算的是字节数** 

2. 所以，达梦中 varchar\(128\) 是存不下 128 个汉字的，最多只能存 42 个（跟字符集有关）。 

3. 达梦中的 length 函数计算的也是字符数，如果要计算字节数，应该使用函数：OCTET\_LENGTH

## 将表结构文件导到`MySQL`中做中转

**为什么不直接生成 `DM` 适配的数据库初始化脚本？**

如上文所述，DM 和 MySQL 语法略有不同，是不能将 MySQL 导出的 sql 语句直接导入到 DM 中直接使用的。

试图在 sql 语句层面转化，必然需要调研、测试甚至是开发、改写转化脚本，比较麻烦，也难免会有遗漏。

既然 DM 软件包中提供了 DTS，我们也愿意相信官方的工具，就没必要浪费那个人力了。

**为什么选择 MySQL 作为中转？**

* 一直以来我们都是使用 MySQL 作为标准版的数据库，所以无论如何都要准备 MySQL 版本的数据库初始化脚本。
* 在开发和测试过程中，我们主要基于 MySQL 进行，所以库、表、数据可以说是现成的，只需要做些标准化处理，简单易得。

中转操作： 

1. 将数据初始化脚本，使用 source 命令导入到 MySQL 中 

2. 将长度超过 975 的 varchar 类型字段修改为 text 类型



### **为什么要将 &gt;975 的 varchar 转为 text？**

我们可以通过 `varchar 类型长度含义不同` 这个模块看到， 为了让 MySQL 中的数据能够顺利的导入到 DM 中，我们需要对 varchar 类型的字段做 `扩充` 处理。 扩充后，有些字段会超出长度限制，所以要升级为 text 类型

**为什么选择 975 这个长度？**

DM 对于 varchar 最大长度的支持，和页的大小有关：

* 8k 的页，varchar 最大长度是：3900
* 16k 的页， varchar 最大长度是：8000

我们的页大小是 16k 的，最长是可以支持到 8000 的。 只是 toB 业务，要在影响范围可控的情况下，考虑到兼容更多的客户的环境，所以我们设 3900 为上限。

按 MySQL 中最占字节的 emoji 字符计算： 3900/4 = 975 （emoji 1 个字符占 4 个字节）

且，经过排查可以确定，产品对 &gt;975 的这些字段没有 `索引`、`条件查询` 等需求，所以改成 text 也不会有严重影响。

**为什么要在 MySQL 中做 text 转化操作？**

在 `将 varchar 类型修改为 text 类型的语法` 部分，已经向大家阐述， DM 是无法进行这个修改的，所以这步必须在 MySQL 中完成。

在 MySQL中执行如下命令：

```text
-- 从系统表中查看长度超过 975 的字段都有哪些
select  * from information_schema.columns where table_schema = 'test' and data_type = 'varchar' and CHARACTER_MAXIMUM_LENGTH > 975;

-- 生成修改语句
select concat('alter table ',TABLE_SCHEMA, '.', TABLE_NAME, ' MODIFY ' ,COLUMN_NAME, ' TEXT;')  from information_schema.columns where table_schema = 'test' and  DATA_TYPE = 'varchar'  and CHARACTER_MAXIMUM_LENGTH > 975;

-- 执行生成的修改语句
alter table test.test_tbl MODIFY person_desc TEXT;
alter table ...
```

## 在达梦中对表结构进行修改

### 将 char 类型字段修改为 varchar 类型

**为什么需要将 char 改为 varchar？**

在测试的过程中，我们发现了这样的一个问题：DM8 对 char 类型的字符串在末尾给补了空格，致使数据内容与实际有所出入。

请看下面的示例：

\[DM8\] 现在有表 tbl\_test 里面的字段 name 为 char 类型，定义的长度为 64，使用 `isql` 工具进行查看， 可以看到不足 64 位的 name 被补了空格：

```text
SQL>  select concat('--',name,'--') from tbl_test where id = 1;
+-----------------------------------------------------------------------------------+
| concat(concat('--',name),'--')                                                    |
+-----------------------------------------------------------------------------------+
| --name000000000000000000                                          --              |
+-----------------------------------------------------------------------------------+
```

经测试发现**这些空格实际上是在查询的时候补上的**，而非就是这样存储的。

验证方法，使用 `update` 对全表的 name 字段去空格，之后再次查询（结果没有一丝丝改变）：

```text
SQL> update tbl_test set name = trim(name)
SQLRowCount returns 1741

SQL>  select concat('--',name,'--') from tbl_test where id = 1;
+-----------------------------------------------------------------------------------+
| concat(concat('--',name),'--')                                                    |
+-----------------------------------------------------------------------------------+
| --name000000000000000000                                          --              |
+-----------------------------------------------------------------------------------+
```

对于这个问题，我们有三个解决方案： 

1. 找找系统参数，看有没有可以不补0的

2. 把字段长度修改为值的实际长度：32 

3. 把char修改为varchar，规避掉这个问题

对于方案1，由于我们做的是 toB 业务，客户环境千奇百怪，有的是死活不让你改配置，这么做就是给自己留坑。所以不选择。

对于方案2，由于我们的业务和体量，varchar 和 char 对我们而言无论是从性能还是从使用上区别都不大，考虑后续还有可能会有超过32位的数据，所以直接选择方案3，一劳永逸。

**为什么选择在 DM 中做这个操作？**

因为我希望 MySQL 中的表结构最能够贴近业务的需求，最能够保持"原汁原味"，不要因为后续的适配而被随意修改。

具体修改方法：

```text
-- 从系统表中查出所有 char 类型的字段
SELECT * FROM SYS.DBA_TAB_COLUMNS WHERE OWNER ='TEST' AND DATA_TYPE = 'CHAR';


-- 生成修改语句
select
        concat('alter table ', owner, '.', table_name, ' modify ', column_name, ' varchar(', data_length, ');')
FROM
        SYS.DBA_TAB_COLUMNS
WHERE
        OWNER     ='TEST'
    AND DATA_TYPE = 'CHAR'


-- 批量执行生成出来的修改语句
alter table TEST.TBL_TEST modify NAME varchar(64);
alter table ...
```

### 将 varchar 类型字段长度扩大为之前的四倍

同：上文 `varchar 类型长度含义不同` 和 `MySQL 中转` 部分所述

**为什么选择在 DM 中做这个操作？**

除了希望MySQL中的表结构"原汁原味"外，还有两个重要原因：

1.受到了索引的约束。MySQL 中的索引限制：

> 唯一索引的字段长度要小于 757 , 组合索引要小于 3072

贸然增加字段长度，可能会引发 `ERROR 1071 (42000): Specified key was too long; max key length is 3072 bytes` 报错

2.使用 modify 在 MySQL 中修改字段长度会覆盖掉其他约束和属性。比如：

```text
-- 创建带有非空约束、默认值、注释的表：
create table tinatest (id int, name varchar(30) not null default 'tina' comment 'tina');

-- 修改字段长度：
alter table tinatest modify name varchar(120);

-- 查看表结构，可以看到：约束、默认值、注释都没了
mysql> show create table tinatest;
+----------+-----------------------------------------------------------+
| Table    | Create Table                                              |
+----------+-----------------------------------------------------------+
| tinatest | CREATE TABLE `tinatest` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(120) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+----------+-----------------------------------------------------------+
1 row in set (0.02 sec)
```

而 DM 中没有这个问题：

```text
-- 列的初始状态
"NAME" VARCHAR(128) DEFAULT 'TEST' NOT NULL,

-- 修改列的长度
ALTER TABLE TEST.TBL_TEST  MODIFY NAME VARCHAR(496)

-- 当前列的状态
"ACCOUNT_KEY" VARCHAR(496) DEFAULT 'TEST' NOT NULL,
```

可以看到 default 值，和 not null 约束都还在。

达梦中操作:

```text
-- 生成修改语句

SELECT
        concat('alter table ', owner, '.', table_name, ' modify ', column_name, ' varchar2(', DATA_LENGTH*4, ');' )
FROM
        SYS.DBA_TAB_COLUMNS
WHERE
        OWNER ='TEST'
    AND
        (
                DATA_TYPE ='VARCHAR'
             OR DATA_TYPE ='VARCHAR2'
        )
    AND DATA_LENGTH <=975


-- 批量执行修改语句
alter table TEST.TBL_TEST modify NAME VARCHAR2(512);
alter table ...
```

**注意：** 

1. 修改 char -&gt; varchar 

2. 修改长度超过 975 的 varchar -&gt; text 

3. 修改长度小于等于 975 的 varchar length \*4

**这三步要按顺序执行**

## 使用脚本对表结构和数据进行校验

表结构和约束校验： 我们使用 MySQL 和 DM 中的系统表对表结构和约束做校验。常用的系统表、视图：

MySQL: 

1. information\_schema.tables 表信息 

2. information\_schema.columns 列信息 

3. information\_schema.statistics 索引信息



DM: 

1. SYS.All\_ALL\_TABLES 表信息 

2. SYS.DBA\_TAB\_COLUMNS 列信息 

3. SYS.DBA\_INDEXES 索引信息

  
数据校验：

我之前 fork 了 beego ORM，开发了兼容达梦版本，我们的数据校验就是使用这个 ORM 同时连接 DM 和 MySQL ，进行数据的查询和比对的。之后这个项目会开源，欢迎大家使用。

