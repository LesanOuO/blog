---
title: "Nginx笔记"
date: "2022-02-16"
tags: ['Nginx']
categories: ['编程知识']
---

本文为个人学习和实践 Nginx 的笔记

## Nginx 常用功能

Nginx 有以下几个常用功能

1. 反向代理

   这是它的主要功能之一，客户端向服务器发送请求时，会先通过 Nginx 服务器，由服务器将请求分发到相应的Web服务器。正向代理是代理客户端，而反向代理则是代理服务器，Nginx 在提供反向代理服务方面，通过使用正则表达式进行相关配置，采取不同的转发策略，配置相当灵活，而且在配置后端转发请求时，完全不用关心网络环境如何，可以指定任意的IP地址和端口号，或其他类型的连接、请求等。

2. 负载均衡

   这也是 Nginx 最常用的功能之一，负载均衡，一方面是将单一的重负载分担到多个网络节点上做并行处理，每个节点处理结束后将结果汇总返回给用户，这样可以大幅度提高网络系统的处理能力；另一方面将大量的前端并发请求或数据流量分担到多个后端网络节点分别处理，这样可以有效减少前端用户等待相应的时间。而 Nginx 负载均衡都是属于后一方面，主要是对大量前端访问或流量进行分流，已保证前端用户访问效率，并可以减少后端服务器处理压力。

3. Web缓存

   在很多优秀的网站中，Nginx 可以作为前置缓存服务器，它被用于缓存前端请求，从而提高 Web服务器的性能。Nginx 会对用户已经访问过的内容在服务器本地建立副本，这样在一段时间内再次访问该数据，就不需要通过 Nginx 服务器向后端发出请求。减轻网络拥堵，减小数据传输延时，提高用户访问速度。

## Nginx 安装

### 下载地址

Nginx 下载地址：http://nginx.org/en/download.html

### Windows 版本安装

解压下载的文件后

![目录](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20220216084221.png)

下面对上面文件夹进行介绍：

1. conf 目录：存放 Nginx 的主要配置文件，很多功能实现都是通过配置该目录下的 nginx.conf 文件，后面我们会详细介绍。
2. docs目录：存放 Nginx 服务器的主要文档资料，包括 Nginx 服务器的 LICENSE、OpenSSL 的 LICENSE 、PCRE 的 LICENSE 以及 zlib 的 LICENSE ，还包括本版本的 Nginx服务器升级的版本变更说明，以及 README 文档。
3. html目录：存放了两个后缀名为 .html 的静态网页文件，这两个文件与 Nginx 服务器的运行相关。
4. logs目录：存放 Nginx 服务器运行的日志文件。
5. nginx.exe：启动 Nginx 服务器的exe文件，如果 conf 目录下的 nginx.conf 文件配置正确的话，通过该文件即可启动 Nginx 服务器。

关闭 nginx 的方法：

进入到 nginx 目录并且输入以下命令：`nginx.exe -s stop`

### Linux 版本安装

首先需要安装 nginx 的依赖环境：

```bash
yum install gcc-c++
yum install -y pcre pcre-devel
yum install -y zlib zlib-devel
yum install -y openssl openssl-devel
```

1. 对于 gcc，因为安装nginx需要先将官网下载的源码进行编译，编译依赖gcc环境，如果没有gcc环境的话，需要安装gcc。
2. 对于 pcre，prce(Perl Compatible Regular Expressions)是一个Perl库，包括 perl 兼容的正则表达式库。nginx的http模块使用pcre来解析正则表达式，所以需要在linux上安装pcre库。
3. 对于 zlib，zlib库提供了很多种压缩和解压缩的方式，nginx使用zlib对http包的内容进行gzip，所以需要在linux上安装zlib库。
4. 对于 openssl，OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及SSL协议，并提供丰富的应用程序供测试或其它目的使用。nginx不仅支持http协议，还支持https（即在ssl协议上传输http），所以需要在linux安装openssl库。

编译及安装：

```bash
1. 首先将下载的文件放到 Linux 系统中，然后解压：
tar -zxvf nginx-1.14.0.tar.gz

2. 接着进入解压之后的目录进行编译安装
./configure --prefix=/usr/local/nginx
make
make install

3. 进入到/usr/local/nginx目录，再进入sbin目录，通过以下命令启动nginx:
./nginx
通过 ps -ef | grep nginx 查看nginx的进程

4. 关闭nginx：
快速关闭：cd /usr/local/nginx/sbin ./nginx -s stop 相当于直接kill掉nginx的进程id
平缓关闭：cd /usr/local/nginx/sbin ./nginx -s quit 等nginx服务处理完所有请求后再关闭连接，停止工作

5. 重启nginx
先停止再启动：./nginx -s quit ./nginx
重新加载配置文件：./nginx -s reload

6. 检测配置文件语法是否正确
指定需要检测的配置文件：nginx -t -c /usr/local/nginx/conf/nginx.conf
检测默认nginx.conf配置文件：nginx -t

```



