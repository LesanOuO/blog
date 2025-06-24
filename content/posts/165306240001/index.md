---
title: "Linux增加SSH端口"
date: "2022-05-21"
tags: ['Linux']
categories: ['动手实践']
---

## 简介

在大型企业中，服务器网络一般都会有各种各样的限制，常见的就会有堡垒机或者运维网关，以达到操作审计等目的。但对于开发者来讲，这些限制大大降低了效率，所以我们需要一个方便的方式来解决这个问题。

本篇文章主要通过为Linux增加SSH端口的方式来解决这个问题，通过增加SSH访问端口来绕开22端口的限制。

## 实现

1. SSH配置文件

通过 `vi /etc/ssh/sshd_config` 来配置SSH端口，在`Port 22`后添加一行`Port XX`即可

2. 配置SELinux

通过 `semanage port -l | grep ssh` 可以看到：
```bash
ssh_port_t                     tcp      22
```
可以看到并没有我们添加的端口

可以执行`semanage port -a -tssh_port_t -p tcp XX`来添加

再次检查：
```bash
ssh_port_t                     tcp      XX, 22
```

3. 重启SSH

```bash
systemctl restart sshd.service
```

4. 通过telnet可以验证

```bash
telnet IP地址  XX
```

通过以上操作后，就可以通过新开放的端口来实现对SSH服务的访问了