# leveldb-Write|sstable|Compaction

Created: June 5, 2022 4:25 PM

### Write Batch成员变量及结构

1. rep_ 的编码格式：
    1. 8字节【sequence】＋4字节【count】＋变长records 数组
    2. records数组元素：1字节【value类型（是否delete）】＋变长【key大小】＋key大小【key内容】＋变长【value大小】＋value大小【value内容】

# Write主流程

- LevelDB 对外提供的写接口 为 put(options,key,value) 和 delete(options,key)
    - 由于delete的删除方式是添加一块与key相关的tomb，因此两个函数最终都收敛到Write实现
    - 两个函数 都会构造一个write batch = sequence  + count + record[count]，然后调用Write函数
- Write 流程：Part 1 —— 准备和检查阶段
    - 构造一个local的 writer，并且持有当前db的metux，将writebatch 和 option赋值给writer
        - mutex保证了同一时刻只有一个写线程能够执行。
    - 将 writerbatch 加入 到writers队列（只有持有metux锁的情况下才能加入）【一个很RAII的设计在MutexLock对象构造的时候上锁，在对象销毁的unlock，对象的生命周期自动管理】
    - if（当前writer 没有done，并且不是不是队首的writer）在cv上wait 等待唤醒；if（writer done）返回writer的状态
    - 此时，是队列head，需要执行write 的流程（此时 持有db的锁）
- writer：Part 2 —— 写入流程
    - 给 写入 创建room，作用：确保memtable 有足够的空间可以写入，如果memtable空间不足，需要创建新的memtable，旧的memtable编程immutable
    - 获取最新的sequence（也就是version信息），因为当前的writer的队首的，基于这个writer，构造一个 writer group，相当于合并多个writer任务（合并之后还是writer batch 的结构）。这时，last writer的指针会指向这个group中最后一个合并的writer，将这个writer batch的sequence+1，更新sequence
        - 目的合并写入，提高吞入量，并且当前是 持有metux的状态，可以安全的将writers队列中的writer进行合并
        - 这个合并后的writer batch的sequence 是 最新的加一，然后last sequence 会加上这个writer batch中的writer count，作为下一组的sequence
    - 在写 log的时候释放mutex，使得写log和其他对memtable的写入可以并发
        - log文件中加入records，如果options强制落盘，此时需要将log文件落盘
        - 然后将这个 write batch 写入到memtable中
    - 最后 错误检查和用sequence更新version
- writer：Part3 —— 唤醒队列中的等待writers
    - 把队列中这个等待的writer出队，设置状态，唤醒，直到遇到之间的writer
    - 如果队列里还有新进来的writer，唤醒队列头的writer，然后返回，自动释放锁

### Build Batch Group

- 要求writer队列不为空，并且第一个writer要有非空的batch
    - 确保已经持有锁，调整maxsize
    - 从第二个writer开始遍历，并且用一个temp的batch合并所有的batch
    - 返回temp batch，并且 last writer 为最后一个writer

### Make Room For Write

- 为write 创建写空间 也就memtable的空间
- 检查后台线程是否正常，如果异常，那么直接返回
- 如果 允许 delay（有数据）并且level0层文件数量超过慢写触发值，解锁，等待1ms，然后不允许延迟（目的是为了积累写，并且给compaction线程执行时间  level0文件太多了）
- 如果不是强制写（有数据），也就是没有数据，并且memtable还有空间，就跳过
- 如果当前memtable已经写满了，并且immutable 还没落盘，需要先等immutable落盘
- 如果level0层文件太多，还需要等level0层文件太多，也需要等待整理
- 至此，需要尝试 切换新的memtable了，给新的memtable创建log文件，然后尝试调度compaction线程

# sstable 结构

- 把内存的table持久化都磁盘的目的
    - 降低内存空间，同时维持数据的持久性
    - 定期compaction的原因？避免WAL太长，系统恢复时间过长

## sstable——物理结构

![Untitled](leveldb-Write%20sstable%20Compaction%2086a8351f5a614e15a9772def7d0fdaf0/Untitled.png)

- 每个文件中按照4KB对齐，拆分为若干个block，除了文件末尾的footer
    - 每个block除了data 还包括1字节压缩类型以及4字节crc校验码
    - 校验码默认使用snappy算法，CRC校验码是循环冗余校验校验码，校验范围包括数据以及压缩类型。