## nginx.conf 配置文件

根据默认配置文件，我们可以将nginx.conf配置文件分为三部分：

### 全局块

从配置文件开始到 events 块之间的内容，主要会设置一些影响 nginx 服务器整体运行的配置指令，主要包括配置运行 Nginx 服务器的用户（组）、允许生成的 worker process 数，进程 PID 存放路径、日志存放路径和类型以及配置文件的引入等。

比如：`worker_processes 1;` 这是Nginx服务器并发处理服务的关键配置，worker_processes 值越大，可以支持的并发处理量也越多，但是会受到硬件、软件等设备的制约。

### events 块

比如：

```
events {
    worker_connections  1024;
}
```

events 块涉及的指令主要影响 Nginx 服务器与用户的网络连接，常用的设置包括是否开启对多 work process 下的网络连接进行序列化，是否允许同时接收多个网络连接，选取哪种事件驱动模型来处理连接请求，每个 word process 可以同时支持的最大连接数等。

上述例子就表示每个 work process 支持的最大连接数为 1024.

这部分的配置对 Nginx 的性能影响较大，在实际中应该灵活配置。

### http 块

```
http {
    include       mime.types;
    default_type  application/octet-stream;


    sendfile        on;

    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }

}
```

这是Nginx服务器配置中最频繁的部分，代理、缓存和日志定义等绝大多数功能和第三方模块的配置都在这里。

http 块也可以包括 **http全局块**、**server 块**：

1. http全局块

   http全局块配置的指令包括文件引入、MIME-TYPE 定义、日志自定义、连接超时时间、单链接请求数上限等。

2. server块

   这块和虚拟主机有密切关系，虚拟主机从用户角度看，和一台独立的硬件主机是完全一样的，该技术的产生是为了节省互联网服务器硬件成本。后面会详细介绍虚拟主机的概念。

   每个 http 块可以包括多个 server 块，而每个 server 块就相当于一个虚拟主机。

   而每个 server 块也分为全局 server 块，以及可以同时包含多个 locaton 块。

   全局server块：最常见的配置是本虚拟机主机的监听配置和本虚拟主机的名称或IP配置。

   location块：一个 server 块可以配置多个 location 块。

   这块的主要作用是基于 Nginx 服务器接收到的请求字符串（例如 server_name/uri-string），对虚拟主机名称（也可以是IP别名）之外的字符串（例如 前面的 /uri-string）进行匹配，对特定的请求进行处理。地址定向、数据缓存和应答控制等功能，还有许多第三方模块的配置也在这里进行。



## 反向代理

### 代理定义

Nginx 主要能够代理如下几种协议，其中用到的最多的就是做Http代理服务器。

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20220216105556.png)

在Java设计模式中，代理模式是这样定义的：给某个对象提供一个代理对象，并由代理对象控制原对象的引用。

代理简单来说，就是如果我们想做什么，但又不想直接去做，那么这时候就找另外一个人帮我们去做。那么这个例子里面的中介公司就是给我们做代理服务的，我们委托中介公司帮我们找房子。

### 正向代理定义

　　弄清楚什么是代理了，那么什么又是正向代理呢？

　　这里我再举一个例子：大家都知道，现在国内是访问不了 Google的，那么怎么才能访问 Google呢？我们又想，美国人不是能访问 Google吗（这不废话，Google就是美国的），如果我们电脑的对外公网 IP 地址能变成美国的 IP 地址，那不就可以访问 Google了。你很聪明，VPN 就是这样产生的。我们在访问 Google 时，先连上 VPN 服务器将我们的 IP 地址变成美国的 IP 地址，然后就可以顺利的访问了。

　　这里的 VPN 就是做正向代理的。正向代理服务器位于客户端和服务器之间，为了向服务器获取数据，客户端要向代理服务器发送一个请求，并指定目标服务器，代理服务器将目标服务器返回的数据转交给客户端。这里客户端是要进行一些正向代理的设置的。

　　PS：这里介绍一下什么是 VPN，VPN 通俗的讲就是一种中转服务，当我们电脑接入 VPN 后，我们对外 IP 地址就会变成 VPN 服务器的 公网 IP，我们请求或接受任何数据都会通过这个VPN 服务器然后传入到我们本机。这样做有什么好处呢？比如 VPN 游戏加速方面的原理，我们要玩网通区的 LOL，但是本机接入的是电信的宽带，玩网通区的会比较卡，这时候就利用 VPN 将电信网络变为网通网络，然后在玩网通区的LOL就不会卡了（注意：VPN 是不能增加带宽的，不要以为不卡了是因为网速提升了）。

