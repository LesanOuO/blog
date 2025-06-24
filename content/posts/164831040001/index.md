---
title: "构建一个带libgdiplus的.NET基础Docker镜像"
date: "2022-03-27"
tags: ["Docker", ".NET"]
categories: ["动手实践"]
---

默认.NET的Docker镜像是不包含libgdiplus的，所以本文介绍如何通过在原有镜像基础上构建一个满足个人需求的Docker镜像。

## Docker 简介

Docker 是一个开源的应用容器引擎，基于 **Go 语言** 并遵从 Apache2.0 协议开源。Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口（类似 iPhone 的 app）,更重要的是容器性能开销极低。

**Docker 的应用场景：**

- Web 应用的自动化打包和发布。
- 自动化测试和持续集成、发布。
- 在服务型环境中部署和调整数据库或其他的后台应用。
- 从头编译或者扩展现有的 OpenShift 或 Cloud Foundry 平台来搭建自己的 PaaS 环境。

**Docker 的优点：**

Docker 是一个用于开发，交付和运行应用程序的开放平台。Docker 使您能够将应用程序与基础架构分开，从而可以快速交付软件。借助 Docker，您可以与管理应用程序相同的方式来管理基础架构。通过利用 Docker 的方法来快速交付，测试和部署代码，您可以大大减少编写代码和在生产环境中运行代码之间的延迟。

- 快速，一致地交付您的应用程序
- 响应式部署和扩展
- 在同一硬件上运行更多工作负载

## Docker 命令

[详细命令可以查看Docker 命令大全](https://www.runoob.com/docker/docker-command-manual.html)

### 常用镜像命令

```bash
# 1. 查看服务器中 Docker 镜像列表
docker images
# 2. 搜索镜像
docker search 镜像名
# 3. 拉取镜像（不加tag(版本号)就默认拉取Docker仓库中该镜像的最新版本latest; 加:tag则是拉取指定版本）
docker pull 镜像名
docker pull 镜像名:v1
# 4. 运行镜像
docker run -itd --name="nginx" --restart=always -p 80:80 -v /data:/data nginx:latest
# 5. 删除镜像
docker rmi -f 镜像名/镜像ID
# 6. 保存镜像
docker save nginx -o /nginx.tar
# 7. 加载镜像
docker load -i 镜像文件位置
```

### 常用容器命令

```bash
1. 查看正在运行容器列表
docker ps
2. 查看所有容器
docker ps -a
3. 停止容器
docker stop 容器名/容器ID
4. 删除容器
docker rm -f 容器名/容器ID
5. 进入容器方式
docker exec -it 容器名/容器ID /bin/bash
exit/ctl+p+q #退出
6. 重启容器
docker restart 容器名/容器ID
7. 启动容器
docker start 容器名/容器ID
8. kill 容器
docker kill 容器名/容器ID
9. 容器文件拷贝
docker cp 容器名/ID:容器内路径 容器外路径 #容器内拷出
docker cp 容器外路径 容器名/ID:容器内路径 #容器外拷入
10. 查看容器日志
docker logs -f --tail=100 容器 #tail查看末尾多少行 默认all
11. 修改存在容器的启动配置
docker update --restart=always 容器
12. 更换容器名
docker rename 容器 容器新名字
13. 通过容器提交镜像！！！###
docker commit -a="提交作者" -m="提交信息" 容器 提交镜像:tag
```

### Docker 运维命令

```bash
1. 查看Docker工作目录
sudo docker info | grep "Docker Root Dir"
2. 查看Docker磁盘占用总体情况
du -hs /var/lib/docker/
3. 查看Docker的磁盘使用具体情况
docker system df
4. 删除无用的容器和镜像
docker rm `docker ps -a | grep Exited | awk '{print $1}'`
docker rmi -f `docker images | grep '<none>' | awk '{print $3}'`
5. 清除所有无容器使用的镜像
docker system prune -a
6. 查找大文件
find / -type f -size +100M -print0 | xargs -0 du -h | sort -nr
7. 查找指定Docker使用目录下大于指定大小文件
find / -type f -size +100M -print0 | xargs -0 du -h | sort -nr | grep '/var/lib/docker/overlap2/*'
```

## 构建一个带libgdiplus的DotNetCore基础镜像

通过Docker拉取一个.netcore3.1基础镜像：`docker pull mcr.microsoft.com/dotnet/aspnet:3.1`

进入容器：`docker run -it mcr.microsoft.com/dotnet/aspnet:3.1 /bin/bash `

安装libgdiplus：

```bash
apt-get update -y
apt-get install -y libgdiplus
apt-get clean
ln -s /usr/lib/libgdiplus.so /usr/lib/gdiplus.dll
```

提交为新镜像：

`docker commit -a="Lesan" -m="added libgdiplus based on .netcore3.1" 28a66ebccd55 dotnetcore-with-libgdiplus:v3.1`

修改项目Dockerfile基础镜像为刚刚构建的自定义镜像 `dotnetcore-with-libgdiplus:v3.1`

> 借鉴参考以下文章：\
> https://blog.csdn.net/leilei1366615/article/details/106267225\
> https://www.runoob.com/docker/docker-command-manual.html\
> https://blog.csdn.net/u014374975/article/details/115436174