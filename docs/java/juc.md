## 线程安全-锁-JUC

### 一、线程安全

1.1 `保证线程安全几种方式`

* 互斥同步-阻塞同步：synchronized、Lock、悲观并发策略
* 非阻塞同步：乐观并发策略、产生冲突，补偿机制，CPU的CAS，Unsafe类
* 无同步方案，方法无状态，不依赖堆上数据和公共系统资源
* ThreadLocal


### 二、锁

* `synchronized`
    * 原理
        * 
    * 优化

* Lock类
    * `AQS`
    * `CAS`
    * `ReentrantLock`