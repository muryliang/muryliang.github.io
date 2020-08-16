---
title: leveldb 概述(一)
toc: true
categories: 
    - leveldb
tags:
    - leveldb
---

# 概述

写在前面，优秀的leveldb分析已经很多了，想要看梗概的人可以直接看[庖丁解LevelDB](https://catkang.github.io/2017/01/07/leveldb-summary.html)这个系列的文章。

花了一周时间把leveldb的所有代码逐行解析了一遍，除了那些test文件，可累死我了，大多数文件只有不到300行还行，`db_impl.cc` 和`version_set.cc`这两个1500行的文件最累，虽说文件函数逻辑清晰，但自己写的时候还是避免这种过长的文件吧，前后联系看到时候特别累，当然比起ceph的bluestore单个文件1w多行而言仍算是小巫见大巫了，这个留作下次分析。这个level系列的几个blog主要是对之前详细分析的一个总结，并不会在这里逐行解析，也没这个必要，leveldb的精髓是其主要思想，而不是里面的辅助函数，如env_posix层。代码总有效行数，从`cloc`来看，大概在17000行左右，分析的路线从和用户最相关的地方开始，也就是读写，然后引出读写中牵涉的部分，如wal，memtable，table，block，cache，之后讨论version以及compaction，最后通过dbimpl进行总结。
<!--more-->

# 缺点

在分析开始之前，先谈一下缺点，再好的项目，也只能在其特定的应用场景大放光彩。

## sdd

leveldb的设计包括其基于的LSM的存储结构，完全是为了优化机械硬盘存储而生的,尽可能避免随机写，读也是以block为单位进行读取并缓存的(默认4k，这个包括其他option都在`option.h`中进行调整),compact的时机选取和估算都是以hdd的磁盘寻道延迟为考量的，也没有考虑过利用ssd随机存取的特性，所以如果以sdd为存储介质，并不是很推荐使用leveldb。

## 写放大

写入可能会造成compaction，也就是一次的写入可能会造成后台读入十多个文件，然后在内存中进行合并操作并输出到某几个文件，具体的写放大没有算过，但是对于kv尺寸比较大的数据业务而言，compact的触发会比较频繁，即使调高memtable容量或者加大单个文件容量，也会在compaction时造成很大的延迟, 可以查到的改进策略主要是kv分开存储，key走lsm，value走wal，[如这篇论文](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf), 优点是减少写放大，缺点的话应该是value的gc处理可能会引起较多的读写惩罚了。据说rocksdb对ssd是有优化的，ceph默认的bluefs使用的也是rocksdb，下次研究一下它是如何改进的。

## 缺点总结

不过总的来说，在ssd，nvme，甚至是pmem大行其道的今天，以及未来，我相信随机读写的开销会越来越小，以至于为高性能而生的LSM的这种化随机为顺序的存储模式将会成为一种过去。
