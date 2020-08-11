---
title: leveldb_skiplist
tags:
---

# 跳表

## 概述

leveldb 的 Memtable使用了

## Memtable

memtable是控制skiplist插入和查询操作的包装类。

- Add: 定义了每个插入的entry是 (var)key_size | key | seq | type | (var)val_size | val。
- Get: 查找的时候，通过构建的LookupKey(通过userkey+seq构造出的与skiplist entry的头部key部分直到seq|type为止的部分)得到的key在skiplist的iterator中(注意不是同文件的memtable的iterator，这两个不一样，后者会加一层头部len的)进行搜索，利用InternalKeyComparator进行比较，只比较userkey部分，因为seq已经根据skiplist内部的大小比较过了,对于snapshot的情况，过大的seq会直接过滤掉。找到对应的key中seq最高的那个，然后根据val的type，如果未删除则返回，删除则返回notfound,同时返回true.只有在key从来未存在的情况下才是false。
- Compare: 利用外部传入的compare函数，包装成可调用，供给内部skiplist用来比较去除头部的len之后的key部分(不会比较到value部分),见`MemTable::KeyComparator::operator()`

### memtable iterator

这个其实不同于上面的Get过程中使用的iterator，这里的iterator是直接供给外部使用的，他会包装传入的用于查询的internalkey，加入长度头部,然后供skiplist搜索，对于搜索结果的key(),value()获取，也会全部接触头部的len prefix，所以对外接口全部是slice的key，value值本身。
