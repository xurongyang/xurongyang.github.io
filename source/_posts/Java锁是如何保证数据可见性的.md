---
title: Java锁是如何保证数据可见性的
date: 2017-03-06 14:53:56
tags:
---
首先来看Java doc中Lock接口中的这样一段话：
> All Lock implementations must enforce the same memory synchronization semantics as provided by the built-in monitor lock:

> * A successful lock operation acts like a successful monitorEnter action
> * A successful unlock operation acts like a successful monitorExit action

由此可知，j.u.c.Lock接口的实现类和synchronized关键字一样，都能够保证内存中数据的可见性，这是怎么实现的呢？下面进行一下分析。

### 内存数据可见性
什么是**内存数据可见性**？

如果一个线程对于另外一个线程是可见的，那么这个线程的修改就能够被另一个线程立即感知到。用《Java并发编程实践》中的一个例子来说明：

```Java
public class VisibilityTest {

    private static int number;

    private static boolean ready = false;

    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready) {
                // 可能一直循环下去
                System.out.println(Thread.currentThread().getName() + " waiting ...");
            }
            // 输出的number的值不一定是10
            System.out.println(number);
        }
    }

    public static void main(String[] args) {
        new ReaderThread().start();
        number = 10;
        ready = true;
    }
}
```
ReaderThread中的循环可能会一直持续下去，因为main方法中设置的ready的值并不一定立即被ReaderThread感知到，并且输出的number的值也不一定是10。

在没有正确同步时，数据的可见性是没有办法保证的。

### Happens-Before规则
从JDK 5开始，JSR-133定义了新的内存模型，提出了Happens-before的偏序规则。简而言之，当一个行为happens before另一个行为，这意味着第一个动作的操作结果对第二个动作都是**可见的**。在本文中我们主要用到了以下两条Happens-before规则：

1. 内置锁的释放锁操作发生在该锁随后的加锁操作之前
2. 一个volatile域的写操作发生在这个volatile域随后的读操作之前

### synchronized提供的数据可见性
synchronized内置锁不仅仅只提供了互斥访问。从上面的Happens-before规则可以看出，一个线程在synchronized块之前或者中间的内存写操作的结果对于其它线程是可见的。下面举例说明。
### volatile关键字提供的数据可见性
根据Happens-before规则：同一个volatile域的写发生于读之前。这意味着volatile域的写操作之前的操作对读操作也是可见的。下面来看一个例子：

```Java
volatile ok = false;

// Thread a
a = 1
ok = true

// Thread b
if (ok) {
    System.out.println(a)
}
```
根据我们之前的描述，ok变量的读发生在写之后，倘若ok判断为true，那么输出的a的值一定是1。
### j.u.c.Lock提供的数据可见性
j.u.c.Lock接口定义了六个方法：

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
根据开头提到的Java Doc可知，lock方法对应内置锁的monitorEnter操作，unlock方法对应内置锁的monitorExit操作。由Happens-before规则，unlock操作发生在lock操作之前，unlock操作之前的内存写操作对lock操作是可见的。不同于synchronized关键字是由JVM支持的，Lock接口的实现类都是由Java代码完成的，那么**可见性**是如何保证的？

在j.u.c包中实现Lock接口的类主要有ReentrantLock和ReentrantReadWriteLock，下面主要以ReentrantLock为例来说明。

先来看ReentrantLock类的lock方法和unlock方法的实现（ReentrantReadWriteLock同理）：

```Java
public void lock() {
    sync.lock();
}

final void lock() {
    acquire(1);
}

public void unlock() {
    sync.release(1);
}
```
lock方法和unlock方法都代理给了sync对象，根据ReentrantLock的构造参数，sync对象可以是FairSync（公平锁）或者是NonfairSync（非公平锁）。

```Java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

下面以FairSync为例（NonfairSync原理一样）：

```Java
abstract static class Sync extends AbstractQueuedSynchronizer

static final class FairSync extends Sync
```
从上面可以看出，lock方法和unlock方法的具体实现都是由acquire和release完成的，而这两个方法都是AbstractQueuedSynchronizer实现的。

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
acquire和release方法的实现比较复杂，因为我们只想说明可见性这一个问题，所以不用完全理解这两个方法的实现，只需要关注tryAcquire和tryRelease即可。tryAcquire和tryRelease都是在FairSync类实现的。

```Java
// state变量是用volatile修饰的
private volatile int state;

protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 读State
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            // 写state
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
简而言之，tryAcquire会先读state，再写state，tryRelease会写state。

将上面的代码做个简化：

```Java
private volatile int state;

void lock() {
    read state
    write state
}

void unlock() {
    write state
}
```
根据Happens-before规则：volatile域的写操作之前的操作结果对读操作是可见的，可知unlock操作发生在lock操作之前，unlock操作之前的内存写操作对lock操作是可见的，由此保证了j.u.c.Lock接口的实现类能实现和synchronized关键字一样的内存数据可见性。