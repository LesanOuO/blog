---
title: "MAVEN私有仓库搭建与配置"
date: "2023-09-22"
tags: ['MAVEN']
categories: ['动手实践']
---

在工作中，公司内部往往需要一个私有的MAVEN仓库进行统一管理，本篇文章就带大家如何通过nexus搭建一个自己的MAVEN私有仓库

## 下载 nexus 安装包

nexus官网地址：https://www.sonatype.com/products/sonatype-nexus-oss-download

由于官网下载有一定的限制，提供百度网盘[下载地址](https://pan.baidu.com/s/17NFbqcupR062GDmNvT1VAw)，提取码：797P

其中：
- `win64`后缀为window安装包
- `unix`后缀为linux安装包
- `mac`后缀为mac安装包

## 安装 nexus

由于服务器一般为linux系统（centos/ubuntu），下面演示为linux系统安装nexus

1. 将`unix`后缀安装包上传至服务器任意目录
2. 解压安装包：`tar -zxvf nexus-3.19.1-01-unix.tar.gz`
3. 进入解压完成的目录下的`bin`目录，执行：`./nexus start`启动nexus
4. 启动成功后通过浏览器访问：http://IP:8081（8081为默认端口，可在`etc/nexus-default.properties`修改）

> ./nexus start启动成功后无法访问http://ip:port时，可以使用./nexus run命令启动，该命令会打印启动日志进行排查启动失败原因，但该命令在退出命令行时同时会停止nexus进程

## 配置 nexus

### 初始化密码

首次登陆后需要更新初始化密码
![](images/1.png)

- nexus3以前的默认用户名密码 admin / admin123
- nexus3的默认用户名依然是admin， 密码在admin.password文件中，可以通过`find / | grep 'admin.password'`查找路径，一般在解压后的`sonatype-work`目录下

登录成功后会多一个**设置**菜单
![](images/2.png)

### 配置仓库

系统默认有以下几个maven仓库：
- maven-central   (proxy) : 远程中央仓库
- maven-releases  (hosted): 私库发行仓库
- maven-snapshots (hosted): 私库快照仓库
- maven-public    (group) : 仓库组

仓库类型：
- proxy：可以自主配置使用的远程仓库地址
- hosted：内部项目构件发布的仓库类型
- virtual：虚拟仓库类型（基本不用）
- group：可以自由顺序组合多个仓库使用

#### 创建阿里云maven远程仓库

菜单选择路径：

Repository --> Repositories --> Create repository --> maven2(proxy)

![](images/3.png)

> 阿里云仓库地址：http://maven.aliyun.com/nexus/content/groups/public/

#### 配置仓库组：

菜单选择路径（默认已有maven-public）：

Repository --> Repositories --> Create repository --> maven2(group)

![](images/4.png)

⚠️ 注意仓库顺序，maven查找依赖时会依次遍历仓库组中的仓库

### 创建角色及用户

#### 创建角色并分配权限

菜单选择路径：

Security --> Roles --> Create

![](images/5.png)

#### 创建用户并分配角色

菜单选择路径：

Security --> Users --> Create

![](images/6.png)

## 测试 nexus

### settings.xml 配置

![](images/7.png)

### pom.xml 配置

![](images/8.png)

> 实际使用中distributionManagement可以配置在parent项目中，子项目无需重复配置

### 测试

上述配置全部完成后就可以在项目中使用mven clean deploy将项目的jar包上传到自己的私服上了

> 本篇文章引用自：https://blog.csdn.net/z562743237/article/details/108852509