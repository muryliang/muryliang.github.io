---
title: leveldb_db
tags:
---

# todo
- db_impl
- version_set/edit
- More pictures

# db

## dbiter

主要用来NewDBIterator中遍历所有加入的文件及mem imm合并的merge iterator，注意所有key首先根据userkey升序，然后内部根据seq降序从iterator中吐出来。

### Next

对于reverse的情况，saved_key里是当前key，iter需要做个next操作，从下面描述的FindPrevUserEntry的iter的位置向后一格到达和当前userkey相同的userkey区段，之后正常FindNextUserEntry。

#### FindNextUserEntry

首先来考虑FindNextUserEntry的情况,跳过所有delete的key，以及高seq的key，返回第一个不用跳过的。
- Seektofirst,到第一个(对于Seek同理)，然后FindNextUserEntry(false,empty_str),这里面会parse每个iter下的key，忽略每个key中高seq超过指定seq数的那些,然后
    1. 首先碰到kTypeValue, 那么由于false，直接返回值，意思上这个是第一个满足seq条件的first key。
    2. 首先碰到kTypeDeletion， 把key名字保存在skip 中，设置skipping为true，继续循环，对于接下来的同一个userkey(低seq，满足`user_comparator_->Compare(ikey.user_key, *skip) <= 0`)全部跳过，直到下一个不一样的key出现为止，设置valid，然后return, 此时iter中就是所要的next之后的key了，外面可以字节iter->key(), iter->value()的得到kv
- 对于Next, 先不考虑reverse的部分，正常next的情况下，需要iter_->Next()之后FindNextUserEntry(true, curkey),这样在FindNextUserEntry中如果碰到和next之前相同的那个key(肯定是seq偏小的),如果不是delete类型,也会由于skipping参数而忽略，如果delete则走上面的路，一样忽略。直接进入下一个不同的key进行检测。

### Prev

Prev 内部结束后saved_key位置保存当前key，iter位置保存在此之前的不同key。对于forward的情况, saved_key保存当前key，然后不断prev直到达到userkey不同的前一个，这样和进入FindPrevUserEntry需要的条件就一样了。

#### FindPrevUserEntry

- 再来考虑FindPrevUserEntry，对于这个函数,外部的saved_key里保存的是当前key，是没有用的，会被里面覆盖。内部照常会在遍历中跳过所有高seq的，因为是prev遍历，所以seq在同userkey的情况下是越来越高。首先第一个满足seq的key一定不会进入第一个if检测，从而会被下面的del或者savekey记录。
    1. 如果第一个满足seq的是del，那么清空savedkey，如果后面一直是delete，那么会循环直到结束，最后返回eof。如果中间某个非delete，记录下来，继续循环，直到找到下一个不同的key为止(是不是del无所谓)
    2. 如果第一个是非del，同样记录下来，继续循环，直到找到下一个不同的key(这个key是不是delete无所谓，到此就停止了，记录在iter内)
- 函数结束后saved_key中保存的是prev之后的key，而iter则是其之前另一个不同userkey的位置，用于后面FindPrevUserEntry使用。


## db_impl

这个是直接和用户接触的部分，也是控制后台所有操作的部分，放在了所有分析的最后来讲。这里插一下里面mutex的使用，内部的mutex和backgroundsignal用的是同一个，我之前看代码总是幻想可能在while判断的之后，wait之前，判定条件就被另一个线程改掉了，导致无限等待，但实际上所有判断都是在mutex lock内的，不会造成竞争判断。

