---
title: Operating-Systems读书笔记-Persistence
date: 2017-02-09 15:58:28
tags:
categories:
- 操作系统
---
## Persistence
### I/O设备
> HOW TO INTEGRATE I/O INTO SYSTEMS

![](/images/QQ20170209-164105@2x.png)
#### 设备抽象
![](/images/QQ20170209-165209@2x.png)

* status register, which can be read to see the current status
of the device.
* command register, to tell the device to perform a certain
task.
* a data register to pass data to the device, or get data from
the device.

#### 设备协议
	While (STATUS == BUSY)
		; // wait until device is not busy
	Write data to DATA register
	Write command to COMMAND register
	(Doing so starts the device and executes the command)
	While (STATUS == BUSY)
		; // wait until device is done with your request

#### Device Driver
> HOW TO BUILD A DEVICE-NEUTRAL OS

![](/images/QQ20170209-170624@2x.png)
### 硬盘
一个扇区的写是原子的
####  Disk Scheduling
* SSTF: Shortest Seek Time First
* Elevator (a.k.a. SCAN or C-SCAN)
* SPTF: Shortest Positioning Time First

#### 文件系统实现
> HOW TO IMPLEMENT A SIMPLE FILE SYSTEM

data structure and access methods(open, read, write, close)
#### 整体结构
![](/images/QQ20170209-195624@2x.png)
superBlock包含了该文件系统包含多少个inode和多少个data block，当mount文件系统时，操作系统会首先读取superBlock，然后将该卷连接到文件系统上。
#### Inode
每个Inode都有一个唯一的inumber对应，当给定一个inumber时，就能够计算出来inode在磁盘上的位置。
![](/images/QQ20170209-200243@2x.png)

	blk = (inumber * sizeof(inode_t)) / blockSize;
	sector = ((blk * blockSize) + inodeStartAddr) / sectorSize;
![](/images/QQ20170209-200631@2x.png)
#### 读写过程
![](/images/QQ20170210-151259@2x.png)
![](/images/QQ20170210-151323@2x.png)
#### The Multi-Level Index
Inode节点中的block字段包含了该文件所有block的地址，为了支持更大的文件，采用了多级结构。
####  Directory Organization
目录中的内容是一个Pair<entryName，inode number>的列表
#### Cache
虚拟内容和文件系统使用的缓存是动态决定的。写文件的时候不会直接写磁盘，会在内容中进行缓存（5s~30s）。这样能获得更好的效率，但是存在丢数据的情况。可以使用fsync()强制写磁盘。
#### 性能优化
> HOW TO ORGANIZE ON-DISK DATA TO IMPROVE PERFORMANCE

磁盘硬件结构，

#### 日志式
> HOW TO UPDATE THE DISK DESPITE CRASHES
> Journaling

1. Data write: Write data to final location; wait for completion
(the wait is optional; see below for details).
2. Journal metadata write: Write the begin block and metadata to the log; wait for writes to complete.
3. Journal commit: Write the transaction commit block (containing TxE) to the log; wait for the write to complete; the transaction (including data) is now committed.
4. Checkpoint metadata: Write the contents of the metadata update to their final locations within the file system.
5. Free: Later, mark the transaction free in journal superblock.
