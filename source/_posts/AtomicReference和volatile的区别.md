---
title: AtomicReference和volatile的区别
date: 2017-02-17 14:11:17
tags:
---
AtomicReference中引用的变量是volatile，get和set方法跟volatile一样，只不过提供了compareAndSet
