## Pipeline与ChannelHandler组件

### 一、`Pipeline`

1.1 首先看下Pipeline与Handler之间的关系
![image](https://user-gold-cdn.xitu.io/2018/8/17/1654526f0a67bb52?imageView2/0/w/1280/h/960/ignore-error/1)

1.2 解释

* `Pipeline`由多个`ChannelHandlerContext`组成，并且由一个`双向链表`维护上下文结构
* ChannelHandlerContext里面装了一个`Handler`，那么可以认为Handler是具体逻辑，向外暴露的是上下文对象，这一串Handler上下文对象组成channel
* 由上可知，pipeline和channel`一一对应`

### 二、`ChannelHandler` UML类图关系

2.1 分类
![image](https://user-gold-cdn.xitu.io/2018/8/17/1654526f0a8f2890?imageView2/0/w/1280/h/960/ignore-error/1)

2.2 解释

* 第一个子接口：`ChannelInboundHandler`，负责处理读数据逻辑，例如，我们在一端读到数据，首先要解析数据，然后对这些数据做一系列逻辑处理，最终把响应写到对端。在开始组装响应之前的所有逻辑都可以放到这个接口下处理，一个重要方法 `channelRead()`
* 第二个子接口：`ChannelOutboundHandler`，负责处理写数据逻辑，与上者相反，定义我们一端在组装完响应之后，把数据写到对端的逻辑。比如，我们封装好了一个 `response` 对象，接下来可能对这个对象进行一些其他的逻辑，然后编码成ByteBuf，写到对端，最核心方法 `write()`
* 而这两个接口都有一个适配器类，默认实现上述接口方法，`ChannelInboundHandlerAdapter` 和 `ChanneloutBoundHandlerAdapter`，默认情况下会把读写事件 `传播` 到下一个handler