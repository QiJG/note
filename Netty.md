# 第一章 Netty入门

## 1.1 Netty概述

### 1.1.1 Netty简介

* Netty是一个异步事件驱动的网络应用框架，用于快速开发可维护的高性能服务器和客户端
* Netty是一个NIO客户机-服务器框架，它支持快速简单的开发网络应用程序，如服务器和客户机。他大大的简化了网络编程，如TCP和UDP套接服务器。
* ”快速和简单“并不意味着申城的应用程序将收到可维护性或性能问题的影响。Netty经过精心设计，并积累许多协议（如 FTP，SMTP，HTTP）的实施经验以及各种二进制和基于文本的一流协议、

### 1.1.2 Netty中的核心概念

#### （1）Channel

​	管道，其实是对 Socket 的封装，其包含了一组API，大大简化了直接与Socket进行操作的复杂性

#### （2）EventLoopGroup

​	EventLoopGroup 是一个EventLoop池，包含很多EventLoop。

​	Netty 为每个 Channel 分配了一个 EventLoop，用于处理用户的连接请求、对用户请求的处理等所有事件。EventLoop 本身只是一个线程驱动，在其生命周期只会绑定一个线程，让其线程处理一个 Channel 的所有 IO事件。

​	Channel 与 EventLoop 的关系是 n:1，而EventLoop 与线程的关系是1:1。

#### （3）ServerBootStrap

​	用于配置整个 Netty代码，将各个组件关联起来。服务端用的是 ServerBootStrap，客户端使用的是BootStrap。

#### （4）ChannelHandler 与 ChannelPipeline

​	ChannelHandler 是对 Channel 中数据的处理器，这些处理器可以是系统本身定义好的编解码器，也可以是用户自定义的。这些处理器会被统一添加到一个 ChannelPipeline 的对象中，然后按照添加的顺序 对 Channel中的数据进行依次处理。

#### （5）ChannelFuture

​	Netty 当中所有的I/O操作都是异步的，所以 Netty 中定义了一个 ChannelFuture 对象作为这个异步操作的 “代言人”，表示异步操作本身。如果想要获取该异步操作的返回值，可以通过该对象的 addListener() 方法为该异步操作添加监听器，为其注册回调，当结果出来后马上调用执行。

### 1.1.3 Netty 的执行流程

![image-20221024223148684](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221024223148684.png)

1. Server 启动 Netty 从 parentGroup 中选出一个 NioEventLoop对其指定的 Port 的连接进行监听
2. Client 启动 Netty 从 eventLoopGroup 中选出一个 NioEventLoop 连接 Server 并处理 Server发送过来的数据。
3. Client 连接指定 Server 的 Port，创建 Channel
4. Netty 从 childGroup 中选出一个 NioEventLoop 与该 Channel 进行绑定，用于处理该 Channel中所有的操作。
5. Client 通过 Channel 向 Server 发送数据包。
6. 服务端的Pipleline 中的处理器依次对 Channel中的数据包进行处理。
7. Server 如需向 Client 发送数据，则需将数据通过Pipeline 中的处理器形成 ByteBuf 数据包。
8. Server 将数据包 通过 Channel 发送该 Client
9. 客户端的 Pipeline 中的处理器依次对 Channel 中的数据包进行处理