---
title: "SSE实现服务器推送"
date: "2022-11-26"
tags: ['消息推送', 'SSE']
categories: ['动手实践']
---

## 简介

服务端向客户端推送消息，其实除了可以用`WebSocket`这种耳熟能详的机制外，还有一种服务器发送事件(`Server-sent events`)，简称`SSE`。

`SSE`它是基于`HTTP`协议的，我们知道一般意义上的`HTTP`协议是无法做到服务端主动向客户端推送消息的，但`SSE`是个例外，它变换了一种思路。

SSE在服务器和客户端之间打开一个单向通道，服务端响应的不再是一次性的数据包而是`text/event-stream`类型的数据流信息，在有数据变更时从服务器流式传输到客户端。

整体的实现思路有点类似于在线视频播放，视频流会连续不断的推送到浏览器，你也可以理解成，客户端在完成一次用时很长（网络不畅）的下载。

`SSE`与`WebSocket`作用相似，都可以建立服务端与浏览器之间的通信，实现服务端向客户端推送消息，但还是有些许不同：

- SSE 是基于HTTP协议的，它们不需要特殊的协议或服务器实现即可工作；WebSocket需单独服务器来处理协议。
- SSE 单向通信，只能由服务端向客户端单向通信；webSocket全双工通信，即通信的双方可以同时发送和接受信息。
- SSE 实现简单开发成本低，无需引入其他组件；WebSocket传输数据需做二次解析，开发门槛高一些。
- SSE 默认支持断线重连；WebSocket则需要自己实现。
- SSE 只能传送文本消息，二进制数据需要经过编码后传送；WebSocket默认支持传送二进制数据。

**SSE 与 WebSocket 该如何选择？**

SSE好像一直不被大家所熟知，一部分原因是出现了WebSocket，这个提供了更丰富的协议来执行双向、全双工通信。对于游戏、即时通信以及需要双向近乎实时更新的场景，拥有双向通道更具吸引力。

但是，在某些情况下，不需要从客户端发送数据。而你只需要一些服务器操作的更新。比如：站内信、未读消息数、状态更新、股票行情、监控数量等场景，`SEE`不管是从实现的难易和成本上都更加有优势。此外，`SSE` 具有`WebSocket`在设计上缺乏的多种功能，例如：`自动重新连接`、`事件ID`和`发送任意事件`的能力。

## 实践

### 前端

前端只需进行一次HTTP请求，带上唯一ID，打开事件流，监听服务端推送的事件就可以了

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script src="http://libs.baidu.com/jquery/2.0.0/jquery.min.js" type="text/javascript"></script>
</head>
<body>
<div>
    <ul>
        <li class="active"><i class=""></i> 未读消息 <span id="count" class="">0</span></li>
    </ul>
    <div>
        <input style="height: 25px; width: 180px;" maxlength="60" value="" id="message" />
        <button class="button" id="mySendBtn" onclick="sendMessage()"> 点击发送</button>

        <hr />
        <div id="arrivedDiv" style="height:200px; width:300px; overflow:scroll; background:#EEEEEE;">
            <br />
        </div>
    </div>
</div>
<script>
    let source = null;
    let userId = "111";

    if (window.EventSource) {
        // 建立连接
        source = new EventSource('http://localhost:8080/sse/sub/'+userId);
        setMessageInnerHTML("连接用户=" + userId);
        // 连接一旦建立，就会触发open事件，另一种写法：source.onopen = function (event) {}
        source.addEventListener('open', function (e) {
            setMessageInnerHTML("建立连接。。。");
        }, false);
        // 客户端收到服务器发来的数据，另一种写法：source.onmessage = function (event) {}
        source.addEventListener('message', function (e) {
            setMessageInnerHTML(e.data);
        });
        // 如果发生通信错误（比如连接中断），就会触发error事件，另一种写法：source.onerror = function (event) {}
        source.addEventListener('error', function (e) {
            if (e.readyState === EventSource.CLOSED) {
                setMessageInnerHTML("连接关闭");
            } else {
                console.log(e);
            }
        }, false);
    } else {
        setMessageInnerHTML("你的浏览器不支持SSE");
    }

    // 监听窗口关闭事件，主动去关闭sse连接，如果服务端设置永不过期，浏览器关闭后手动清理服务端数据
    window.onbeforeunload = function () {
        closeSse();
    };

    // 关闭Sse连接
    function closeSse() {
        source.close();
        const httpRequest = new XMLHttpRequest();
        httpRequest.open('GET', 'http://localhost:8080/sse/close/' + userId, true);
        httpRequest.send();
        console.log("close");
    }

    // 将消息显示在网页上
    function setMessageInnerHTML(innerHTML) {
        $("#arrivedDiv").append("<br/>"+innerHTML);
        var count = $("#count").text();
        count = Number(count) + 1;
        $("#count").text(count);
    }

    // 发送信息
    function sendMessage() {
        var content = $("#message").val();
        $.ajax({
            url: 'http://localhost:8080/sse/push',
            type: 'GET',
            data: { "id": userId, "content": content },
            success: function (data) {
                console.log(data)
            },
            error: function (err) {
            },
            done: function () {
            }
        })
    }
