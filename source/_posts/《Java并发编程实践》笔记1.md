---
title: 《Java并发编程实践》笔记1
date: 2017-03-05 22:49:47
tags: [Java, 并发]
categories:
- 并发

---
### 并发的优点
* 资源利用率
* 公平性
* 便利性（不是很靠谱）

### 线程带来的风险
1. 安全性问题
2. 活跃性问题
3. 性能问题

### 可见性问题
Java内存模型要求变量的读取和写入都是源自的，但对于非volatile类型的64位数值变量（long和double），JVM允许其读操作和写操作分解为两个32位的操作。
#### 加锁与可见性
内置锁既保证了互斥行为，也保证了内存可见性。**Lock呢？存疑**
![](/images/QQ20170305-230327@2x.png)
### 对象不可变的条件
1. 对象创建后其状态就不能修改
2. 对象的所有域都是final类型
3. 对象是正确创建的（this引用没有逸出）

### final域
final类型的变量是不可变的，但是如果final引用的对象是可变的，那么这些被引用的对象是可以修改的。**不可变对象是线程安全的**。

### 安全发布的常用模式
要安全的发布一个对象，对象的引用以及对象的状态必须同时对其它线程可见。

一个正确构造的对象可以通过以下方式来安全的发布：

* 在静态初始化函数中初始化一个对象引用
* 将对象的引用保存到volatile类型的域或者AtomicReference对象中
* 将对象的引用保存到某个正确构造对象的final类型域中
* 将对象的引用保存到一个由锁保护的域中

第一条可见下面的例子，因为静态变量是在类的初始化阶段进行的，有内部同步机制：

    public static Holder holder = new Holder()

### 安全共享对象的方式
![](/images/QQ20170305-232736@2x.png)
### ConcurrentModificationException
容器在迭代过程中被修改时，就会抛出ConcurrentModificationException。如果不希望在迭代期间对容器加锁，一种办法是**克隆容器**。

容器的toString，equals和hashCode方法都会迭代容器。
### ConcurrentHashMap
* 分段锁，提高并发度
* 弱一致性，不会抛出ConcurrentModificationException
* size和isEmpty返回近似值

### SynchronousQueue
SynchronousQueue是一个没有数据缓冲的BlockingQueue，要将一个元素放入SynchronousQueue，必须有另一个线程正在等待接受这个元素。
### FutureTask
Future是接口，FutureTask是实现类。在1.8中FutureTask不再基于AQS实现，FutureTask的run方法会阻塞执行，get方法会等待执行完成才会返回。
### 栅栏
线程必须同时到达栅栏位置，才能继续执行。闭锁（CountDownLatch）用于等待事件，栅栏用于等待线程（await方法）
### 总结
* 尽量将域声明为final类型，除非需要它们是可变的
* 不可变对象一定是线程安全的
