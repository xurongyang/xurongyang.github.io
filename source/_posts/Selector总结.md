---
title: Selector总结
date: 2017-02-22 15:20:09
tags:
---
### Selector
SelectableChannel的多路开关选择器，其select方法对应于epoll，select，kqueue
### SelectionKey
一个绑定了某个selector的可被选择的channel对象的包装（a selectable channel's registration with a selector is represented by a SelectionKey object.A selection key is created each time a channel is registered with a selector)。

SelectionKey是线程安全的。

一个selector有三个selectionKey的Set集合。

* key集合代表当前已注册的管道，通过keys方法获得。
* selectedKey集合代表在上次selection操作时已经被检测到准备好进行下一步的动作（The selected-key set is the set of keys such that each key's channel was detected to be ready for at least one of the operations identified in the key's interest set during a prior selection operation)，通过selectedKeys方法获得，selectedKeys是keySet的子集。
* cancelledKey集合是已经取消但是没有删除的管道对象集合（have been cancelled but whose channels have not yet been deregistered）

所有的三个集合在一个新的selector中都是空的。

### Selection操作

1. 清空cancelledKey集合
2. 咨询操作系统每一个channel当前的状态，是否已经ready。

	* If the channel's key is not already in the selected-key set then it is added to that set and its ready-operation set is modified to identify exactly those operations for which the channel is now reported to be ready. Any readiness information previously recorded in the ready set is discarded.

    * Otherwise the channel's key is already in the selected-key set, so its ready-operation set is modified to identify any new operations for which the channel is reported to be ready. Any readiness information previously recorded in the ready set is preserved; in other words, the ready set returned by the underlying system is bitwise-disjoined into the key's current ready set.
3. 如果在步骤 (2) 的执行过程中要将任意键添加到已取消键集中，则处理过程如步骤 (1)

 select(), select(long), selectNow() 三个方法的区别在于是否等待一个或多个channel的状态变为ready，以及多久。

### 并发
Selectors是线程安全的，但是keySet不是。

在selection的操作中对selectionKey的修改没有影响，在下次selection才会提现出来。

### SelectableChannel
SelectableChannel通过registe方法和一个selector对象绑定，register方法返回selectionKey对象来代表channel的注册。

A channel cannot be deregistered directly; instead, the key representing its registration must be cancelled.

一个channel只能和一个selector注册上。SelectableChannel是线程安全的。

**在JAVA NIO中的三个核心的组件：Channels、Buffers和Selectors**
