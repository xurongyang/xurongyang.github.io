---
title: kafka文件存储
date: 2017-02-23 16:49:16
tags: [存储]
---

[Kafka文件存储机制那些事](http://tech.meituan.com/kafka-fs-design-theory.html)

每个topic的partition会分散成segment来存储，每个segment是固定大小，大概500M

![](/images/kafka-fs-segment-file-list-small.png)

segment的存储分为索引文件和内容文件，分别为index文件和log文件，**有意思的是index文件名包含了消息条数的信息**

![](/images/kafka-fs-index-correspond-data.png)
> 上述图中索引文件存储大量元数据，数据文件存储大量消息，索引文件中元数据指向对应数据文件中message的物理偏移地址。
其中以索引文件中元数据3,497为例，依次在数据文件中表示第3个message(在全局partiton表示第368772个message)、以及该消息的物理偏移地址为497。

> 从上述图了解到segment data file由许多message组成，下面详细说明message物理结构如下：

![](/images/kafka-fs-partiton-segmentfile-message-structure.png)

#### 在partition中如何通过offset查找message
1. 第一步查找segment file
上述图2为例，其中00000000000000000000.index表示最开始的文件，起始偏移量(offset)为0.第二个文件00000000000000368769.index的消息量起始偏移量为368770 = 368769 + 1.同样，第三个文件00000000000000737337.index的起始偏移量为737338=737337 + 1，其他后续文件依次类推，以起始偏移量命名并排序这些文件，只要根据offset **二分查找**文件列表，就可以快速定位到具体文件。
当offset=368776时定位到00000000000000368769.index | log

2. 第二步通过segment file查找message
通过第一步定位到segment file，当offset=368776时，依次定位到00000000000000368769.index的元数据物理位置和00000000000000368769.log的物理偏移地址，然后再通过00000000000000368769.log顺序查找直到offset=368776为止。

#### 总结
Kafka高效文件存储设计特点

* Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。
* 通过索引信息可以快速定位message和确定response的最大大小。
* 通过index元数据全部映射到memory，可以避免segment file的IO磁盘操作。
* 通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小。
