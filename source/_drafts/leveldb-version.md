---
title: leveldb 版本控制及compaction实现(四)
tags:
- leveldb
---

# 概述

leveldb的版本控制是通过snapshot完成的。snapshot其实即使一个数字，代表了当前允许查询的最大seq，相当轻量级。和版本相关的有几个数据结构:`version`, `version_set`, `version_edit`

## Version

这个代表一个单个的version，一个version中包含了本version的每个层级存在的文件,以及compaction相关的信息，这是一个纯内存的数据结构，链接进入`version_set`中，在读写的时候会被引用，然后保持在内存里，一旦无任何人引用，就会自动销毁。最新的version永远被current引用，保持不会销毁。

## Version Set

这个是所有处于被其他某些对象引用状态的version的集合，在remove file的时候，会参考这里面所有的文件，只有不在这里面且指定需要删除的文件才会被删除。

## Version Edit

这个表示了两个相邻version之间的文件差异，也是manifest文件中存储的每条记录的格式，最开始的一条version记录，无论是newdb开始的，还是某次打开之后重新写新的manifest以加速下次打开后recover过程的，也是通过转换成version edit结构写入manifest的。

## compaction

由于version控制实际理念很简单，这里就一并把compaction也讲掉吧, 每次触发compaction后，具体的执行线程里会查看是否手动指定了范围，如果是，那么在那个范围内的所有level从上到下都要执行一遍compaction；否则就是通过内部的Pick函数去选择哪个层的那个文件执行compaction。 这个函数会根据seek之后的状态修改(VersionSet::Builer::Apply中)，或者上一次compact之后当前version重新计算的每层的score(VersionSet::Finalize中)决定选择的key的范围。

具体来说只要是和选定key范围重叠的当前层(L0会有这种情况，因为L0文件不是顺序不重叠的)和下一层的文件，都会包含进去，具体的选择策略中，还会尝试如果下一层中包含文件的整体范围应用到当前层后，加入的新文件造成的当前层compact key范围扩大之后，不会再次造成下层文件范围扩大，那么就选用这个更大的范围，主要目的应该是把level+1中选到的文件所对应的key更彻底的一次性和当前level合并掉。

选择完成后，就会利用mergingiterator把所有输入文件中的key，通过排除不在需要保存的seq范围内的，其他的都一并保存下来，放入输出文件，要注意，输出之后，同层的文件都是按照internal key的顺序保存下来的,不存在范围重叠，但是存在同userkey但是不同seq的那些key，这是供不同version访问的。完成后，不需要的文件会unref，最终在无人引用的时候被删除。
