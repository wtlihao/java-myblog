## CyclicBarrier

### 一、源码分析

### 1.1 类属性
```java
 private static class Generation {
        boolean broken = false;
 }
// 栅栏入口锁
private final ReentrantLock lock = new ReentrantLock();
// 开放条件
private final Condition trip = lock.newCondition();
// 到达线程的阈值
private final int parties;
// 优先执行的command
private final Runnable barrierCommand;
// 当前栅栏情况
private Generation generation = new Generation();
// 等待的线程数量
private int count;
```

### 1.2 构造器
```java
public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
 }
```