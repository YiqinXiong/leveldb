# RocksDB笔记

## 总体结构

![levelDB结构](https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/levelDB%E7%BB%93%E6%9E%84.jpg)

1. **MemTable**：内存数据结构，具体实现是 SkipList。 接受用户的读写请求，新的数据会先在这里写入。
2. **Immutable MemTable**：当 MemTable 的大小达到设定的阈值后，会被转换成 Immutable MemTable，只接受读操作，不再接受写操作，然后由后台线程 flush 到磁盘上 —— 这个过程称为 minor compaction。
3. **Log**：数据写入 MemTable 之前会先写日志，用于防止宕机导致 MemTable 的数据丢失。一个日志文件对应到一个 MemTable。
4. **SSTable**：Sorted String Table。分为 level-0 到 level-n 多层，每一层包含多个 SSTable，文件内数据有序。<u>除了 level-0 之外，每一层内部的 SSTable 的 key 范围都不相交。</u>
5. **Manifest**：Manifest 文件中记录 SSTable 在不同 level 的信息，包括每一层有哪些 SSTable，每个 SSTable 的文件大小、最大 key、最小 key 等信息。
6. **Current**：重启时，LevelDB 会重新生成 Manifest，所以 Manifest 文件可能同时存在多个，Current 记录的是当前使用的 Manifest 文件名。
7. **TableCache**：TableCache 用于缓存 SSTable 的文件描述符、索引和 filter。
8. **BlockCache**：SSTable 的数据是被组织成一个个 block。BlockCache 用于缓存这些 block（解压后）的数据。



## MemTable

每个levelDB实例最多维护两个MemTable，即MemTable（mem\_）和Immutable MemTable（imm\_）。mem\_可读写，imm\_只读。

