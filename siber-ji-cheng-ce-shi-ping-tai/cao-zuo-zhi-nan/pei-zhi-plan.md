# 配置 plan

![plan &#x7F16;&#x8F91;&#x754C;&#x9762;](../../.gitbook/assets/image%20%2814%29.png)

## 测试场景（plan）介绍

plan 是一系列 flow 的合集，flow 间并发执行。如有任一 flow 失败，则 plan 整体状态置为失败。但 flow 的失败不会影响 plan 的运行，在所有的 flow 都运行至最终状态（成功或失败）后，plan 方才结束运行。

## 自动执行

应用\(process\) 上到测试环境后，siber 会自动触发所有相关的 plan。

自动执行不用配置，有 siber 根据 uri（http接口）及 method 信息（grpc）接口自动进行绑定。

plan 中包含任意该服务的 case，则定义为相关 plan。

关联关系:  plan -&gt; flow -&gt; case -&gt; method -&gt; process

## 编辑 plan

**plan 名称**：自定义 plan 名称，建议见名知意。

**plan 版本**：plan 中的 case 会执行不高于此版本的最高版本，详见：[case 版本](https://liu-tongtong.gitbook.io/dba/siber-ji-cheng-ce-shi-ping-tai/cao-zuo-zhi-nan/pei-zhi-case#case-ban-ben)

* 通常跟产品迭代统一
* 私有部署时，为私有部署的版本
* 回滚时，为回滚的目标版本

**接口类型**：想要以什么协议运行该 plan 下的 case，详见：[可适配的 case 协议](https://liu-tongtong.gitbook.io/dba/siber-ji-cheng-ce-shi-ping-tai/chan-pin-gai-shu#ke-kuo-pei-de-case-xie-yi)

* 接口类型一旦选定，该 plan 下的所有 case 均按照此协议执行。

**环境配置**：即产品线名称，选中后，默认该 plan 可在该产品线下的所有环境（例如：开发、测试、灰度、生产）执行。

**执行顺序**：

*  顺序执行：即按顺序执行 flow，但即便 flow 失败，plan 也不会终止。如果有强依赖的需求请将这些 case 配置在同一个 flow 中。
  * 有些服务配置低，或是有限流，并发执行会有意料外的错误。此时建议选择顺序执行
* 并发执行：默认并发执行，提高执行效率

服务名称： siber 自动关联的项目名称，与[强制执行](https://liu-tongtong.gitbook.io/dba/siber-ji-cheng-ce-shi-ping-tai/cao-zuo-zhi-nan/pei-zhi-qiang-zhi-zhi-hang)中的 process 名称一致。

**运行配置**：

![plan &#x8FD0;&#x884C;&#x914D;&#x7F6E;](../../.gitbook/assets/image%20%2816%29.png)

* 运行环境：指开发、测试、灰度、生产等，可以分别为不同的环境设置不同的 crontab 触发和 CI 触发
* 定时触发：即 crontab 触发，详情参考[配置方法](https://www.jianshu.com/p/626acb9549b1)，或[官方文档](https://godoc.org/github.com/robfig/cron)
* 服务发布触发：即 CI 触发，作为[强制执行](https://liu-tongtong.gitbook.io/dba/siber-ji-cheng-ce-shi-ping-tai/cao-zuo-zhi-nan/pei-zhi-qiang-zhi-zhi-hang)和[自动执行](https://liu-tongtong.gitbook.io/dba/siber-ji-cheng-ce-shi-ping-tai/cao-zuo-zhi-nan/pei-zhi-plan#zi-dong-zhi-hang)的补充

**flow 列表**：

* 左边框内为所有已配置 flow
* 右边框内为该 plan 所包含的 flow
* 可任意增删

## 复制 plan

在 plan 列表页，点击“复制”，进行 plan 复制。除名称和创建及最后修改时间不一样外，所有信息与 plan 保持一致。

复制出的 plan 名称为：源 plan 名称 + 时间戳。

