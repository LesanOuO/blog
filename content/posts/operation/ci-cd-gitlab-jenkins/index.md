---
title: "记一次搭建GitLab+Jenkins实现CI&CD"
date: "2022-04-01"
tags: ["CI&CD", "GitLab", "Jenkins"]
categories: ["动手实践"]
weight:
---

本篇文章记录本人搭建CI&CD实现持续集成和持续部署

## 1.使用docker安装gitlab

### 下载镜像 （使用中文社区版）

![](images/1.png)

`docker pull twang2218/gitlab-ce-zh`

### 创建所需目录为后续挂载文件

进入所需目录后，打开PowerShell，通过以下命令进行目录创建

`mkdir -p gitlab/etc ` 、 `mkdir -p gitlab/etc` 、 `mkdir -p gitlab/etc`

目录结构如下图所示

![](images/2.png)

### 启动容器

镜像下载完成后可通过 `docker images`查看下载结果，再通过镜像启动为容器

```bash
docker run -d -p 9443:443 -p 9080:80 -p 9022:22 --restart always --name testgitlab -v D:\testgitlab\gitlab\etc:/etc/gitlab -v D:\testgitlab\gitlab\log:/var/log/gitlab -v D:\testgitlab\gitlab\data:/var/opt/gitlab --privileged=true twang2218/gitlab-ce-zh
# 执行完成后会返回一串字符串
```

其中：

`-d：`后台执行

`-p：`端口映射

`--restart：`重启机制

`--name：`容器名称

`-v：`挂载文件，使得容器内文件在宿主机内有映射

`--privileged：`使得容器获取宿主机root权限

### 进入容器，修改配置

输入命令 `docker exec -it testgitlab bash` 即可进入刚刚创建好的容器

1.修改 `gitlab.rb`配置的两种方式：1. 进入挂载好的etc目录下找到 `gitlab.rb`文件进行修改；2. 通过进入容器内进行命令行 `vi /etc/gitlab/gitlab.rb `修改

```
# 整个gitlab.rb都是注释了的，我们可以按需加入我们的配置
# 1. gitlab访问地址，可以写域名。如果端口不写的话默认为80端口
eaxternal_url 'http://192.168.3.12:9080'
# 2. ssh主机ip
gitlab_rails['gitlab_ssh_host'] = '192.168.3.12'
# 3. ssh连接端口
gitlab_rails['gitlab_shell_ssh_port'] = 9022
# 4. 防止内存占用过大，限制线程数
unicorn['worker_processes'] = 2
```

2.修改 `gitlab.yml`配置（这一步原本不是必须的，因为 `gitlab.rb`内配置会覆盖这个，为了防止没有成功覆盖所以我在这里进行配置，当然你也可以选择不修改 `gitlab.rb`直接修改这里）

通过命令行 `vi /opt/gitlab/embedded/service/gitlab-rails/config/gitlab.yml`修改，或者找到挂载文件修改

![](images/3.png)

![](images/4.png)

修改上图红框配置

3.让修改后的配置生效，并重启

`gitlab-ctl reconfigure` 、 `gitlab-ctl restart` 、 `exit`（退出容器命令行）

或者重启容器 `docker restart testgitlab`

### 访问gitlab

输入http://192.168.3.12:9080打开页面（ip请输入前面设置的），默认账户root，密码需要重新设置至少8位

![](images/5.png)

## 2.使用docker安装jenkins

### 下载镜像

`docker pull jenkins/jenkins`

### 创建所需目录为后续挂载文件

在服务器上先创建一个jenkins工作目录 /var/jenkins_mount，赋予相应权限，稍后我们将jenkins容器目录挂载到这个目录上，这样我们就可以很方便地对容器内的配置文件进行修改。如何后续在容器内修改的话会非常麻烦，由于容器中无vi命令。

`mkdir -p /var/jenkins_mount` 、 `chmod 777 /var/jenkins_mount`

