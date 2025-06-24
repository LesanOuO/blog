---
title: "Docker为已启动容器添加挂载目录或端口映射"
date: "2022-06-14"
tags: ['Docker']
categories: ['动手实践']
---

在使用docker时，常常需要对已启动的容器进行相应的配置修改，其中最常见的就是挂载目录或端口映射。其实配置也并不是非常复杂，可以通过修改docker中的json配置文件即可。

以下为详细修改步骤

## 1. 先查看需要修改的容器的id号

首先通过 `docker ps -a` 查看 **CONTAINER ID**

再通过 `docker inspect <container_id>` 查看 **Id** （一般在最开始的位置）

## 2. 关闭docker服务

在做相应配置前，一定要先**停止docker**服务，否则会修改不成功。
```bash
systemctl stop docker
```

## 3. 前往docker配置文件目录

通过以下命令即可进入到配置文件目录：
```bash
cd /var/lib/docker/containers/<container_id>
```

## 4. 修改hostconfig.json文件

通过 `vim hostconfig.json` 即可修改配置文件。

> vim中可以通过 /字符串 快速定位字符串位置

- 修改以下内容配置端口映射
```json
{
    ...
    "PortBindings": {
        "80/tcp": [
        {
            "HostIp": "",
            "HostPort": "8080"
        }
        ]
    }
    ...
}
```

- 修改以下内容配置挂载目录
```json
{
    ...
    "Binds": [
        "/home/docker/www:/var/www"
    ]
    ...
    // 其中 `/home/docker/www` 为宿主机目录，`/var/www` 为容器目录
}
```

## 4. 修改config.v2.json文件

通过 `vim config.v2.json` 即可修改配置文件。

- 修改以下内容配置端口映射
```json
{
    ...
    "ExposedPorts": {
        "80/tcp": {},
    }
    ...
}
```

- 修改以下内容配置挂载目录
```json
{
    ...
    "MountPoints": {
        "/var/www": {
            "Source": "/home/docker/www",
            "Destination": "/var/www",
            "RW": true,
            "Name": "",
            "Driver": "",
            "Type": "bind",
            "Propagation": "rprivate",
            "Spec": {
                "Type": "bind",
                "Source": "/home/docker/www",
                "Target": "/var/www",
            }
            "SkipMountpointCreation": false,
        }
    }
    ...
    // 其中 `/home/docker/www` 为宿主机目录，`/var/www` 为容器目录
}
```

## 5. 重启docker服务

    `systemctl start docker`