　　可能听到这里大家还是很抽象，没关系，和下面的反向代理对比理解就简单了。

### 反向代理定义

　　反向代理和正向代理的区别就是：**正向代理代理客户端，反向代理代理服务器。**

　　反向代理，其实客户端对代理是无感知的，因为客户端不需要任何配置就可以访问，我们只需要将请求发送到反向代理服务器，由反向代理服务器去选择目标服务器获取数据后，在返回给客户端，此时反向代理服务器和目标服务器对外就是一个服务器，暴露的是代理服务器地址，隐藏了真实服务器IP地址。

　　下面我们通过两张图来对比正向代理和方向代理：

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20220216110708.png)

理解这两种代理的关键在于代理服务器所代理的对象是什么，正向代理代理的是客户端，我们需要在客户端进行一些代理的设置。而反向代理代理的是服务器，作为客户端的我们是无法感知到服务器的真实存在的。

总结起来还是一句话：**正向代理代理客户端，反向代理代理服务器。**

### Nginx 反向代理

#### 相关指令

##### listen

该指令用于配置网络监听。主要有如下三种配置语法结构：

① 配置监听的IP地址

`listen address[:port] [default_server] [setfib=number] [backlog=number] [rcvbuf=size] [sndbuf=size] [deferred] [accept_filter=filter] [bind] [ssl];`

② 配置监听端口

`listen port [default_server] [setfib=number] [backlog=number] [rcvbuf=size] [sndbuf=size] [accept_filter=filter] [deffered] [bind] [ipv6only=on|off] [ssl];`

③ 配置 UNIX Domain Socket

`listen unix:path [default_server] [backlog=number] [rcvbuf=size] [sndbuf=size] [accept_filter=filter] [deffered] [bind] [ssl]`

上面的配置看似比较复杂，其实使用起来是比较简单的：

```
listen *:80 | *:8080 #监听所有80端口和8080端口
listen  IP_address:port   #监听指定的地址和端口号
listen  IP_address     #监听指定ip地址所有端口
listen port     #监听该端口的所有IP连接
```

下面分别解释每个选项的具体含义：

1、address:IP地址，如果是 IPV6地址，需要使用中括号[] 括起来，比如[fe80::1]等。

2、port:端口号，如果只定义了IP地址，没有定义端口号，那么就使用80端口。

3、path:socket文件路径，如 var/run/nginx.sock等。

4、default_server:标识符，将此虚拟主机设置为 address:port 的默认主机。（在 nginx-0.8.21 之前使用的是 default 指令）

5、 setfib=number:Nginx-0.8.44 中使用这个变量监听 socket 关联路由表，目前只对 FreeBSD 起作用，不常用。

6、backlog=number:设置监听函数listen()最多允许多少网络连接同时处于挂起状态，在 FreeBSD 中默认为 -1,其他平台默认为511.

7、rcvbuf=size:设置监听socket接收缓存区大小。

8、sndbuf=size:设置监听socket发送缓存区大小。

9、deferred:标识符，将accept()设置为Deferred模式。

10、accept_filter=filter:设置监听端口对所有请求进行过滤，被过滤的内容不能被接收和处理，本指令只在 FreeBSD 和 NetBSD 5.0+ 平台下有效。filter 可以设置为 dataready 或 httpready 。

11、bind:标识符，使用独立的bind() 处理此address:port，一般情况下，对于端口相同而IP地址不同的多个连接，Nginx 服务器将只使用一个监听指令，并使用 bind() 处理端口相同的所有连接。

12、ssl:标识符，设置会话连接使用 SSL模式进行，此标识符和Nginx服务器提供的 HTTPS 服务有关。



##### server_name

该指令用于虚拟主机的配置。通常分为以下两种：

**1、基于名称的虚拟主机配置**

语法格式如下：

```
server_name   name ...;
```

一、对于name 来说，可以只有一个名称，也可以有多个名称，中间用空格隔开。而每个名字由两段或者三段组成，每段之间用“.”隔开。

```
server_name 123.com www.123.com
```

二、可以使用通配符“*”，但通配符只能用在由三段字符组成的首段或者尾端，或者由两端字符组成的尾端。

```
server_name *.123.com www.123.*
```

三、还可以使用正则表达式，用“~”作为正则表达式字符串的开始标记。

```
server_name ~^www\d+\.123\.com$;
```