### 启动容器

```bash
docker run -d -p 10240:8080 -p 10241:50000 -v D:\testjenkins\jenkins_mount:/var/jenkins_home -v /etc/localtime:/etc/localtime --name testjenkins jenkins/jenkins
# 其中-v /etc/localtime:/etc/localtime让容器使用和服务器同样的时间设置
```

可以通过 `docker ps`来查看启动情况

可以通过 `docker logs testjenkins`来查看容器日志

### 修改配置

进入刚刚挂载的文件，修改 `hudson.model.UpdateCenter.xml`文件

将 url 修改为 清华大学官方镜像：https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json ，配置镜像加速

![](images/6.png)

还需要修改 `default.json`文件，位置为 `cd /var/jenkins_home/updates`

使用sed命令修改 `default.json`

linux下：

```bash
sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```

mac下：

```bash
sed -i "" 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i "" 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```

重启容器 `docker restart testjenkins`

### 访问jenkins

输入http://localhost:10240打开页面

![](images/7.png)

选择默认插件安装即可

## 3.gitlab + jenkins

为了实现自动持续构建, 不需要人工操作 ( 留人工操作用于处理特殊情况 )，通过gitlab+jenkins实现CI&CD，具体流程如下

![](images/8.png)

1. 开发提交代码
2. 开发对需要发布的版本打上 **Tag**
3. 触发 **GitLab** 的 **tag push** 事件, 调用 **Webhook**
4. **Webhook** 触发 **Jenkins** 的构建任务
5. **Jenkins** 构建完项目可以按版本号上传到仓库、部署、通知相关人员等等

### 配置gitlab

1. 建一个测试项目 **test** ，随便 **commit** 一些内容，比如通过网页添加 `README.md`

![](images/9.png)

2. 创建账号的 **access token** ，用于 **Jenkins** 调用 **GitLab** 的 **API**

![](images/10.png)

3. 记下生成的 **access token**  , 后面需要用到！！！且它只会展示一次，请记录好！！！

![](images/11.png)

### 配置jenkins

#### 安装环境所需插件