- 在block的data内部，最先记录 restart pointer 的个数n，然后就是n个restart pointer，在之后才是真正的entry
    - 可以细分为n个group，每个restart 指向一个group，group内每个key按照key-value存储
    - 具体的entry格式：shared【和restart pointer共享了多长的key前缀】＋unshared【没有共享的长度】＋value的长度＋【未共享的内容】＋【value的内容】
    - 根据共享的key前缀拼接上没有共享的 = Key
    - 为什么用共享key？因为key的有序，使得key有大量重复部分，减少空间占用，为什么使用restart point？一方面为了数据不会因为一个start pointer损坏而丢失 另一方面缩短临近共享前缀的读取距离 可以加速读取，直接比较restart point处的key
    - 当读到shared = 0 的时候。意味着开启了新的一个group，并且这里的unshared是完整的key长度

## sstable——逻辑结构

![Untitled](leveldb-Write%20sstable%20Compaction%2086a8351f5a614e15a9772def7d0fdaf0/Untitled%201.png)

- 每个sstable中，由五部分组成：1个footer、1个数据索引、n个数据、1个过滤器、1个过滤器索引
- handle = offset ＋ size

### data block 与 index block

- data block 是按照上述物理结构来存储所有属于这个ssstable的有效数据
    - 默认的重启点有16个，也就是拆分了16个group
    - Key =  `UserKey + SequenceNum + Type`  `Type := kDelete or kValue`
- index block 也是使用上述物理结构存储数据，不过 存储的key-value 则是 data block的索引
    - key【data block最后一条数据key】、value【指向这个data block的hanle，即如何找到这个data block】。这里的handle = offset ＋size
    - 这里的重启点也就一个，不考虑复用key的情况

### metaindex block 与 meta block【filter block】

![Untitled](leveldb-Write%20sstable%20Compaction%2086a8351f5a614e15a9772def7d0fdaf0/Untitled%202.png)

- metaindex block的物理结构与 data block 和 index block一样
    - key-value，key【filter block name】 value【handle】
- meta block：过滤器结构
    - base表示按照每2^(base)个字节创建一个filter
    - filter offset的offset表示，filter offsets 的位置，然后每个filter offset指向一个filter data

### footer结构

- 8字节 magic code +16字节index handle【8offset 8size】＋16字节metaindex block【8offset 8size】

# sstable build流程

## BlockBuilder

- 在sstable中的五种block，其中三种都是kv存储结构，使用同一个builder来进行构造
- 对于一个builder结构，记录着options，其中buffer是真正build之后的data block，还包括last key、count、finish等信息。并且一个uint32 的数组记录重启点

### 关键函数 Add

- 输入参数 一个 key 一个value，且要求finish没有完成、同时key一定顺序上大于之前任何一个已经add过的key，在实现中会有assert
- 首先判断这个group内的kv数有没有到达重启点之间的间隔数【默认16】
    - 如果没到达，计算当前key和重启点key的shared前缀长度，
    - 如果达到了，记录当前的restart点，并count清空
- 根据shared size 算出不相同部分的size，然后在buffer后追加shared size、nonshared size、value size，再append nonshared 部分的key 和value
- 更新last key、count

### 关键函数 Finish

- 先Reset一个builder信息之后，一直调用add函数，添加完后，调用finish 将 一个block的重启点和重启点个数 以及 crc 信息加入buffer
- 最后返回这个被buffer填充的slice

## filter block

- data block紧跟着就是filter block，其包括许多filter（bloom filter），每个data block对应一个filter，但是一个filter可能对应多个filter，查看sstable中是否含有key时，先检查filter，如果filter结果是没有，那一定没有，如果有，则可能有（bloom filter的结论）

### FilterBlockBuilder

- const 的filter policy，keys 已经过滤过的key内容，以及每个key的开始位置，filter data，以及filter offsets的记录

### FilterBlockBuilder:AddKey

- 记录当前kyes长度的位置，添加一个新的key

### FilterBlockBuilder：StartBlock

