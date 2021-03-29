# 预定义的 FUNCTION

## 随机字符串、随机数

### {{FUNCTION.random\_string\(\*\)}}

* 随机生成字符串 “\*_”号可替换对应位数_号不填
* \(\)未填写则默认生成16位字符串

### {{FUNCTION.random\_phone}}

* 随机生成手机号 （例如使用在注册场景，修改个人信息场景）
* 返回结果string类型



### {{FUNCTION.random\_uuid}}

* 随机生成UUID 例如:16fd2706-8baf-433b-82eb-8c7fada847da 
* 返回结果string类型



### {{FUNCTION.random\_url}}

* 随机生成邮箱地址 
* 返回结果string类型



### {{FUNCTION.name}}

* 随机生成姓名 
* 暂时只支持英文名称
* 返回结果string类型

## 时间函数

### {{FUNCTION.unix\_second}}

* 秒级时间戳
* 返回结果int类型



### {{FUNCTION.unix\_millisecond}}

* 毫秒级时间戳
* 返回结果int类型



### {{FUNCTION.unix\_microsecond}}

* 微秒级时间戳
* 返回结果int类型



### {{FUNCTION.unix\_nanosecond}}

* 纳秒级时间戳
* 返回结果int类型

## 其他函数

### {{FUNCTION.base\_64\(xxxx\)}}

* xxxx 替换为对应图片地址
* 图片类型支持 "image/gif", "image/jpeg", "image/jpg", "image/png", "image/tiff"

