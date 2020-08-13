---
title: leveldb_db
tags:
---

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

