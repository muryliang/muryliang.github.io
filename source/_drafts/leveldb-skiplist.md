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

### memory order

- acquire load, release store, 保证了对于同一个atomic变量，store之前的所有其他write操作都在store被与其对应的load观察到值之前全部完成并对相应cpu可见，同时保证store不会下移乱序超过atomic的store，load不会上移
- relaxed这个只保证操作本身的原子性，不存在中间状态为外部可见。在设置max_height的时候使用的是relaxed,因为这里就算读到的时候新height时，如果在insert中接下来的插入操作没有完成，也只是得到了旧head节点，直接会走到下一层，如果插入完成了，那么会得到得到新的插入节点(受益于接下来的acquire load),从而永远保证不会出现其他中间状态。

### skiplist 结构

skiplist由node组成，高度最高定义kMaxHeight, 每个节点高度都是maxheight，头部节点key是0,这样默认最小,尾部节点是一个null指针。node间的排列由memtable传入的compare判定。每次插入的节点利用随机数设定高度，利用findgreateroequal找到每层中在自己前面的节点，从低到高利用atomic的release store设定，这样可以保证设定的值被读取之后，对应的节点确实已经插入(使用next_[n].load的时候)，支持并发读。

- 遍历操作就是遍历level 0的节点,持续Next即可
- Seek操作就是FindGreatOrEqual,找到的不一定是本身，可能不存在,得到之后的节点。