- 根据block的offset 创建filter，默认每2KB创建一个filter
- 每一个data block会调用一次start block 创建若干个filter，最后调用finish 完成一个meta block的创建

## TableBuilder

- 将内存中的immutable转储为sstable的实现，使用Table Builder
- 只有rep 一个struct来操作，包括要写的file、当前位置、状态、datablock的builder和index block的builder、filter block的builder等信息

### TableBuilder：WriteRawBlock

- 将一段block contents 写入file，首先记录这个block 的 handle 即offset和size
    - 先记录offset表示这个block开始的位置，最后将offset加上这块size+tail的五字节 修改记录
- 然后再原先的file之后append 这块block contents，如果append成功，追加上压缩type和crc校验值

### TableBuilder：WriteBlock

- 传入一个blockbuilder，先用这个builder，调用finish，得到raw block，根据压缩类型，对raw block进行压缩，然后调用WriteRawBlock写入文件
- 最后清空block builder

### TableBuilder：Add

- 接收 一个key和一个value，当前的rep不能已经close，要可以接收kv，写入文件，并且key大于last-key，要保证key是升序进入
- 如果当前有正在等待加入index block的index entry，需要先再将这个index entry写入index block
    - 使用last-key 与 handle，然后将等待变量置为false
- 并且在filter block中加入这个key，也在datablock 中加入这个kv，如果当前的data block大于了一个data block的大小，flush这个data block到文件中

### TableBuilder：Flush

