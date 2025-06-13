---
title: "在.NET中实现消息推送的三种方式"
date: "2022-03-19"
tags: ["消息推送", ".NET"]
categories: ["动手实践"]
---

消息推送是大部分系统都需要做到的功能，在.NET中我分别通过RabbitMQ、MQTT、SignalR实现消息推送功能，本篇文章将通过它们实现简单的推送功能，手把手带大家完成编程。本文环境为 `.NET Core 3.1`下

## SignalR实现

### 简介

**SignalR是一个开源的库，跨平台**；让Web应用与其他应用通讯变得很简单，**Web服务端可以实时的将内容推送给对应的客户端，客户端发送的信息也可以实时到其他客户端**。

**SignalR提供了一种远程过程调用(RPC)的方式，使得客户端可以调用服务器的方法，同样在服务器端的方法中也能调用客户端的方法**。

### 示例

Nuget包为：
`<PackageReference Include="Microsoft.AspNetCore.SignalR.Core" Version="1.1.0" />`

- 首先我们需要创建一个自己的SignalR Hub

```csharp
using Microsoft.AspNetCore.SignalR;
using System.Threading.Tasks;

namespace KBEAM.Hubs
{
    public class ChatHub : Hub    // 继承自SignalR Hub库
    {
        public async Task SendMessage(string user, string message)
        {
            await Clients.All.SendAsync("ReceiveMessage", user, message);
        }

        public async Task SendMessageToGroup(string group, string message)
        {
            await Clients.Group(group).SendAsync("ReceiveMessageFromGroup", group, message);
        }

        public async Task AddToGroup(string groupName)
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, groupName);

            await Clients.Group(groupName).SendAsync("ReceiveMessage", $"{Context.ConnectionId} has joined the group {groupName}.");
        }

        public async Task RemoveFromGroup(string groupName)
        {
            await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);

            //await Clients.Group(groupName).SendAsync("ReceiveMessage", $"{Context.ConnectionId} has left the group {groupName}.");
        }
    }
}

```

- 在Startup.cs文件中注册相关服务及管道

```csharp
// 1.在ConfigureServices函数中添加以下语句，注册相关服务
services.AddSignalR();
// 2.在Configure函数中添加以下语句，配置管道终结点
app.UseEndpoints(endpoints =>
{
    // ...
    endpoints.MapHub<ChatHub>("/chatHub");
    // ...
});
```

- 编写服务端业务，推送消息

```csharp
// 定义一个上下文
private readonly IHubContext<ChatHub> hubContext;
// 通过构造函数注入依赖
public MonitorService(IHubContext<ChatHub> hub)
{
    hubContext = hub;
}
//在需要的地方调用方法，进行消息推送
await hubContext.Clients.Group(group).SendAsync("ReceiveMessageFromGroup",group, "需要发送的消息");

```

- JS客户端程序编写

首先，需要通过npm来安装SignalR封装好的JS文件，`npm install @microsoft/signalr`

```js
// 1.首先进行SignalR客户端连接
const signalR = require("@microsoft/signalr")
let conn = new signalR.HubConnectionBuilder()
    .withUrl("http://localhost:8988/chatHub")
    .withAutomaticReconnect()
    .configureLogging(signalR.LogLevel.Error)
    .build()
export default conn

import signalR from "@/utils/signalR";
// 2.客户端调用服务端方法（RPC）
signalR.invoke("AddToGroup", "groupName").catch(function (err) {   // 加入用户组
    return console.error(err.toString());
});
signalR.invoke("RemoveFromGroup", "大屏").catch(function (err) {   // 移除用户组
    return console.error(err.toString());
});
// 3.客户端监听服务器消息
signalR.on("ReceiveMessageFromGroup", function (group, message) {
    console.log(group + " " + message);
    that.lineChartData = JSON.parse(message);
});
```

## RabbitMQ实现

### 简介

**RabbitMQ**是实现了高级消息队列协议（AMQP）的开源消息代理软件（亦称面向消息的中间件），由以高性能、健壮以及可伸缩性出名的 Erlang 写成。

### 示例

首先您需要在电脑上安装好 Erlang及RabbitMQ服务器，这一步大家就自行搜索解决吧，网上应该有很多的解决方法

- RabbitMQ准备

首先开启Stomp插件

```bash
rabbitmq-plugins enable rabbitmq_management  # 开启此插件后有管理界面http://localhost:15672/
rabbitmq-plugins enable rabbitmq_web_stomp
rabbitmq-plugins enable rabbitmq_web_stomp_examples
```

- 服务端发送消息

所需Nuget包：
`<PackageReference Include="RabbitMQ.Client" Version="6.2.4" />`

