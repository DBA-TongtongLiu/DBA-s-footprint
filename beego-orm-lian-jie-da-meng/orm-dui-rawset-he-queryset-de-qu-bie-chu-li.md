# orm 对 rawSet 和 querySet 的区别处理

beego orm 支持两种查询：

[raw sql](https://beego.me/docs/mvc/model/rawsql.md)

```text
type User struct {
    Id   int
    Name string
}

var users []User
num, err := o.Raw("SELECT id, name FROM user WHERE id = ?", 1).QueryRows(&users)
```

[ORM](https://beego.me/docs/mvc/model/query.md)

```text
var user User
err := o.QueryTable("user").Filter("name", "slene").One(&user)
```

通常我们建议使用 orm，不仅规范，而且安全，比如：可以防止 SQL 注入。参考文档：（补简书连接）

可有些时候，SQL 语句过于复杂，或者出于其他可能考虑，我们必须要使用 raw sql，如果您使用的是 MySQL 数据库，那么不需要任何适配，beego-orm 已经做得很好了。

但，假如您使用的是达梦，或是任何自动将表名和列名转化为大写的数据库，则可以参考本文的处理方式。

在 beego-orm 适配达梦，[名列名大小写问题](https://liu-tongtong.gitbook.io/dba/beego-orm-lian-jie-da-meng#biao-ming-lie-ming-da-xiao-xie-de-wen-ti)这个模块，我们阐述了自己遇到的问题和解决办法，接下来我们给大家分析下这两种查询方式并同步下我们踩过那些坑：

## MySQL 和 DM 表、列名大小写背景知识

在我们的环境中，二者的表名和列名都是大小写不敏感的。

MySQL 中是使用 `lower_case_table_names` 参数来控制：

```text
mysql>  show global variables like 'lower_case_table_names';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| lower_case_table_names | 1     |
+------------------------+-------+
1 row in set (0.02 sec)
```

顾名思义，我们可以知道，MySQL 是默认把表名、列名中的大写字母转化为小写字母的。

达梦则不然，达梦是默认将表、列名转化为大写字母的。

如果你想使用小写名称，需要在建表时使用双引号固定下来。在查询的时候也要使用双引号，不然就可能汇报表、列不存在的问题，比较麻烦。

由于数据库本身已设置大小写不敏感，所以我们不必担心会同时存在：Table 和 table 两张表，或者同一个表中存在 Column 或者 column 两个列的情况，因为根本创建不上，会报 `duplicate name` 的错误。

在我们内部，一直是以 MySQL 作为基准开发的，在建表时有约束表名使用全小写字母加下划线的格式。但是未对应用操作 DML 做硬性要求，所以可能存在大小写混用的问题。

## RegisterModel

在使用 beego-orm 之前需要使用 RegisterModel 函数进行注册。

```text
type User struct {
    Id   int
    name string
}

func init(){
    orm.RegisterModel(new(User))
}
```

在 RegisterModel 的时候会调用  将所有列名注册为小写：

```text
func snakeString(s string) string {
   data := make([]byte, 0, len(s)*2)
   j := false
   num := len(s)
   for i := 0; i < num; i++ {
      d := s[i]
      if i > 0 && d >= 'A' && d <= 'Z' && j {
         data = append(data, '_')
      }
      if d != '_' {
         j = true
      }
      data = append(data, d)
   }
   return strings.ToLower(string(data[:]))
}
```

调用链：

1. RegisterModel -&gt; RegisterModelWithPrefix -&gt;  registerModel -&gt; newModelInfo
2. addModelFields -&gt; newFieldInfo：处理列信息，列名、类型、struct 嵌套等
3. getColumnName -&gt; snakeString：将驼峰式的命名转化为蛇形命名，比如：ColumnName 会被转化成 column\_name

所以在后续使用时，都以这种 column\_name 格式作为表中的字段名。这对 raw sql 和 orm 的查询，都造成了隐患：

## rawSet

当我们使用 raw sql 这种执行方式时，我们使用的是 `rawSet` 这个结构体，这个结构体实现了 `RawSeter` 这个接口，我们常用的：`QueryRow`，`QueryRows`，`Values` 等方法都是被包含在 RawSeter 这个接口中的：

```text
type rawSet struct {

	query string
	args  []interface{}
	orm   *orm
	
}



type RawSeter interface {

	Exec() (sql.Result, error)
	QueryRow(containers ...interface{}) error
	QueryRows(containers ...interface{}) (int64, error)
	SetArgs(...interface{}) RawSeter
	Values(container *[]Params, cols ...string) (int64, error)
	ValuesList(container *[]ParamsList, cols ...string) (int64, error)
	ValuesFlat(container *ParamsList, cols ...string) (int64, error)
	RowsToMap(result *Params, keyCol, valueCol string) (int64, error)
	RowsToStruct(ptrStruct interface{}, keyCol, valueCol string) (int64, error)
	Prepare() (RawPreparer, error)
}
```

我们以 `QueryRows` 为例来说明： 

QueryRows 会首先将 raw sql 查询出一个结果集：

```text
rows, err := o.orm.db.Query(query, args...)
```

然后遍历结果集中的每一“行”，将它们按照列的名称赋值到对应的列中：

```text
	for rows.Next() {
	
		if structMode {
			columns, err := rows.Columns()
	    ……
		}
		
		……
		
    if sMi != nil {
			for _, col := range columns {
				if fi := sMi.fields.GetByColumn(col); fi != nil {
				……
				}
			}
		……
	}
```

这个 columns 中的值是从 rows 中获得的，也就是达梦中返回的列名，诸如：**COLUMN\_NAME** 这种全大写的格式。

而 fields 中的 key 则是 `RegisterModel` 时转化的全小写的格式，诸如：**column\_name** 。

所以**这样直接操作，肯定都是空值**。我们选择将

```text
sMi.fields.GetByColumn(col)
```

修改为：

```text
sMi.fields.GetByColumn(strings.ToLower(col)
```

来解决问题。

这样做的优点：

1. 最少的代码修改量；
2. 业务无感知；
3. 不必修改达梦或是 MySQL 中的表结构，大家都知道这意味着要重头重新测试 🙃 

## querySet

当我们使用 ORM 这种执行方式时，我们使用的是 `querySet` 这个结构体，这个接口体实现了 `QuerySeter` 这个接口。我们常用的 `Filter`, `All`, `One` 等方法，都是包含在这个结构体中的。

```text
type QuerySeter interface {

	Filter(string, ...interface{}) QuerySeter
	Exclude(string, ...interface{}) QuerySeter
	SetCond(*Condition) QuerySeter
	GetCond() *Condition
	Limit(limit interface{}, args ...interface{}) QuerySeter
	Offset(offset interface{}) QuerySeter
	GroupBy(exprs ...string) QuerySeter
	OrderBy(exprs ...string) QuerySeter
	RelatedSel(params ...interface{}) QuerySeter
	Distinct() QuerySeter
	ForUpdate() QuerySeter
	Count() (int64, error)
	Exist() bool
	Update(values Params) (int64, error)
	Delete() (int64, error)
	PrepareInsert() (Inserter, error)
	All(container interface{}, cols ...string) (int64, error)
	One(container interface{}, cols ...string) error
	Values(results *[]Params, exprs ...string) (int64, error)
	ValuesList(results *[]ParamsList, exprs ...string) (int64, error)
	ValuesFlat(result *ParamsList, expr string) (int64, error)
	RowsToMap(result *Params, keyCol, valueCol string) (int64, error)
	RowsToStruct(ptrStruct interface{}, keyCol, valueCol string) (int64, error)
}
```

我们以 One\(\) 为例来说明：

使用 One\(\) 这个 function，不会不会也存在值渲染不上的情况？

答案是不会的

为什么？

因为 **One\(\) 这个function 是通过顺序进行 column 赋值的**，源码分析如下：

调用链：One -&gt; ReadBatch -&gt; setColsValues 

在 `setColsValues` 中，我们可以看到是根据 filedIndex（int类型）获得到 filed 的信息，并根据此信息将查询出来的内容赋值到对应的列中。所以大小写是影响不到 ORM 的查询的。

```text
fi := mi.fields.GetByColumn(column)

field := ind.FieldByIndex(fi.fieldIndex)
……
_, err = d.setFieldValue(fi, value, field)
```

_`All()` 也一样，因为他们都走了 `ReadBatch`_

那么是否使用了ORM 就万事大吉了呢？

ORM 也有一个注意事项，那就是在使用 Filter 的时候，注意指定的 key 要与 `RegisterModel` 后的 FieldName 保持一致：

```text
type User struct {
    Id   int
    name string
}

num, err := o.QueryTable("user").Filter("name", "slene").Delete()
```

如果 FiledName 是大写（比如通过 column 属性指定），那么 Filter 中的 key 也需要保大写。否则，会报：找不到对应的列。

```text
type User struct {
    Id   int
    name string `orm:"column(NAME)"`
}

num, err := o.QueryTable("user").Filter("NAME", "slene").Delete()
```

## 



