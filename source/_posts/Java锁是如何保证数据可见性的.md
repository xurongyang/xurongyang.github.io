---
title: Java锁是如何保证数据可见性的
date: 2017-03-06 14:53:56
tags:
---
### 引言
在 [java.util.concurrent.locks.Lock](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/Lock.html) 接口的Javadoc中有这样一段话：
> All Lock implementations must enforce the ***same memory synchronization semantics as provided by the built-in monitor lock*** :

> * A successful lock operation acts like a successful monitorEnter action
> * A successful unlock operation acts like a successful monitorExit action
 
> Unsuccessful locking and unlocking operations, and reentrant locking/unlocking operations, do not require any memory synchronization effects.

这段话的核心是j.u.c.locks.Lock接口的实现类具有和synchronized内置锁一样的内存同步语义。

不同于由JVM底层实现的内置锁，Lock接口的实现类是直接用Java代码实现的。如何保证了内存中数据的可见性？下面进行一下分析。

### 可见性
什么是可见性？如果一个线程对于另外一个线程是可见的，那么这个线程的修改就能够被另一个线程立即感知到。用一个简单的例子来说明：

```Java
boolean ok = false;
int a = 0;

// Thread a
a = 1
ok = true

// Thread b
// 可能一直循环下去
while (ok) {
    // 输出的number的值不一定是1
    System.out.println(a)
}
```
Thread b中的循环可能会一直持续下去，因为Thread a设置的ok的值并不一定立即被Thread b感知到，并且输出的a的值也不一定是1。

在多线程程序中，没有做**正确的同步**是无法保证内存中数据的可见性的。

### 用锁来保证可见性
我们可以利用Java锁来保证多线程程序中数据的可见性，来看下面这个例子：

```Java
public class Counter {
    private static int counter = 0;
    
    // 第一种方式：没有做同步操作
    public static int incrCounter1() {
        return counter++;
    }
    
    // 第二种方式：使用synchronized同步
    public synchronized static int incrCounter2() {
        return counter++;
    }
    
    // 第三种方式：使用ReentrantLock同步
    private static ReentrantLock lock = new ReentrantLock();
    
    public static int incrCounter3() {
        try {
            lock.lock();
            return counter++;
        } finally {
            lock.unlock();
        }
    }
}
```
显然，incrCounter1不是线程安全的，在一个线程写入一个最新的值后，无法保证另外一个线程能立即读到最新写入的值，incrCounter2和incrCounter3分别利用内置锁synchronized和ReentrantLock来保证了其它线程能看到最新的counter的值，达到我们想要的效果。

### Java锁保证可见性的具体实现

#### Happens-before规则

从JDK 5开始，JSR-133定义了新的内存模型，内存模型描述了多线程代码中的哪些行为是合法的，以及线程间如何通过内存进行交互。

新的内存模型语义在内存操作（读取字段，写入字段，加锁，解锁）和其他线程操作上创建了一些偏序规则，这些规则又叫作Happens-before规则。它的含义是当一个动作happens before另一个动作，这意味着第一个动作被保证在第二个动作之前被执行并且结果对其可见。我们利用Happens-before规则来解释Java锁到底如何保证了可见性。

Java内存模型一共定义了八条Happens-before规则，和Java锁相关的有以下两条：

1. 内置锁的释放锁操作发生在该锁随后的加锁操作之前
2. 一个volatile变量的写操作发生在这个volatile变量随后的读操作之前

#### synchronized提供的可见性
synchronized有两种用法，一种可以用来修饰方法，另外一种可以用来修饰代码块。我们以synchronized代码块为例：

```Java
synchronized(SomeObject) {
    // code
}
```
因为synchronized代码块是互斥访问的，只有一个线程释放了锁，另一个线程才能进入代码块中执行。

由上述Happens-before规则第一条：
> 内置锁的释放锁操作发生在该锁随后的加锁操作之前

假设当线程a释放锁后，线程b拿到了锁并且开始执行代码块中的代码时，线程b必然能够看到线程a看到的所有结果，所以synchronized能够保证线程间数据的可见性。

#### j.u.c.locks.Lock提供的可见性

##### volatile关键字的可见性
对第一个代码样例做一下改造，用volatile关键字来修饰ok，其余不变：

