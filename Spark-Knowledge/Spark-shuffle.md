# Spark-shuffle



## 概念

在理解Spark shuffle的过程前，先回顾一下MapReduce的shuffle过程：

在MapReduce框架中，Shuffle是连接Map和Reduce之间的桥梁，Map阶段的数据通过shuffle输出到对应的Reduce中，然后Reduce对数据执行计算。**在整个shuffle过程中，往往伴随这大量的磁盘和网络I/O，所以shuffle性能的高低也直接决定了整个程序的性能高低。**

![img](..\big-data\resouces\v2-7922486f9a5b271abe91e63f17cf3ca3_720w.jpg)

上面就是MapReduce shuffle流程，数据会被拉到不同的节点上进行聚合处理，会产生大量的磁盘和网络IO。

![img](..\big-data\resouces\v2-92a6e538e69e3ef92aca7278288ff541_720w.jpg)

上图是Spark DAG schedula的任务划分，从最后一个RDD往前追溯，遇到宽依赖就划分一个Stage，触发Shuffle操作。Shuffle 操作可能涉及的过程包括数据的排序，聚合，溢写，合并，传输，磁盘IO，网络的 IO 等等。Shuffle 是连接 MapTask 和 ReduceTask 之间的桥梁，Map 的输出到 Reduce 中须经过 Shuffle 环节，Shuffle 的性能高低直接影响了整个程序的性能和吞吐量。

通常 Shuffle 分为两部分：Map 阶段的数据准备( ShuffleMapTask )和Reduce(ShuffleReduceTask) 阶段的数据拷贝处理。一般将在 Map 端的 Shuffle 称之为 Shuffle Write，在 Reduce 端的 Shuffle 称之为 Shuffle Read。



## 未优化的HashShuffle

每一个 ShuffleMapTask 都会为每一个 ReducerTask 创建一个单独的文件，总的文件数是 M * R，其中 M 是 ShuffleMapTask 的数量，R 是 ShuffleReduceTask 的数量。

![img](..\big-data\resouces\hashShuffleManager1.jpg)



在处理大数据时，ShuffleMapTask 和 ShuffleReduceTask 的数量很多，创建的磁盘文件数量 M*R 也越多，大量的文件要写磁盘，再从磁盘读出来，不仅会占用大量的时间，而且每个磁盘文件记录的句柄都会保存在内存中（每个人大约 100k），因此也会占用很大的内存空间，频繁的打开和关闭文件，会导致频繁的GC操作，很容易出现 OOM 的情况。



## 优化后的HashShuffle

在 Spark 0.8.1 版本中，引入了 Consolidation 机制，该机制是对 HashShuffle 的一种优化。



![img](..\big-data\resouces\hashShuffle2.jpg)



可以明显看出，在一个 core 上连续执行的 ShuffleMapTasks 可以共用一个输出文件 ShuffleFile。

先执行完的 ShuffleMapTask 形成 ShuffleBlock i，后执行的 ShuffleMapTask 可以将输出数据直接追加到 ShuffleBlock i 后面，形成 ShuffleBlock i'，每个 ShuffleBlock 被称为 FileSegment。下一个 stage 的 reducer 只需要 fetch 整个 ShuffleFile 就行了。

这样，每个 worker 持有的文件数降为 cores * R。cores 代表核数，R 是 ShuffleReduceTask 数。

## Sort-Based Shuffle



由于 HashShuffle 会产生很多的磁盘文件，引入 Consolidation 机制虽然在一定程度上减少了磁盘文件数量，但是不足以有效提高 Shuffle 的性能，适合中小型数据规模的大数据处理。为了让 Spark 在更大规模的集群上更高性能处理更大规模的数据，因此在 Spark 1.1 版本中，引入了 SortShuffle。

![img](..\big-data\resouces\sortShuffleManager.jpg)



该机制每一个 ShuffleMapTask 都只创建一个文件，将所有的 ShuffleReduceTask 的输入都写入同一个文件，并且对应生成一个索引文件。

**以前的数据是放在内存缓存中，等到数据完了再刷到磁盘，现在为了减少内存的使用，在内存不够用的时候，可以将输出溢写到磁盘，结束的时候，再将这些不同的文件联合内存的数据一起进行归并，从而减少内存的使用量。一方面文件数量显著减少，另一方面减少Writer 缓存所占用的内存大小，而且同时避免 GC 的风险和频率。**

但对于 Rueducer 数比较少的情况，Hash Shuffle 要比 Sort Shuffle 快，因此 Sort Shuffle 有个 “fallback” 计划，对于 Reducers 数少于 “spark.shuffle.sort.bypassMergeThreshold” (200 by default)，将使用 fallback 计划，hashing 相关数据到分开的文件，然后合并这些文件为一个。



