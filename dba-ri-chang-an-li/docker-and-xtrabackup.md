---
description: 本文记录了在使用 debian OS 系统的 docker 上安装及使用 xtrabackup 的实践
---

# docker 上安装及使用 XtraBackup

## 查看 MySQL 版本号

```text
mysql> \s
--------------
mysql  Ver 14.14 Distrib 5.7.30, for Linux (x86_64) using  EditLine wrapper

Connection id:        11
Current database:
Current user:
SSL:
Current pager:        stdout
Using outfile:        ''
Using delimiter:    ;
Server version:        5.7.32-log MySQL Community Server (GPL)
Protocol version:    10
Connection:
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    utf8
Conn.  characterset:    utf8
TCP port:        3306
Uptime:            1 hour 28 min 13 sec

Threads: 2  Questions: 31389  Slow queries: 0  Opens: 126  Flush tables: 1  Open tables: 119  Queries per second avg: 5.930
--------------
```

我们使用的是 5.7 版本，选择 percona-xtrabackup-24 进行备份、恢复

## 安装 lsb\_release

使用命令 `lsb_release -sc` 查看系统版本

倘若提示命令不存在，使用`apt-get install lsb-release`安装

如果提示找不到包，例如：

```text
# apt-get install lsb-release

Reading package lists... Done
Building dependency tree
Reading state information... Done
E: Unable to locate package lsb-release
```

使用命令 `apt-get update` 更新 apt-get，之后重新安装： `apt-get install lsb-release`

安装成功后，查看版本号

```text
# lsb_release -sc

buster
```

## 安装 xtrabackup

1 下载 deb 包

```text
wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb
```

2 使用 dpkg 安装下载下来的包

```text
sudo dpkg -i percona-release_latest.$(lsb_release -sc)_all.deb
```

3 按照提示，执行更新

```text
apt-get update
```

4 安装 percona-xtrabackup-24

```text
sudo apt-get install percona-xtrabackup-24
```

5 安装压缩插件

```text
sudo apt-get install qpress
```

## 使用 XtraBackup 还是 innobackupex？

在我们使用的2.4 版本中，xtrabackup 已经支持 MyISAM的备份了。原文：

> It can back up data from _InnoDB_, _XtraDB_, and _MyISAM_ tables on _MySQL_ 5.1 [\[1\]](https://www.percona.com/doc/percona-xtrabackup/2.4/index.html#n-1), 5.5, 5.6 and 5.7 servers, as well as _Percona Server for MySQL_ with _XtraDB_.

## 备份操作

xtrabackup 是支持备份运行中的实例的

```text
xtrabackup --backup --target-dir=/data/backups/
```

· 如果 target-dir 不存在，xtrabackup 可以自动创建 · 但是如果 target-dir 里有数据，则会报错 `file exists`

在 backup 的过程中，实例可能随时会有写入。xtrabackup 每描检查一次 log 查看是否有需要拷贝的新记录。

如果此时写入量过大，覆写了 log，但是xtrabackup 还未来得及读取，会引发报错。建议选择低峰时期再操作。

## prepare 操作

prepare 操作是在 backup 和 restore 之间操作的。

```text
xtrabackup --prepare --target-dir=/data/backups/
```

这是因为由于是热拷贝， backup 出来的数据不是时间点一致\(point-in-time consistent\)的。如果直接拿这个阶段的备份去恢复，很可能因为数据'受损'引发崩溃。

**prepare 是解决不一致性的必要手段**。从操作层面来说，可以由备份程序备份完成后直接执行，也可以在恢复程序执行前操作。但我本人建议由备份程序操作： 1. 有问题可以更早发现，避免恢复时才发现备份是不可用的 1. prepare 只能操作由相同或更新版本 xtrabackup 备份出来的数据，反之不行。在备份后立即操作可以减少因备份恢复使用的 xtrabackup 版本不统一引发的问题

## 恢复操作

恢复操作需要 data 目录为空，通常： 1. 创建一个新 docker 1. 关闭实例 1. rm -rf 原始的数据路径

```text
 xtrabackup --copy-back --target-dir=/data/backups/
```

完成后，检查文件属组是否正确，没问题的话，启动实例即可

## 校验操作

保险起见通常会做些校验操作，简单的诸如： 1. checksum 开始备份之后未改变的 table，检查数据是否齐全 2. 根据 update\_time 抽查备份过程中有修改的 table，数据是否符合预期 3. 检查 table 和 column 的数量。这个虽然有可能不一致，因为在备份后，可能会有新建表，新增列的操作。但即便操作，通常也是少量，方便确认。

  
  
 参考文档：

  [https://www.percona.com/doc/percona-xtrabackup/2.4/installation/apt\_repo.html](https://www.percona.com/doc/percona-xtrabackup/2.4/installation/apt_repo.html)

  [https://www.percona.com/doc/percona-xtrabackup/2.4/backup\_scenarios/compressed\_backup.html\#compressed-backup](https://www.percona.com/doc/percona-xtrabackup/2.4/backup_scenarios/compressed_backup.html#compressed-backup)

