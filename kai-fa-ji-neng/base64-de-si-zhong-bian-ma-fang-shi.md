---
description: 到处都是学问呐~
---

# base64 的四种编码方式

事情的起因是这样的：

在做 [siber](https://liu-tongtong.gitbook.io/dba/siber-ji-cheng-ce-shi-ping-tai/chan-pin-gai-shu) 项目的时候，有接口使用 `[]byte` 类型数据接收文件流。

```text
 // proto 定义
 bytes file_data = 1;
```

但是 siber 是统一使用 json 格式进行的 request body 定义，不能直接传输。

一个通用的解决方案是将 `[]byte` 转化为 `base64` 进行传输。我们选用这个包：

```text
"encoding/base64"
```

这个包下面带有不同的 base64 编码格式：

* StdEncoding：常规编码
* URLEncoding：URL safe 编码
* RawStdEncoding：常规编码，末尾不补 =
* RawURLEncoding：URL safe 编码，末尾不补 =

跟常规编码相比， URL safe替换掉字符串中的特殊字符，`+` 和 `/`

以`[]byte("Hello world. 你好，世界！")` 为例：

```text
base64.StdEncoding.EncodeToString(msg)
// SGVsbG8gd29ybGQuIOS9oOWlve+8jOS4lueVjO+8gQ==

base64.RawStdEncoding.EncodeToString(msg)
// SGVsbG8gd29ybGQuIOS9oOWlve+8jOS4lueVjO+8gQ

base64.URLEncoding.EncodeToString(msg)
// SGVsbG8gd29ybGQuIOS9oOWlve-8jOS4lueVjO-8gQ==

base64.RawURLEncoding.EncodeToString(msg)
// SGVsbG8gd29ybGQuIOS9oOWlve-8jOS4lueVjO-8gQ

```

