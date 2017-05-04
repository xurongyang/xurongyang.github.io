---
title: Java锁是如何保证数据可见性的
date: 2017-03-06 14:53:56
tags:
---
在 [java.util.concurrent.locks.Lock](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/Lock.html) 接口的Javadoc中有这样一段话：
> All Lock implementations must enforce the ***same memory synchronization semantics as provided by the built-in monitor lock*** :

> * A successful lock operation acts like a successful monitorEnter action
> * A successful unlock operation acts like a successful monitorExit action
 
> Unsuccessful locking and unlocking operations, and reentrant locking/unlocking operations, do not require any memory synchronization effects.

简而言之，j.u.c.locks.Lock接口的实现类具有和synchronized关键字一样的内存同步语义。

synchronized关键字的用法众所周知，它可以对方法或代码块加锁，同一时刻最多只有一个线程能执行方法或者代码块中的代码。然而，synchronized不仅支持互斥访问，它还能确保一个线程在同步方法或同步块之前或之中的内存写入对其它线程是可见的。

同样，Lock接口实现类可以通过加锁和解锁操作实现线程间的互斥访问，同时能够保证内存中数据的可见性。

不同于synchronized关键字是由JVM支持的，Lock接口的实现类是直接用Java代码实现的。如何保证内存中数据的可见性？下面进行一下分析。

### 可见性
#### 什么是可见性

如果一个线程对于另外一个线程是可见的，那么这个线程的修改就能够被另一个线程立即感知到。

用一个简单的例子来说明：

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

#### 如何用锁来保证可见性
我们可以利用Java锁来保证内存中数据的可见性，来看下面这个例子：

```Java
public class Counter {
	private static int counter = 0;
	
	public static int incrCounter1() {
		return counter++;
	}
	
	public synchronized static int incrCounter2() {
		return counter++;
	}
	
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
很明显，incrCounter1不是线程安全的，incrCounter2和incrCounter3分别利用内置锁synchronized和ReentrantLock来保证了其它线程能看到最新的counter的值。

### Java锁保证可见性的具体实现

#### Happens-before规则
从JDK 5开始，JSR-133定义了新的内存模型，提出了Happens-before的偏序规则。当一个操作happens before另一个操作，这意味着第一个操作的操作结果及它之前的操作的操作结果对第二个操作是**可见的**，换句话说第二个操作能看到第一个操作及它之前对内存数据的修改结果。

Java内存模型一共定义了八条Happens-before规则，和Java锁相关的有以下两条：

1. 内置锁的释放锁操作发生在该锁随后的加锁操作之前
2. 一个volatile域的写操作发生在这个volatile域随后的读操作之前

从第一条Happens-before规则可知，synchronized内置锁本身就能够保证可见性。

#### j.u.c.locks.Lock提供的可见性

##### volatile关键字的可见性
对第一个代码样例做一下改造，用volatile关键字来修饰ok：

```Java
volatile boolean ok = false
int a = 0

// Thread a
a = 1
ok = true

// Thread b
if (ok) {
    System.out.println(a)
}
```
根据Happens-before规则第二条：同一个volatile域的写发生在读之前，ok的读操作发生在写操作之后，倘若ok判断为true，那么输出的a的值一定是1。

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

// sync.lock()
final void lock() {
    acquire(1);
}

public void unlock() {
    sync.release(1);
}
```
lock方法和unlock方法的具体实现都代理给了sync对象，根据ReentrantLock的构造参数，sync对象可以是FairSync（公平锁）或者是NonfairSync（非公平锁），以FairSync为例（NonfairSync原理类似）：

```Java
abstract static class Sync extends AbstractQueuedSynchronizer

static final class FairSync extends Sync

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```
从上面可以看出，lock方法和unlock方法的具体实现都是由acquire和release完成的，而FairSync类中并没有定义acquire方法和release方法，这两个方法都是在AbstractQueuedSynchronizer类中实现的。

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
acquire和release方法的实现比较复杂，因为我们只想说明可见性这一个问题，所以不用完全理解这两个方法的实现，只需要关注tryAcquire和tryRelease即可。tryAcquire和tryRelease在父类AbstractQueuedSynchronizer中没有定义，留给子类FairSync去实现。

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
将上面的代码做个简化，只留下关键步骤：

```Java
private volatile int state;

void lock() {
    read state
    if (获取到锁)
    	write state
}

void unlock() {
    write state
}
```
如果获取到锁，根据Happens-before规则：同一个volatile域的写发生在读之前，可以推断出unlock操作发生在lock操作之前。lock方法可以对应于内置锁的monitorEnter操作，unlock方法对应于内置锁的monitorExit操作，这和Javadoc中的描述是一致的。

由此可以得出结论Lock接口的实现类能实现和synchronized关键字一样的内存数据可见性。

### 结束语
ReentrantLock及其它Lock接口实现类实现内存数据可见性的方式相对比较隐秘，借助volatile关键字间接的实现了可见性。其实不光是Lock接口实现类，因为j.u.c包中大部分同步器的实现都是基于AbstractQueuedSynchronizer类来实现的，因此这些同步器也能够提供一定的可见性，有兴趣的同学可以尝试用类似的思路去分析。