---
title: leveldb_log
tags:
---

# WAL

写入memtable之前，会先刷入log文件，这里分析一下这个log文件的格式。

首先文件名字通过LogFileName得到，配有一个唯一的数字以及.log后缀。文件内部是以Record为单位进行记录的，每条记录大小不能超过kBlockSize大小。record的格式是 crc(32bit) | len(16bit)|type(full|first|middle|last)|data, 对于剩余空间少于一个头部的(根据这种模式，单条空记录或者只能放得下头部部分的记录也会单独存储在这里)，利用全0填充。这样确保没有任何记录是横快kBlockSize边界的

传入的initial_offset只有在单个block大小里有效。
