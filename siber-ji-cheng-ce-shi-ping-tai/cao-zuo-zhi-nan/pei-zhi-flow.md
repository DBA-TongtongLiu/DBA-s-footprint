# 配置 flow

![&#x7F16;&#x8F91;&#x6D4B;&#x8BD5;&#x573A;&#x666F;](../../.gitbook/assets/image%20%285%29.png)

## 测试场景（flow）介绍

flow 是一系列测试用例（case）的合集，flow 中的 case 按顺序执行。

## 编辑 flow

点击左边栏“测试场景管理”进入编辑界面。

**flow 名称**：自定义 flow 名称。建议：见名知意。

**case 运行控制**：

* 错误终止：执行 flow 时，case 失败，则停止运行整个 flow。
  * 适用于强依赖之前 case 成功的 flow
* 错误忽略：执行 flow 时，case 失败，flow 继续执行，直至所有 case 均被执行。
  * 适用于没有依赖关系的 case 集合

**case 列表：**已配置的 case 列表，支持按关键字筛选，支持多选。

**设置流程**：为本 flow 添加的 case 列表，可上下拖动调整顺序。

**case 设置**：展示选择的 case 配置详情，只可查看，不可修改。

## 复制 flow

在 flow 列表页，点击“复制”，进行 flow 复制。除名称和创建及最后修改时间不一样外，所有信息与 源flow 保持一致。

复制出的 flow 名称为：源 flow 名称 + 时间戳。



