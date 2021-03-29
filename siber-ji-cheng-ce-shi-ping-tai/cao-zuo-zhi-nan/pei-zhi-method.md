# 配置 method

当前 siber 支持三种格式的接口配置：HTTP，gRPC，GraphQL

## GRPC

![gRPC method &#x914D;&#x7F6E;](../../.gitbook/assets/image%20%2810%29.png)

**proto 文件相对路径**：

来也科技的所有 proto 文件，是放在一个仓库中管理的。这个路径就是要添加的method 所在的 proto 的相对路径。

siber会实时从仓库中拉取。

**method 列表**：

选好 proto file 后，点击“解析”按钮，siber 会自动解析 proto file 中的所有 method， 并以列表形式返回展示。

选中需要的 method 即可。

**method 描述**：仅展示，不可修改

这部分的所有信息都是从proto file 文件中解析出来的，配置者用户可以在这里做一个清晰的查看，而不必回头对照 proto file。

这部分信息不做保存，proto file 随着版本迭代有所变化时，这部分信息会随之变化。

来也科技设计的 proto 都是向后兼容的。可以认为最高版本的 proto 支持所有版本的 case 运行。如若不然，需要切到指定版本分支上运行。

## HTTP

![http &#x63A5;&#x53E3;&#x914D;&#x7F6E;&#x754C;&#x9762;](../../.gitbook/assets/image%20%2815%29.png)

**http 请求方式**：选择 post、get、put、copy ……等请求方式

**uri**：访问地址，与环境配置中的环境拼接为一个完整的 url

**method 名称**：自定义名称，全局唯一

* 第一段：默认Siber，如果公司大的话，可以填部门名称
* 第二段：产品线名称，例如：OpenAPI、Mage、Commander……
* 第三段：用户自定义，建议清晰明了



## GraphQL

![GraphQL &#x63A5;&#x53E3;&#x914D;&#x7F6E;&#x754C;&#x9762;](../../.gitbook/assets/image%20%284%29.png)

GraphQL  method 支持两种录入方式：

* 手动配置
* GraphQuery 注入：所有信息均只可查看，不可修改。

