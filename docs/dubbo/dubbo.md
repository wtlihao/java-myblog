1. Dubbo架构
    ![](http://dubbo.apache.org/img/architecture.png)     
2. 图解
    1. Registry采用ZK
    2. Consumer和Provider都是为Web工程
    3. Monitor为Dubbo提供的监控平台
3. 详解
    1. ZK首先启动
    2. Provider与Consumer第一次连接ZK时，C端拿到P端服务注册的元信息，并缓存，即使ZK挂了也不会影响C端调用P端的接口
    3. 之后，ZK与C和P端三端相互之间都建立了长连接（Netty实现）
4. 箭头解释
    1. P端将元信息（IP、Port、类全名）初始化到ZK（ZK即创建了一个Node保存下来）
    2. C端订阅P端注册的服务（通过P端暴露的接口去ZK找这个节点）
    3. ZK返回找到的Node的信息反馈给C端
    4. C端通过Spring接口代理模式生成一个代理类去做RPC调用接口
    5. 同时异步还要发送调用次数同步到Dubbo的监控平台