```csharp
// 建立RabbitMQ连接
private static readonly ConnectionFactory rabbitMqFactory = new ConnectionFactory()
{
    HostName = "localhost",
    UserName = "用户名",
    Password = "密码",
    Port = 5672,
    VirtualHost = "虚拟主机配置"
};

// 发送消息
using (IConnection conn = rabbitMqFactory.CreateConnection())
{
    using (IModel channel = conn.CreateModel())
    {
        channel.ExchangeDeclare("monitor", ExchangeType.Direct, durable: true, autoDelete: false, arguments: null);
        channel.QueueDeclare("message", durable: true, autoDelete: false, exclusive: false, arguments: null);
        channel.QueueBind("message", "monitor", routingKey: "message");

        var props = channel.CreateBasicProperties();
        props.Persistent = true;
        channel.BasicPublish(exchange: "monitor", routingKey: "message", basicProperties: props, body: Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(new { expectedData, actualData })));
    }
}
```

- 客户端接受消息

首先需要安装npm库文件，`npm install stompjs`

```js
import Stomp from "stompjs";

// 定义一个RabbitMQ客户端
data() {
    return {
        client: Stomp.client("ws://localhost:15674/ws"),
    };
},

// 初始化连接操作
created() {
     this.client.connect(
       "用户名",
       "密码",
       this.onConnected,
       this.onFailed,
       "虚拟主机名称"
     );
}
methods: {
  onConnected: function () {
    //订阅频道
    // const topic = localStorage.getItem("Lesan");
    console.log("连接成功");
    this.client.subscribe(
      "/exchange/monitor/message",
      this.responseCallback,
      this.onFailed
    );
  },
  onFailed: function (frame) {
    console.log("MQ Failed: " + frame);
    this.$message.error("连接失败");
  },
  // 回传消息
  responseCallback: function (frame) {
    console.log("MQ msg=>" + frame.body);
    this.lineChartData = JSON.parse(frame.body);
    //接收消息处理
  },
  // 断开相应的连接
  close: function () {
    this.client.disconnect(function () {
      console.log("已退出账号");
    });
  },
},
```

## MQTT实现

### 简介

MQTT是IBM开发的一个即时通讯协议，该协议支持所有的平台，几乎可以把所有联网的物品和外部连接起来。

1. 使用发布/订阅消息模式，提供一对多的消息发布，解除应用程序耦合。这一点很类似于XMPP，但是MQTT的信息冗余远小于XMPP。
2. 使用TCP/IP提供网络连接。主流的MQTT是基于TCP连接进行数据推送的，但是同样有基于UDP的版本，叫做MQTT-SN。这两种版本由于基于不同的连接方式，优缺点自然也就各有不同了。
3. 三种消息传输方式QoS：
   0代表“至多一次”，消息发布完全依赖底层 TCP/IP 协议。会发生消息丢失或重复。这一级别可用于如下情况，环境传感器数据，丢失一次读记录无所谓，因为不久后还会有第二次发送
   1代表“至少一次”，确保消息到达，但消息重复可能会发生
   2代表“只有一次”，确保消息到达一次。这一级别可用于如下情况，在计费系统中，消息重复或丢失会导致不正确的结果

### 示例

- 服务端开发

所需Nuget包：`<PackageReference Include="MQTTnet.AspNetCore" Version="3.1.2" />`

对Program.cs修改：

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseKestrel(o =>
            {
                o.ListenAnyIP(8988);
                o.ListenAnyIP(1884, t => t.UseMqtt());
            });
            webBuilder.UseStartup<Startup>();
            //webBuilder.UseUrls("http://0.0.0.0:8988");
        });
```

对Startup.cs修改：

```csharp
// 在ConfigureServices函数中添加
services.AddHostedMqttServer(mqttServer => mqttServer.WithoutDefaultEndpoint().WithConnectionValidator(c =>
{
    if (c.Username != "admin" || c.Password != "123456")
    {
        c.ReasonCode = MqttConnectReasonCode.BadUserNameOrPassword;
        return;
    }
    c.ReasonCode = MqttConnectReasonCode.Success;

}))
    .AddMqttConnectionHandler()
    .AddConnections();

// 在Configure函数中添加
app.UseEndpoints(endpoints =>
{
    endpoints.MapConnectionHandler<MqttConnectionHandler>(
    "/mqtt",
    httpConnectionDispatcherOptions => httpConnectionDispatcherOptions.WebSockets.SubProtocolSelector =
                                        protocolList =>
                                            protocolList.FirstOrDefault() ?? string.Empty);
});
app.UseMqttServer(S =>
{
    MqttHelper.Server = S;
    S.ClientConnectedHandler = new MqttServerClientConnectedHandlerDelegate(OnConnected);
    S.StartedHandler = new MqttServerStartedHandlerDelegate(OnStarted);
});

// 添加事件处理函数
private void OnStarted(EventArgs obj)
{
    RecurringJob.AddOrUpdate<MonitorService>("MonitorData", p => p.UpdateDataAsync("大屏"), "0/15 * * * * *");
}