- [Git Parameter](https://links.jianshu.com/go?to=https%3A%2F%2Fwiki.jenkins.io%2Fdisplay%2FJENKINS%2FGit%2BParameter%2BPlugin) ( 用于参数化构建中动态获取项目分支 )
- [Generic Webhook Trigger](https://links.jianshu.com/go?to=https%3A%2F%2Fwiki.jenkins-ci.org%2Fdisplay%2FJENKINS%2FGeneric%2BWebhook%2BTrigger%2BPlugin) ( 用于解析 **Webhook** 传过来的参数 )
- [GitLab](https://links.jianshu.com/go?to=https%3A%2F%2Fwiki.jenkins-ci.org%2Fdisplay%2FJENKINS%2FGitLab%2BPlugin) ( 用于推送构建结果给 **GitLab** )

#### 添加 GitLab 凭据

![](images/12.png)

![](images/13.png)

![](images/14.png)

#### 在系统配置中配置gitlab

![](images/15.png)

![](images/16.png)

#### 创建新的FreeStyle任务

- **General**

**勾选** 参数化构建过程, 添加 **Git Parameter** 类型的参数 **ref** , 这样构建的时候就可以指定分支进行构建

![](images/17.png)

- **源码管理**

选择 **Git** , 添加项目地址和授权方式 ( **帐号密码** 或者 **ssh key** ) , 分支填写构建参数 **$ref**

![](images/18.png)

- **构建触发器**

选择 **Generic Webhook Trigger** 方式用于解析 **GitLab** 推过来的详细参数 ( [jsonpath 在线测试](https://links.jianshu.com/go?to=http%3A%2F%2Fjsonpath.com) ) 。其他触发方式中: [Trigger builds remotely](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Fjwentest%2Fp%2F8204421.html) 是 **Jenkins** 自带的, **Build when a change is pushed to GitLab** 是 **GitLab 插件** 提供的, 都属于简单的触发构建, 无法做复杂的处理

![](images/19.png)

- **Optional filter**

虽然 **Generic Webhook Trigger** 提供了 **Token** 参数进行鉴权, 但为了避免不同项目进行混调 ( 比如 A 项目提交代码却触发了 B 项目的构建) , 还要对请求做下过滤。**Optional filter** 中 **Text** 填写需要校验的内容 ( 可使用变量 ) , **Expression** 使用正则表达式对 **Text** 进行匹配, 匹配成功才允许触发构建

![](images/20.png)

- **构建**

构建内容按自己实际的项目类型进行调整, 使用 **Maven 插件** 或 **脚本** 等等

![](images/21.png)

- **构建后操作**

 构建后操作添加 **Publish build status to GitLab** 动作, 实现构建结束后通知构建结果给 **GitLab**

![](images/22.png)

#### 在GitLab的项目页面中, 添加一个Webhook

添加一个 **Webhook** ( [http://JENKINS_URL/generic-webhook-trigger/invoke?token=](https://links.jianshu.com/go?to=http%3A%2F%2FJENKINS_URL%2Fgeneric-webhook-trigger%2Finvoke%3Ftoken%3D)<上面 **Jenkins** 项目配置中的 **token**> ) , 触发器选择 **标签推送事件**。因为日常开发中 **push** 操作比较频繁而且不是每个版本都需要构建, 所以只针对需要构建的版本打上 **Tag** 就好了

`http://172.20.10.7:10240//generic-webhook-trigger/invoke?token=d63ad84eb18cb04d4459ec347a196dce`

![](images/23.png)

创建完使用 **test 按钮** 先测试下, 可能会出现下面的错误

`Requests to the local network are not allowed` 通过下面方法解决

![](images/24.png)

### 测试效果

将代码拉下来在本地操作通过IDEA进行操作

![](images/25.png)

然后使用快捷键 **Cmd + Shift + K** 调出 **Push 窗口** , 把 **Tag** 推送到 **GitLab** 中

![](images/26.png)

回到 **GitLab** 页面可以看到触发了 **Webhook** , **View details** 查看请求详情, **Response body** 中 `triggered` 字段值为 `true` 则表示成功触发了 **Jenkins** 进行构建

![](images/27.png)

**注意:** 每添加一个 **Tag** 就会触发一次事件, 不管是不是一起 **push** 的。所以一次 **push** 多个 **Tag** 会触发 **Jenkins** 进行多次构建。不过 **Jenkins** 已经做了处理, 默认串行执行任务 ( 一个任务结束再执行下一个 ) , 而且在构建前有一个 **pending** 状态, 此时被多次触发会进行合并, 并取首次触发的参数, 如下图所示:

![](images/28.png)

### 关于 Tag 的几点说明

- 推送 **Tag** 到远端的时候, 远端已存在 ( 同名 ) 的 **Tag** 不会被添加到远端
- 拉取远端的 **Tag** 时, 本地已存在 ( 同名 ) 的 **Tag** 不会添加到本地
- 拉取远端的 **Tag** 时, 本地不会删除远端已删除的 **Tag** , 需要同步远端的 **Tag** 可以先删除本地所有 **Tag** 再 **pull**
- 删除 **Tag** 也会推送事件, 要做好过滤 ( 上面配置中已使用 **commitsId** 字段进行过滤 )

> 本篇文章产考下列文章：\
> [docker安装gitlab](https://www.cnblogs.com/hanease/p/15690227.html)\
> [docker安装jenkins](https://www.cnblogs.com/fuzongle/p/12834080.html)\
> [docker中jenkins插件加速](https://juejin.cn/post/6844904137570648072)\
> [整合gitlab+jenkins](https://www.jianshu.com/p/7e8037c63d63)
