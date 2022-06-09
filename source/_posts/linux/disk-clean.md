---
title: "Linux 磁盘清理"
date: 2022-05-07T14:10:25+08:00
lastmod: 2022-05-07T14:10:25+08:00
categories: ["Linux"]
---

Linux 系统磁盘大文件定位、分析、处理。

<!--more-->

## 查看系统磁盘信息

使用 `df` 命令查看占用情况：

+ `-l` 选项只显示本机的文件系统。
+ `-h` 选项以人类可读的方式打印输出数据。

```
df -lh
```

输出如下结果：

```shell
文件系统                 容量  已用  可用 已用% 挂载点
devtmpfs                 7.7G     0  7.7G    0% /dev
tmpfs                    7.7G     0  7.7G    0% /dev/shm
tmpfs                    7.7G  818M  6.9G   11% /run
tmpfs                    7.7G     0  7.7G    0% /sys/fs/cgroup
/dev/mapper/centos-root  853G  224G  630G   27% /
/dev/sda1               1014M  199M  816M   20% /boot
/dev/mapper/centos-home   20G  836M   20G    5% /home
tmpfs                    1.6G     0  1.6G    0% /run/user/1002
```

从结果可以看见文件系统 `/dev/mapper/centos-root` 下的挂载点 `/` 已使用 `224G`。接下来就是定位到占用磁盘空间大但又无用的文件进行删除操作释放空间。

## 定位最大文件目录

使用 `du` 命令以递归方式汇总根目录下每个文件或文件夹的磁盘使用情况：

+ `-h` 选项以人类可读的方式打印输出数据。
+ `--max-depth=1` 选项将汇总的目录深度指定为 `1` 层（即汇总当前指定目录下的二级目录的磁盘占用情况）。

```shell
sudo du -h --max-depth=1 /
```

输出如下结果：

```shell
166M    ./boot
0       ./dev
805M    ./home
0       ./proc
826M    ./run
0       ./sys
36M     ./etc
729M    ./root
36G     ./var
32K     ./tmp
2.8G    ./usr
0       ./media
0       ./mnt
8.3G    ./opt
184G    ./srv
234G    .
```

可以看到占用磁盘空间最大的文件目录为: `/srv` 和 `/var`。

## 分析最大文件目录

首先使用 `tree` 命令分析 `/srv` 和 `/var` 这两个目录的结构树：
+ `-L 2` 选项指定目录深度为 `2`。
+ `-d` 选项只展示文件夹（不展示文件）。
+ `-h` 选项以人类可读的方式打印输出数据。 

```shell
tree -L 2 -d -h /srv/
```

输出如下结果：

```shell
/srv/
└── [ 162]  volume
    ├── [  41]  gitea
    ├── [8.0K]  jenkins
    ├── [  32]  mysql
    ├── [  32]  mysql_backup
    ├── [  20]  nacos
    ├── [ 265]  nexus3
    ├── [  30]  redis
    ├── [  30]  rocketmq
    ├── [4.0K]  rocketmq-dashboard
    └── [  30]  x-job

11 directories
```

从结果得知 `/srv` 目录下只有单个 `volume` 目录，分析 `/srv/volume` 目录的占用情况即可：

```shell
sudo du -h --max-depth=1 /srv/volume/
```

输出如下结果：

```shell
20G     /srv/volume/mysql
141G    /srv/volume/nexus3
4.0G    /srv/volume/jenkins
96K     /srv/volume/redis
535M    /srv/volume/gitea
908M    /srv/volume/rocketmq
0       /srv/volume/nacos
3.1M    /srv/volume/rocketmq-dashboard
487M    /srv/volume/x-job
18G     /srv/volume/mysql_backup
184G    /srv/volume/
```

几番折腾，最终发现了一个无用 Mysql 备份目录的 `/srv/volume/mysql_backup` 占用 `18G`；虽然 Nexus 的 Docker 容器卷目录 `/srv/volume/nexus3` 占用 `141G`，但属于正常使用。

此处只删除 `/srv/volume/mysql_backup` 即可。

`/var` 目录分析同理，不再赘述。

## 确认文件未被占用

删除文件是个人都会使用：`rm –rf` ；但是在 Linux 或者 Unix 系统中，通过 `rm` 或者文件管理器删除文件将会从文件系统的目录结构上解除链接；然而如果文件是被打开的（有一个进程正在使用），那么进程将仍然可以读取该文件，磁盘空间也一直被占用。

正确的做法是删除文件后，一定要查看文件是否被进程占用，根据实际情况中止占用该文件的进程或者采用其他处理方式。

```shell
lsof -w | grep deleted
```

根据输出结果中的 `pid` 来中止进程才能真正的删除文件。