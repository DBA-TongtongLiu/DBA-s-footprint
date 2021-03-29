# 常见错误说明

## Selector got nil response

排查方法：

1. 检查log中response body 是否存在目标字段
2. 检查selector语法是否正确

常见原因：

* 选错接口类型/协议类型

## error getting request data：_\*_value out of range

常见原因：

* 用户测试传入异常超长值的返回

## Lack of necessary conditions

常见原因：

* 运行plan时，未传planID或运行环境

