---
title: AQS
date: 2017-01-17 14:25:12
tags:
---
### 锁分类
* 自旋锁：自旋锁是一种非阻塞锁，也就是说，如果某线程需要获取自旋锁，但该锁已经被其他线程占用时，该线程不会被挂起，而是在不断的消耗CPU的时间，不停的试图获取自旋锁。
* 互斥锁：互斥锁是阻塞锁，当某线程无法获取互斥量时，该线程会被直接挂起，该线程不再消耗CPU时间，当其他线程释放互斥锁后，操作系统会激活那个被挂起的线程，让其投入运行。

### 同步器定义
* 内部同步状态的管理(例如：表示一个锁的状态是获取还是释放)
* 同步状态的更新和检查操作，且至少有一个方法会导致调用线程在同步状态被获取时阻塞，以及在其他线程改变这个同步状态时解除线程的阻塞

### 同步器支持的操作
* 阻塞和非阻塞（例如tryLock）同步。
* 可选的超时设置，让调用者可以放弃等待
* 通过中断实现的任务取消，通常是分为两个版本，一个acquire可取消，而另一个不可以。

### 同步器的实现
#### 伪代码
acquire操作阻塞调用的线程，直到或除非同步状态允许其继续执行
##### acquire操作实现

    while (没有获得执行许可) {
    	if (当前线程没有加入等待队列) {
    		将当前线程入队
    	}
    	可能阻塞当前队列
    }
    如果当前线程在等待队列中，将其移除队列
    
##### release操作实现
release操作是通过某种方式改变同步状态，使得一或多个被acquire阻塞的线程继续执行

	更新同步状态
	if (某一被阻塞线程被允许执行acquire操作) 
		激活一个或多个等待队列中的线程
#### 实现的关键点
* 同步状态的原子性管理；
* 线程的阻塞与解除阻塞；
* 队列的管理；

##### 同步状态
AQS类使用单个int（32位）来保存同步状态，并暴露出getState、setState以及compareAndSet操作来读取和更新这个状态。这些方法都依赖于j.u.c.atomic包的支持，这个包提供了兼容JSR133中volatile在读和写上的语义，并且通过使用本地的compare-and-swap或load-linked/store-conditional指令来实现compareAndSetState，使得仅当同步状态拥有一个期望值的时候，才会被原子地设置成新值。

基于AQS的具体实现类必须根据暴露出的状态相关的方法定义tryAcquire和tryRelease方法，以控制acquire和release操作。当同步状态满足时，tryAcquire方法必须返回true，而当新的同步状态允许后续acquire时，tryRelease方法也必须返回true。这些方法都接受一个int类型的参数用于传递想要的状态。例如：可重入锁中，当某个线程从条件等待中返回，然后重新获取锁时，为了重新建立循环计数的场景。很多同步器并不需要这样一个参数，因此忽略它即可。

#### 阻塞
用LockSuport类解决阻塞问题，方法LockSupport.park阻塞当前线程除非/直到有个LockSupport.unpark方法被调用（unpark方法被提前调用也是可以的）。unpark的调用是没有被计数的，因此在一个park调用前多次调用unpark方法只会解除一个park操作。park方法同样支持可选的相对或绝对的超时设置，以及与JVM的Thread.interrupt结合 —— 可通过中断来unpark一个线程。

#### 队列
整个框架的关键就是如何管理被阻塞的线程的队列，该队列是严格的FIFO队列，因此，框架不支持基于优先级的同步。

CLH队列是一个链表队列，通过两个字段head和tail来存取，这两个字段是可原子更新的，两者在初始化时都指向了一个空节点。
![](/images/CLHNode.png)

一个新的节点，node，通过一个原子操作入队：

    do {
		pred = tail;
	} while(!tail.compareAndSet(pred, node));
	
每一个节点的“释放”状态都保存在其前驱节点中。因此，自旋锁的“自旋”操作就如下：
	
	while (pred.status != RELEASED); // spin
自旋后的出队操作只需将head字段指向刚刚得到锁的节点：

	head = node;

CLH锁的优点在于其入队和出队操作是快速、无锁的，以及无障碍的（即使在竞争下，某个线程总会赢得一次插入机会而能继续执行）；且探测是否有线程正在等待也很快（只要测试一下head是否与tail相等）；同时，“释放”状态是分散的（译者注：几乎每个节点都保存了这个状态，当前节点保存了其后驱节点的“释放”状态，因此它们是分散的，不是集中于一块的。），避免了一些不必要的内存竞争。    
#### 最终实现

	if(!tryAcquire(arg)) {
		node = new Node();
	    pred = node's effective predecessor;
	    while (pred is not head node || !tryAcquire(arg)) {
	        if (pred's signal bit is set)
	            park()
	        else
	            compareAndSet pred's signal bit to true;
	        pred = node's effective predecessor;
	    }
	    head = node;
	}
