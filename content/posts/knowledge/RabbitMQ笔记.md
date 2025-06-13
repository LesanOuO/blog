---
title: "RabbitMQ笔记"
date: "2022-02-17"
tags: ['RabbitMQ','.NET']
categories: ['编程知识']
---

## RabbitMQ简述

MQ全称为Message Queue, 消息队列（MQ）是一种应用程序对应用程序的通信方法。RabbitMQ是一个在AMQP基础上完整的，可复用的企业消息系统。他遵循Mozilla Public License开源协议。AMQP(高级消息队列协议) 是一个异步消息传递所使用的应用层协议规范，作为线路层协议，而不是API（例如JMS），AMQP 客户端能够无视消息的来源任意发送和接受信息。AMQP的原始用途只是为金融界提供一个可以彼此协作的消息协议，而现在的目标则是为通用消息队列架构提供通用构建工具。因此，面向消息的中间件 （MOM）系统，例如发布/订阅队列，没有作为基本元素实现。AMQP当中有四个概念非常重要（一个虚拟主机持有一组交换机、队列和绑定）：

1. `virtual host`，虚拟主机
2. `exchange`，交换机
3. `queue`，队列
4. `binding`，绑定

## 消息传递过程

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/rabbitmq1.jpg)

上图为RabbitMQ中一些重要名词的概述，其中包括Connection、Channel、Exchange、Queue、Bind、Routing Key

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/rabbitmq2.jpg)

上图为消息从生产到消费的整个流程，其中Exchange，与Queue都是可以设置相关属性，队列的持久化，交换器类型制定

这个过程走分三个部分，1、客户端（生产消息队列），2、RabbitMQ服务端（负责路由规则的绑定与消息的分发），3、客户端（消费消息队列中的消息）

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/rabbitmq3.jpg)

由图可以看出，一个消息可以走一次网络却被分发到不同的消息队列中，然后被多个的客户端消费，那么这个过程就是RabbitMQ的核心机制，RabbitMQ的路由类型与消费模式

## RabbitMQ中Exchange的类型

类型有4种，direct，fanout，topic，headers。其中headers不常用，本篇不做介绍，其他三种类型，会做详细介绍。

那么这些类型是什么意思呢？就是Exchange与队列进行绑定后，消息根据exchang的类型，按照不同的绑定规则分发消息到消息队列中，可以是一个消息被分发给多个消息队列，也可以是一个消息分发到一个消息队列。具体请看下文。

介绍之初还要说下RoutingKey，这是个什么玩意呢？他是exchange与消息队列绑定中的一个标识。有些路由类型会按照标识对应消息队列，有些路由类型忽略routingkey。具体看下文。

### Exchange类型direct

根据交换器名称与routingkey来找队列的

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20220217105315.png)

Note:消息从client发出，传送给交换器ChangeA，RoutingKey为routingkey.ZLH,那么不管你发送给Queue1，还是Queue2一个消息都会保存在Queue1，Queue2，Queue3，三个队列中。这就是交换器的direct类型的路由规则。只要找到路由器与routingkey绑定的队列，那么他有多少队列，他就分发给多少队列。

### Exchange类型fanout

这个类型忽略Routingkey，他为广播模式

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20220217105352.png)

Note:消息从客户端发出，只要queue与exchange有绑定，那么他不管你的Routingkey是什么他都会将消息分发给所有与该exchang绑定的队列中。

### Exchange类型topic

这个类型的路由规则如果你掌握啦，那是相当的好用，与灵活。他是根据RoutingKey的设置，来做匹配的，其中这里还有两个通配符为：

\*，代表任意的一个词。例如topic.zlh.*，他能够匹配到，topic.zlh.one ,topic.zlh.two ,topic.zlh.abc, ....

\#，代表任意多个词。例如topic.#，他能够匹配到，topic.zlh.one ,topic.zlh.two ,topic.zlh.abc, ....

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20220217105423.png)

Note：这个图看上去很乱，但是他是根据匹配符做匹配的，这里我建议你自己做下消息队列的具体操作。

## 消息队列的消费与消息确认

### 消息队列的消费

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20220217110443.png)

Note:如果一个消息队列中有大量消息等待操作时，我们可以用多个客户端来处理消息，这里的分发机制是采用负载均衡算法中的轮询。第一个消息给A，下一个消息给B，下下一个消息给A，下下下一个消息给B......以此类推

### 消息确认

保证消息的安全性，保证此消息被正确处理后才能在服务端的消息队列中删除。那么rabbitmq提供啦ack应答机制，来实现这一功能。

ack应答有两种方式：1、自动应答，2、手动应答



## 使用例子

### 生产者

