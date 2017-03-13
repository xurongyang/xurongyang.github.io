---
title: Java锁是如何保证内存可见性的
date: 2017-03-06 14:53:56
tags:
---
首先来看Java doc中Lock接口中的这样一段话：
> All Lock implementations must enforce the same memory synchronization semantics as provided by the built-in monitor lock:

> * A successful lock operation acts like a successful monitorEnter action
> * A successful unlock operation acts like a successful monitorExit action

很明显，lock的实现类和synchronized一样，都能够保证内存中数据的可见性，这是怎么实现的呢？下面进行一下分析。

### 内存可见性
什么是**内存可见性**？如果一个线程对于另外一个线程是可见的，那么这个线程的修改就能够被另一个线程立即感知到。举个例子：

```Java
public class VisibilityTest {

    private static int number;

    private static boolean ready = false;

    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready) {
                System.out.println(Thread.currentThread().getName() + " waiting ...");
            }
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

### synchronized提供的数据可见性
正如上面Javadoc中提到了，synchronized作为内置锁，是能够提供内存数据可见性的。它是如何做到的？