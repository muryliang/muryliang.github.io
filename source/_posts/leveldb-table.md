---
title: leveldb 表(三)
tags:
  - leveldb
toc: true
categories:
  - leveldb
date: 2020-08-16 17:44:54
---


# 概述

这篇主要分析一下table，block相关的读写操作，同时一并分析各种iterator的使用。
<!--more-->
## block

具体结构见block_builder.cc 的开头注释描述。<br/>
要谈table，就要先谈table的最小一次性内存读入单位，也就是block，每个table都有自己的block cache,这样就不用频繁读取磁盘了。每个block内根据`restart_interval`切分为几块，这样做的目的是减少每个key需要存储的部分的长度，由于sstable中是顺序存储的，所以前后key都有相似的前缀，所以相邻key中后面那个会记录和之前那个相同的部分长度，之后才是自身独特的部分，然后是value，当然这样的后果是，要找到一个key的完整部分，需要遍历所有key，这就失去了顺序排列带来的可以二分查找的好处了，作为速度和空间利用的折中，使用restartpoint，也就是每隔n个记录，就强制放一个完整记录。所有restart index都记录在当前block的尾部。在查找的时候，如果是顺序依次查找，则只会一个一个向下解析，如果是反向查找，则需要利用restartindex跳回早先最近的完整key，然后向下查找，所以最坏情况需要查找interval个key。

## table

table 的具体结构就是多个datablock的组合, 可以从`doc/table_format.md`中看到。在文件的结尾还会放上每个block在文件中偏移以及内部最大key对应的index block，这个的largestkey主要是seek的时候使用，因为iterator的seek操作的功能是返回比指定target更大的最小的key。除了这两种block，还有filterblock以及metaindexblock，这两个是用来实现bloomfilter过滤的工具，在构建table的时候也会同步构建，每个block对应一个fitlerblock中的一个entry，实际查找的时候会使用indexblock进行二分查找，找到对应的可能包含key的block，然后根据这个找到对应的filter，判断key是否存在，如果判断存在，那么再进入指定的block，利用restart index查找key，最后检查是否存在(因为bloomfilter会有false positive的情形)

## cache

每次读取的block内容，以及每次读取的table打开文件的描述符，都会保存在不同的lrucache中，cache由leveldb自己实现,也可以设置option跳过cache，使用的时候，增大cache可以减少读取需要消耗的时间。

## iterator

iterator 是leveldb中一个很重要的数据遍历模型，在read，write，compaction中均处于核心地位，是各个组件如memtable，sstable, block之间交互的统一接口，使用继承来实现各自内部不同的细节，还通过嵌套实现比如多个文件中所有key的顺序遍历，整个db中数据的顺序非重复遍历等。

iter的思想其实很简单，就是遍历内部的数据，leveldb多数iter就是很简单的返回内存数据结构中存在的下一条或者上一条数据,顶多就是跳跃到上一个restartpoint之类的，但有两个特殊的iterator。
- TwoLevelIterator, 这个利用两层iterator实现一个系列的文件的访问(如所有level1中文件的顺序访问, 首先是文件meta信息的iter的遍历，然后是对于某个iter信息，打开文件，返回内部的table的iter，用来遍历内部的每一条entry。  同样table的iter其实也是一个twoleveliterator，首先是根据indexblock找到block，然后是block内部的iterator。
- MergingIterator, 这个用于融合多个iterator中的数据，每次返回最小的那个，如同merge排序法所做的一样。主要用于dbimple中的整体iterator，以及compaction的时候对于多个层级中文件的合并输出的过程，这个过程需要按顺序得到key，同时忽略掉seq上小的那个，保留最大的，也就是最近的那个，由于merge会把相同userkey的放在相邻的输出，所以过滤旧key变成了一件很容易的事情，而不是需要set这种数据结构去辅助的工作了。
这两个是可以互相嵌套的，像twolevel其实就套了两层twolevel在使用，设计的很巧妙，使得整体架构一致性很高。