```csharp
using RabbitMQ.Client;
using System;
using System.Text;

namespace RabbitMQProduct
{
    internal class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Welcome to RabbitMQ Product!");
            DirectExchangeSendMsg();
            // TopicExchangeSendMsg();
            Console.WriteLine("按任意值，退出程序");
            Console.ReadKey();
        }

        /// <summary>
        /// 连接配置
        /// </summary>
        private static readonly ConnectionFactory rabbitMqFactory = new ConnectionFactory()
        {
            UserName = "guest",
            Password = "guest",
            Port = 5672,
            //VirtualHost = "LesanVirtualHost"
        };
        /// <summary>
        /// 路由名称
        /// </summary>
        const string ExchangeName = "Lesan.exchange";

        //队列名称
        const string QueueName = "Lesan.queue";

        /// <summary>
        /// 路由名称
        /// </summary>
        const string TopExchangeName = "topic.Lesan.exchange";

        //队列名称
        const string TopQueueName = "topic.Lesan.queue";


        /// <summary>
        ///  单点精确路由模式
        /// </summary>
        public static void DirectExchangeSendMsg()
        {
            using (IConnection conn = rabbitMqFactory.CreateConnection())
            {
                using (IModel channel = conn.CreateModel())
                {
                    //设置交换器的类型
                    channel.ExchangeDeclare(ExchangeName, ExchangeType.Direct, durable: true, autoDelete: false, arguments: null);
                    //声明一个队列，设置队列是否持久化，排他性，与自动删除
                    channel.QueueDeclare(QueueName, durable: true, autoDelete: false, exclusive: false, arguments: null);
                    //绑定消息队列，交换器，routingkey
                    channel.QueueBind(QueueName, ExchangeName, routingKey: QueueName);

                    var props = channel.CreateBasicProperties();
                    //队列持久化
                    props.Persistent = true;
                    string vadata = Console.ReadLine();
                    while (vadata != "exit")
                    {
                        var msgBody = Encoding.UTF8.GetBytes(vadata);
                        //发送信息
                        channel.BasicPublish(exchange: ExchangeName, routingKey: QueueName, basicProperties: props, body: msgBody);
                        Console.WriteLine(string.Format("***发送时间:{0}，发送完成，输入exit退出消息发送",
                            DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss")));
                        vadata = Console.ReadLine();
                    }
                }
            }
        }
        /// <summary>
        /// topic 模糊匹配模式，符号“#”匹配一个或多个词，符号“*”匹配不多不少一个词。因此“log.#”能够匹配到“log.info.oa”，但是“log.*” 只会匹配到“log.error”
        /// </summary>
        public static void TopicExchangeSendMsg()
        {
            using (IConnection conn = rabbitMqFactory.CreateConnection())
            {
                using (IModel channel = conn.CreateModel())
                {
                    channel.ExchangeDeclare(TopExchangeName, ExchangeType.Topic, durable: false, autoDelete: false, arguments: null);
                    channel.QueueDeclare(TopQueueName, durable: false, autoDelete: false, exclusive: false, arguments: null);
                    channel.QueueBind(TopQueueName, TopExchangeName, routingKey: TopQueueName);
                    //var props = channel.CreateBasicProperties();
                    //props.Persistent = true;
                    string vadata = Console.ReadLine();
                    while (vadata != "exit")
                    {
                        var msgBody = Encoding.UTF8.GetBytes(vadata);
                        channel.BasicPublish(exchange: TopExchangeName, routingKey: TopQueueName, basicProperties: null, body: msgBody);
                        Console.WriteLine(string.Format("***发送时间:{0}，发送完成，输入exit退出消息发送", DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss")));
                        vadata = Console.ReadLine();
                    }
                }
            }
        }
    }
}

```

### 消费者

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System;
using System.Text;

