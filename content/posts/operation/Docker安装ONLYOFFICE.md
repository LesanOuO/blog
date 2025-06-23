---
title: "Docker安装ONLYOFFICE"
date: "2025-06-23"
tags: ['Docker','ONLYOFFICE']
categories: ['动手实践']
---

## 简介

`ONLYOFFICE`是一个值得推荐的软件，它能够帮助各种规模和类型的团队安全地在线协作处理文档。

1. 文档编辑功能
- 支持多种格式：可以编辑Word、Excel、PowerPoint、PDF等多种常见格式的文件。
- 兼容性强：与Microsoft Office和OpenOffice格式兼容，保证文件的互通性。
- 多种编辑工具：提供丰富的文本编辑、表格功能、图形和图像插入等多种编辑工具。
2. 云办公支持
- 文档在线协作：支持多人同时编辑同一个文档，可以实时看到其他人的编辑内容。
- 文件共享与管理：通过ONLYOFFICE平台，可以方便地管理和共享文件。
- 文件版本控制：自动保存文档版本，支持版本回溯，确保文件的完整性和可追溯性。
3. 高度集成的工作平台
- 邮件、日历和任务管理：集成了邮件、日历、任务等管理功能，帮助团队协作和工作流优化。
- 云存储：支持将文件存储在ONLYOFFICE云端，也支持第三方云存储（如Google Drive、Dropbox等）的集成。
- API接口：提供丰富的API，允许开发者定制或集成ONLYOFFICE到其他业务系统中。
4. 安全性
- 文件加密：支持文件加密功能，保护文件隐私。
- 权限控制：可以设置不同用户的访问权限，确保敏感信息的安全。
- 双重身份验证：增强安全性，防止未经授权的访问。
5. 跨平台支持
- 桌面端与移动端支持：提供Windows、macOS和Linux版本的桌面应用，还支持iOS和Android移动端应用。
- Web版：支持通过浏览器直接访问，避免设备限制。
6. 开放源码与自托管
- 开源版本：ONLYOFFICE提供免费开源版本，用户可以根据需要自定义和修改代码。
- 自托管：允许企业将ONLYOFFICE部署在自己的服务器上，确保数据的安全和控制。
7. 集成与兼容性
- 与其他工具集成：可以与Microsoft Office、Google Workspace、Nextcloud等工具进行集成。
- 插件支持：支持第三方插件，增强功能的灵活性。

更多`ONLYOFFICE`相关信息可访问其[官网](https://www.onlyoffice.com/zh/)、[GitHub](https://github.com/ONLYOFFICE/DocumentServer)、[开发文档](https://api.onlyoffice.com/)

## Docker安装

### 准备镜像

通过Docker我们可以很快速的安装`ONLYOFFICE`

>https://helpcenter.onlyoffice.com/docs/installation/docs-community-install-docker.aspx


```bash
docker pull onlyoffice/documentserver:latest
```

### 运行镜像

```bash
docker run -itd -p 8881:80 --restart=always \
    -v /data/onlyoffice/DocumentServer/prod/logs:/var/log/onlyoffice  \
    -v /data/onlyoffice/DocumentServer/prod/data:/var/www/onlyoffice/Data  \
    -v /data/onlyoffice/DocumentServer/prod/lib:/var/lib/onlyoffice \
    -v /data/onlyoffice/DocumentServer/prod/db:/var/lib/postgresql \
    -e JWT_SECRET=gie_jwt_secret -e ALLOW_META_IP_ADDRESS=true -e ALLOW_PRIVATE_IP_ADDRESS=true \
    --name=onlyoffice --privileged onlyoffice/documentserver
```

文件说明：

- `/var/log/onlyoffice` for ONLYOFFICE Docs logs
- `/var/www/onlyoffice/Data` for certificates
- `/var/lib/onlyoffice` for file cache
- `/var/lib/postgresql` for database

变量说明：

- `-e JWT_SECRET=gie_jwt_secret` JWT验证密钥，JWT_ENABLED默认为true
- `-e ALLOW_META_IP_ADDRESS=true``-e ALLOW_PRIVATE_IP_ADDRESS=true` 允许内网访问

### 结果

通过浏览器输入`http://电脑IP:8881`，**一定要用电脑IP**，防止出现无法正常试用的问题。

访问成功后会自动跳转welcome页面！

## 一些问题！！！

### 内网访问

修改容器中的`/etc/onlyoffice/documentserver/default.json`以下字段

```json
"request-filtering-agent" : {
        "allowPrivateIPAddress": true,
        "allowMetaIPAddress": true
},
```

可以通过`docker cp`命令进行修改，容器内没有`vi`或`vim`命令

```bash
docker cp onlyoffice:/etc/onlyoffice/documentserver/default.json ./
vim default.json

docker cp ./default.json onlyoffice:/etc/onlyoffice/documentserver/default.json
```

### 修改local.json文件

如设置token，修改 `local.json` 文件时，需要执行如下命令，否则镜像重启会恢复原内容：

```bash
supervisorctl restart all
```

### 添加中文字体

#### 准备中文字体

先准备好自己需要安装的字体，可以直接从windows系统C:\Windows\Fonts下选择拷贝，也可以参考这个网址，热心人已经给整理好了，地址：[onlyoffice-chinese-fonts](https://github.com/funggtopp/onlyoffice-chinese-fonts)

#### 删除onlyoffice自带字体

先要进入容器删除onlyoffice原有字体，**注意：如果只是添加字体，且想使用原有字体的话那么就不用删除原有字体**

```bash
# 进入容器
docker exec -it onlyoffice bash
# 进入容器后执行
cd /usr/share/fonts/
rm -rf *
cd /var/www/onlyoffice/documentserver/core-fonts/
rm -rf *
```

#### 添加新字体

用`docker cp`命令将下载的字体都放到容器内的`/usr/share/fonts`目录下。

#### 生成字体

进入容器，执行命令：`/usr/bin/documentserver-generate-allfonts.sh`


> 本文参考至 https://blog.csdn.net/fire_in_java/article/details/138905726、https://blog.csdn.net/XINYANGXINNIAN/article/details/135288590