- 将一个data block flush到磁盘文件中，调用Write Block 将这个data block写入文件，并且得到要写入index block的handle
- 把pending_index_entry置为true，并调用WritableFile::Flush把WritableFile内部的buffer写入磁盘(注意，不是Sync）
- 然后调用filter block builder的startBlock来创建filter

### TableBuilder：Finish

- 完成sstable的创建，此时最后data block被flush到file中，将rep 关闭
- 写入filter block，且不压缩，写入metaindex block，也就是filter block的索引
    - key = filter. filter policy name  value = handle
- 然后写入index block，每个index entry用来索引data block
- 最后写入 footer，完成一个sstable的创建

# Compaction流程

## Minor Compaction —— Immutable → Disk

### CompactMemTable

- 需要持有mutex锁，记录当前的version，并将其的引用计数自增，调用writelevel0，将immutable写入，将引用计数减一
- 更新version edit
- 调用RemoveObsoleteFiles删除废弃的文件(比如Memtable的预写日志文件），RemoveObsoleteFiles会检查数据库下所有的文件，删除废弃的文件。

### WriteLevel0Table

- 确认持有mutex锁，获取memtable的遍历迭代器，写入Log中
- 调用BuildTable完成Memtable到SST文件的转换，meta.number是文件要使用的number，最终生成的sst文件名为$number.ldb，比如000012.ldb。
- 调用PickLevelForMemTableOutput为新的SST文件挑选level，最低放到level 0，最高放到level 2。这里还会修改 VersionEdit，添加一个文件记录。包含文件的number、size、最大和最小key。
- 更新第level层的统计信息，level是刚才选出的level。

### BuildTable

- 创建一个file，然后遍历memtable，调用TableBuilder::Add把key-value写入文件。因为Memtable是有序的，所以第一个key和最后一个key分别赋值给TableBuilder的smallest和largest，即最小key和最大key。
- 调用TableBuilder::Finish完成最后的写入操作，至此整个SST文件的内容构造完成。
- 调用Sync把缓冲区数据强制写到磁盘上，至此Memtable完成落盘。
- 调用table_cache->NewIterator读取刚刚生成的sst文件，一方面检查写入是否OK，另一封面把新文件加入到leveldb的缓存中。

### PickLevelForMemTableOutput

- 根据这个sstable的最大和最小key，来选择落盘的level
- 首先检查，level0和这个区间是否有重叠key如果有，则直接加入level0
- 然后检查 level+1层是否有重叠，如果有，返回level，如果没有，再检查level+2层，是否有key范围重叠size 超过阈值，如果有，返回level，如果没有返回level+1
    - 即，在level升1层的条件是在level+1层没有重叠，并且key在level+2层的重叠不超过阈值

## Major Compaction —— Merge Sstables

- Compaction当中，inputs 的两个元素，指向需要整理的两层文件

### LevelDB的分层逻辑

- 每层的单个文件最大size相同，通过限制不同层的文件总size来限制每层的文件数量，L1 的最大size为1MB，往下每层扩大十倍
    - level0 层通过文件数量限制，，默认达到4个就进行合并，达到12个，就暂停Memtable落盘。L6层是最高层，L6层的size其实是没有限制的，因为就算超了，也不可能继续往上合并了。
    - 合并发生在相邻两层之间

### PickCompaction

- 当不是手动整理时，需要选择 Compaction的level 和文件等
- 首先 选择 要compaction的level及文件。第二步是选择level+1层的文件。第一步size_compaction和seek_compaction两种挑选机制，优先size_compaction，size_compaction未触发才会检查seek_compaction。
- 基于size_compaction时，首先在当前层遍历最新version的file，找到大于compact指针记录的最大key 的 下一个file。如果没有或者上次compaction的最大key已经是最后一个文件，那么选择第一个文件
- 基于seek_compaction时，直接从version信息中获取下一个要compaction的level和文件
- 选好了当前level和file，需要选择 level+1 的文件 调用 SetupOtherInputs
    - 调用getRange 获取input0层的最大key和最小key，然后再level+1层找到和[smallest，largest]范围有重叠的文件，放入input1，然后获取两层文件中的最大和最小key。这一步都使用filemetadata，不需要遍历文件
    - 然后统计level+2层 和 最大最小有重合的文件，放入compaction中。选择本层最大的key放入compaction pointer，更新version

### Background Compaction

- 从pick compaction得到compaction之后，判断整理的文件和下一层有没有重叠，如果没有重叠，从本层中删除文件，加入到下一层，修改log
- 如果有重叠，需要整理 调用Do Compaction Work，然后删除文件，改变leveldb状态

### DoCompactionWork

- 获取最旧的仍然在被使用的快照(snapshot)。快照，指的是版本号，即某个sequence。这里检查snapshots_列表，如果为空，就把smallest_snapshot置为最新的sequence，如果snapshots_不为空，就把smallest_snapshot置为最老的snapshot。smallest_snapshot在后面将被用于判断重复的user key是否可以删除。
- 获取一个所有input文件的迭代器，每次获取这些文件的最小key的记录，类似多路归并，每次只获取next，然后准备一些遍历开始Compaction
    - 如果imm_(即Immutable Memtable）不为空，就先把imm_落盘，这里和合并逻辑没有关系，因为落盘imm_和合并SST都是要加锁的，两者不能同时进行，所以优先落盘imm_，以免后台合并阻塞了imm_落盘，进而影响Memtable的写入。
    - ShouldStopBefore用于判断是否结束当前输出文件的，ShouldStopBefor比较当前文件已经插入的key和level+2层重叠的文件的总size是否超过阈值，如果超过，就结束文件。
        - 如果和leve2重叠的太多，会导致level1合并的时候，引入太多level2的文件
    - 首先解析当前的这个key，如果没有解析出来，也就不需要删除
    - 然后查看是否key的一条record是否要被删除，删除条件如下
        - 如果当前的key在之前出现过（说明相同的key但早出现的seq大于当前的key），且之前的seq已经比最小的snapshot_seq小了，这个key就可以丢弃
        - 或者 如果当前的key类型是 deletion，并且key的seq小于最小的snapshot，关键在于，更高的level里是否还有key的记录，如果有则不可以删除，因为删除意味着，下次找的时候会发现原本已经删除了数据，在高层还能找到，数据不一致
    - 如何需要丢失，就不管了，如果不能丢弃，查看当前是否有sstable的builder，如果没有则创建一个，如果有，将记录添加。当文件大小超过最大文件size时候，结束这个compaction的文件
    - 最后上锁，记录compaction的结果

# 发生Compaction的时候

- 首先，等write在写入一个write batch时，需要make room，如果memtable空间不足，需要变为immutable，并创建新的memtable
    - 此时immutable不为空，最后会调度后台compaction线程，传入一个后台执行的任务background call
    - 进入后台整理的流程，在此，如果immutable不是空，minor compaction把immutable 转储到磁盘上，然后return
    - 如果此时没有immutable，会尝试major compaction