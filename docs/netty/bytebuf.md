## ByteBuf结构

一、图解结构
![image](https://user-gold-cdn.xitu.io/2018/8/5/1650817a1455afbb?imageslim)

二、分析

1、`ByteBuf` 是一个字节容器，容器里面的的数据分为三个部分，第一个部分是已经丢弃的字节，这部分数据是无效的；第二部分是可读字节，这部分数据是 ByteBuf 的主体数据， 从 ByteBuf 里面读取的数据都来自这一部分;最后一部分的数据是可写字节，所有写到 ByteBuf 的数据都会写到这一段。最后一部分虚线表示的是该 ByteBuf 最多还能扩容多少容量（`问题:为啥废弃字节还要保留？`）

2、三段内容，从左至右分别为：`readerIndex`指针、`writerIndex`指针、以及`capacity`容量

3、从 `ByteBuf` 中每读取一个字节，readerIndex 自增1，ByteBuf 里面总共有 writerIndex-readerIndex 个字节可读, 由此可见当 readerIndex 与 writerIndex
相等的时候，ByteBuf 不可读

4、写数据是从 writerIndex 指向的部分开始写，每写一个字节，writerIndex 自增1，直到增到 `capacity`，这个时候，表示 ByteBuf 已经不可写了

5、`ByteBuf` 里面还有一个参数 `maxCapacity`，当向 ByteBuf 写数据的时候，如果容量不足，那么这个时候可以进行扩容，直到 capacity 扩容到 maxCapacity，超过 maxCapacity 就会报错

6、`Tips`：由于 `Netty` 使用了堆外内存，而堆外内存是不被 jvm 直接管理的，也就是说申请到的内存无法被垃圾回收器直接回收，所以需要我们手动回收。有点类似于c语言里面，申请到的内存必须手工释放，否则会造成内存泄漏。

7.`内存管理`：Netty 的 ByteBuf 是通过`引用计数`的方式管理的，如果一个 ByteBuf 没有地方被引用到，需要回收底层内存。默认情况下，当创建完一个 ByteBuf，它的引用为1，然后每次调用 retain() 方法， 它的引用就加一， release() 方法原理是将引用计数减一，减完之后如果发现引用计数为0，则直接回收 ByteBuf 底层的内存。