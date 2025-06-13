---
title: "Linux中修改Docker存储位置"
date: "2022-07-02"
tags: ['Linux', 'Docker']
categories: ['动手实践']
---

在维护服务器时，发现docker所在盘容量已满，导致mongodb插入数据失败从而崩溃。由此记录修改Linux中Dockder位置时遇到的问题

## 一些命令

### `df` 命令
df命令来自于英文词组”Disk Free“的缩写，其功能是用于显示系统上磁盘空间的使用量情况。

常常使用 `df -h` 以容易阅读的方式显示

### `ln` 命令
Linux ln（英文全拼：link files）命令是一个非常重要命令，它的功能是为某一个文件在另外一个位置建立一个同步的链接。

Linux文件系统中，有所谓的链接(link)，我们可以将其视为档案的别名，而链接又可分为两种 : 硬链接(hard link)与软链接(symbolic link)，硬链接的意思是一个档案可以有多个名称，而软链接的方式则是产生一个特殊的档案，该档案的内容是指向另一个档案的位置。硬链接是存在同一个文件系统中，而软链接却可以跨越不同的文件系统。

通常我们都使用 `ln -s log2013.log link2013` 来创建软链接

### `lsof` `netstat` 命令
Linux 查看端口占用情况可以使用 lsof 和 netstat 命令。

可使用 `lsof -i:端口号` `netstat -tunlp | grep 端口号`

### `kill` 命令
Linux kill 命令用于删除执行中的程序或工作。

可使用 `kill -9 PID` 彻底杀死进程

### `top` 命令
Linux top命令用于实时显示 process 的动态。

### `free` 命令
Linux free命令用于显示内存状态。

### `du` 命令
Linux du （英文全拼：disk usage）命令用于显示目录或文件的大小。

通常使用 `du -h` 提高信息的可读性

## 修改 Docker 存储位置

1. 通过软链接

```bash
// 停止 docker 服务
systemctl stop Docker
// 移动整个 /var/lib/docker 目录到目标路径(/data/docker)
mv /var/lib/docker /data/docker
// 创建软链接
ln -s /root/docker /var/lib/docker
// 重启 docker
systemctl start docker
```

2. 修改 docker 配置文件

```bash
// 停止 docker 服务
systemctl stop docker
// 移动整个 /var/lib/docker 目录到目标路径(/data/docker)
mv /var/lib/docker /data/docker
// 修改 docker.service 文件
vim /usr/lib/systemd/system/docker.service
// 重启 docker 服务
systemctl daemon-reload
systemctl start docker
// 查看配置是否生效
docker info | grep "Docker Root Dir"
```

其中 docker.service 修改的内容为：

在 `ExecStart=/usr/bin/dockerd` 后面添加参数 `--graph /data/docker`

结果如下：

`ExecStart=/usr/bin/dockerd --graph /data/docker -H fd:// --containerd=/run/containerd/containerd.sock
`

## Docker 中 mongodb 出现错误
在更换了 docker 存储位置后， mongodb 启动后就一直报错 `Failed to set up listener: SocketException: Permission denied` ，原因是以为启动mongo时,无法写入mongo.socket文件到/tmp目录下

查阅了网络上的多个教程，最终有效的是下面的解决方案：

将本地的/tmp目录挂在到mongo容器中，如果是已启动的容器，可通过我的 `docker为已启动容器添加挂载目录或端口映射` 这篇文章进行挂载；如果是新启动的容器，只需要 `-v /tmp:/tmp` 即可