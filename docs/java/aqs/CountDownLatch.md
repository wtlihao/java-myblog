## CountDownLatch

### 一、源码分析

### 1.1构造器
```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```
- 可以看出创建CountDownLatch必须声明参数

### 1.2设置资源
```java
Sync(int count) {
     setState(count);
}
```
- 可以看出是通过内部类的Sync同步器set，由CountDownLatch调用这个构造器，这个就是主线程需要等多少次

### 1.3 重写tryAcquireShared
```java
protected int tryAcquireShared(int acquires) {
   return (getState() == 0) ? 1 : -1;
}
```
- 可以看出是共享模式获取资源，只要没人占用就获取锁，但是读者很疑惑，这个方法只有返回值啊，其实AQS类会调用这个，可以在后面会讲

### 1.4 释放tryReleaseShared
```java
 protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
     }
```
- 这个方法由AQS调用，然后doReleaseShared方法唤醒其他线程

### 1.5 CountDownLatch的await方法
- 这个方法就是主线程调用await方法获取state是否为0，如果没有就阻塞了
```java
public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```
- tryAcquireSharedNanos，这个方法是AQS自己实现的
```java
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
}
```
- 可以看出调用一个获取state的方法，也就是CountDownLatch重写的方法，当state为0会返回1。

## 二、总结

### 2.1 CountDownLatch 的三种典型用法
①某一线程在开始运行前等待n个线程执行完毕。将 CountDownLatch 的计数器初始化为n ：new CountDownLatch(n) ，每当一个任务线程执行完毕，就将计数器减1 countdownlatch.countDown()，当计数器的值变为0时，在CountDownLatch上 await() 的线程就会被唤醒。一个典型应用场景就是启动一个服务时，主线程需要等待多个组件加载完毕，之后再继续执行。

②实现多个线程开始执行任务的最大并行性。注意是并行性，不是并发，强调的是多个线程在某一时刻同时开始执行。类似于赛跑，将多个线程放到起点，等待发令枪响，然后同时开跑。做法是初始化一个共享的 CountDownLatch 对象，将其计数器初始化为 1 ：new CountDownLatch(1) ，多个线程在开始执行任务前首先 coundownlatch.await()，当主线程调用 countDown() 时，计数器变为0，多个线程同时被唤醒。

③死锁检测：一个非常方便的使用场景是，你可以使用n个线程访问共享资源，在每次测试阶段的线程数目是不同的，并尝试产生死锁。不是很理解这句话？？？

### 2.2CountDownLatch 的不足
CountDownLatch是一次性的，计数器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当CountDownLatch使用完毕后，它不能再次被使用。