## shuffle过程

Shuffle 的整个生命周期由 ShuffleManager 来管理，Spark 2.3中，唯一的支持方式为 SortShuffleManager，SortShuffleManager 中定义了 writer 和 reader 对应shuffle 的 map 和 reduce 阶段。reader 只有一种实现 `BlockStoreShuffleReader`，writer 有三种运行实现：

- BypassMergeSortShuffleWriter：当前 shuffle 没有聚合， 并且分区数小于 `spark.shuffle.sort.bypassMergeThreshold`（默认200）
- UnsafeShuffleWriter：当条件不满足 BypassMergeSortShuffleWriter 时， 并且当前 rdd 的数据支持序列化（即 UnsafeRowSerializer），也不需要聚合， 分区数小于 2^24
- SortShuffleWriter：当以上条件都不满足时，选择SortShuffleWriter。

### BypassMergeSortShuffleWriter

BypassMergeSortShuffleWriter 的运行机制的触发条件如下：

- shuffle reduce task(即partition)数量小于`spark.shuffle.sort.bypassMergeThreshold` 参数的值。
- 没有map side aggregations。

note: map side aggregations是指在 map 端的聚合操作，通常来说一些聚合类的算子都会都 map 端的 aggregation。不过对于 groupByKey 和combineByKey， 如果设定 mapSideCombine 为false，就不会有 map side aggregations。

BypassMergeSortShuffleHandle 算法适用于没有聚合，数据量不大的场景。 给每个分区分配一个临时文件，对每个 record 的 key 使用分区器（模式是hash，如果用户自定义就使用自定义的分区器）找到对应分区的输出文件句柄，写入文件对应的文件。



因为写入磁盘文件是通过 Java的 BufferedOutputStream 实现的，BufferedOutputStream 是 Java 的缓冲输出流，**首先会将数据缓冲在内存中，当内存缓冲满溢之后再一次写入磁盘文件中**，这样可以减少磁盘 IO 次数，提升性能。最后，会将所有临时文件合并成一个磁盘文件，并创建一个索引文件标识下游各个 reduce task 的数据在文件中的 start offset与 end offset。

![img](..\big-data\resouces\BypassMergeSortShuffleWriter.jpg)



该过程的磁盘写机制其实跟未经优化的 HashShuffleManager 是一样的，也会创建很多的临时文件（所以触发条件中会有 reduce task 数量限制），只是在最后会做一个磁盘文件的合并，对于 shuffle reader 会更友好一些。



**BypassMergeSortShuffleWriter 所有的中间数据都是在磁盘里，并没有利用内存。而且它只保证分区索引的排序，而并不保证数据的排序。**



### SortShuffleWriter



该模式下，数据首先写入一个内存数据结构中，此时根据不同的 shuffle 算子，可能选用不同的数据结构。有些 shuffle 操作涉及到聚合，对于这种需要聚合的操作，使用 PartitionedAppendOnlyMap 来排序。对于不需要聚合的，则使用 PartitionedPairBuffer 排序。

**在进行 shuffle 之前，map 端会先将数据进行排序。**排序的规则，根据不同的场景，会分为两种。首先会根据 Key 将元素分成不同的 partition。**第一种只需要保证元素的 partitionId 排序，但不会保证同一个 partitionId 的内部排序。第二种是既保证元素的 partitionId 排序，也会保证同一个 partitionId 的内部排序**。

接着，往内存写入数据，每隔一段时间，当向 MemoryManager 申请不到足够的内存时，或者数据量超过 `spark.shuffle.spill.numElementsForceSpillThreshold` 这个阈值时 （默认是 Long 的最大值，不起作用），就会进行 Spill 内存数据到文件，然后清空内存数据结构。假设可以源源不断的申请到内存，那么 Write 阶段的所有数据将一直保存在内存中，由此可见，PartitionedAppendOnlyMap 或者 PartitionedPairBuffer 是比较吃内存的。

在溢写到磁盘文件之前，会先根据 key 对内存数据结构中已有的数据进行排序。排序过后，会分批将数据写入磁盘文件。默认的 batch 数量是 10000 条，也就是说，排序好的数据，会以每批 1 万条数据的形式分批写入磁盘文件。写入磁盘文件也是通过 Java 的 BufferedOutputStream 实现的。

一个 task 将所有数据写入内存数据结构的过程中，会发生多次磁盘溢写操作，也就会产生多个临时文件。在将最终排序结果写入到数据文件之前，需要将内存中的 PartitionedAppendOnlyMap 或者 PartitionedPairBuffer 和已经 spill 到磁盘的 SpillFiles 进行合并。

