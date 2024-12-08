+++
title = "支持 .NET Core 以及 Unity 的轻量级可靠 UDP 库 LiteNetLib 介绍"
date = 2019-12-15 20:20:31
slug = "201912152020"

[taxonomies]
tags = [".NET Core"]
+++

之前做 Unity 网络游戏开发的时候，想着后端也用 C# 就很方便，于是看上了开源跨平台的 .NET Core，当时网络模块就是用了 LiteNetLib 实现的。

<!-- more -->

这个东西依然支持 .NET Core 3.1 和 .NET Framework 4.x（虽然官网都没写）。功能不多，用起来方便，很适合小规模的实时联机游戏。

<https://github.com/RevenantX/LiteNetLib>

## 概括

跟随本文实现 Unity 前端与 .NET Core 后端互相发送信息

Unity 的 .NET 版本为.NET Framework 4.x，后端为 .NET Core 3.1，LiteNetLib 库使用 0.8.x 稳定版本

## Unity 前端

注意：Unity 里应当用源码而不是预编译的 DLL 文件，因为源码中使用了条件编译以避开 Unity 的 Bug。

直接从 GitHub 上 Clone 下来，把其中的 `LiteNetLib` 拷贝到 Unity 项目里就能使用了。

为了方便，我们简单写一个实现 `INetEventListener` 接口的 `MonoBehaviour` 就好，注释比较详细。

`LiteNetLibHello.cs` 如下：

```cs
using System.Net;
using System.Net.Sockets;
using LiteNetLib;
using LiteNetLib.Utils;
using UnityEngine;

/// <summary> 作为 MonoBehaviour，该类也是监听器 </summary>
public class LiteNetLibHello : MonoBehaviour, INetEventListener {

    /// <summary> server 的监听端口 </summary>
    private const int c_serverPort = 23333;

    /// <summary> 连接口令 </summary>
    private const string c_connectKey = "HelloWorld";

    /// <summary> 发送 hello 消息的间隔 </summary>
    private const float c_helloInterval = 1;

    /// <summary> hello 计时器 </summary>
    private float m_helloTimer;

    /// <summary> LiteNetLib 核心的网络管理器，用来主动连接、断开、触发事件回调等 </summary>
    private NetManager m_clientManager;

    /// <summary> 服务器 Peer </summary>
    private NetPeer m_server;

    /// <summary> Writer </summary>
    private NetDataWriter m_writer = new NetDataWriter ();

    void Start () {
        // 将自身作为事件监听器传入
        m_clientManager = new NetManager (this);

        // 接收断线信息
        m_clientManager.UnconnectedMessagesEnabled = true;

        m_clientManager.Start ();

        // 主动连接后端
        m_clientManager.Connect ("localhost", c_serverPort, c_connectKey);
    }

    void Update () {
        // 调用该函数会遍历并清空 Manager 的消息队列，并回调到监听器的相应位置
        m_clientManager.PollEvents ();

        // 定时
        m_helloTimer -= Time.deltaTime;
        if (m_helloTimer <= 0) {
            m_helloTimer = c_helloInterval;

            var peer = m_clientManager.FirstPeer;
            if (peer != null && peer.ConnectionState == ConnectionState.Connected) { // 若已连接到 Server 则发送 Hello World
                m_writer.Put ("Hello, I'm client.");
                // 不可靠 UDP 方式直接发送消息
                peer.Send (m_writer, DeliveryMethod.Unreliable);
                m_writer.Reset ();
            } else // 未连接则尝试连接
                m_clientManager.Connect ("localhost", c_serverPort, c_connectKey);
        }
    }

    void OnDestroy () {
        if (m_clientManager != null)
            m_clientManager.Stop ();
    }

    /// <summary> 有 peer 连接成功的事件回调 </summary>
    public void OnPeerConnected (NetPeer peer) {
        Debug.Log ("连接到了 " + peer.EndPoint);
    }

    /// <summary> 网络错误 </summary>
    public void OnNetworkError (IPEndPoint endPoint, SocketError socketErrorCode) {
        Debug.Log ("网络错误：" + socketErrorCode);
    }

    /// <summary> 接收信息 </summary>
    public void OnNetworkReceive (NetPeer peer, NetPacketReader reader, DeliveryMethod deliveryMethod) {
        var msg = reader.GetString ();
        Debug.Log ("接收到消息：" + msg);
    }

    public void OnNetworkReceiveUnconnected (IPEndPoint remoteEndPoint, NetPacketReader reader, UnconnectedMessageType messageType) { }

    public void OnNetworkLatencyUpdate (NetPeer peer, int latency) { }

    public void OnConnectionRequest (ConnectionRequest request) { }

    /// <summary> 断线 </summary>
    public void OnPeerDisconnected (NetPeer peer, DisconnectInfo disconnectInfo) {
        Debug.Log ("断线，原因：" + disconnectInfo.Reason);
    }
}
```

