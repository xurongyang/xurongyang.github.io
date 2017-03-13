---
title: 《Java并发编程实践》笔记2
date: 2017-03-12 18:15:29
tags: [Java, 并发]
categories:
- 并发
---
## 创建一个线程的开销到底有多大
* Java栈
* native栈

## Executor框架
### Executor接口
```Java
public interface Executor {
	void execute(Runnable command);
}
```
### 几个常见的线程池
* newSingleThreadExecutor
* newFixedThreadPool
* newCachedThreadPool
* newScheduleredThreadPool

### ExecutorService接口
扩展了Executor接口，添加了一些生命周期管理的方法

```Java
public interface ExecutorService extends Executor {
	void shutdown();
	List<Runnable> shutdownNow();
	boolean isShutdown();
	boolean isTerminated();
	boolean awaitTermination(long timeout, Timeunit unit) throws InterruptedException;
``` 
ExecutorService的生命周期有三种状态：

1. 运行
2. 关闭
3. 已终止

shutdown方法将执行平缓的关闭过程：不再接收新任务，同时等待已提交的任务执行完成（包括那些还未开始运行的）。

shutdownNow将执行粗暴的的关闭过程：尝试取消所有运行中的任务，并且不再启动队列中尚未开始执行的任务。
### Callable和Future
Executor执行的任务有4个生命周期：1.创建，2.提交，3.开始，4.完成。

Future表示一个任务的生命周期，提供了相应的方法来判断是否已经完成或者取消，以及获取任务的结果和取消任务等。
####CompletionService
```Java
// 提交任务
completionService.submit(new Callable())

// 获取已完成的任务
for (int t = 0; t < tasks.size();i++) {
	Future f = completionService.take();
	Result r = f.get()
}
```
#### invokeAll
```Java
List<Future> futures = executorService.invokeAll(tasks, time, unit);
```
### 取消与关闭
#### interrupt
调用interrupt并不会立即停止线程正在进行中的工作，只是传递了请求中断的消息。

> **通常，中断是实现取消最合理的方式**

#### 响应中断
响应中断有两种方式：1.传递异常（interruptedException），2.恢复中断状态
![](/images/QQ20170312-184507@2x.png)
#### 超时取消
![](/images/QQ20170312-184916@2x.png)
#### 采用newTaskFor封装非标准的取消
####关闭ExecutorService
```Java
executorService.shutdown()
executorService.awaitTermination(timeout, unit)
```
采用毒丸对象，读到该对象时就终止。
#### UncaughtExceptionHandler
![](/images/QQ20170312-190301@2x.png)
#### 关闭钩子
![](/images/QQ20170312-190359@2x.png)
### 线程池使用
#### 最佳线程数
对于计算密集型任务，在N<sub>cpu</sub>的系统中，当线程池大小为N<sub>cpu</sub>+1时通常能实现最佳的利用率（计算密集型线程会由于偶尔的页缺失中断或其他原因暂停时，这个线程可以补上）。

![](/images/QQ20170312-190953@2x.png)
#### 饱和策略
* 中止，默认策略，会抛出RejectExecutionException
* 调用者运行，主线程不会accept，到达的请求会保存在TCP队列中而不是应用程序的队列中。如果持续过载，TCP层将最终发现它的请求队列被填满，同样会开始抛弃请求。
