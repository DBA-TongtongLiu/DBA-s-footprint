# 配置case

![siber case &#x914D;&#x7F6E;&#x754C;&#x9762;](../../.gitbook/assets/image.png)

## 基础配置

### **case 名称**

自定义的 case 名称，建议：见名知意。

case 名称全局唯一。

### **所在的 method**

在 method 管理界面注册过的method。

如果是通过 proto file 或者 graphQL 注册的method，会自动渲染初始结构。字段值默认为该类型的空值，或者指定的默认值：

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

## 

## 

## 

## 

## 

## 

## 

## case tag

## case 版本说明



