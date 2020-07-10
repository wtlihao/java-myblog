## Semaphore

### 一、源码分析

### 1.1 构造方法
```java
public Semaphore(int permits) {
        sync = new NonfairSync(permits);
}
```

```java
public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```
- 可以看出上面构造器调用了父类Sync构造器，而父类构造器也只能setState(permits)，看的出来就时设置了许可证的数量

### 1.2 获取许可证
- 先从获取一个许可看起，非公平模式下。
```java
public void acquire(int permits) throws InterruptedException { 
   if (permits < 0) throw new IllegalArgumentException();
   sync.acquireSharedInterruptibly(permits); 
}
```

- 从上面可以看到，调用了Sync的acquireSharedInterruptibly方法，该方法在父类AQS中
```java
public final void acquireSharedInterruptibly(int arg) throws InterruptedException { 
   //如果线程被中断了，抛出异常 
   if (Thread.interrupted()) throw new InterruptedException(); 
   //获取许可失败，将线程加入到等待队列中 
   if (tryAcquireShared(arg) < 0) 
   doAcquireSharedInterruptibly(arg); 
}
```

- AQS子类如果使用共享模式的话，需要实现tryAcquireShared方法，非公平下
```java
protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
}
```
- 看的出来调用父类的非公平方法，如下
```java
final int nonfairTryAcquireShared(int acquires) { 
   for (;;) { 
    //获取剩余许可数量 
    int available = getState(); 
    //计算给完这次许可数量后的个数 
    int remaining = available - acquires; 
    //如果许可不够或者可以将许可数量重置的话，返回 
    if (remaining < 0 || compareAndSetState(available, remaining)) 
    return remaining; 
  } 
}
```

- 再看下公平的获取，代码如下：
```java
protected int tryAcquireShared(int acquires) {
    for (;;) { 
     //如果前面有线程再等待，直接返回-1 
     if (hasQueuedPredecessors()) return -1; 
     //后面与非公平一样 
     int available = getState(); 
     int remaining = available - acquires; 
     if (remaining < 0 || compareAndSetState(available, remaining)) 
      return remaining; 
   } 
}
```
- 从上面可以看到，FairSync与NonFairSync的区别就在于会首先判断当前队列中有没有线程在等待，如果有，就老老实实进入到等待队列；而不像NonfairSync一样首先试一把，说不定就恰好获得了一个许可，这样就可以插队了。

### 1.3 释放许可
- Semaphore.Sync的方法
```java
public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
}
```
- releaseShared是AQS抽象类的抽象方法，下面代码是Semaphore.Sync重载方法
- 可以看出这个方法里面调用一个模板方法tryReleaseShared，由子类去实现
```java
public final boolean releaseShared(int arg) { 
  //如果改变许可数量成功 
  if (tryReleaseShared(arg)) { 
      doReleaseShared(); 
      return true; 
  } 
  return false; 
}
```

- AQS子类实现共享模式的类需要实现tryReleaseShared类来判断是否释放成功
```java
protected final boolean tryReleaseShared(int releases) { 
  for (;;) { 
   //获取当前许可数量 
   int current = getState(); 
   //计算回收后的数量 int next = current + releases; 
   if (next < current) 
   // overflow 
   throw new Error("Maximum permit count exceeded");
   //CAS改变许可数量成功，返回true 
   if (compareAndSetState(current, next))
   return true;
 } 
}

```

### 总结
Semaphore是信号量，用于管理一组资源。其内部是基于AQS的共享模式，AQS的状态表示许可证的数量，在许可证数量不够时，线程将会被挂起；而一旦有一个线程释放一个资源，那么就有可能重新唤醒等待队列中的线程继续执行。