private void OnConnected(MqttServerClientConnectedEventArgs args)
{
    Console.WriteLine(args.ClientId);
}

// 消息推送方法
MqttHelper.PublishAsync("monitor", JsonConvert.SerializeObject(new { expectedData, actualData }));
```

添加MqttHelper类：

```csharp
using MQTTnet.Server;
using System.Text;

namespace Common
{
    public class MqttHelper
    {
        public static IMqttServer Server { get; set; }

        public static void PublishAsync(string topic, byte[] payload)
        {
            if (Server != null)
            {
                Server.PublishAsync(new MQTTnet.MqttApplicationMessage()
                {
                    Topic = topic,
                    Payload = payload
                }, new System.Threading.CancellationToken(false));
            }
        }
        public static void PublishAsync(string topic, string payload)
        {
            if (Server != null)
            {
                Server.PublishAsync(new MQTTnet.MqttApplicationMessage()
                {
                    Topic = topic,
                    Payload = Encoding.UTF8.GetBytes(payload)
                }, new System.Threading.CancellationToken(false)); ;
            }
        }
    }
}

```

- 客服端开发

首先需要引入npm包：`npm install mqtt`

```js
import mqtt from "mqtt";

// 连接Mqtt服务器，并订阅消息
  mounted() {
    let that = this;
    let mqttClient = mqtt.connect("ws://localhost:8988/mqtt", {
      username: "用户名",
      password: "密码",
      clientId: "客户端ID",
    });
    mqttClient.on("connect", (e) => {
      console.log("connected", e);
      mqttClient.subscribe("monitor", { qos: 1 }, (err) => {
        if (!err) {
          console.log("subscribed");
        }
      });
    });
    mqttClient.on("message", (topic, message) => {
      console.log(topic, message.toString());
      that.lineChartData = JSON.parse(message.toString());
    });
  },
```

## 其他

Vue前端测试客户端全部代码，通过Vue-cli搭建的项目，修改App.vue即可

```html
<template>
  <div id="app">
    <img alt="Vue logo" src="./assets/logo.png" />
    <HelloWorld msg="Welcome to Your Vue.js App" />
  </div>
</template>

<script>
import HelloWorld from "./components/HelloWorld.vue";
import mqtt from "mqtt";
import Stomp from "stompjs";

export default {
  name: "App",
  components: {
    HelloWorld,
  },
  data() {
    return {
      rabbitClient: Stomp.client("ws://172.21.30.233:15674/ws"),
    };
  },
  created() {},
  mounted() {
    // this.useMqtt();
    // this.useRabbitMQ();
    this.useSignalR();
  },
  methods: {
    useSignalR() {
      const signalR = require("@microsoft/signalr");
      const conn = new signalR.HubConnectionBuilder()
        .withUrl("http://172.21.30.233:8988/chatHub")
        .withAutomaticReconnect()
        .configureLogging(signalR.LogLevel.Error)
        .build();
      conn
        .start()
        .then(() => {
          console.log("conneted");
          conn.invoke("AddToGroup", "大屏").catch(function (err) {
            return console.error(err.toString());
          });
          conn.on("ReceiveMessageFromGroup", function (group, message) {
            console.log(group + " " + message);
          });
        })
        .catch((err) => {
          return console.error(err.toString());
        });
    },
    useMqtt() {
      let mqttClient = mqtt.connect("ws://172.21.30.233:8988/mqtt", {
        username: "admin",
        password: "123456",
        // clientId: "Lesan",
      });
      mqttClient.on("connect", (e) => {
        console.log("connected", e);
        mqttClient.subscribe("monitor", { qos: 1 }, (err) => {
          if (!err) {
            console.log("subscribed");
          }
        });
      });
      mqttClient.on("message", (topic, message) => {
        console.log(topic, message.toString());
      });
    },
    useRabbitMQ() {
      // this.rabbitClient.heartbeat.outgoing = 0;
      // this.rabbitClient.heartbeat.incoming = 0;
      this.rabbitClient.connect(
        "KB",
        "KB",
        this.onConnected,
        this.onFailed,
        "kb_monitor"
      );
    },
    onConnected: function () {
      //订阅频道
      // const topic = localStorage.getItem("Lesan");
      console.log("连接成功");
      this.rabbitClient.subscribe(
        "/exchange/monitor/message",
        this.responseCallback,
        this.onFailed
      );
    },
    onFailed: function (frame) {
      console.log("MQ Failed: " + frame);
      this.$message.error("连接失败");
    },
    // 回传消息
    responseCallback: function (frame) {
      console.log("MQ msg=>" + frame.body);
      //接收消息处理
    },
    // 断开相应的连接
    close: function () {
      this.rabbitClient.disconnect(function () {
        console.log("已退出账号");
      });
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>

```