该表达式“~”表示匹配正则表达式，以www开头（“^”表示开头），紧跟着一个0~9之间的数字，在紧跟“.123.co”，最后跟着“m”($表示结尾)

以上匹配的顺序优先级如下：

```
1 ①、准确匹配 server_name
2 ②、通配符在开始时匹配 server_name 成功
3 ③、通配符在结尾时匹配 server_name 成功
4 ④、正则表达式匹配 server_name 成功
```

**2、基于 IP 地址的虚拟主机配置**

语法结构和基于域名匹配一样，而且不需要考虑通配符和正则表达式的问题。

```
server_name 192.168.1.1
```



##### location

该指令用于匹配 URL。

　　语法如下：

```
location [ = | ~ | ~* | ^~] uri {

}
```

　　1、= ：用于不含正则表达式的 uri 前，要求请求字符串与 uri 严格匹配，如果匹配成功，就停止继续向下搜索并立即处理该请求。

　　2、~：用于表示 uri 包含正则表达式，并且区分大小写。

　　3、~*：用于表示 uri 包含正则表达式，并且不区分大小写。

　　4、^~：用于不含正则表达式的 uri 前，要求 Nginx 服务器找到标识 uri 和请求字符串匹配度最高的 location 后，立即使用此 location 处理请求，而不再使用 location 块中的正则 uri 和请求字符串做匹配。

　　注意：如果 uri 包含正则表达式，则必须要有 ~ 或者 ~* 标识。



##### proxy_pass

　　该指令用于设置被代理服务器的地址。可以是主机名称、IP地址加端口号的形式。

　　语法结构如下：

```
proxy_pass URL;
```

　　URL 为被代理服务器的地址，可以包含传输协议、主机名称或IP地址加端口号，URI等。

```
proxy_pass  http://www.123.com/uri;
```



##### index

　　该指令用于设置网站的默认首页。

　　语法为：

```
index  filename ...;
```

　　后面的文件名称可以有多个，中间用空格隔开。

```
index  index.html index.jsp;
```

　　通常该指令有两个作用：第一个是用户在请求访问网站时，请求地址可以不写首页名称；第二个是可以对一个请求，根据请求内容而设置不同的首页。

### 配置案例

```
server {
    listen       80;
    server_name  localhost;
    charset utf-8;

    location / {
        root   /home/ruoyi/projects/ruoyi-ui;
        try_files $uri $uri/ /index.html;
        index  index.html index.htm;
    }

    location /prod-api/ {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://localhost:8080/;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
    	root   html;
    }
}
```



## 负载均衡

### 负载均衡的由来

早期的系统架构，基本上都是如下形式的：

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20220216113322.png)

客户端发送多个请求到服务器，服务器处理请求，有一些可能要与数据库进行交互，服务器处理完毕后，再将结果返回给客户端。

这种架构模式对于早期的系统相对单一，并发请求相对较少的情况下是比较适合的，成本也低。但是随着信息数量的不断增长，访问量和数据量的飞速增长，以及系统业务的复杂度增加，这种架构会造成服务器相应客户端的请求日益缓慢，并发量特别大的时候，还容易造成服务器直接崩溃。很明显这是由于服务器性能的瓶颈造成的问题，那么如何解决这种情况呢？

我们首先想到的可能是升级服务器的配置，比如提高CPU执行频率，加大内存等提高机器的物理性能来解决此问题，但是我们知道[摩尔定律](https://www.cnblogs.com/ysocean/p/7641540.html)的日益失效，硬件的性能提升已经不能满足日益提升的需求了。最明显的一个例子，天猫双十一当天，某个热销商品的瞬时访问量是极其庞大的，那么类似上面的系统架构，将机器都增加到现有的顶级物理配置，都是不能够满足需求的。那么怎么办呢？

上面的分析我们去掉了增加服务器物理配置来解决问题的办法，也就是说纵向解决问题的办法行不通了，那么横向增加服务器的数量呢？这时候集群的概念产生了，单个服务器解决不了，我们增加服务器的数量，然后将请求分发到各个服务器上，将原先请求集中到单个服务器上的情况改为将请求分发到多个服务器上，将负载分发到不同的服务器，也就是我们所说的**负载均衡**。

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20220216113509.png)

负载均衡完美的解决了单个服务器硬件性能瓶颈的问题，但是随着而来的如何实现负载均衡呢？客户端怎么知道要将请求发送到那个服务器去处理呢？

### Nginx 实现负载均衡

Nginx 服务器是介于客户端和服务器之间的中介，通过上一节的反向代理的功能，客户端发送的请求先经过 Nginx ，然后通过 Nginx 将请求根据相应的规则分发到相应的服务器。

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20220216114059.png)

