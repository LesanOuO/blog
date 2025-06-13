---
title: "关于.NET的面试笔记"
date: "2022-11-26"
tags: [".NET"]
categories: ["实用分享"]
---

## .net 基础
1、Task和Thread有什么区别

2、当执行遇到await的时候会执行什么动作

3、事件和委托之间的区别，Action和func的区别

https://www.cnblogs.com/RaindayXia/p/5261797.html

http://mikeblog.cn/article/details/5236

4、Dictionary在多线程并发的情况下会出现什么问题，ConcurrentDictionary之间的区别

https://blog.csdn.net/xiaouncle/article/details/122011337

5、EF 在没有执行前，比如还没有Tolist()等情况下，做了什么事情

6、使用EF有没有遇到过什么问题？当时我的回答是在多线程的情况下更新实体，更新不了，必须重新取一次，才能更新，也就是跨线程了

7、IQueryable接口的作用

https://zhuanlan.zhihu.com/p/47776558

https://www.cnblogs.com/zhaopei/p/IQueryable-IQueryProvider.html

8、说说垃圾回收机制

https://www.jianshu.com/p/cc57aa7f3824

9、MVC里面注册管道处理模型的步骤，MVC有哪些过滤器，在什么样的场景下适合使用，分别介绍下。过滤器的注册方式

https://blog.csdn.net/qq_36330228/article/details/95766103

https://docs.microsoft.com/zh-cn/aspnet/core/mvc/controllers/filters?view=aspnetcore-6.0

10、AOP、IOC的作用

https://www.cnblogs.com/wswind/p/aop_in_dotnet.html

https://www.cnblogs.com/jlion/p/12392461.html

11、设计模式，一般都是问比较常用的，比如单例的三要素、工厂、代理、适配器，最好举几个例子

12、泛型里面的协变、逆变的作用

https://www.cnblogs.com/wangbaicheng1477865665/p/OutIn.html

13、C#8.0新增的语法，你知道的有哪些

## .net Core
1、.net Core启动执行顺序

2、三个依赖注入方式，生命周期、泛型注入、获取实例有哪几种方式

3、.net Core和 .net framework之间有什么不同

4、读取配置文件里面的接口IOptionsSnapshot、IOptionsMonitor之间的区别

https://www.cnblogs.com/wenhx/p/ioptions-ioptionsmonitor-and-ioptionssnapshot.html

5、读取配置文件，怎么保证修改后每次都是读取到最新的

6、配置环境变量的作用

https://zhuanlan.zhihu.com/p/361004968

7、JWT和Cookie之间的区别

https://www.cnblogs.com/gj0151/p/15333826.html

8、什么是跨域、同源，怎么防止跨域请求

## 框架
1、Quartz.net创建任务的三大对象，具体说下每个对象是用来做什么的，自己有做过扩展么，说下IJobListener、ITriggerListener接口里面的方法，怎么防止Job在同一时间重复执行

https://www.cnblogs.com/PatrickLiu/p/10193973.html

https://www.coder.work/article/1575162

2、Abp里面集成了哪些组件，事件总线你是怎么用的，注册实例有哪几种方式，说下Abp的启动流程， 你知道Application层暴露服务是怎么实现的么，需要注意哪些点
！！！

3、Autofac和.net Core原生的注入，有什么区别，什么样的场景下适合用Autofac属性注入

https://www.cnblogs.com/liang24/p/14541650.html

## 中间件
1、说下Redis里面的五大类型吧，分别适合什么样的场景使用，什么是缓存穿透、缓存雪崩，怎么避免这种情况发生，有搭建过集群么，集群的模式有哪几种，说下持久化RDB和AOF的方式区别、什么是哈希槽

https://developer.aliyun.com/article/712113

https://cloud.tencent.com/developer/article/1769953

2、介绍下RabbitMQ的四种路由，怎么设置插队，说下消息确认的方式，Tx、Confirm二种事务模式的区别

https://developer.aliyun.com/article/769883

https://cloud.tencent.com/developer/article/1487995

https://writing-bugs.blog.csdn.net/article/details/102935393

3、说下ElasticSearch的常用的数据类型，keyWord和text的区别，说下分词器的作用

4、 Apollo有哪些功能，大概的说下

https://www.cnblogs.com/edisonchou/p/netcore_microservice_apollo_foundation.html

## 微服务

1、如何设计一个高可用的微服务架构？一开始可以简单的说下，不要说的太细了，如果是问到里面的细节的话，最好仔细的去回答。这个问题是我面试架构师的时候被问到的，所以普通的开发可能不会问。

2、说说你对DDD了解的一个深度吧，说下领域是怎么划分的，介绍下值对象、聚合、聚合根，实体之间的联系。什么样的情况下聚合根当做一个整体使用，限界上下文的作用，你有没有自己独立搭建过项目，大概说下你是怎么分层的，每一层之间的联系是什么

https://zhuanlan.zhihu.com/p/91525839

https://developer.aliyun.com/article/716908

3、介绍下Polly里面的策略，失败重试、服务降级、失败降级、超时处理等

https://www.cnblogs.com/willick/p/polly.html

4、HttpClient创建的三种模式，请求拦截器的作用

5、gRPC有什么特点，Protocol Buffers序列化的原理

6、用过identityserver4么，如果说用过，会问你在项目里面是怎么用的，和JWT有什么区别，需要注意哪些点

7、dockerfile的编写，编写dockerfile常用的指令，分别介绍下

8、优化镜像有哪几种方式？我当时回答是 From基础镜像找小一点的，尽量使用alpine，或者使用多阶段构建。

9、 kubernetes和Docker之间的区别？搭建一个高可用的集群，需要用到哪些组件？

这是我的回答：nginx、Keepalived、HAProxy、Calico、metricsServer、CoreDNS、KubeProxy

10、什么是Restful？说下你在项目里面是怎么设计Api的。

## 数据库
1、 聚集索引和非聚集索引的区别

https://cloud.tencent.com/developer/article/1541265

2、 说下存储过程吧，你平时用的多么，说下优点与缺点

https://cloud.tencent.com/developer/article/1338545

3、 说下建表的三大范式

4、 怎么看执行计划，你平时是怎么优化SQL语句

https://www.cnblogs.com/powercto/p/14382208.html

5、 事务隔离级别，什么是悲观锁，这个问题出现率比较低

https://developer.aliyun.com/article/743691

## 算法
一般的公司很少有算法题，出现比较高的是一些外企、一线互联网公司，像我最近去面的微软、iHerb、lexisnexis，直接在白板、白纸上写，如果写不出，可能下面的面试没法继续聊了， Leetcode上面的简单和中等的题要通过。每天抽时间多刷一刷。这样长期积累对自己的算法功底、代码的健壮性能提高很多。

下面是今年面试遇到的一些算法题

N叉树的最大深度
平衡二叉树
爬楼梯
寻找两个正序数组的中位数
两个数组的交集
重新排列字符串
排序链表
排序算法
最近的请求次数
斐波那契数列

我目前已经积累了较多工作经验 原公司个人成长空间比较局限，工作内容更偏向于重复性的作业无法满足我的个人职业发展需求，所以决定看看其他机会
