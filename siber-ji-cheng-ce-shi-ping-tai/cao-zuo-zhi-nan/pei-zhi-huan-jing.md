# 配置环境

![&#x73AF;&#x5883;&#x914D;&#x7F6E;&#x754C;&#x9762;](../../.gitbook/assets/image%20%286%29.png)

用于配置不同产品线的不同环境。诸如：

* 吾来：（开发、测试、灰度、生产）\* （grpc、http）
* Mage：（开发、测试、灰度、生产）\* （grpc、http）
* ……



**配置名称**：自定义的环境名称，建议见名知意

**gRPC、HTTP**：参考[可适配的 case 协议](https://liu-tongtong.gitbook.io/dba/siber-ji-cheng-ce-shi-ping-tai/chan-pin-gai-shu#ke-kuo-pei-de-case-xie-yi)

* 如该产品线支持两种协议，则两种均配
* 如仅支持一种协议，则只配一种即可
* 也可以只配一个环境，比如：测试
* 总之就是用到什么配置什么就行，其他的可以为空

**应用配置**：当前支持 pubkey 和 secret 的配置





