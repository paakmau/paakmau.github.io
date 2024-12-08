+++
title = "使用 Vue.js 和 Spring Boot 配置 WebSocket"
date = 2020-07-07 18:21:46
slug = "202007071821"

[taxonomies]
tags = ["Spring Boot", "Vue.js", "WebSocket"]
+++

HTTP 协议中的请求只能由客户端发起，如果需要服务端主动向客户端发送信息，就会用到 WebSocket

<!-- more -->

## 后端部分

首先需要一个配置类用于指定 WebSocket 的端口号

```java
@Configuration
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

然后我们写一个类用来管理 WebSocket 连接，处理连接建立、收到消息、连接断开等的回调

```java
@ServerEndpoint(value = "/wstest")
@Component
public class WebSocketTest {
    private static int onlineCount = 0;
    private static CopyOnWriteArraySet<WebSocketTest> webSocketSet = new CopyOnWriteArraySet<>();

    private Session session;

    @OnOpen
    public void onOpen(Session session) {
        this.session = session;
        webSocketSet.add(this);
        onlineCount++;
        System.out.println("WS: New connection");

        sendText("Hello, I'm server.");
    }

    @OnClose
    public void onClose() {
        webSocketSet.remove(this);
        onlineCount--;
        System.out.println("WS: Connection closed");
    }

    @OnMessage
    public void onMessage(String msg, Session session) {
        System.out.println("WS: Message received, " + msg);
        sendText("Your msg is " + msg);
    }

    @OnError
    public void onError(Session session, Throwable error) {
        System.out.println("WS: err");
        error.printStackTrace();
    }

    public static int GetOnlineCount() {
        return onlineCount;
    }

    private void sendText(String text) {
        try {
            session.getBasicRemote().sendText(text);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

然后可以找一些在线测试网站测一下后端

## 前端部分

跟后端差不多，把回调函数绑定一下就能用了

```html
<template>
  <button @click="sendTestMsg">发送测试信息</button>
</template>

<script>
export default {
  data() {
    return {
      path: "ws://127.0.0.1:8080/wstest",
      socket: undefined
    };
  },
  mounted() {
    this.socket = new WebSocket(this.path);
    this.socket.onopen = this.onOpen;
    this.socket.onclose = this.onClose;
    this.socket.onerror = this.onError;
    this.socket.onmessage = this.onMessage;
  },
  methods: {
    onOpen() {
      console.log("连接成功");
    },
    onClose() {
      console.log("连接关闭");
    },
    onErr() {
      console.log("错误发生");
    },
    onMessage(msg) {
      console.log("收到消息: ", msg);
    },
    sendTestMsg() {
      let msg = "Hello";
      this.socket.send(msg);
      console.log("发送消息: ", msg);
    }
  }
};
</script>

<style>
</style>
```
