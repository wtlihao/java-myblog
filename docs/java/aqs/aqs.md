### 一、AQS学习思维导图
![image](https://camo.githubusercontent.com/75950bb0b6b7ba5672236abf0c7c096a822b1d39/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d31302d33312f36313131353836352e6a7067)

### 二、概念
1、AQS是用来构建锁和同步器的框架，使用AQS能简单且高效地构造出应用广泛的同步器，比如ReentrantLock,Semaphore，还有其他的ReenReadWriteLock、SynchronusQueue，FutureTask等等都是基于AQS。

### 三、原理
1、AQS原理：AQS核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制AQS是用CLH队列锁实现的，即将暂时获取不到锁的线程加入到队列中。CLH队列如下。

![image](https://camo.githubusercontent.com/55090fdc22963d41a3e56d5dadb17dfa7ad6379f/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f4a6176612532302545372541382538422545352542412538462545352539312539382545352542462538352545352541342538372545462542432539412545352542392542362545352538462539312545372539462541352545382541462538362545372542332542422545372542422539462545362538302542422545372542422539332f434c482e706e67)

### 四、AQS对资源的共享方式
- Exclusive(独占)：只能有一个线程能执行，如ReentrantLock。在独占的模式下又有公平和非公平
    - 公平锁：按照线程在同步队列的排队顺序去拿锁
    - 非公平锁：当线程要获取锁时，无视队列顺序，直接去拿，抢到就是谁的
- Share（共享）：多个线程可同时执行，如Semaphore/CountDownLatch。

### 五、AQS框架基于模板方法设计模式
- 同步器的设计是基于模板方式模式，如果需要自定义同步器一般也是这样
    1. 使用者继承AQS类并重写指定的方法（这些重写方法很简单，无非就是对于共享资源的获取和释放）
    2. 将AQS组合在自定义同步组件的实现中，并调用模板方法，而这些模板方法会调用使用者重写的方法。重写的模板方法有如下几个。
```java
isHeldExclusively()//该线程是否正在独占资源。只有用到condition才需要去实现它。
tryAcquire(int arg)//独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int arg)//独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryAcquireShared(int arg)//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryReleaseShared(int arg)//共享方式。尝试释放资源，成功则返回true，失败则返回false。
```

### 六、源码分析
1、Node节点
```java
static final class Node {
        /** waitStatus值，表示线程已被取消（等待超时或者被中断）*/
        static final int CANCELLED =  1;
        /** waitStatus值，表示后继线程需要被唤醒（unpaking）*/
        static final int SIGNAL    = -1;
        /**waitStatus值，表示结点线程等待在condition上，当被signal后，会从等待队列转移到同步到队列中 */
        static final int CONDITION = -2;
       /** waitStatus值，表示下一次共享式同步状态会被无条件地传播下去*/
        static final int PROPAGATE = -3;
        /** 等待状态，初始为0 */
        volatile int waitStatus;
        /**当前结点的前驱结点 */
        volatile Node prev;
        /** 当前结点的后继结点 */
        volatile Node next;
        /** 与当前结点关联的排队中的线程 */
        volatile Thread thread;
        /** ...... */
    }
```

2、获取资源
```java
public final void acquire(int arg){
    if(!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE),arg)){
        selfInterrupt();
    }
}
```
- 梳理一下逻辑（即分析这三个函数内部做了什么）
    - 1、首先，调用使用者重写的tryAcquire方法，若返回true，意味着获取资源成功，后面逻辑不会执行；反之，也就是获取失败，进入b步骤
    - 2、此时，获取资源失败，构造独占式同步节点，通过addWaiter将此节点添加到同步队列的尾部(此时可能会有多个线程节点试图加入同步队列尾部，需要以线程安全的方式添加)
    - 3、该节点已在队列中尝试同步状态，若获取不到，则阻塞节点线程，知道被前驱节点唤醒或者被中断
- addWaiter函数分析：
```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);//构造结点
        //指向尾结点tail
        Node pred = tail;
        //如果尾结点不为空，CAS快速尝试在尾部添加，若CAS设置成功，返回；否则，eng。
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
- 先cas快速设置，若失败，进入enq方法,将结点添加到同步队列尾部这个操作，同时可能会有多个线程尝试添加到尾部，是非线程安全的操作。
```java
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { //如果队列为空，创建结点，同时被head和tail引用
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {//cas设置尾结点，不成功就一直重试
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
- enq内部是个死循环，通过CAS设置尾结点，不成功就一直重试。很经典的CAS自旋的用法，我们在之前关于原子类的源码分析中也提到过。这是一种乐观的并发策略。

- acquireQueued
```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {//死循环
                final Node p = node.predecessor();//找到当前结点的前驱结点
                if (p == head && tryAcquire(arg)) {//如果前驱结点是头结点，才tryAcquire，其他结点是没有机会tryAcquire的。
                    setHead(node);//获取同步状态成功，将当前结点设置为头结点。
                    p.next = null; // 方便GC
                    failed = false;
                    return interrupted;
                }
                // 如果没有获取到同步状态，通过shouldParkAfterFailedAcquire判断是否应该阻塞，parkAndCheckInterrupt用来阻塞线程
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
- acquireQueued内部也是一个死循环，只有前驱结点是头结点的结点，也就是老二结点，才有机会去tryAcquire；若tryAcquire成功，表示获取同步状态成功，将此结点设置为头结点；若是非老二结点，或者tryAcquire失败，则进入shouldParkAfterFailedAcquire去判断判断当前线程是否应该阻塞，若可以，调用parkAndCheckInterrupt阻塞当前线程，直到被中断或者被前驱结点唤醒。若还不能休息，继续循环。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //获取前驱结点的wait值 
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)//若前驱结点的状态是SIGNAL，意味着当前结点可以被安全地park
            return true;
        if (ws > 0) {
        // ws>0，只有CANCEL状态ws才大于0。若前驱结点处于CANCEL状态，也就是此结点线程已经无效，从后往前遍历，找到一个非CANCEL状态的结点，将自己设置为它的后继结点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {  
            // 若前驱结点为其他状态，将其设置为SIGNAL状态
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
- 如果循环一次获取状态失败，就需要判断是否可以休息，判断规则为需要找到前驱节点是sinal的才可以，假如第一次的前驱==sinal，那么直接休息，或者需要从后往前遍历，找到一个非cancel节点，将它只为这个节点的后继

3、释放资源
```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
}
```
- 如果当前线程释放资源成功，那么就获取头结点，唤醒头结点的后继，调用unparkSuccessor方法唤醒
```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
    }
```
- 首先CAS将头结点的ws状态为0，然后获取后继，如果后继节点为空或者状态出于cancel，那么从后往前遍历找到一个<=0的节点，然后调用unpark唤醒这个线程
- 往后往前遍历有两点：一是为了照顾新入队的节点，因为入队操作没有CAS操作，对于释放资源的线程来讲并没有感知有新入队的节点，那么AQS采取的就是从后往前策略；二是回想一下cancelAcquire方法的处理过程，cancelAcquire只是设置了next的变化，没有设置prev的变化，在最后有这样一行代码：node.next = node，如果这时执行了unparkSuccessor方法，并且向后遍历的话，就成了死循环了，所以这时只有prev是稳定的。

### 3、共享模式
1、获取资源

```java
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)//返回值小于0，获取同步状态失败，排队去；获取同步状态成功，直接返回去干自己的事儿。
            doAcquireShared(arg);
}
```
- 如果尝试获取失败，关键看doAcquireShared(arg)函数
```java
private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);//构造一个共享结点，添加到同步队列尾部。若队列初始为空，先添加一个无意义的傀儡结点，再将新节点添加到队列尾部。
        boolean failed = true;//是否获取成功
        try {
            boolean interrupted = false;//线程parking过程中是否被中断过
            for (;;) {//死循环
                final Node p = node.predecessor();//找到前驱结点
                if (p == head) {//头结点持有同步状态，只有前驱是头结点，才有机会尝试获取同步状态
                    int r = tryAcquireShared(arg);//尝试获取同步装填
                    if (r >= 0) {//r>=0,获取成功
                        setHeadAndPropagate(node, r);//获取成功就将当前结点设置为头结点，若还有可用资源，传播下去，也就是继续唤醒后继结点
                        p.next = null; // 方便GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&//是否能安心进入parking状态
                    parkAndCheckInterrupt())//阻塞线程
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
- 分析：首先构造一个共享节点（判断队列是否为空根据tail==null），进入自旋，找到node前驱，也就是老二节点才能尝试获取同步状态，然后尝试获取同步状态，如果获取到就setHead然后传播，如果失败那么久找到一个安心休息的地方，与独占模式一样的。关键看下setHeadAndPropagate函数

```java
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        if (propagate > 0 || h == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
}
```
- 保存之前的head，然后将获取同步状态的node设置为node，然后判断传播值、h、h的ws值，如果三者满足一个则传播下去唤醒，反之需要三个条件都不满足就不做唤醒。

2、释放资源

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();//释放同步状态
        return true;
    }
    return false;
}
```
- 关键看doReleaseShared函数

```java
private void doReleaseShared() {
        for (;;) {//死循环，共享模式，持有同步状态的线程可能有多个，采用循环CAS保证线程安全
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;          
                    unparkSuccessor(h);//唤醒后继结点
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                
            }
            if (h == head)              
                break;
        }
    }
```