新写入数据进入mem\_，当mem\_超过阈值（**[write_buffer_size](https://github.com/google/leveldb/blob/1.22/include/leveldb/options.h#L82)**）时，mem\_切换为imm\_并生成新的mem\_。imm\_会由levelDB的后台线程经过compact变成SSTable存在磁盘上。

如果mem\_大小达到阈值将要切换为imm\_，而之前的imm\_还来不及完成compact过程，就会阻塞写请求（ **[stall write](https://github.com/google/leveldb/blob/1.22/db/db_impl.cc#L1336)**）

其支持的操作为：

- 单次插入：[Add](https://github.com/google/leveldb/blob/1.22/db/memtable.cc#L75)
- 单次查询：[Get](https://github.com/google/leveldb/blob/1.22/db/memtable.cc#L100)
- 范围查询：[MemTableIterator](https://github.com/google/leveldb/blob/1.22/db/memtable.cc#L46)

### MemTable的内部结构

MemTable中保存的单元为memkey，如下图所示：

<img src="https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/memkey.jpg" alt="memkey" style="zoom:67%;" />

memkey是一个由key和value拼接编码成的字符串，由以下四个部分组成：

- klength: 变长的 32 位整数（**[varint 的编码](https://link.zhihu.com/?target=https%3A//developers.google.com/protocol-buffers/docs/encoding%23varints)**），表示 internal key 的长度。
- internal key: 长度为 klength 的字符串。
- vlength: 变长的 32 位整数，表示 value 的长度。
- value: 长度为 vlength 的字符串。

MemTable内部对于memkey按internal key进行排序，排序规则为：

1. 按userkey排序
2. userkey相同时按sequence降序排序（sequence表明同一个key出现的先后顺序）
3. userkey和sequence相同时按type降序排序（理论上不会到达这一步，因为一个levelDB实例中sequence单调递增）

因此，在一个 MemTable 中相同userkey的多个版本，新的排在前面，旧的排在后面。

### MemTable的内存分配

![img](https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/rocksdb_mempool.png)

使用 [Arena类](https://github.com/google/leveldb/blob/1.22/util/arena.h#L16) 进行内存分配和使用统计。

每次申请 [kBlockSize](https://github.com/google/leveldb/blob/1.22/util/arena.cc#L9) 的内存，申请时再从这块内存中划分使用。

若申请时剩余内存不够，且申请的大小＞ kBlockSize / 4，则直接new一块内存；否则再申请 kBlockSize 的一块内存，并从这个 block 划分一部分内存。

### SkipList

> SkipList 是一种可以用于替代查找树的内存数据结构，其基本原理是在一个有序的链表之上增加一些索引，通过一个随机规则（保证一定概率）来“模拟”二分查找。

<img src="https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/SkipList.jpg" alt="SkipList" style="zoom:80%;" />

上图是SkipList的示意图，类似于有4条线路的地铁，节点看作地铁站，（1）为起点，NIL为终点，则可以通过换乘来更快到达目的地。坐车的过程也就是在SkipList里搜索的过程。

例如上图中要到达（9）站，可以先在（1）站坐2号线经过（4）到达（6），再在（6）换乘3号线到达（9）。这实际上就是在SkipList里搜索（9）号节点的过程。

#### 原理

<img src="https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/SkipList1.jpg" alt="SkipList1" style="zoom: 67%;" />

首先从基础链表开始，由于链表节点在内存中离散存在，因此无法二分查找。

<img src="https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/SkipList2.jpg" alt="SkipList2" style="zoom: 67%;" />

在基础链表上增加一层索引，就可以将查找量减半。

<img src="https://pic2.zhimg.com/v2-c2c468617c753a721baee588cf574171_r.jpg" alt="preview" style="zoom: 67%;" />

再递归建立索引，类似于构建搜索树。注意每次增加索引时**并不要求精准**地对搜索范围一分为二，只使用一个随机的概率规则来确定一个节点的高度。

#### 查找

查找的过程只需要从顶层开始查找，类似二分查找。

#### 插入

根据概率规则确定将要插入的节点的高度，然后在每一层插入一个该节点（或指针）。假设p=1/2，则新插入节点占1层的概率为1，占2层的概率为1/2，占3层的概率为1/4，以此类推。

#### 删除

从上到下删除该节点在每一层的指针，最后删除最下层的节点。每一层中删除的方法就是删除单链表中节点的经典方法。



## Log

> Log就是**WAL**（**Write Ahead Log**）文件。
>
> LevelDB的数据在写到MemTable之前会先将数据持久化到log文件中。

### Log的组成

<img src="https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/image-20201116152709725.png" alt="image-20201116152709725" style="zoom: 80%;" />

每个log文件包含若干个定长的block（由[kBlockSize](https://github.com/google/leveldb/blob/1.22/db/log_format.h#L27)决定），特例是文件尾部可能会有一个不足kBlockSize的部分块。

每个block里面可能有1个或几个record，特例：

1. 若block末尾不足7字节（小于下文中的header大小），则全填0x0；
2. 若block末尾刚好7字节，则填入一个长度为0（无data）的record。

每个record由下面的部分组成：

```go
record :=
  checksum: uint32     	// 校验和，小端
  length: uint16       	// 小端
  type: uint8          	// 类型包括FULL, FIRST, MIDDLE, LAST
  data: uint8[length]	// 实际数据
```

如上，一个record可以看作：

1. **7字节**长的**header**（checksum: uint32 + length: uint16 + type: uint8）
2. **length字节**长的**实际数据**（data: uint8[length]）

如果将上层写入的数据称为user record，由于 block 是定长的，而 user record 是变长的，一个 user record 有可能被截断成多个 record，保存到一段连续的 block 中。因此，在 header 中有一个 [type](https://github.com/google/leveldb/blob/1.22/db/log_format.h#L14) 字段用来表示record的类型。

record的类型有FULL, FIRST, MIDDLE, LAST：

```c++
enum RecordType {
  // Zero is reserved for preallocated files
  kZeroType = 0,

  kFullType = 1,

  // For fragments
  kFirstType = 2,
  kMiddleType = 3,
  kLastType = 4
};
```

1. **kFullType**： 一个完整的 user record；
2. **kFirstType**： 是 user record 的第一个 record；
3. **kMiddleType**： 是 user record 中间的 record。如果写入的数据比较大，kMiddleType 的 record 可能有多个；
4. **kLastType**： 是 user record 的最后一个 record。

### Log的例子

初始化整个 log 为空，假设我们有 3 个 user records：

1. **A** 大小为 1000 字节
2. **B** 大小为 97270 字节
3. **C** 大小为 8000 字节

**A** 小于 32KB，会被保存到第一个 block，长度为 1000，类型为 kFullType，占用空间为 7 + 1000 = 1007。

**B** 比较大，会被切分成 3 个分片，保存到不同的 block：

1. 第一个分片保存到第一个 block，长度为 31754 字节，类型为 kFirstType。因为保存 A 之后，这个 block 剩余的空间为 32768 - 7 - 1000 = 31761 字节。除去 header，可以保存 B 的前 31761 - 7 = 31754 字节。此时 B 还有 97270 - 31754 = 65516 字节需要保存；
2. 65516 字节超过了一个 block 的大小，所以第二个分片需要完整占用第二个 block，长度为 32768 - 7 = 32761 字节，类型为 kMiddleType。此时 B 还有 65516 - 32761 = 32755 字节需要保存；
3. B 的第三个分片保存到第三个 block ，长度为 32755，类型为 kLastType。第三个 block 剩余的空间为 32768 - 7 - 32755 = 6 字节。由于 6 字节小于一个 header 的大小（7 字节），会被进行 padding（填 0）。

**C** 会被保存到第四个 block，长度为 8000 字节，类型为 kFullType，占用空间 7 + 8000 = 8007。

综上，A、B、C 在 log 文件中的结构如下：

<img src="https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/log_block.jpg" alt="log_block" style="zoom: 50%;" />



## SSTable

> SSTable 全称 Sorted String Table，顾名思义，里面的 key-value 都是有序保存的。除了两个 MemTable，LevelDB 中的大部分数据是以 SSTable 的形式保存在外存上。

### 总体结构

1. SSTable提供了从键到值的持久，有序的映射，其中键和值都是任意字节字符串。 
2. 提供单点查询和范围查询操作。 
3. 内部结构：每个SSTable包含一系列块（通常每个块的大小为64KB，但这是可配置的）； 块索引（存储在SSTable的末尾）用于定位块； 当SSTable打开时，索引被加载到内存中。
4. 查找过程：我们首先通过在内存索引中执行二分查找来找到适当的块，然后从磁盘读取适当的块。 可选地，SSTable也可以完全映射到内存中，避免访问磁盘。

<img src="https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/image-20201117115326844.png" alt="image-20201117115326844" style="zoom:80%;" />

文件末尾的footer是定长的，其余block都是变长的。

- **footer**：大小为48B（8B的 [magic number](https://github.com/google/leveldb/blob/1.22/table/format.h#L77) 、两个20B的 [BlockHandle](https://github.com/google/leveldb/blob/1.22/table/format.h#L24) ）。两个 BlockHandle 分别为 index handle 和 meta index handle，index handle 指向 index block，meta index handle 指向 meta index block。
- **index block**：以 key-value 形式存放了 data block 的索引信息。key 是一个大于等于当前 data block 中最大的 key 且小于下一个 block 中最小的 key；value 就是对应的 data block 的 BlockHandle。
- **meta index block**：以 key-value 形式存放了 meta block 的索引信息。key 是 meta index 的名字（也就是 Filter 的名字）；value 就是对应的 data block 的 BlockHandle。
- **meta block**：当前LevelDB中仅有一个 meta block ，保存的是这个 SSTable 中的 key 组成的 bloom filter。RocksDB还有`[meta block 1: filter block]`、`[meta block 2: stats block]`、`[meta block 3: compression dictionary block]` 等 meta block。
- **data block**：直接存储有序key-value对，是 SSTfile 的数据实体。

### Block内部结构

>  [index block](https://github.com/google/leveldb/blob/1.22/table/table_builder.cc#L44)、[meta index block](https://github.com/google/leveldb/blob/1.22/table/table_builder.cc#L43)和[data block](https://github.com/google/leveldb/blob/1.22/table/table_builder.cc#L214) 都是通过 [BlockBuilder](https://github.com/google/leveldb/blob/1.22/table/block_builder.h#L18) 来生成，通过 [Block](https://github.com/google/leveldb/blob/1.22/table/block.h#L18) 来读取的。

~~不同的是，meta block（bloom filter）由 [FilterBlockBuilder](https://github.com/google/leveldb/blob/1.22/table/filter_block.h#L31) 来生成，通过 [FilterBlockReader](https://github.com/google/leveldb/blob/1.22/table/filter_block.h#L53) 来读取。下面主要讨论通过 BlockBuilder 来生成的三种 block。~~

#### 前缀压缩

最简单的方式，block 里面只需要将一个个 key-value 有序保存。但是为了节省空间，LevelDB 在 block 的内部实现了**[前缀压缩](https://github.com/google/leveldb/blob/1.22/table/block_builder.cc#L80-L84)**。

前缀压缩利用了 key 的有序性（前缀相同的有序 key 会聚集在一起）对 key 进行压缩，每个 key 与前一个 key 相同的前缀部分可以不用保存。读取的时候再根据规则进行解码即可。

LevelDB 将 block 的一个 key-value 称为一条 entry。每条 entry 的格式如下：

![v2-73468f410dcde74c3841070211c9dac8_720w](https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/v2-73468f410dcde74c3841070211c9dac8_720w.png)

- **shared_bytes**：和前一个 key 相同的前缀长度。
- **unshared_bytes**：和前一个 key不同的后缀部分的长度。
- **value_length**：value 数据的长度。
- **key_delta**：和前一个 key不同的后缀部分。
- **value**：value 数据。

一个 **data block** 的数据格式如下：

<img src="https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/image-20201117121517288.png" alt="image-20201117121517288" style="zoom:80%;" />

- **restarts[ ]**：在 LevelDB 中，默认**[每 16 个 key](https://github.com/google/leveldb/blob/1.22/include/leveldb/options.h#L105)** 就会重新计算前缀压缩，每次重新开始计算前缀压缩的第一个 key 称之为重启点（restart point）。restarts 数组记录了这个 block 中所有重启点的 offset。
- **num_restarts**：是 restarts 数组的长度。

#### Block中查找过程

在 block 中查找一个 key（[Block::Iter::Seek](https://github.com/google/leveldb/blob/1.22/table/block.cc#L163)）：

1. 先在 restarts 数组的基础上进行[二分查找](https://github.com/google/leveldb/blob/1.22/table/block.cc#L166-L189)，确定 restart point。
2. 从 restart point 开始[遍历查找](https://github.com/google/leveldb/blob/1.22/table/block.cc#L192-L200)。



## Manifest

> 为了避免进程崩溃或机器宕机导致的数据丢失，LevelDB 需要将元数据持久化到磁盘，承担这个任务的就是 Manifest 文件，每当有新的Version产生都需要更新 Manifest。
>
> 内容上，Manifest 文件保存了整个 LevelDB 实例的元数据，比如：每一层有哪些 SSTable。 
>
> 格式上，Manifest 文件其实就是一个 log 文件，一个 log record 就是一个 VersionEdit 。

### VersionEdit

用 [**VersionEdit**](https://github.com/google/leveldb/blob/1.22/db/version_edit.h#L29) 来表示这种相邻 Version 的差值。

用来表示一次 [metadata](https://github.com/google/leveldb/blob/1.22/db/version_edit.h#L88-L97)（元数据） 的变更。VersionEdit 可以使用成员函数 [**EncodeTo**](https://github.com/google/leveldb/blob/1.22/db/version_edit.cc#L41) 和 **[DecodeFrom](https://github.com/google/leveldb/blob/1.22/db/version_edit.cc#L107)** 进行序列化和反序列化。

### Version

RocksDB 的 [**Version**](https://github.com/google/leveldb/blob/1.22/db/version_set.h#L60) 表示一个版本的 metadata，其主要内容是 FileMetaData 指针的二维数组，分层记录了所有的SST文件信息。

**[FileMetaData](https://github.com/google/leveldb/blob/1.22/db/version_edit.h#L18-L27)** 数据结构用来维护一个文件的元信息，包括文件大小，文件编号，最大最小值，引用计数等信息，其中引用计数记录了被不同的Version引用的个数，保证被引用中的文件不会被删除。

<img src="https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/image-20201117141756080.png" style="zoom:80%;" />

Version 是 VersionEdit 进行 [apply](https://github.com/google/leveldb/blob/1.22/db/version_set.cc#L801) 之后得到的数据库状态——当前版本[包含哪些 SSTable](https://github.com/google/leveldb/blob/1.22/db/version_set.h#L154)，并通过[引用计数](https://github.com/google/leveldb/blob/1.22/db/version_set.h#L151)保证多线程并发访问的安全性。读操作要读取 SSTable 之前需要调用 **[Version::Ref](https://github.com/google/leveldb/blob/1.22/db/version_set.cc#L474)** 增加引用计数，不使用时需要调用 **[Version::UnRef](https://github.com/google/leveldb/blob/1.22/db/version_set.cc#L476)** 减少引用计数。

### VersionSet

![image-20201117145727829](https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/image-20201117145727829.png)



**[VersionSet](https://github.com/google/leveldb/blob/1.22/db/version_set.h#L167)** 结构如上图所示，它是一个 Version 构成的**[双向链表](https://github.com/google/leveldb/blob/1.22/db/version_set.h#L310)**，这些 Version 按**时间顺序先后产生**，记录了当时的元信息。

每生成一个 Version 就使用成员函数 **[AppendVersion](https://github.com/google/leveldb/blob/1.22/db/version_set.cc#L784)** 往这个链表插入一个节点，链表头指向最新的Version，同时维护了每个Version的引用计数，被引用中的Version不会被删除，其对应的SST文件也因此得以保留。



## Cache

> 根据功能的不同，分为两种 cache：
>
> 1. Block cache：缓存解压后的 data block，可以加快热数据的查询。
> 2. Table cache：缓存打开的 SSTable 文件描述符和对应的 index block、meta block 等信息。
>

### Block Cache

一个 SSTable 在被打开的时候，会通过 `options.block_cache` 的 **[NewId](https://github.com/google/leveldb/blob/1.22/util/cache.cc#L378)** 为其分配一个唯一的 **[cache_id](https://github.com/google/leveldb/blob/1.22/table/table.cc#L74)**。

每个 data block 保存到 block cache 的 **key** 为 **[cache_id + offset](https://github.com/google/leveldb/blob/1.22/table/table.cc#L172)**，**value** 就是该 data block ：

```c++
        s = ReadBlock(table->rep_->file, options, handle, &contents);
        if (s.ok()) {
          block = new Block(contents);
          if (contents.cachable && options.fill_cache) {
            cache_handle = block_cache->Insert(key, block, block->size(),
                                               &DeleteCachedBlock);
          }
        }
```

由于 SSTable 是只读的，block 的读取和 block cache 的维护非常简单，具体参考 **[Table::BlockReader](https://github.com/google/leveldb/blob/1.22/table/table.cc#L155)**。

### Table Cache

Table cache 的 **key** 是 **[SSTable 的 file_number](https://github.com/google/leveldb/blob/1.22/db/table_cache.cc#L45)**，**value** 是一个 **[TableAndFile ](https://github.com/google/leveldb/blob/1.22/db/table_cache.cc#L14)**对象：

```c++
        char buf[sizeof(file_number)];
        EncodeFixed64(buf, file_number);
        Slice key(buf, sizeof(buf));
        ······
        TableAndFile* tf = new TableAndFile;
        tf->file = file;
        tf->table = table;
        *handle = cache_->Insert(key, tf, 1, &DeleteEntry);
```

Table 内部封装了**filter**、 **index **和 **metaindex_handle** ，以及其他一些 SSTable 的元数据：

```c++
struct Table::Rep {
  ~Rep() {
    delete filter;
    delete[] filter_data;
    delete index_block;
  }

  Options options;
  Status status;
  RandomAccessFile* file;
  uint64_t cache_id;
  FilterBlockReader* filter;
  const char* filter_data;

  BlockHandle metaindex_handle;  // Handle to metaindex_block: saved from footer
  Block* index_block;
};
```

## Filter

> LevelDB（RocksDB） 可以设置通过 **[bloom filter](https://www.cnblogs.com/liyulong1982/p/6013002.html)** 来减少不必要的读 I/O 次数。

### 基本概念

![img](https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/2012071317402283.png)

可以通过一个Hash函数将一个元素映射成一个位阵列（Bit Array）中的一个点。这样一来，我们只要看看这个点是不是 1 就知道可以集合中有没有它了。这就是布隆过滤器的基本思想。

Hash面临的问题就是冲突。假设 Hash 函数是良好的，如果我们的位阵列长度为 m 个点，那么如果我们想将冲突率降低到例如 1%, 这个散列表就只能容纳 m/100 个元素。显然这就不叫空间有效了（Space-efficient）。可以使用多个 Hash解决，如果它们有一个说元素不在集合中，那肯定就不在。如果它们都说在，虽然也有一定可能性不在，但误判概率比较低。

### 用处

布隆过滤器可以对请求的key快速进行判断是否存在一个对应的value，若不存在就直接返回，避免了在磁盘中查找该key。

示意：

![img](https://xyq6785665.oss-cn-shenzhen.aliyuncs.com/img/2012071317513278.png)

假设 m 为 bitmap 的长度，n 是元素的总数，k 是哈希函数的个数，则平均每个 key 消耗的内存 bits_per_key = m / n。 对于给定的 bits_per_key，要使误识别率最低，则 k 的取值为 bits_per_key * ln2。 推导过程：[链接的 #False positives 概率推导这一章](https://www.cnblogs.com/liyulong1982/p/6013002.html)

误判只有False Positive（实际没有，但报告有），不会出现False Negative（实际有，但报告没有），也就是不会遗漏实际存在的key。

- 如果 key 存在，一定返回 true；
- 如果 key 不存在，大概率返回 false，小概率返回true。

过滤器与SSTable绑定，一个SSTable对应一个过滤器。



## 读操作流程

> 1. 点查询（Point Query）：读一个 key 的数据。
> 2. 范围查询（Range Query）：有序读一段 key 范围的数据。

### 点查询



### 范围查询

