---
title: Operating Systems读书笔记(Virtualization)
date: 2016-08-13 13:25:34
tags: Operating Systems
categories:
- 操作系统
---
[《Operating Systems:Three Easy Pieces》](https://book.douban.com/subject/19973015/)

### 操作系统的核心问题
> HOW DOES THE OPERATING SYSTEM VIRTUALIZE RESOURCES?

* 虚拟化CPU支持多任务同时执行
* 虚拟化内存提供无限大的内存空间，进程内存互相不影响

>HOW TO BUILD CORRECT CONCURRENT PROGRAMS?

>HOW TO STORE DATA PERSISITENTLY?

### CPU虚拟化
#### 进程
> HOW TO PROVIDE THE ILLUSION OF MANY CPUS?

进程的上下文：

1. 内存，指令与数据
2. 寄存器
3. PC,Stack Pointer，Frame Pointer
4. I/O 

![](/images/QQ20170130-221344@2x.png)

堆，栈

在unix系统中，每个进程都会默认有3个文件描述符，标准输入，输出，错误

main方法？

进程的状态：

1. 运行
2. 准备
3. 阻塞，A common example: when a process initiates an I/O request to a disk, it becomes blocked and thus some other process can use the processor.

##### 进程状态转换图
![](/images/QQ20170130-222049@2x.png)
![](/images/QQ20170130-222252@2x.png)
##### 进程数据结构
![](/images/QQ20170130-222716@2x.png)
#### 虚拟化CPU的挑战
> HOW TO EFFICIENTLY VIRTUALIZE THE CPU WITH CONTROL? 

##### 限制性行为
* 用户模式
* 内核模式

操作系统在启动的时候注册系统调用表，操作系统会通知硬件当发生系统调用时应该去执行什么代码。

##### 进程切换
> HOW TO REGAIN CONTROL OF THE CPU
> 
> HOW TO GAIN CONTROL WITHOUT COOPERATION

利用timer interrupt，操作系统可以在任何情况下获得控制权。这个时间设备可被编程实现为每n毫秒进行一次中断，当中断发生时，当前正在运行的进程会停止，而一个之前被设置好的OS handler会开始执行，这样操作系统就获得了CPU的控制权，然后就可以停止当前进程，并开始另一个进程。
##### 流程
![](/images/QQ20170204-115301@2x.png)
###### 保存上下文
![](/images/QQ20170131-163101@2x.png)

#### 进程调度策略

> HOW TO DEVELOP SCHEDULING POLICY

##### 评价标准
The turnaround time of a job is defined
as the time at which the job completes minus the time at which the job
arrived in the system

T<sub>turnaround=T<sub>completion</sub>-T<sub>arrival</sub>

Response time is defined as the time from when the job arrives in a
system to the first time it is scheduled

T<sub>response</sub>=T<sub>firstturn</sub>-T<sub>arrival</sub>
##### 具体策略
* FIFO（延时太大，公平）
* Shortest Job First（响应时间）
* Shortest Time-to-Completion First（响应时间）
* Round Robin（实时）

##### The Multi-Level Feedback Queue
> HOW TO SCHEDULE WITHOUT PERFECT KNOWLEDGE?
 
###### 基本规则
* **Rule 1:** If Priority(A) > Priority(B), A runs (B doesn’t).
* **Rule 2:** If Priority(A) = Priority(B), A & B run in RR.
* **Rule 3:** When a job enters the system, it is placed at the highest priority (the topmost queue).
* **Rule 4:** Once a job uses up its time allotment at a given level (regardless of how many times it has given up the CPU), its priority is reduced (i.e., it moves down one queue).
* **Rule 5:** After some time period S, move all the jobs in the system to the topmost queue.

![](/images/QQ20170204-140202@2x.png)

##### Lottery Scheduling
>  HOW TO SHARE THE CPU PROPORTIONALLY

完全准确的按照权重调度，比如A:B:C=100:50:250，用一个大整数比如10000除以它们，分别是100，200和40，每run一次加100，200和40

	current = remove_min(queue); // pick client with minimum pass
	schedule(current); // use resource for quantum
	current->pass += current->stride; // compute next 	pass using stride
	insert(queue, current); // put back into the queue

![](/images/QQ20170204-165637@2x.png)

##### Multiprocessor Scheduling
> HOW TO SCHEDULE JOBS ON MULTIPLE CPUS
 
### 内存虚拟化
> HOW TO VIRTUALIZE MEMORY
> HOW TO EFFICIENTLY AND FLEXIBLY VIRTUALIZE MEMORY

为每个进程创造出一个从地址0开始，几乎无限大的内存空间。所以，需要翻译地址，虚拟地址->物理地址，还有之后的虚拟内存。
#### 地址空间
![](/images/QQ20170205-105419@2x.png)

堆和栈的扩张方向正好相反，堆从小到大，栈从大到小
JVM地址空间？
![](/images/WX20170205-191412@2x.png)
#### 地址翻译
* base，基地址
* bound(limit)，边界，防止地址越界

physical address = virtual address + base
#### 空间分配
操作系统需要维护空闲内存列表，当进程创建的时候，找到空闲的内存块分配给它。当进程销毁的时候，从进程回收内存，放入空闲内存列表。
#### 操作系统与硬件的交互
![](/images/QQ20170205-210855@2x.png)
#### 内存分段
整个进程没必要物理上在一起，代码段，堆和栈可以物理隔离，这样可以保持灵活性，同时降低对大块内存的要求。
![](/images/QQ20170205-212800@2x.png)
如何知道是哪个段？通过在虚拟地址中加两位来解决。

栈需要指明扩展的方向，再加一位。

![](/images/QQ20170205-215102@2x.png)
保护位再加一位

![](/images/QQ20170205-215500@2x.png)
#### 内存碎片问题
> HOW TO MANAGE FREE SPACE
 
FreeList策略

* Best Fit
* Worst Fit
* First Fit
* Next Fit

#### 分页
> HOW TO VIRTUALIZE MEMORY WITH PAGES

分段可以给每一个进程分配不同的线性地址空间，而分页可以把同一线性地址空间映射到不同的物理空间

固定大小使得空闲空间管理变得简单，分页和分段的区别在于：分页会造成页内空间浪费，分段会造成段外空间碎片化，相对而言，分页的问题更好解决。

#### 分页地址翻译
![](/images/QQ20170206-191917@2x.png)

> HOW TO SPEED UP ADDRESS TRANSLATION
> 
> TLB

![](/images/QQ20170206-192852@2x.png)
>HOW TO MANAGE TLB CONTENTS ON A CONTEXT SWITCH

![](/images/QQ20170206-194007@2x.png)
> HOW TO DESIGN TLB REPLACEMENT POLICY
>
> LRU

> HOW TO MAKE PAGE TABLES SMALLER?

![](/images/QQ20170206-194834@2x.png) 
#### 交换分区
Present Bit代表是否交换

![](/images/QQ20170207-132350@2x.png)
操作系统会维护high watermark (HW)和low watermark (LW)，当内存低于LW，会启动后台线程swap daemon or page daemon进行内存页清理，直到达到HW。
