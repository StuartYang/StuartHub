---
title: 一篇吃透WebSocket
mermaid: false
math: false
comments: true
hide: false
date: 2024-12-19 19:07:52
tags: [WebSocket, 网络协议]
categories: [网络协议]
---





# WebSocket 诞生背景

早期，很多网站为了实现推送技术，所用的技术都是轮询（也叫短轮询）。轮询是指由浏览器每隔一段时间向服务器发出 HTTP 请求，然后服务器返回最新的数据给客户端。

**常见的轮询方式分为短轮询与长轮询，它们的区别如下图所示：**

![mermaid-ai-diagram-2024-12-19-110504.png](https://img.stuarthub.us.kg/2024/12/410df9cf7e9abf9371f2cdfc6b186a7b.png)


​
- 短轮训：客户端定期发送请求，服务器立即响应（无论有无数据），请求频率固定，资源消耗较大。
- 长轮训：请求挂起等待数据，服务器有数据才响应，减少无效请求，实时性较好

这种传统的模式带来很明显的缺点，即浏览器需要不断的向服务器发出请求，然而 HTTP 请求与响应可能会包含较长的头部，其中真正有效的数据可能只是很小的一部分，所以这样会消耗很多带宽资源。

在这种情况下，HTML5 定义了 WebSocket 协议，能更好的节省服务器资源和带宽，并且能够更实时地进行通讯。

WebSocket 与 HTTP 和 HTTPS 使用相同的 TCP 端口，可以绕过大多数防火墙的限制。

**默认情况下：**

-  WebSocket 协议使用 80 端口；
-  若运行在 TLS 之上时，默认使用 443 端口。

## 1.1 WebSocket 简介


WebSocket 使得客户端和服务器之间的数据交换变得更加简单，<mark style="background: #FFB8EBA6;">允许服务端主动向客户端推送数据。</mark>在 WebSocket API 中，浏览器和服务器只需要完成一次握手，两者之间就可以创建持久性的连接，并进行双向数据传输。


## 1.2 WebSocket 优点

**普遍认为，WebSocket的优点有如下几点：**

- 较少的控制开销：在连接创建后，服务器和客户端之间交换数据时，用于协议控制的数据包头部相对较小；
- 更强的实时性：由于协议是<mark style="background: #FFB8EBA6;">全双工</mark>的，所以服务器可以随时主动给客户端下发数据。相对于 HTTP 请求需要等待客户端发起请求服务端才能响应，延迟明显更少；
- 保持连接状态：与 HTTP 不同的是，WebSocket 需要先创建连接，这就使得其成为一种有状态的协议，之后通信时可以省略部分状态信息；
- 更好的二进制支持：WebSocket 定义了二进制帧，相对 HTTP，可以更轻松地处理二进制内容；
- 可以支持扩展：WebSocket 定义了扩展，用户可以扩展协议、实现部分自定义的子协议。

由于 WebSocket 拥有上述的优点，所以它被广泛地应用在即时通讯领域。


# 手写 WebSocket 服务器（Java+JavaScript）

## 2.1 Java 写服务端


 在 pom.xml 文件中添加 WebSocket 所需的依赖
 
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
<dependency>
    <groupId>javax.websocket</groupId>
    <artifactId>javax.websocket-api</artifactId>
</dependency>
```


 **配置 WebSocket**
 
```java

import jakarta.websocket.*;
import jakarta.websocket.server.ServerEndpoint;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.concurrent.CopyOnWriteArraySet;

@Component
@ServerEndpoint("/ws")
public class MyWebSocketServer {

    // 用于存储连接的会话
    private static final CopyOnWriteArraySet<MyWebSocketServer> webSocketSet = new CopyOnWriteArraySet<>();
    private Session session;

    @OnOpen
    public void onOpen(Session session) {
        this.session = session;
        webSocketSet.add(this);
        System.out.println("新的连接加入，当前在线人数：" + webSocketSet.size());
    }

    @OnMessage
    public void onMessage(String message, Session session) {
        System.out.println("收到消息：" + message);
        // 广播消息到所有连接
        for (MyWebSocketServer webSocket : webSocketSet) {
            try {
                webSocket.sendMessage("广播消息：" + message);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    @OnClose
    public void onClose() {
        webSocketSet.remove(this);
        System.out.println("连接关闭，当前在线人数：" + webSocketSet.size());
    }

    @OnError
    public void onError(Session session, Throwable error) {
        System.err.println("发生错误：" + error.getMessage());
    }

    private void sendMessage(String message) throws IOException {
        this.session.getBasicRemote().sendText(message);
    }
}

```

> @ServerEndpoint 是一种轻量级的 WebSocket 实现方式，适合简单的实时通信应用场景。如果需要更多高级功能（如订阅、消息路由），可以结合 STOMP 协议和 Spring WebSocket 来实现复杂的实时通信系统。

**配置 ServerEndpointExporter**

```java

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

@Configuration
public class WebSocketConfig {

    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}

```

## 2.2 JavaScript测试WebSocket

启动 Spring Boot 项目，服务会监听 /ws 路径。

```JavaScript
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WebSocket Client</title>
</head>
<body>
    <h1>WebSocket 客户端</h1>
    <textarea id="messages" cols="50" rows="10" readonly></textarea><br>
    <input type="text" id="inputMessage" placeholder="输入消息">
    <button id="sendButton">发送</button>

    <script>
        const messages = document.getElementById("messages");
        const inputMessage = document.getElementById("inputMessage");
        const sendButton = document.getElementById("sendButton");

        // 创建 WebSocket 连接
        const ws = new WebSocket("ws://localhost:8080/ws");

        // 监听连接打开事件
        ws.onopen = () => {
            messages.value += "连接已建立\n";
        };

        // 监听消息事件
        ws.onmessage = (event) => {
            messages.value += "收到消息: " + event.data + "\n";
        };

        // 监听连接关闭事件,
        ws.onclose = () => {
            messages.value += "连接已关闭\n";
            console.log("连接关闭，尝试重连...");
	        setTimeout(() => createWebSocket(url), 1000); // 1秒后重连
        };

        // 监听错误事件
        ws.onerror = (error) => {
            messages.value += "发生错误\n";
            console.error("WebSocket 错误:", error);
        };

        // 点击按钮发送消息
        sendButton.addEventListener("click", () => {
            const message = inputMessage.value;
            if (message && ws.readyState === WebSocket.OPEN) {
                ws.send(message);
                messages.value += "发送消息: " + message + "\n";
                inputMessage.value = "";
            } else {
                messages.value += "连接未打开或消息为空\n";
            }
        });
    </script>
</body>
</html>
```

# WebSocket 与长轮询有什么区别

- 长轮询的本质基于 HTTP 协议，它仍然是一个一问一答（请求 — 响应）的模式。
- WebSocket 在<mark style="background: #FFB8EBA6;">握手成功后，就是全双工的 TCP 通道</mark>，数据可以主动从服务端发送到客户端。


![mermaid-ai-diagram-2024-12-19-110620.png](https://img.stuarthub.us.kg/2024/12/65ae6345a2b5115660c97d9b4a673df3.png)

​


# WebSocket和Socket的区别

| **特性** | Socket      | WebSocket             |
| ------ | ----------- | --------------------- |
| 所属层级   | 传输层         | 所属层级                  |
| 通讯模式   | 点对点，全双工     | 点对点，全双工               |
| 协议支持   | TCP/UDP     | 自定义协议（基于Http/1.1或者更高） |
| 开发复杂度  | 高           | 低                     |
| 使用场景   | 游戏，底层网络通讯协议 | 实时推送，聊天，在线协做工具        |


- 如果开发浏览器应用或实时性要求高的服务，**WebSocket** 是首选；如果需要完全控制通信流程和协议，选择 **Socket**。


# 参考资料

- [WebSocket 教程](https://www.runoob.com/w3cnote/websocket-tutorial.html)
- [WebSocket 协议](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket)
- [WebSocket 教程](https://www.runoob.com/w3cnote/websocket-tutorial.html)
- [WebSocket 教程](https://www.runoob.com/w3cnote/websocket-tutorial.html)