此外，由于一个 task 就只对应一个磁盘文件，也就意味着该 task 为下游 stage 的 task 准备的数据都在这一个文件中，因此还会单独写一份索引文件，其中标识了下游各个 task 的数据在文件中的 start offset 与 end offset。

### UnsafeShuffleWriter

触发条件有三个：

- Serializer 支持 relocation。Serializer 支持 relocation 是指，Serializer 可以对已经序列化的对象进行排序，这种排序起到的效果和先对数据排序再序列化一致。支持 relocation 的 Serializer 是 KryoSerializer，Spark 默认使用 JavaSerializer，通过参数 spark.serializer 设置
- 没有指定 aggregation 或者 key 排序， 因为 key 没有编码到排序指针中，所以只有 partition 级别的排序。
- partition 数量不能大于指定的阈值(2^24)，因为 partition number 使用24bit 表示的。

UnsafeShuffleWriter 首先将数据序列化，保存在 MemoryBlock 中。然后将该数据的地址和对应的分区索引，保存在 ShuffleInMemorySorter 内存中，利用ShuffleInMemorySorter 根据分区排序。当内存不足时，会触发 spill 操作，生成spill 文件。最后会将所有的 spill文 件合并在同一个文件里。

整个过程可以想象成归并排序。ShuffleExternalSorter 负责分片的读取数据到内存，然后利用 ShuffleInMemorySorter 进行排序。排序之后会将结果存储到磁盘文件中。这样就会有很多个已排序的文件， UnsafeShuffleWriter 会将所有的文件合并。

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\unsafeShuffleWriter)

UnsafeShuffleWriter 是对 SortShuffleWriter 的优化，大体上也和 SortShuffleWriter 差不多。从内存使用角度看，主要差异在以下两点：

- 一方面，在 SortShuffleWriter 的 PartitionedAppendOnlyMap 或者 PartitionedPairBuffer 中，存储的是键值或者值的具体类型，也就是 Java 对象，是反序列化过后的数据。而在 UnsafeShuffleWriter 的 ShuffleExternalSorter 中数据是序列化以后存储到实际的 Page 中，而且在写入数据过程中会额外写入长度信息。总体而言，序列化以后数据大小是远远小于序列化之前的数据。
- 另一方面，UnsafeShuffleWriter 中需要额外的存储记录（LongArray），它保存着分区信息和实际指向序列化后数据的指针（经过编码的Page num 以及 Offset）。相对于 SortShuffleWriter， UnsafeShuffleWriter 中这部分存储的开销是额外的

### Shuffle中的数据结构

SortShuffleWriter 中使用 ExternalSorter 来对内存中的数据进行排序，ExternalSorter 中缓存记录数据的数据结构有两种：**一种是Buffer，对应的实现类PartitionedPairBuffer，设置mapSideCombine=false 时会使用该结构；另一种是Map，对应的实现类是PartitionedAppendOnlyMap，设置mapSideCombine=true时会使用该结构。**

两者都是使用了 hash table 数据结构， 如果需要进行 aggregation， 就使用 PartitionedAppendOnlyMap（支持 lookup 某个Key，如果之前存储过相同 key 的 K-V 元素，就需要进行 aggregation，然后再存入aggregation 后的 K-V）， 否则使用 PartitionedPairBuffer（只进行添K-V 元素） 。

#### PartitionedPairBuffer

**设置 mapSideCombine=false 时** ，这种情况在 Map 阶段不进行 Combine 操作，在内存中缓存记录数据会使用 PartitionedPairBuffer 这种数据结构来缓存、排序记录数据，它是一个 Append-only Buffer，仅支持向 Buffer 中追加数据键值对记录，PartitionedPairBuffer 的结构如下图所示：

![img](..\big-data\resouces\partitionPairBuffer.jpg)

默认情况下，PartitionedPairBuffer 初始分配的存储容量为 capacity = initialCapacity = 64，实际上这个容量是针对key的容量，因为要存储的是键值对记录数据，所以实际存储键值对的容量为 2 * initialCapacity = 128。PartitionedPairBuffer 是一个能够动态扩充容量的 Buffer，内部使用一个一维数组来存储键值对，每次扩容结果为当前Buffer容量的 2 倍，即 2*capacity，最大支持存储 2^31-1个键值对记录（1073741823个） 。

PartitionedPairBuffer 存储的键值对记录数据，键是(partition, key)这样一个Tuple，值是对应的数据 value，而且 curSize 是用来跟踪写入 Buffer 中的记录的，key 在 Buffer 中的索引位置为 2 * curSize，value 的索引位置为 2 * curSize+1，可见一个键值对的 key 和 value 的存储在 PartitionedPairBuffer 内部的数组中是相邻的。

https://www.jianshu.com/p/286173f03a0b

https://www.cnblogs.com/itboys/p/9201750.html







