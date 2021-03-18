---
description: 据说是因为中美关系恶化，于是乎，我们开始适配一些国产的操作系统和数据库~
---

# 麒麟服务器上安装 DM8.md

**查看麒麟服务器版本：**

```text
root@Kylin:~# uname -a
Linux Kylin 4.4.131-20200704.kylin.server-generic #kylin SMP Sat Jul 4 19:29:27 CST 2020 aarch64 aarch64 aarch64 GNU/Linux
```

**查看DM8安装文档：**

1. 登录达梦官网，找到 DM8（飞腾版本64位）进行下载。
2. 进入到"文档下载"界面，下载安装文档：《达梦数据库管理系统安装手册》

具体安装步骤，这里不再赘述。

  
 DM8版本的达梦，按照操作手册安装完成后，并不能直接投入使用。 这是因为，安装手册只讲到"安装达梦应用"。   
  


**我们需要自己配置并启动一个instance，步骤如下：**

1 查看是否有dms进程。如果存在该进程，则直接跳向步骤4，若不存在，则按顺序执行以下步骤：

```text
root@Kylin:~# ps -ef | grep dms
dmdba     2384     1  0 12月07 ?      00:00:41 /opt/dmdbms/bin/dmserver /opt/dmdbms/data/tinaliu/dm.ini -noconsole
```

2 使用 dminit 来初始化 instance 的配置信息：

```text
root@Kylin:/opt/dmdbms/bin# cd /opt/dmdbms/bin
root@Kylin:/opt/dmdbms/bin# ./dminit path=/opt/dmdbms/data EXTENT_SIZE=32 PAGE_SIZE=32 CASE_SENSITIVE=N  CHARSET=1  DB_NAME=tinaliu  INSTANCE_NAME=tinaliu  PORT_NUM=5236
```

说明：

* /opt/dmdbms/bin 为 DM8 的安装目录
* CASE\_SENSITIVE\(\)=大小敏感\(Y\)，可选值：Y/N，1/0 
* CHARSET=字符集\(0\)，可选值：0\[GB18030\]，1\[UTF-8\]，2\[EUC-KR\] 
* DB\_NAME=数据库名\(DAMENG\) 
* INSTANCE\_NAME=实例名\(DMSERVER\) 
* PORT\_NUM=监听端口号\(5236\)

注：

* 实例名，数据库名 端口号如无特殊要求可以默认。
* 字符集大小写敏感、初始页、簇大小制定后无法修改

3 使用前端命令启动 instance,观察是否正常：

```text
root@Kylin:/opt/dmdbms/bin# cd /opt/dmdbms/bin
root@Kylin:/opt/dmdbms/bin# ./dmserver /opt/dmdbms/data/tinaliu/dm.ini
```

说明：

* /opt/dmdbms/data/tinaliu/dm.ini 是执行步骤2 dminit 后生成的
* 前端启动只是为了观察是否可以正常启动，session 退出则会停止 instance

4 使用 disql 或其他客户端检查是否可以连接到 instance。disql 命令：

```text
root@Kylin:/opt/dmdbms/bin# disql SYSDBA:SYSDBA@localhost:5236
```

5 若连接正常，使用ctrl + c 终止前端进程，然后开始后端启动  
 6 执行installer脚本：

```text
root@Kylin:/opt/dmdbms/bin# cd /opt/dmdbms/script/root
root@Kylin:/opt/dmdbms/script/root# ./dm_service_installer.sh -t dmserver -p tinaliu -dm_ini /opt/dmdbms/data/tinaliu/dm.ini -server localhost:5236
```

成功后，可以收到提示：  
 _Created symlink from /etc/systemd/system/multi-user.target.wants/DmServicetinaliu.service to /lib/systemd/system/DmServicetinaliu.service._  
 _创建服务\(DmServicetinaliu\)完成_

7 启动实例：

```text
root@Kylin:/opt/dmdbms/bin# cd /opt/dmdbms/bin
root@Kylin:/opt/dmdbms/bin# ./DmServicetinaliu start
```

说明：

* DmServicetinaliu 是根据 dminit 时输入的实例信息生成的，不要选错了

8 查看实例运行信息：

```text
 root@Kylin:/opt/dmdbms/bin# ./DmServicetinaliu status
```

9 至此，DM8实例已经成功安装并启动