namespace RabbitMQConsumer
{
    internal class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Welcome to RabbitMQ Consumer!");
            //DirectAcceptExchange();
            //DirectAcceptExchangeEvent();
            DirectAcceptExchangeTask();
            //TopicAcceptExchange();
            Console.WriteLine("按任意值，退出程序");
            Console.ReadKey();
        }

        /// <summary>
        /// 连接配置
        /// </summary>
        private static readonly ConnectionFactory rabbitMqFactory = new ConnectionFactory()
        {
            UserName = "guest",
            Password = "guest",
            Port = 5672,
            //VirtualHost = "LesanVirtualHost"
        };
        /// <summary>
        /// 路由名称
        /// </summary>
        const string ExchangeName = "Lesan.exchange";

        //队列名称
        const string QueueName = "Lesan.queue";

        /// <summary>
        /// 路由名称
        /// </summary>
        const string TopExchangeName = "topic.Lesan.exchange";

        //队列名称
        const string TopQueueName = "topic.Lesan.queue";

        /// <summary>
        /// 基于时间轮询的，每隔一段时间获取一次
        /// </summary>
        public static void DirectAcceptExchange()
        {
            using (IConnection conn = rabbitMqFactory.CreateConnection())
            {
                using (IModel channel = conn.CreateModel())
                {
                    //设置交换器的类型
                    channel.ExchangeDeclare(ExchangeName, ExchangeType.Direct, durable: true, autoDelete: false, arguments: null);
                    //声明一个队列，设置队列是否持久化，排他性，与自动删除
                    channel.QueueDeclare(QueueName, durable: true, autoDelete: false, exclusive: false, arguments: null);
                    //绑定消息队列，交换器，routingkey
                    channel.QueueBind(QueueName, ExchangeName, routingKey: QueueName);
                    while (true)
                    {
                        BasicGetResult msgResponse = channel.BasicGet(QueueName, true);
                        if (msgResponse != null)
                        {
                            var msgBody = Encoding.UTF8.GetString(msgResponse.Body.ToArray());
                            Console.WriteLine(string.Format("***接收时间:{0}，消息内容：{1}", DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss"), msgBody));
                        }

                        System.Threading.Thread.Sleep(TimeSpan.FromSeconds(1));
                    }
                }
            }
        }
        /// <summary>
        /// 基于事件的，当消息到达时触发事件，获取数据
        /// </summary>
        public static void DirectAcceptExchangeEvent()
        {
            using (IConnection conn = rabbitMqFactory.CreateConnection())
            {
                using (IModel channel = conn.CreateModel())
                {
                    //channel.ExchangeDeclare(ExchangeName, "direct", durable: true, autoDelete: false, arguments: null);
                    channel.QueueDeclare(QueueName, durable: true, autoDelete: false, exclusive: false, arguments: null);
                    //channel.QueueBind(QueueName, ExchangeName, routingKey: QueueName);
                    var consumer = new EventingBasicConsumer(channel);
                    consumer.Received += (model, ea) =>
                    {
                        var msgBody = Encoding.UTF8.GetString(ea.Body.ToArray());
                        Console.WriteLine(string.Format("***接收时间:{0}，消息内容：{1}", DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss"), msgBody));
                    };
                    channel.BasicConsume(QueueName, true, consumer: consumer);
                    Console.WriteLine("按任意值，退出程序");
                    Console.ReadKey();
                }
            }
        }
        /// <summary>
        /// 基于事件的，当消息到达时触发事件，获取数据
        /// </summary>
        public static void DirectAcceptExchangeTask()
        {
            using (IConnection conn = rabbitMqFactory.CreateConnection())
            {
                using (IModel channel = conn.CreateModel())
                {
                    //channel.ExchangeDeclare(ExchangeName, "direct", durable: true, autoDelete: false, arguments: null);
                    channel.QueueDeclare(QueueName, durable: true, autoDelete: false, exclusive: false, arguments: null);
                    channel.BasicQos(prefetchSize: 0, prefetchCount: 1, global: false);//告诉broker同一时间只处理一个消息
                    //channel.QueueBind(QueueName, ExchangeName, routingKey: QueueName);

                    //定义这个队列的消费者
                    var consumer = new EventingBasicConsumer(channel);
                    consumer.Received += (model, ea) =>
                    {
                        var msgBody = Encoding.UTF8.GetString(ea.Body.ToArray());
                        Console.WriteLine(string.Format("***接收时间:{0}，消息内容：{1}", DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss"), msgBody));
                        int dots = msgBody.Split('.').Length - 1;
                        System.Threading.Thread.Sleep(dots * 1000);
                        //处理完成，告诉Broker可以服务端可以删除消息，分配新的消息过来
                        channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
                    };
                    //noAck设置false,告诉broker，发送消息之后，消息暂时不要删除，等消费者处理完成再说
                    //false为手动应答，true为自动应答
                    channel.BasicConsume(QueueName, false, consumer: consumer);

                    Console.WriteLine("按任意值，退出程序");
                    Console.ReadKey();
                }
            }
        }
        /// <summary>
        /// topic 模糊匹配模式，符号“#”匹配一个或多个词，符号“*”匹配不多不少一个词。因此“log.#”能够匹配到“log.info.oa”，但是“log.*” 只会匹配到“log.error”
        /// </summary>
        public static void TopicAcceptExchange()
        {
            using (IConnection conn = rabbitMqFactory.CreateConnection())
            {
                using (IModel channel = conn.CreateModel())
                {
                    channel.ExchangeDeclare(TopExchangeName, ExchangeType.Topic, durable: false, autoDelete: false, arguments: null);
                    channel.QueueDeclare(TopQueueName, durable: false, autoDelete: false, exclusive: false, arguments: null);
                    channel.BasicQos(prefetchSize: 0, prefetchCount: 1, global: false);
                    channel.QueueBind(TopQueueName, TopExchangeName, routingKey: TopQueueName);
                    var consumer = new EventingBasicConsumer(channel);
                    consumer.Received += (model, ea) =>
                    {
                        var msgBody = Encoding.UTF8.GetString(ea.Body.ToArray());
                        Console.WriteLine(string.Format("***接收时间:{0}，消息内容：{1}", DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss"), msgBody));
                        int dots = msgBody.Split('.').Length - 1;
                        System.Threading.Thread.Sleep(dots * 1000);
                        Console.WriteLine(" [x] Done");
                        channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
                    };
                    channel.BasicConsume(TopQueueName, false, consumer: consumer);

                    Console.WriteLine("按任意值，退出程序");
                    Console.ReadKey();
                }
            }
        }
    }
}

```



## 引用

以上笔记摘录自网络

> https://www.cnblogs.com/knowledgesea/p/5296008.html
> https://www.cnblogs.com/personblog/p/10681741.html