```Java
volatile boolean ok = false
int a = 0

// Thread a
a = 1
ok = true

// Thread b
if (ok) {
    // 确保a的值为1 
    System.out.println(a)
}
```
根据上述Happens-before规则第二条：
> 一个volatile变量的写操作发生在这个volatile变量随后的读操作之前
 
假设线程a将ok的值设置为true，那么如果线程b看到ok的值为true，一定可以保证输出的a的值是1。

##### ReentrantLock可见性保证的具体实现
j.u.c.locks.Lock接口定义了六个方法：

```Java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```
在j.u.c包中实现Lock接口的类主要有ReentrantLock和ReentrantReadWriteLock，下面以ReentrantLock为例来说明（ReentrantReadWriteLock原理相同）。

先来看ReentrantLock类的lock方法和unlock方法的实现：

```Java
public void lock() {
    sync.lock();
}

// sync.lock()实现
final void lock() {
    acquire(1);
}

public void unlock() {
    sync.release(1);
}
```
lock方法和unlock方法的具体实现都代理给了sync对象，来看一下sync对象的定义：

```Java
abstract static class Sync extends AbstractQueuedSynchronizer

static final class FairSync extends Sync

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

根据ReentrantLock的构造参数，sync对象可以是FairSync（公平锁）或者是NonfairSync（非公平锁），我们以FairSync为例（NonfairSync原理类似）来说明。

从上面代码中可以看出，lock方法和unlock方法的具体实现都是由acquire和release方法完成的，而FairSync类中并没有定义acquire方法和release方法，这两个方法都是在Sync的父类AbstractQueuedSynchronizer类中实现的。

```Java
public final void acquire(int arg) {
    // 只关注tryAcquire即可
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

public final boolean release(int arg) {
    // 只关注tryRelease即可
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

```
acquire方法的大致步骤：tryAcquire会尝试获取锁，如果获取失败会将当前线程加入等待队列，并挂起当前线程。当前线程会等待被唤醒，被唤醒后再次尝试获取锁。

release方法的大致步骤：tryRelease会尝试释放锁，如果释放成功可能会唤醒其它线程，释放失败会抛出异常。

我们可以看出，获取锁和释放锁的具体操作是在tryAcquire和tryRelease中实现的，而tryAcquire和tryRelease在父类AbstractQueuedSynchronizer中没有定义，留给子类FairSync去实现。

我们来看一下FairSync类的tryAcquire和tryRelease的具体实现：

```Java
// state变量定义在AbstractQueuedSynchronizer中，表示同步状态。
private volatile int state;

protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 读State
    int c = getState();
    if (c == 0) {
        // 获取到锁会写state
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        // 写state
        setState(nextc);
        return true;
    }
    return false;
}

protected final boolean tryRelease(int releases) {
    // 读state
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    // 写state
    setState(c);
    return free;
}
```
从上面的代码中可以看到有一个volatile state变量，这个变量用来表示同步状态，获取锁时会先读取state的值，获取成功后会把值从0修改为1。当释放锁时，也会先读取state的值然后进行修改。也就是说，无论是成功获取到锁还是成功释放掉锁，都会先读取state变量的值，再进行修改。

我们将上面的代码做个简化，只留下关键步骤：

```Java
private volatile int state;

void lock() {
    read state
    if (can get lock)
        write state
}

void unlock() {
    write state
}
```

假设线程a通过调用lock方法获取到锁，此时线程b也调用了lock方法，因为a尚未释放锁，b只能等待。a在获取锁的过程中会先读state，再写state。当a释放掉锁并唤醒b，b会尝试获取锁，也会先读state，再写state。

我们注意到上述提到的Happens-before规则的第二条：
> 一个volatile变量的写操作发生在这个volatile变量随后的读操作之前

可以推测出，当线程b执行获取锁操作，读取了state变量的值后，线程a在写入state变量之前的任何操作结果对线程b都是可见的。

由此，我们可以得出结论Lock接口的实现类能实现和synchronized内置锁一样的内存数据可见性。

### 结束语
ReentrantLock及其它Lock接口实现类实现内存数据可见性的方式相对比较隐秘，借助了volatile关键字间接地实现了可见性。其实不光是Lock接口实现类，因为j.u.c包中大部分同步器的实现都是基于AbstractQueuedSynchronizer类来实现的，因此这些同步器也能够提供一定的可见性，有兴趣的同学可以尝试用类似的思路去分析。