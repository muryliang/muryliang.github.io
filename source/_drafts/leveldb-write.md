---
title: leveldb_write
tags:
---
# todo

- refs
- data format
- bloom filter in util/bloom.cc
- code style
- thread annotation: GUARDED_BY
- dumpfile.cc how to dump
- repair
- table build builder.cc + table/table_builder.cc
- table read(with cache, already known) and block read
- iterator
- compaction


# todo current blog

- 

# 写操作

## 概述

leveldb分为读写两种操作，读在下一次分析，这次分析写操作的流程。

无论是Put还是Delete都只是一个不同类型的写操作，在内部两者都是通过形成一个`WriteBatch`然后执行`DBImpl->Write()`来实现的。Write本身是一个**单线程**操作, 通过DBImpl的mutex_锁来维持序列化。每个`WriteBatch`都会被首先包装成`Writer`类(这里的是dbimpl中的内部Writer，不是单独的Writer.cc中的Writer)，然后获取锁，将自己通过`push_back`放入全局的`writers`的队列尾部，等待写入。这个实际写入由自己或者其他排在自己前面的人来完成。也就是说，如果当前`writer`是`writers`队列的第一个(也就是队列里只有自己一个写入操作)，那么这里代码会直接继续执行下去。如果队列前面还有其他写操作，那么这里会利用初始化时获得的全局`mutex_`执行cond_wait,等待其他人来完成自己，或者自己成为第一个，这两种情况的区别在于下面的一个`BuildBatchGroup`中的一个条件判断，这个下面再讲。

如果不是第一个,在wait的过程中被唤醒之后，判断是否已经被之前的writer给顺带一起写入了。这里是利用batch实现批量写入，可以批量写入WAL(log_->AddRecord, 同时在写入的时候，释放mutex，其他并发的writer会在这个时候加入writers，当时和当前的batchgroup没有关系了，会在之后由这一批结束之后的接下来第一个writer统一group之后写入)，这里应该是为机械硬盘优化的，一次性写入越多可以减少寻道时间，而之后的写入memtable的操作是纯内存的,内部也是逐条插入skiplist，不会受到batch的影响，唯一可以说有影响的应该是减少了mutex的开销以及压缩了多个重叠的函数调用路径。如果已经被写入了，那么直接返回写入结果的状态，操作结束。

如果是第一个，且尚未写入，则继续执行下面的操作。在DBImpl::MakeRoomForWrite中会根据当前sstable数量，memtable大小以及当前写入状态，决定是否需要执行compaction操作，是否需要等待compaction完成，在这里面是可能发生主动睡眠以及compaction等待睡眠的。这个下面分析。完成之后, 开始BuildBatchGroup,这里会从writers里收集尽可能多的writer尽量一次性发送给wal。完成group之后,会释放mutex_(之前wait之后直到现在都是持有mutex_的，所以不会对list以及background compaction产生竞争)。 然后开始实际的写入操作，由于上面已经分配的空间，做了compaction，所以这里的写入不会涉及其他线程的触发，直接就是写入wal，然后一次加入skiplist节点，然后直接返回。写入首先是wal，然后按需求进行sync(异步不会sync)，之后写入memtable，最后更新sequence，之后signal通知每一个已经写入的writer(通过之前buildgroup的时候返回队列最后一个来标记结尾),最后还要通知剩余的writers队列的头部继续这个执行流程，然后当前写也返回。这整个过程都是单线程的，所以不存在争抢的问题。

## MakeRoomForWrite

此函数必须在locked且writers非空的状态下运行。参数force 对应的是writebatch为null的情况，同时表示`allow_delay == false`.这里force的意思的必须进行一次compaction，而allow delay的意思是允许下面的超过softlimit之后进行平摊延迟的操作，这两者情况是相反的，强制compaction的时候不允许delay(为何?)

1. 首先处理backgroundError, 如果之前compaction出现error，会在其后某个writer的时候返回错误。
2. 只要writebatch不为null，允许delay，这时如果L0层文件数量超过softlimit，就会开始平摊预留给后端compaction的延迟时间，减少方差，每次1ms，同时让出cpu，避免share core的时候阻塞compaction，但是每个writer只允许delay一次。
3. 如果已用内存数量少于指定的大小，则room已经预留足够，直接返回
4. 再接下来就是预留不足的情况了, 查看imm 如果非null，代表正在进行compaction,代表结束后可以预留出空间，所有在这里wait，之后再次循环，会进入选项3, 然后break退出
5. 仍然是预留不足，但是imm为空，如果L0文件超过hardlimit，那么同样compaction一定正在进行，同上等待。
6. 如果L0文件少于softlimit，没有imm，却仍然已用空间超过限定空间，或者指定了force而导致尽管没有超过限定空间，仍然没有在3中跳出。那就生成新logfile，替换mem为imm，然后触发compaction(这里imm必定之前为空，不然在选项4就会被catch。如果是force的情况，那么会force=false，重新loop会从选项3跳出

## BuildBatchGroup

在这里会进行多个writebatch的合并，意在一次可以写入多个writebatch。主要合并结束点为
- 总batch大小超过规定大小，这个大小默认1M,但是如果head的batch大小小于128k，那么会缩减为head + 128k.
- head是async而当前需要并入的是sync的batch，这样会影响async的操作，不允许

另外，当需要合并时(即总数至少2个batch时), 为避免破坏作为输入掺水的head的batch，会使用一个固定的tmp_batch作为收集用的batch，这个会返回给上级函数，同时返回最后一个合并进入的writer。在上层函数中会把headbatch的sequence设置为`last_sequence+1`,同时设置之后的`last_sequence+1`为`last_sequence+count(batch)`

## WriteBatch

WriteBatchInternal是用来作为WriteBatch中操作的工具类，同时不想暴露给外部。

WriteBatch内部是 start_seq(64) | count(32) | key(varstring) | val(varstring) .....的模式，实际内部通过InsertInto经由的MemTableInserter实现每个entry的插入，对于delete插入的是空的value。每进行一次Add操作，就会基于sequence++,所有每个entry的seq都不相同，外部db_impl.cc中只会在writebatch头部记录一个起始seq，然后在write完成后更新最终seq。不管内部出错与否，都不能够重复使用这里已经分配出去的seq，只能使用超过当前writebatch的seq。


# 读操作

读操作按照memtable，imm，sstable的顺序进行访问，根据传入的snapshot的seq或者当前seq决定最新允许的结果状态。具体的遍历操作在下面描述。需要注意的是在需要访问sstable层次的时候，会涉及更新Version::GetStats，直接导致在read操作的时候也会产生compaction。预测持续随机读冷数据的情况下会产生较多compaction，占用过多cpu，同时根据write的代码，在write需要compaction的时候，如果仍然正在compaction(由于read),就会导致额外的延迟。
