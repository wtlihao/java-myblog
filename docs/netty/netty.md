## Netty介绍

### 一、Netty重要类

1. `NioEventLoopGroup`：事件组
2. `ServerBootstrap`：服务端启动类
3. `Bootstrap`：客户端启动类
4. `NioServerSocketChannel`
    1. 服务端Channel类
    2. 相当于服务端与客户端建立连接通道，这个类为服务端处理逻辑，十分重要，有一个生命周期，在不同周期做不同事情
5. `NioSocketChannel`
    1. 客户端Channel类
    2. 同服务端Channel一样
6. `ChannelInboundHandlerAdapter`：通道类添加的逻辑抽象类，需要客户端和服务端去实现生命周期函数
7. `ByteBuf`：数据传输载体