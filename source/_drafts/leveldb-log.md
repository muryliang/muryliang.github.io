---
title: leveldb_log
tags:
---

# WAL

## Write

写入memtable之前，会先刷入log文件，这里分析一下这个log文件的格式。

首先文件名字通过LogFileName得到，配有一个唯一的数字以及.log后缀。文件内部是以Record为单位进行记录的，每条记录大小不能超过kBlockSize大小。record的格式是 crc(32bit) | len(16bit)|type(full|first|middle|last)|data, 对于剩余空间少于一个头部的(根据这种模式，单条空记录或者只能放得下头部部分的记录也会单独存储在这里)，利用全0填充。这样确保没有任何记录是横快kBlockSize边界的

传入的initial_offset只有在单个block大小里有效。

## Read

initial offset这里的作用和write不一样，用来跳过整块的数量，而小于一块的部分，在ReadPhysicalRecord中会每次读取后检测跳过，检测InitialOffset跳过, 然后在ReadRecord中空转一个循环，继续读取，直到超过offset的位置为止(这里对于不到offset的地方报出的corrept在实际处理的地方会检测offset，这种的会忽略报告，所有不会产生异常)。

有几个变量作用要记录一下
- resyncing_: 这个用于在initial offset存在的时候，跳过开头部分的middle last等记录，用于快速处理。
- last_record_offset_: 上一条读取成功的记录的开头偏移，
- prospective_record_offset同样的作用，不过用于横跨一条记录的不同部分，最后在last的时候会赋给last_record_offset_
- physical_record_offset:用于在每次ReadPhysicalRecor之后记录每条record的起始位置，用于在full和first的时候赋予prospective_record_offset
- end_of_buffer_offset_:这个初始赋予跳过的整块的offset，然后每次读ReadPhysicalRecord之后都会增加相应大小，用于计算每次physical_record_offset,由于每次读取后都会从内部buffer中删掉，所有每次end_of_buffer_offset_ - buffer_.size() - kHeaderSize - fragment.size()都可以得到当前record的开头。

实际每次返回的record的slice所承载的buf，在full的时候是使用的ReadPhysicalRecor内的backingstor，而在具有first，last的情形下是使用的传入的scratch


先描述一下内部实际读盘的函数ReadPhysicalRecord, 每次读取一个kBlockSize的块，放入内部的backingstore空间，然后一次读取一个record. 在不满足整块的时候会记录eof事件，在下次读取的时候返回，如果普通的read一样。正常情况下每次读取一个record之后会检测type然后构建slice返回，如果有错误的record,会主动清除buffer，其后果就是下次读取的时候会fetch一个新的kBlockSize，也就是出错会放弃当前block剩余所有字节。

ReadRecord 则利用上述函数，实现一次返回一条完整的逻辑记录，在遇到eof的时候返回false，遇到bad_record的时候则利用reporter报告，然后继续读取。
