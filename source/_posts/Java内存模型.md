---
title: Java内存模型
date: 2017-02-14 14:26:07
tags:
categories:
- Java
---
## 起源
最早接触到**Java内存模型**这个概念的源于[深入理解Java内存模型](http://www.infoq.com/cn/articles/java-memory-model-1)系列文章，第一次并没有看懂，有很多的概念理解不了，后来有了更多的经验积累，对Java内存模型的认识也逐渐深刻，故有此篇文章。
## 定义
先来解释一下Java内存模型，Java内存模型是

## 定义
Java内存模型规范了Java虚拟机与计算机内存是如何协同工作的。Java虚拟机是一个完整的计算机的一个模型，因此这个模型自然也包含一个内存模型——又称为Java内存模型。

Java内存模型规定了如何和何时可以看到由其他线程修改过后的共享变量的值，以及在必须时如何同步的访问共享变量。


## 参考
* [Java内存模型](http://ifeve.com/java-memory-model-6/)
* [源码剖析AQS在几个同步工具类中的使用](http://ifeve.com/abstractqueuedsynchronizer-use/)
