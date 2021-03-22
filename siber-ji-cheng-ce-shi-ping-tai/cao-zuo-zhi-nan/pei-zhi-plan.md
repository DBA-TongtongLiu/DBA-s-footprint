# 配置 plan

![](../../.gitbook/assets/image%20%287%29.png)

## 测试场景（plan）介绍

plan 是一系列 flow 的合集，flow 间并发执行。如有任一 flow 失败，则 plan 整体状态置为失败。但 flow 的失败不会影响 plan 的运行，在所有的 flow 都运行至最终状态（成功或失败）后，plan 方才结束运行。

## 编辑 plan

**plan 名称**：自定义 plan 名称，建议见名知意。

**plan 版本**：plan 中的 case 会执行不高于此版本的最高版本，详见：[case 版本](https://liu-tongtong.gitbook.io/dba/siber-ji-cheng-ce-shi-ping-tai/cao-zuo-zhi-nan/pei-zhi-case#case-ban-ben)

* 通常跟产品迭代统一
* 私有部署时，为私有部署的版本
* 回滚时，为回滚的目标版本

**接口类型**：想要以什么协议运行该 plan 下的 case，详见：[可适配的 case 协议](https://liu-tongtong.gitbook.io/dba/siber-ji-cheng-ce-shi-ping-tai/chan-pin-gai-shu#ke-kuo-pei-de-case-xie-yi)

* 接口类型一旦选定，该 plan 下的所有 case 均按照此协议执行。

**环境配置**：即产品线名称，选中后，默认该 plan 可在该产品线下的所有环境（例如：开发、测试、灰度、生产）执行。

执行顺序：