主要配置指令为上一讲的 pass_proxy 指令以及 upstream 指令。负载均衡主要通过专门的硬件设备或者软件算法实现。通过硬件设备实现的负载均衡效果好、效率高、性能稳定，但是成本较高。而通过软件实现的负载均衡主要依赖于均衡算法的选择和程序的健壮性。均衡算法又主要分为两大类：

静态负载均衡算法：主要包括轮询算法、基于比率的加权轮询算法或者基于优先级的加权轮询算法。

动态负载均衡算法：主要包括基于任务量的最少连接优化算法、基于性能的最快响应优先算法、预测算法及动态性能分配算法等。

静态负载均衡算法在一般网络环境下也能表现的比较好，动态负载均衡算法更加适用于复杂的网络环境。

例子：

1. 普通轮询算法

```
upstream OrdinaryPolling {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}
server {
    listen       80;
    server_name  localhost;

    location / {
        proxy_pass http://OrdinaryPolling;
        index  index.html index.htm index.jsp;

    }
}
```

2. 基于比例加权轮询

```
upstream OrdinaryPolling {
    server 127.0.0.1:8080 weight=5;
    server 127.0.0.1:8081 weight=2;
}
server {
    listen       80;
    server_name  localhost;

    location / {
        proxy_pass http://OrdinaryPolling;
        index  index.html index.htm index.jsp;

    }
}
```

3. 基于IP路由负载

我们知道一个请求在经过一个服务器处理时，服务器会保存相关的会话信息，比如session，但是该请求如果第一个服务器没处理完，通过nginx轮询到第二个服务器上，那么这个服务器是没有会话信息的。

最典型的一个例子：用户第一次进入一个系统是需要进行登录身份验证的，首先将请求跳转到Tomcat1服务器进行处理，登录信息是保存在Tomcat1 上的，这时候需要进行别的操作，那么可能会将请求轮询到第二个Tomcat2上，那么由于Tomcat2 没有保存会话信息，会以为该用户没有登录，然后继续登录一次，如果有多个服务器，每次第一次访问都要进行登录，这显然是很影响用户体验的。

这里产生的一个问题也就是集群环境下的 session 共享，如何解决这个问题？

通常由两种方法：

1、第一种方法是选择一个中间件，将登录信息保存在一个中间件上，这个中间件可以为 Redis 这样的数据库。那么第一次登录，我们将session 信息保存在 Redis 中，跳转到第二个服务器时，我们可以先去Redis上查询是否有登录信息，如果有，就能直接进行登录之后的操作了，而不用进行重复登录。

2、第二种方法是根据客户端的IP地址划分，每次都将同一个 IP 地址发送的请求都分发到同一个 Tomcat 服务器，那么也不会存在 session 共享的问题。

而 nginx 的基于 IP 路由负载的机制就是上诉第二种形式。大概配置如下：

```
upstream OrdinaryPolling {
    ip_hash;
    server 127.0.0.1:8080 weight=5;
    server 127.0.0.1:8081 weight=2;
}
server {
    listen       80;
    server_name  localhost;

    location / {
        proxy_pass http://OrdinaryPolling;
        index  index.html index.htm index.jsp;

    }
}
注意：我们在 upstream 指令块中增加了 ip_hash 指令。该指令就是告诉 nginx 服务器，同一个 IP 地址客户端发送的请求都将分发到同一个 Tomcat 服务器进行处理。
```

4. 基于服务器响应时间负载分配

根据服务器处理请求的时间来进行负载，处理请求越快，也就是响应时间越短的优先分配。

```
upstream OrdinaryPolling {
    server 127.0.0.1:8080 weight=5;
    server 127.0.0.1:8081 weight=2;
    fair;
}
    server {
    listen       80;
    server_name  localhost;

    location / {
        proxy_pass http://OrdinaryPolling;
        index  index.html index.htm index.jsp;

    }
}
通过增加了 fair 指令。
```

5. 对不同域名实现负载均衡

通过配合location 指令块我们还可以实现对不同域名实现负载均衡。

```
upstream wordbackend {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}

upstream pptbackend {
    server 127.0.0.1:8082;
    server 127.0.0.1:8083;
}

server {
    listen       80;
    server_name  localhost;

    location /word/ {
        proxy_pass http://wordbackend;
        index  index.html index.htm index.jsp;

    }
    location /ppt/ {
        proxy_pass http://pptbackend;
        index  index.html index.htm index.jsp;

    }
}
```

## 引用

以上学习笔记多处摘录自网络

> https://www.cnblogs.com/ysocean/p/9392912.html