# 配置 case

![siber case &#x914D;&#x7F6E;&#x754C;&#x9762;](../../.gitbook/assets/image.png)

## 基础配置

### **case 名称**

自定义的 case 名称，建议：见名知意。

case 名称全局唯一。

### **所在的 method**

在 method 管理界面注册过的method。

如果是通过 proto file 或者 GraphQuery 注册的method，会自动渲染初始结构。字段值默认为该类型的空值，或者指定的默认值：

```text
{
    "Token": "",
    "Url": "",
    "Ip": "",
    "System": "IM"
}
```

### 打标签

自定义标签，可用于标识：所属模块，使用范围等。

在 case 列表界面，可以根据标签进行搜索。

在做统计分析时，可以按照标签作为维度分析。

## case 设置

### case 版本

比如：接口 CreateUser 于2020年初上线 V1.0 版本后就一直没有变化，同时接口 UpdateUser 也于 2020年初上线 V1.0 版本，但分别于 2020 年末和上个月上线了 V1.1 和 V1.2 版本。

那么在配置 case 时，只需要对 UpdateUser 接口增加 V1.2 版本的配置。CreateUser 接口无需任何变动。

此时，如果 plan 选择 V1.2 版本运行，则会运行 V1.2 版本的 UpdateUser 和 V1.1 版本的 CreateUser；

如果 plan 选择 V1.1版本运行（比如私有部署，比如回滚回归），则会运行 V1.1 版本的 UpdateUser 和 V1.1 版本的 CreateUser。

即：**plan 会运行不高于自己版本的最高版本 case 。**

### **url 参数**

在许多 http（get、post） 请求中，参数是拼接在url 中的。此时，url 需要可以被参数渲染，例如，Commander \(来也科技业务线之一，详见：UIBot\)，在 siber 上的测试案例：“获得流程下所有数据指标统计” 中对于 url 参数的配置:

`?flowid={{VARIABLE.flow_id}}show`

**url 参数支持 parameter 渲染（ FUNCTION 和 VARIABLE 的解析）。**

### 请求头

支持 http（包括 GraphQL）、gRPC 两种协议的 request header。

**request header 支持 parameter 渲染（ FUCTION、VARIABLE 和 SiberAuth 的解析）。**

#### **SiberAuth**

使用 SiberAuth 可以支持对接口进行鉴权。当前已内置通用鉴权算法，可以自己在 siber 上添加自定义的鉴权算法。

通常而言，一个测试场景，我们倾向于使用一个用户从头测到尾（不妨碍你下次执行的时候换个不同权限的用户）。对于一个产品线，我们可以配置一个或多个用户进行测试。

**自定义鉴权算法的结构**

![&#x81EA;&#x5B9A;&#x4E49;&#x9274;&#x6743;&#x914D;&#x7F6E;&#x65B9;&#x5F0F;](../../.gitbook/assets/image%20%283%29.png)

**三段式的 Key：**（以\#分隔）

Siber\#siber\_open\_source\#siberOpenSource

Siber : 用于标识产品线

siber\_open\_source：环境名，在环境管理中配置

siberOpenSource：应用配置名称，在环境管理中配置

**标识算法的 Value：**

SiberAuth 为内置鉴权算法（详见源码），如果有其他鉴权算法，诸如公司内部的自定义的算法，可以自由在源码上添加。

我们也鼓励大家向 Siber 提交代码。

**SiberAuth 的渲染示例：**

图片左边为请模板，及用户配置的内容。

右边为渲染后的内容，即实际发送的请求内容。

![SiberAuth &#x6E32;&#x67D3;&#x6848;&#x4F8B;](../../.gitbook/assets/image%20%281%29.png)

pubkey 在环境中的配置：

![&#x73AF;&#x5883;&#x4E2D;&#x914D;&#x7F6E; pubkey &#x548C; secret](../../.gitbook/assets/image%20%282%29.png)

### 上传文件

当前 siber 支持对图片做 base64 编码，图片将会上传至 oss 或者 minio 上（取决您配置的是什么）。

上传成功后，前端会返回图片所在路径，然后使用 siber 的 FUNCTION进行说明，下面是一个 request body 的示例：

```text
{
    "img_base64": "{{FUNCTION.base_64(https://laiye-im-saas.oss-cn-beijing.aliyuncs.com/e04f3d27-037c-4b86-9dd0-706a559aaf88.jpeg)}}"
}
```

### 请求体

request body。**支持 parameter 渲染（ FUCTION、VARIABLE 和 SiberAuth ）。**

如果是通过 proto file 或者 GraphQuery（来也科技内部服务），配置的 method，在配置 case 基础信息的时候会自动渲染初始化 request body。字段值默认为该类型的空值，或者指定的默认值。

用户在此基础上修改即可。

**request body 支持 parameter 渲染（ FUCTION、VARIABLE 的解析）。**

### **注入项**

将指定内容保存到变量中，在同一 flow 的后续 case 中，使用 `{{VARIABLE.***}}` 的格式进行访问。

variable 的值可以来自：

1. request header、request body 全部或部分内容
2. response header 、response body 全部或部分内容
3. response status ，即接口响应码值
4. response time，即接口响应时间

**注意：**

* 同一个 flow 中，variable 名称唯一，如果同一个 variable 被保存两次，则后面的覆盖掉前面的。
* variable 在 flow 内共享，不在 flow 之间共享。

### **检查项**

检查指定内容是否符合预期，检查内容包括：

1. request header、request body 全部或部分内容
2. response header 、response body 全部或部分内容
3. response status ，即接口响应码值
4. response time，即接口响应时间

检查方式包括：

1. `=` 等于，支持数字、字符串、数组、嵌套数组、map 等
2. `!=` 不等于，支持数字、字符串、数组、嵌套数组、map 等
3. `>` 大于，支持数组、字符串
4. `>=` 大于等于，支持数组、字符串
5. `<` 小于，支持数组、字符串
6. `<=` 小于等于，支持数组、字符串
7. `exist` 存在，判断检查的内容中是否存在指定的 key
8. `length` 长度：支持字符串长度、数组类型
9. `in` 检查的内容是否被包含在指定内容中，支持字符串、数组类型 
10. `include` 检查的内容是否包含指定内容，支持字符串、数组类型
11. `not include` 检查的内容不包含指定内容为符合预期，支持字符串、数组类型

### **睡眠时间**

执行完此 case 后，休眠多长时间进行后续操作。

### 摘取指定内容

siber 内部使用 [Gjson](https://github.com/tidwall/gjson) 来摘取指定内容，感兴趣的同学，可以点击链接查看详细信息。这里做些简单的示例，假设存在 json 如下：

```text
{
    "name": {
        "first": "Tom",
        "last": "Anderson"
    },
    "age": 37,
    "children": [
        "Sara",
        "Alex",
        "Jack"
    ],
    "fav.movie": "Deer Hunter",
    "friends": [
        {
            "first": "Dale",
            "last": "Murphy",
            "age": 44,
            "nets": [
                "ig",
                "fb",
                "tw"
            ]
        },
        {
            "first": "Roger",
            "last": "Craig",
            "age": 68,
            "nets": [
                "fb",
                "tw"
            ]
        },
        {
            "first": "Jane",
            "last": "Murphy",
            "age": 47,
            "nets": [
                "ig",
                "tw"
            ]
        }
    ]
}
```

使用以下命令获得相关内容：

```text
"name.last"          >> "Anderson"
"age"                >> 37
"children"           >> ["Sara","Alex","Jack"]
"children.#"         >> 3
"children.1"         >> "Alex"
"child*.2"           >> "Jack"
"c?ildren.0"         >> "Sara"
"fav\.movie"         >> "Deer Hunter"

```

```text
"friends.#.first"    >> ["Dale","Roger","Jane"]
"friends.1.last"     >> "Craig"
friends.#(last=="Murphy").first    >> "Dale"
friends.#(last=="Murphy")#.first   >> ["Dale","Jane"]
friends.#(age>45)#.last            >> ["Craig","Murphy"]
friends.#(first%"D*").last         >> "Murphy"
friends.#(first!%"D*").last        >> "Craig"
friends.#(nets.#(=="fb"))#.first   >> ["Dale","Roger"]
```

## 复制 case

在 case 列表页，点击“复制”，进行 case 复制。除名称、创建时间、最后修改时间不一样外，所有信息与源 case 保持一致。

复制出的 case 名称为：源 case 名称 + 时间戳。