## DotNet Core 后端

为了方便，只写一个类并直接运行。

`HelloLiteNetLib.cs` 如下：

```cs
using System.Threading;
using System;
using System.Collections.Generic;
using System.Net;
using System.Net.Sockets;
using LiteNetLib;
using LiteNetLib.Utils;

public class HelloLiteNetLib : INetEventListener {

    private const int c_serverPort = 23333;

    private const string c_connectKey = "HelloWorld";

    private NetManager m_serverManager;

    private List<NetPeer> m_peerList = new List<NetPeer> ();

    NetDataWriter m_writer = new NetDataWriter ();

    public HelloLiteNetLib () {
        m_serverManager = new NetManager (this);
        m_serverManager.Start (c_serverPort);
    }

    public void Tick () {
        m_serverManager.PollEvents ();
        m_writer.Put ("Hello, I'm server.");
        m_serverManager.SendToAll (m_writer, DeliveryMethod.Unreliable);
        m_writer.Reset ();
    }

    public void OnConnectionRequest (ConnectionRequest request) {
        Console.WriteLine ("接收到连接请求");
        request.AcceptIfKey (c_connectKey);
    }

    public void OnNetworkError (IPEndPoint endPoint, SocketError socketError) {
        Console.WriteLine ("网络错误, 客户端：" + endPoint + "，错误信息：" + socketError);
    }

    public void OnNetworkLatencyUpdate (NetPeer peer, int latency) { }

    public void OnNetworkReceive (NetPeer peer, NetPacketReader reader, DeliveryMethod deliveryMethod) {
        Console.WriteLine ("接收到了消息：" + reader.GetString ());
    }

    public void OnNetworkReceiveUnconnected (IPEndPoint endPoint, NetPacketReader reader, UnconnectedMessageType messageType) { }

    public void OnPeerConnected (NetPeer peer) {
        Console.WriteLine ("连接到了 " + peer.EndPoint);
        m_peerList.Add (peer);
    }

    public void OnPeerDisconnected (NetPeer peer, DisconnectInfo info) {
        Console.WriteLine ("断线，EndPoint：" + peer.EndPoint + "，原因：" + info.Reason);
    }

    static void Main (string[] args) {
        var helloLiteNetLib = new HelloLiteNetLib ();
        while (true) {
            Thread.Sleep (2000);
            helloLiteNetLib.Tick ();
        }
    }
}
```

## LiteNetLib 内部实现介绍

会用就行，可以直接跳过

### 接收信息

首先，这个网络库使用事件监听的思路来把网络消息传递给上游模块。
他会把网络消息转化为连接、断开、数据传输、网络错误等事件并适时调用上游的事件监听器的相应接口。

然后，它开一个线程监听端口，接收消息并把转换为事件储存在队列里。
上游模块需要定时 PollEvent，让他一次性地访问整个队列（当然队列会被清空），根据每个事件调用监听器相应的接口。

因此，虽然消息的获取是异步的，但事件监听是同步的，使用时不必担心线程问题。

### 发送信息

简单得多。数据发送，都被放入发送队列，然后内部有一个线程定时把整个队列发送。
有兴趣去看源码，很好理解。

总之就是线程安全、使用简单（瞎几把用就行）
