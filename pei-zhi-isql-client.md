---
description: 安装千万条，环境第一条。Docker 是个好东西，推荐你多用~
---

# 配置 UnixODBC

资源紧张的时候，服务器是大家共用的，上面部署了一堆服务。所以选用docker 进行 unix odbc 的编译和适配。避免牵一发而动全身，影响他人使用。（~~我不会告诉你其实：是服务器 gcc 版本太低了，编译报错~~）

因为最终，我们是使用 golang 进行开发的。所以基于 golang1.14 镜像来构建。

1 创建 Dockerfile

```text
root@Kylin:/data/liutongtong011# cd /data/liutongtong011
root@Kylin:/data/liutongtong011# touch Dockerfile
```

Dockerfile 内容：

```text
FROM golang:1.14

RUN apt-get update && \
    apt-get install -y unixodbc-dev unixodbc && \
    go get github.com/alexbrainman/odbc
```

2 根据 Dockerfile build 镜像

```text
docker build - < Dockerfile -t liutongtong011
```

3 查看镜像是否存在

```text
docker images | grep liutongtong011
```

4 启动容器

```text
docker run -it liutongtong011
```

说明：执行完 docker run 自动就登录到容器内了。如果想要再其他session进入容器，执行以下步骤：

5 查看容器 ID

```text
docker ps | grep liutongtong
```

说明：第一列即为容器 ID

6 登录容器

```text
docker exec -it 017a0f7a3067 bash
```

  
  
 容器启动成功后，在容器内部进行 UnixODBC 的配置：

1 使用 `odbcinst -j` 命令查看 odbc的配置

```text
root@Kylin:/go# odbcinst -j
unixODBC 2.3.6
DRIVERS............: /etc/odbcinst.ini
SYSTEM DATA SOURCES: /etc/odbc.ini
FILE DATA SOURCES..: /etc/ODBCDataSources
USER DATA SOURCES..: /root/.odbc.ini
SQLULEN Size.......: 8
SQLLEN Size........: 8
SQLSETPOSIROW Size.: 8
```

2 配置 `odbc.ini`

```text
[DM8]
Description   = DM ODBC DSN
Driver     = DM8 ODBC DRIVER
SERVER     = 172.0.0.1
UID       = SYSDBA
PWD       = SYSDBA
TCP_PORT   = 5236
```

说明：为了减少出错的可能，我就直接将 `/root/.odbc.ini` 和 `/etc/odbc.ini` 配置成一样的了

3 配置 `odbcinst.ini`

```text
[DM8 ODBC DRIVER]
Description   = ODBC DRIVER FOR DM8
Driver     = /opt/dmdbms/bin/libdodbc.so
Setup     = /lib/libdmOdbcSetup.so
threading = 0
```

说明：

* 这里的 title `[DM8 ODBC DRIVER]` 必须和 odbc.ini 中的 Driver 保持一致
* 这里的 `/opt/dmdbms/bin/libdodbc.so` 是达梦的 so，而非 UnixODBC 自带的

4 使用 `isql` 登录数据库

```text
root@Kylin:/go# isql -v DM8
+---------------------------------------+
| Connected!                            |
|                                       |
| sql-statement                         |
| help [tablename]                      |
| quit                                  |
|                                       |
+---------------------------------------+
```

说明：`-v` 的作用是，一旦报错，可以展示报错详情

5 执行任意 sql 测试

```text
SQL> select * from v$version;
+---------------------------------------------------------------------------------+
| BANNER                                                                          |
+---------------------------------------------------------------------------------+
| DM Database Server 64 V8
                                                       |
| DB Version: 0x7000b                                                             |
+---------------------------------------------------------------------------------+
SQLRowCount returns 2
2 rows fetched
```

如果 `isql` 可以成功连接DB，并能执行测试语句，说明 UnixODBC 配置成功。

这里简单介绍几个Docker命令，熟悉Docker的同学可以跳过这一趴：

```text
docker commit e9f39c7081e0 unixodbc001
```

说明：

* e9f39c7081e0： 正在运行的容器ID，可以使用 `docker ps` 查看
* unixodbc001：自定义的镜像名称

```text
docker tag unixodbc001 registry.cn-beijing.aliyuncs.com/liutongtong/unixodbc001:V0.1
docker push registry.cn-beijing.aliyuncs.com/liutongtong/unixodbc001:V0.1
```

说明：像镜像仓库中提交该镜像，以后用的时候，直接拉取即可。

_现实往往是残酷的，上面简单的几步中可能会遇到许多问题。_

我把自己在安装过程中踩的坑，报的错及解决方案列在下面，供大家参考：

使用源码编译安装UnixODBC:

起初在网上找了达梦大学的官方教程：[http://www.dameng.com/teachers\_view.aspx?TypeId=183&Id=891&FId=t26:183:26](http://www.dameng.com/teachers_view.aspx?TypeId=183&Id=891&FId=t26:183:26)

按照里面的步骤进行源码安装。最先遇到的问题是:

**=&gt; configure 时报错：cannot guess build type**

```text
./configure
UNAME_MACHINE = aarch64
UNAME_RELEASE = 4.4.131-20200704.kylin.server-generic
UNAME_SYSTEM  = Linux
UNAME_VERSION = #kylin SMP Sat Jul 4 19:29:27 CST 2020
configure: error: cannot guess build type; you must specify one
```

解决办法：`./configure --build=arm`

**=&gt; 无法编译出 .so文件&lt;/font&gt;**

即便增加`enable-shared`也无法解决问题

```text
 ./configure  --build=arm --enable-shared
```

编译出来的始终是 `.a` 和 `.la` 文件。

尝试手动合成 `.so`

```text
root@greatwall-os:/usr/local/lib# ar -x libodbcinst.a
root@greatwall-os:/usr/local/lib# gcc -shared *.o -o  libodbcinst.so
```

报错：

```text
/usr/bin/ld: libltdlc_la-ltdl.o: relocation R_AARCH64_ADR_PREL_PG_HI21 against external symbol `__stack_chk_guard@@GLIBC_2.17' can not be used when making a shared object; recompile with -fPIC
/usr/bin/ld: libltdlc_la-ltdl.o(.text+0x6e4): 无法解决 R_AARCH64_ADR_PREL_PG_HI21 重定向于符号 “__stack_chk_guard@@GLIBC_2.17” 有冲突
/usr/bin/ld: 最后的链结失败: 错误的值
collect2: error: ld returned 1 exit status
```

解决办法：清除现有 odbc，增加 configure 参数后重新安装

```text
make uninstall && make clean

CFLAGS="-fPIC" ./configure --build=arm 
# 编译出来还是 .a ，但是这回  .a 是可以合并成 .so 的

ar -x /usr/local/lib/libodbc.a
gcc -shared *.o -o libodbc.so
```

**=&gt; could not determine kind of name for C.SQL\_WLONGVAR**

之前用的UnixODBC2.21版本过低，升级到2.3.2即可解决问题

**=&gt; isql 时报 file not found**

```text
[root@dameng-test001 dameng]# isql -v DM8 SYSDBA SYSDBA
[01000][unixODBC][Driver Manager]Can't open lib '/home/dmdba/dmdbms/bin/libdodbc.so' : file not found
```

使用 `ls -l /home/dmdba/dmdbms/bin/libdodbc.so` 命令可以看到文件存在，且权限为可访问。

使用 `ldd` 命令查看依赖是否有问题：

```text
[root@dameng-test001 dameng]# ldd /home/dmdba/dmdbms/bin/libdodbc.so
    linux-vdso.so.1 =>  (0x00007fff36943000)
    libdmdpi.so => not found
    libdmfldr.so => not found
    librt.so.1 => /lib64/librt.so.1 (0x00007fbcec98a000)
    libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fbcec76e000)
    libdl.so.2 => /lib64/libdl.so.2 (0x00007fbcec56a000)
    libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007fbcec263000)
    libm.so.6 => /lib64/libm.so.6 (0x00007fbcebf61000)
    libc.so.6 => /lib64/libc.so.6 (0x00007fbcebb93000)
    libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007fbceb97d000)
    /lib64/ld-linux-x86-64.so.2 (0x00007fbcece44000)
```

发现是找不到以来的lib

进行全局查找：`find / -name "libdmdpi.so"` 发现实际上时存在的。路径为：/home/dmdba/dmdbms/bin/ （_这里路径和之前不一致是因为，为了保证可行性，先在Linux上做了适配_）

在 `.bash_profile` 中配置 LD\_LIBRARY\_PATH 变量:

```text
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/dmdba/dmdbms/bin/
```

**=&gt;\[S1000\]\[unixODBC\]Encryption module failed to load**

这是因为，在DM8中，加密模块已不在`libdodbc.so` 中

解决办法：把安装目录 `/opt/dmdbms/bin` 下的所有 `.so` 文件都拷贝到 docker 中

```text
for i in  `ls /opt/dmdbms/bin/*.so`; do echo $i ;docker cp $i bed496495fee:/opt/dmdbms/bin/; done
```

**=&gt;\[IM004\]\[unixODBC\]\[Driver Manager\]Driver's SQLAllocHandle on SQL\_HANDLE\_HENV failed**

`odbcinst.ini` 中配置的 Driver `libdodbc.so` 必须得是达梦的，不能是UnixODBC自己生成的

**=&gt; cannot find -lodbc**

报错详情：

```text
root@Kylin:/Users/liutongtong/go/src/dameng-test/main# go run beego-orm.go
# github.com/alexbrainman/odbc/api
/usr/lib/gcc-cross/arm-linux-gnueabi/8/../../../../arm-linux-gnueabi/bin/ld: cannot find -lodbc
/usr/lib/gcc-cross/arm-linux-gnueabi/8/../../../../arm-linux-gnueabi/bin/ld: cannot find -lodbc
```

使用 `ld` 命令查看：

```text
root@Kylin:/Users/liutongtong/go/src/dameng-test/main# ld /usr/local/lib/libodbc.so
ld: warning: cannot find entry symbol _start; not setting start address
ld: /usr/local/lib/libodbc.so: undefined reference to `dlopen'
ld: /usr/local/lib/libodbc.so: undefined reference to `dlclose'
ld: /usr/local/lib/libodbc.so: undefined reference to `dlerror'
ld: /usr/local/lib/libodbc.so: undefined reference to `dlsym'
```

按照网上搜索来的方法进行尝试:

```text
重新编译：
./configure LIBS=-ldl CFLAGS=-fno-strict-aliasing --build=arm

configure的时候提示：
checking for shl_load... (cached) no
checking for shl_load in -ldld... (cached) no
checking for dld_link in -ldld... no
checking for _ prefix in compiled symbols... no

尝试：
root@Kylin:/usr/local/lib# ar -x libodbcinst.a
root@Kylin:/usr/local/lib# gcc -shared *.o -o libodbcinst.so  -ldl

之后 ld 报错：
root@Kylin:/Users/liutongtong/go/src/dameng-test/main# ld /usr/local/lib/libodbc.so
ld: warning: cannot find entry symbol _start; not setting start address
```

总之是越做越错，越尝试越绝望。

这个问题后来通过重新拉镜像解决了。原来是我在疯狂的改环境的时候，无意间造成了破坏。

幸好是在 Docker 里，没有对他人造成影响。

这个故事告诉我们：**一团乱麻的时候，不妨重新来过**

