---
title: leveldb_write
tags:
---
# todo

- WAL
- skiplist
- 丰富batch group
- snapshot, sequence

# todo current blog

- DBImpl::MakeRoomForWrite
- BuildBatchGroup

# 写操作

leveldb分为读写两种操作，读在下一次分析，这次分析写操作的流程。

无论是Put还是Delete都只是一个不同类型的写操作，在内部两者都是通过形成一个`WriteBatch`然后执行`DBImpl->Write()`来实现的。Write本身是一个**单线程**操作, 通过DBImpl的mutex_锁来维持序列化。每个`WriteBatch`都会被首先包装成`Writer`类，然后获取锁，将自己通过`push_back`放入全局的`writers`的队列尾部，等待写入。这个实际写入由自己或者其他排在自己前面的人来完成。也就是说，如果当前`writer`是`writers`队列的第一个(也就是队列里只有自己一个写入操作)，那么这里代码会直接继续执行下去。如果队列前面还有其他写操作，那么这里会利用初始化时获得的全局`mutex_`执行cond_wait,等待其他人来完成自己，或者自己成为第一个，这两种情况的区别在于下面的一个`BuildBatchGroup`中的一个条件判断，这个下面再讲。

如果不是第一个,在wait的过程中被唤醒之后，判断是否已经被之前的writer给顺带一起写入了。这里是利用batch实现批量写入，可以批量写入WAL(log_->AddRecord, 同时在写入的时候，释放mutex，其他并发的writer会在这个时候加入writers，当时和当前的batchgroup没有关系了，会在之后由这一批结束之后的接下来第一个writer统一group之后写入)，这里应该是为机械硬盘优化的，一次性写入越多可以减少寻道时间，而之后的写入memtable的操作是纯内存的,内部也是逐条插入skiplist，不会受到batch的影响，唯一可以说有影响的应该是减少了mutex的开销以及压缩了多个重叠的函数调用路径。如果已经被写入了，那么直接返回写入结果的状态，操作结束。

如果是第一个，且尚未写入，则继续执行下面的操作。在DBImpl::MakeRoomForWrite中会根据当前sstable数量，memtable大小以及当前写入状态，决定是否需要执行compaction操作，是否需要等待compaction完成，在这里面是可能发生主动睡眠以及compaction等待睡眠的。这个下面分析。完成之后, 开始BuildBatchGroup,这里会从writers里收集尽可能多的writer尽量一次性发送给wal，这里面会在group太大或者first是sync但是当前是async的时候停止添加。完成group之后,会释放mutex_(之前wait之后直到现在都是持有mutex_的，所以不会对list以及background compaction产生竞争)。 然后开始实际的写入操作，由于上面已经分配的空间，做了compaction，所以这里的写入不会涉及其他线程的触发，直接就是写入wal，然后一次加入skiplist节点，然后直接返回。之后signal通知每一个已经写入的writer(通过之前buildgroup的时候返回队列最后一个来标记结尾),最后还要通知剩余的writers队列的头部继续这个执行流程，然后当前写也返回。
