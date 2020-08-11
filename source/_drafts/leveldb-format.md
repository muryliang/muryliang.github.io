---
title: leveldb_format
tags:
---

# 数据格式与存储

## key格式

### internalkey

用于内部表示一个userkey，具体格式是 key + seq(56bit)|type(8bit)(del or put), 通过InternalKeyComparator进行互相比较。首先Memtable构造的时候传入的是util/db_options.h中的BytewiseComparator，顾名思义，就是普通的Slice的compare方法实现，然后InternalKeyComparator内部的Compare首先使用BytewiseComparator进行比较，如果不同直接返回结果，相同则比较internalkey后面的seq|type,由于每个key的seq不一样,这里实际比较的就是seq(处于高位56bit),大的表示最新，就应该排在前面。所以是根据userkey升序，根据seq降序。

#### InternalKeyComparator::FindShortestSeparator

用于找到比start大但是比limit小的作为存储中block的界限

#### InternalKeyComparator::FindShortSuccessor

和上面的相似，这里找的是大于等于当前string的最小string

### LookupKey

这个是用于查询的key结构，内部是totallen | userkey | seq | valtypeforseek, 用start_表示整个key的开头，kstart_表示去掉encode的长度之后的userkey开头，end表示包含后面seq|type的结尾。内部做了小数据优化，使用内部数组而不是默认分配

### Slice

用于和go类似的只存储指针和长度的slice，要求原string必须在slice生命期间一直有效，不可修改。内部的compare在相同的情况下短的为小。

## 数据编码方式

leveldb使用的编码致力于节省资源，所以使用的varint这种变长存储长度的模式。

- PutFixedXX 这种需要以小字节序把对应u32/u64放入char中，不是直接强制转换是因为这样可以跨不同架构的机器，据说是从protobuf中摘出来的代码,在这里的用途可能是数据库文件本身平台迁移吧。

- PutVarintXX 类似utf的编码方式，利用每个字节最高位记录是否需要下一个字节的帮助，为1表示下一个字节也需要解析，对于32bit总共需要5字节

- PutLengthPrefixedSlice 用于在string前加入变长的string长度

- GetVarintXX: 从头部取出variant编码的值返回，同时把input修改为其后的部分。

- GetLengthPrefixedSlice: 有两个重载，分别处理char*以及slice，都会返回剩余的部分以及当前的结果，一个用于bool探测，另一个用于直接返回char*

## 内部内存分配

内存分配使用area结构，这个本质还是使用new分配，只不过多了aligned接口保证分配在指定对齐位置，并且至少8字节对齐。这个我觉得有用的地方就是给MakeRoomForWrite提供当前内存使用量信息，并且这个是粗略的当前已分配block数量(32k单位), 何时新block？