- SanitizeOptions 规范option中传入的值，并且对于infolog(记录一些日志的，不是给leveldb用的memtable log), 如果没有传入，创建新的，对于tablecache，没有传入，创建新的，这个在后面version里面会判断是否是这里创建过的，是的话需要在析构里delete掉，否则是不处理的。
- NewDB 会创建一个新的db，从头计数log，manifest文件，设置currentfile
- Recover 会在db不存在的时候创建新的，然后从manifest中恢复version，然后利用RecoverLogFile 加入新的log文件的内容。
- RemoveObsoleteFiles 对于`pending_outputs_` 中没有记录的tempfile， 以及不在version set指定lognumber，manifestnumber范围内的文件，不在live files范围内的文件，统一删除。
- RecoverLogFile: 对每个新的log文件都会调用，load 进入memtable，然后compact进入L0,一只log文件对应一个以上的memtable，根据write_buffer_size 决定写入时机，对于最后一个logfile，需要判断是否需要重用文件,条件是当前最后一个log文件，当前文件没有因为过大而涉及过memtable的L0写入。
- WriteLevel0Table: 由上方调用来写入L0 文件，更新compaction stat状态(涉及哪个level写入了多少字节), 如果有base 的version指针，还会根据那个计算应该把memtable下方到第几个level. 比较要注意的是对于正在创建的文件，会放入pending_output 数组中，这个在RemoveObsoleteFiles的时候会涉及，被避免删除掉。 同时把这一切都放入edit中，供上层记录。
- 在上述的Recover操作中改动的文件，特别是新增的logfile 恢复导致的table文件的产生，都由Recover()带入的edit来记录，最后由外层进行version记录。

- CompactMemTable :被用来通过WriteLevel0Table 把imm写入文件，将对应的version edit apply到version set中，也就是文件中作为一个edit，同时在version set内存结构列表中添加一个新version
- CompactRange: 这个会计算需要compact的range对应的level的最大值，然后等待imm的处理完成，之后从0层开始compact到指定层
- MaybeScheduleCompaction 这个是所有compaction的入口，这个内部会启动后台线程BGWork进行compaction处理，无论是manual还是其他情况引起的需要compaction(如allow seek相关),但是一次只会启动一个。
- BackgroundCall 这个是后端compaction的入口，在结束之前会继续调用MaybeScheduleCompaction，重复创建，自己会正常结束，以此来应对上一次compaction产生过多文件的情况。(每次version中logandapply都会重新计算每层的score，高score会优先是下一次compacti的目标层次)

- BackgroundCompaction: 如果imm存在，只compact这个，然后立刻返回。否则根据是否是手动或者Pick出来的，选择compaction，包装在一个compaction类中。 对于可以简单move的情况，直接move(manual不行)。然后就是docompaction的工作了，完成后，对于manual的情况，需要修改manualcompaction结构的状态，然后上层函数会signal通知。具体的compaction相关在下面。

- CleanupCompaction: 只是用于清理compaction完成后的情况，包括pending_outputs的取消，打开文件的关闭等。
- OpenCompactionOutputFile打开一个output文件用于compact，将打开table的信息放入传入的compactionstate结构体，这个compactionstate收集了所有的outfile，每次调用这个都打开一个新的用于放当前输出。
- FinishCompactionOutputFile 完成了单个输出文件的写入后调用这个，进行table的finish操作
- InstallCompactionResults 这个是完成所有compaction之后，在edit中添加需要删除和添加的文件，然后组成edit输出到manifest中。
- DoCompactionWork 具体的compaction操作,首先通过snapshot建立最晚的可以compact的seq，这个在后面选择是否丢弃key的时候有用。如果存在imm，先compact并固化到manifest中(但是此时这个l0的table不会被放入compaction序列中啊), 这里利用merge iterator 依序遍历input文件中的所有key， compact实际很简单，就是用merge遍历所有key，由于cmp对于seq是降序的，第一次见到的key，必定是seq最高的，如果这个是个delete并且seq比snapshot小，并且高层没有同key了，那么就可以安全的忽略这个key而不会导致找到高层key的不一致，对于已经写入的key，后续的也会忽略。compact工作全部完成后，将compact信息添加到stats中，然后InstallCompactionResults建立version.

- DB::Open 用来打开数据库，会先Recover，然后看是否需要创建mem table以及是否需要保存一个新的manifest文件，这个在manifest不够大的时候不会去创建新的文件的。