</script>
</body>
</html>
```

### 后端

基本步骤，创建一个`SseEmitter`对象放入`sseEmitterMap`进行管理

下面是`SseEmitterUtils`类，里面主要是对`SseEmitter`的基本操作，以及当前所有的连接管理
```java
@Slf4j
@Component
public class SseEmitterUtils {
    /**
     * 当前连接数
     */
    private static AtomicInteger count = new AtomicInteger(0);
    /**
     * 使用map对象，便于根据userId来获取对应的SseEmitter，或者放redis里面
     */
    private static Map<String, SseEmitter> sseEmitterMap = new ConcurrentHashMap<>();

    /**
     * 创建用户连接并返回 SseEmitter
     */
    public static SseEmitter connect(String userId) {
        if (sseEmitterMap.containsKey(userId)) {
            return sseEmitterMap.get(userId);
        }
        try {
            /**
             * 设置超时时间，0表示不过期。默认30秒
             */
            SseEmitter sseEmitter = new SseEmitter(0L);
            /**
             * 注册回调
             */
            sseEmitter.onCompletion(completionCallBack(userId));
            sseEmitter.onError(errorCallBack(userId));
            sseEmitter.onTimeout(timeoutCallBack(userId));
            sseEmitterMap.put(userId, sseEmitter);

            /**
             * 数量+1
             */
            count.getAndIncrement();

            return sseEmitter;
        } catch (Exception e) {
            log.error("创建新的sse连接异常，当前用户：{}", userId);
        }
        return null;
    }

    /**
     * 给指定用户发送消息
     */
    public static void sendMessage(String userId, String message) {

        if (sseEmitterMap.containsKey(userId)) {
            try {
                sseEmitterMap.get(userId).send(message);
            } catch (IOException e) {
                log.error("用户[{}]推送异常:{}", userId, e.getMessage());
                removeUser(userId);
            }
        }
    }

    /**
     * 向同组人发布消息   （要求userId+groupId）
     */
    public static void groupSendMessage(String groupId, String message) {
        if (MapUtil.isNotEmpty(sseEmitterMap)) {
            sseEmitterMap.forEach((k, v) -> {
                try {
                    if (k.startsWith(groupId)) {
                        v.send(message, MediaType.APPLICATION_JSON);
                    }
                } catch (IOException e) {
                    log.error("用户[{}]推送异常:{}", k, e.getMessage());
                    removeUser(k);
                }
            });
        }
    }

    public static List<String> getIds() {
        return new ArrayList<>(sseEmitterMap.keySet());
    }

    public static void removeUser(String userId) {
        sseEmitterMap.remove(userId);
        // 数量-1
        count.getAndDecrement();
        log.info("移除用户：{}", userId);
    }

    private static Runnable completionCallBack(String userId) {
        return () -> {
            log.info("结束连接：{}", userId);
            removeUser(userId);
        };
    }

    private static Runnable timeoutCallBack(String userId) {
        return () -> {
            log.info("连接超时：{}", userId);
            removeUser(userId);
        };
    }

    private static Consumer<Throwable> errorCallBack(String userId) {
        return throwable -> {
            log.info("连接异常：{}", userId);
            removeUser(userId);
        };
    }
}
```

最后是`Controller`，主要是创建连接、发送消息、断开连接对外的接口
```java
@RestController
@CrossOrigin("*")
@RequestMapping("/sse")
public class SSEController {

    /**
     * 基础接口
     */
    @GetMapping("/index")
    public String sse(){
        return "sse";
    }

    /**
     * sse 订阅消息
     */
    @GetMapping(path = "sub/{id}", produces = {MediaType.TEXT_EVENT_STREAM_VALUE})
    public SseEmitter sub(@PathVariable String id) {
        return SseEmitterUtils.connect(id);
    }

    /**
     * sse 发布消息
     */
    @GetMapping("push")
    public void push(String id, String content) {
        SseEmitterUtils.sendMessage(id, content);
    }

    /**
     * sse 断开连接
     */
    @GetMapping("breakConnect")
    public void breakConnect(String id, HttpServletRequest request, HttpServletResponse response) {
        request.startAsync();
        SseEmitterUtils.removeUser(id);
    }
}
```