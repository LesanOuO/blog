---
title: "Linux系统使用systemd配置.NET项目开机自启动"
date: "2024-04-28"
tags: ['Linux','.NET']
categories: ['动手实践']
---

本片文章记录在Linux（CentOS7）中如何使用`systemctl`命令进行系统和服务管理

## `systemctl`命令

```bash
# 设置开机自启动，可在任意目录下执行
systemctl enable xxx.service

# 启动nginx服务
systemctl start xxx.service

# 停止开机自启动
systemctl disable xxx.service

# 查看服务当前状态
systemctl status xxx.service

# 重新启动服务
systemctl restart xxx.service

# 查看所有已启动的服务
systemctl list-units --type=service
```

## 创建xxx.service

在`/etc/systemd/system/`路径下，新增一个`myTest.service`文件，文件内容如下

```
# Unit 文件描述
[Unit]
Description=myTest service

# Service 配置参数
[Service]
Type=simple
GuessMainPID=true

# 自启动项目所在的位置路径
WorkingDirectory=/root/myTestWeb
StandardOutput=journal
StandardError=journal

# 自启动项目的命令，这里用了dotnet启动，所以前面添加了dotnet的路径/usr/bin/
ExecStart=/usr/bin/dotnet myTest.dll --Urls=http://*:端口号
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

## 设置开机自启动myTest.service

```bash
systemctl enable myTest.service
```

## 开启服务，查看状态

```
systemctl start myTest.service

systemctl status myTest.service
```