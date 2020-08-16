---
title: leveldb 读写(二)
tags:
  - leveldb
toc: true
categories:
  - leveldb
date: 2020-08-16 17:44:51
---


# 概述

leveldb的写全都是通过WriteBatch完成的，读则是Get接口，这些内部除了准备阶段，其他全都是可以并行的部分，也不必担心rw的过程中如果有其他的rw发生，或者compaction发生而导致文件的消失，首先写入是一次性的，不存在写入的文件还可以修改；其次，MVCC的模式使得那些正在被查询或者用来compaction的版本不会被意外删除，所以get和compaction不会因为文件内容错乱或者文件不存在而发生错误。
<!--more-->
## Write

不论是Put，还是Delete，都是包装成WriteBatch结构下发的，当然你也可以直接使用writebatch，一个writebatch中的数据可以保证是写在一起的，并且一次性写完的。

- 准备工作: 所有的写会在Write中通过mutex进行序列化，只有排在开头的写可以继续下去，其他写必须等待。而可以继续下去的写也不是只写自己，这样就不能和好的利用批量写的特性了(其实这里的批量写并不是写sstable，只不过是写wal的时候的批量写，但同样也是需要写硬盘的，批量会更快一点), 这里会首先利用`MakeRoomForWrite`检查是否需要启动compaction并且等待完成，这里有一个平摊延迟的策略，就是L0文件多于softlimit的时候，会主动延迟1ms,这个是为了让compaction有机会运行，也是为了不至于写得过快导致compaction后面赶不上，结果导致最后触发hardlimit的write延迟过高，造成整体延迟抖动过大。这段的等待会释放锁，让其他的write有机会在期间加入。在通过size检查后，使用`BuildBatchGroup`把在之前等待期间也加入的writebatch合并到一个writebatch中。

- 开始write: 这里会释放锁，因为memtable是读可并发的。
    1. 写先要写入wal中，这个是为了在每次启动的时候可以恢复memtable的内容。wal的结构也是很简单的，只有一个限制，就是每条记录都必须满足小于kBlockSize大小, 因此，对于过长的record，需要切分，这也就有了full,first,middle,last几种record类型，每条record通过头部的crc，长度，type来分隔。
    2. 写完wal之后，就是memtable了，这个本质上就是一个skiplist，skiplist是一种依据概率形成的数据结构，拥有与红黑树同等级的查询和插入复杂度，但是去除了节点旋转平衡的开销，代价是需要概率来保证查询复杂度O(lg N), leveldb的skiplist不需要支持删除操作，也减少了很多复杂性，skiplist利用了atomic操作实现了一写多读的并发。到了这一步之后本次写就不会触发compaction了，而是在下次写的时候在MakeRoomForWrite里会触发，或者在重新打开db的时候的log recover的时候会触发。

- 收尾工作: 成功写完后，会对group中的每个写都发送signal通知完成，同时通知writers中接下去的head的write继续执行，然后本次write返回。

## Read

Read 由Get操作完成，具体的操作是首先记录本次Get能够接受的最高的seq(从snapshot参数中获取),这个实现了version控制，然后对需要访问的memtable，immtable(这个只有在write的时候才会触发产生，并且只要一产生就会进入compaction流程，所以这是一个暂时性质的东西)和current version进行引用获取，通过这个操作，涉及的所有东西都不会被其他线程擅自删除了，而是会在最终引用归零的时候删除，也正是因为如此，实际内存中存在的memtable在某一时刻可能是多余两个的。假设某个get正在进行并且引用了某个imm，同时一个write触发的imm的compaction结束了，那么write方会unref这个imm，但是imm被Get引用，不会删除，如果Get进程特别慢，在这期间mem又满了，转换了imm，那么内存中当前就会存在超过2个memtable结构，当然这种极高负载的情况在leveldb容量不是非常大或者磁盘故障的情况下不太可能发生。

Get会依序从mem，imm，自低到高每一层的sstable来搜索，同时通过GetStats记录seek操作行为，这个是为了感知本次查询是否遍历了过多的文件，根据`version_set.cc Apply`中的注释，过多的seek时间等同于一次compaction，所以get过程中多余的seek会统计起来，达到一定程度就触发一次compaction，以提高今后Get的命中率。

## 总结

总的来说，读写结构清晰，而且不存在其他隐形的常驻后端进程的干扰，简洁高效, 也有助于代码分析。
