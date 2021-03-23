# 执行日志

## plan 运行日志

![plan &#x65E5;&#x5FD7;&#x5217;&#x8868;](../../.gitbook/assets/image%20%2818%29.png)

点击左边栏“测试计划管理”，进入 plan 列表，点击“日志”，查看所有历史运行日志：

* 支持多维度筛选
* 可以在  plan 级别看到失败原因（点击“+”号）
  * 如果 plan 中有多个 case 失败，展示第一个的失败原因
  * 如果是流程故障，比如某些内置检查失败，也可以在这里查看

点击“详情”，查看本次运行的 flow 运行日志。

**接口类型**：plan 选择何种协议运行，则该 plan 下的所有 case 都以这种协议执行接口。

**触发方式**：本次运行有什么触发，当前支持：手动、定时（cron）、CI 三种。

## flow 运行日志

![flow &#x8FD0;&#x884C;&#x65E5;&#x5FD7;](../../.gitbook/assets/image%20%2813%29.png)



## case 运行日志

### case 日志列表

![case &#x65E5;&#x5FD7;&#x5217;&#x8868;](../../.gitbook/assets/image%20%285%29.png)

### case 日志详情

case 日志详情分为 3 部分：基础信息、请求部分、返回和检查部分

#### 基础信息

![](../../.gitbook/assets/image%20%2817%29.png)

**case 版本**：

实际运行的 case 版本，依据[版本控制](https://liu-tongtong.gitbook.io/dba/siber-ji-cheng-ce-shi-ping-tai/cao-zuo-zhi-nan/pei-zhi-case#case-ban-ben-shuo-ming)，plan 会执行该 case 不高于它的最高版本。那究竟执行的哪版是需要记录的，因为 case 和 plan 都在不断的发展变化。我们需要记录当时的现场。

**请求地址**：

* HTTP ：通常是环境配置和接口配置 uri 的拼接
* GRPC：通常是 envoy 代理地址，或者域名



#### 请求**部分**

![case &#x8FD0;&#x884C;&#x65E5;&#x5FD7;&#xFF1A;&#x8BF7;&#x6C42;&#x90E8;&#x5206;](../../.gitbook/assets/image%20%2825%29.png)

**请求模板**：用户的配置项，比如使用的自定义鉴权、FUNCTION、VARIABLE

**请求列表**：parameter 渲染后的结果，**实际请求内容**

\*\*\*\*

#### 返回和检查部分

![](../../.gitbook/assets/image%20%2821%29.png)

**状态**：接口调用的状态码值

**响应时间**：接口调用所耗时间

**返回头、返回体**：接口实际上的返回内容

**action 执行结果**：注入项、检查项、睡眠的执行结果

* 可以清晰的看到执行状态：SUCCESS、FAILED
* 可以清晰的看到配置内容：检查的是什么，期望值是什么，关系是什么（相等、包含……）
* 可以清晰的看到执行完本 case 后消耗的时间

