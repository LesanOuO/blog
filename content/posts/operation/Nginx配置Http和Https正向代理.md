---
title: "Nginx配置Http和Https正向代理"
date: "2022-11-26"
tags: ['Nginx', '网络代理']
categories: ['动手实践']
---

`Nginx`配置`http`代理非常简单，网上教程也很多，但是无法很方便的配置`https`代理，本片文章将记录搭建可以同时代理`http`和`https`的服务器。

## 准备工作

首先我们得需要下载`Nginx`及第三方模块`ngx_http_proxy_connect_module`
```bash
wget http://nginx.org/download/nginx-1.18.0.tar.gz

wget https://github.com/chobits/ngx_http_proxy_connect_module/archive/master.zip
```

值得注意的是，`ngx_http_proxy_connect_module`第三方模块与`Nginx` 有版本对应关系，需要在其[官网](https://github.com/chobits/ngx_http_proxy_connect_module)确定版本对应关系。

## 安装程序

将刚刚下好的程序进行解压安装
```bash
#解压Nginx
tar -zxvf nginx-1.18.0.tar.gz
#解压ngx_http_proxy_connect_module
unzip master.zip

cd nginx-1.18.0
#打补丁，版本需注意
patch -p1 < /path/to/ngx_http_proxy_connect_module/patch/proxy_connect.patch
#执行编译命令
./configure  --prefix=/usr/share/nginx --add-module=/path/to/ngx_http_proxy_connect_module

make && make install
```

## 配置 Nginx 正向代理

配置`nginx.conf`，在`http`模块添加如下配置即可

```conf
server {
    resolver 114.114.114.114 ipv6=off; #DNS配置
    resolver_timeout 10s;
    listen 8888;
    proxy_connect;                          #启用 CONNECT HTTP方法
    proxy_connect_allow            443 80;  #指定代理CONNECT方法可以连接的端口号或范围的列表
    proxy_connect_connect_timeout  20s;     #定义客户端与代理服务器建立连接的超时时间
    proxy_connect_read_timeout     20s;     #定义客户端从代理服务器读取响应的超时时间
    proxy_connect_send_timeout     20s;     #设置客户端将请求传输到代理服务器的超时时间
    location / {
        proxy_pass $scheme://$http_host$request_uri;
    }
}
```

## 配置客户端代理服务器

### Linux
打开/etc/profile文件，在最下面添加如下配置即可
```profile
#http代理，ip是nginx的ip，端口是nginx配置的监听端口
export http_proxy="http://ip:8888"
#https代理
export https_proxy="http://ip:8888"
#不需要代理的ip,访问这些ip，不会走代理
export no_proxy="127.0.0.1, localhost"
```

## 为`Nginx`配置`systemctl`

### 创建一个`nginx.service`

在 `/usr/lib/systemd/system/` 目录下面新建一个`nginx.service`文件，并赋予可执行的权限：

`chmod +x /usr/lib/systemd/system/nginx.service`

### 编辑service内容
`vim /usr/lib/systemd/system/nginx.service`
```
[Unit] //对服务的说明
Description=nginx - high performance web server //描述服务
After=network.target remote-fs.target nss-lookup.target //描述服务类别

[Service] //服务的一些具体运行参数的设置
Type=forking //后台运行的形式
PIDFile=/usr/local/nginx/logs/nginx.pid //PID文件的路径
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf //启动准备
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf //启动命令
ExecReload=/usr/local/nginx/sbin/nginx -s reload //重启命令
ExecStop=/usr/local/nginx/sbin/nginx -s stop //停止命令
ExecQuit=/usr/local/nginx/sbin/nginx -s quit //快速停止
PrivateTmp=true //给服务分配临时空间

[Install]
WantedBy=multi-user.target //服务用户的模式
```

### 启动服务

```bash
在启动服务之前，需要先重载systemctl命令
systemctl daemon-reload
systemctl start nginx.service
```