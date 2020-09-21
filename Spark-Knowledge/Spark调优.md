## 数据倾斜

### 原因：

在Spark中，同一个Stage的不同Partition可以并行处理，而具有依赖关系的不同Stage之间是串行处理的。假设某个Spark Job分为Stage 0和Stage 1两个Stage，且Stage 1依赖于Stage 0，那Stage 0完全处理结束之前不会处理Stage 1。而Stage 0可能包含N个Task，这N个Task可以并行进行。如果其中N-1个Task都在10秒内完成，而另外一个Task却耗时1分钟，那该Stage的总时间至少为1分钟。换句话说，一个Stage所耗费的时间，主要由最慢的那个Task决定。

由于同一个Stage内的所有Task执行相同的计算，在排除不同计算节点计算能力差异的前提下，不同Task之间耗时的差异主要由该Task所处理的数据量决定。

### 来源：

在Spark中，数据倾斜主要分为两种：

- 从数据源直接读取。如读取HDFS，Kafka
- 读取上一个Stage的Shuffle数据

### 解决方法：

#### 避免数据源倾斜：

离线处理数据中，Spark读取数据基本来源于HDFS。HDFS中存储文件分为可切分文件(如：sequenceFile、text、orc等)和不可切分文件（Gzip、snappy压缩文件等）







### shuffle调优

大多数Spark作业的性能主要消耗在了shuffle环节上，因为该环节包含了大量的磁盘IO、序列化、网络数据传输等操作。影响一个Spark作业性能的因素，主要还是代码开发、资源参数以及数据倾斜，shuffle调优只能在整个Spark的性能调优中占到一小部分而已。



#### ShuffleManager

在Spark的源码中，负责shuffle过程的执行、计算和处理的组件主要就是ShuffleManager。而随着Spark的版本的发展，ShuffleManager也在不断迭代，变得越来越先进。

在Spark 1.2以前，默认的shuffle计算引擎是HashShuffleManager。该ShuffleManager而HashShuffleManager有着一个非常严重的弊端，就是会产生大量的中间磁盘文件，进而由大量的磁盘IO操作影响了性能。

#### HashShuffleManager

下图是HashShuffleManager未经优化前：

![preview](D:\workspace\workspace-github\Knowledge\big-data\resouces\hashShuffleManager1.jpg)



从上图可以看出，在一个stage结束计算之后，shuffle write就会生成每一个对应下游 reduce 的文件（按key分类），所以，每个执行shuffle write的task数量等于下一个stage的task的数量。比如下一个stage总共有100个task，那么当前stage的每个task都要创建100份磁盘文件。如果一个Executor有多个task的话，会创建大量的中间文件。

shuffle read，通常就是一个stage刚开始时要做的事情。此时该stage的每一个task就需要将上一个stage的计算结果中的所有相同key，从各个节点上通过网络都拉取到自己所在的节点上，然后进行key的聚合或连接等操作。

shuffle read的拉取过程是一边拉取一边进行聚合的。每个shuffle read task都会有一个自己的buffer缓冲，每次都只能拉取与buffer缓冲相同大小的数据，然后通过内存中的一个Map进行聚合等操作。聚合完一批数据后，再拉取下一批数据，并放到buffer缓冲中进行聚合操作。以此类推，直到最后将所有数据到拉取完，并得到最终的结果。



#### HashShuffleManager优化

优化后的HashShuffleManager如下图：

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\hashShuffleManager2.jpg)



在设置spark.shuffle.consolidateFiles=true后，在shuffle write过程中，task就不会为每一个下游stage的task创建一个磁盘文件了，此时会有一个shuffleFileGroup概念，每个shuffleFileGroup会对应一批磁盘文件，磁盘文件的数量与下游stage的task数量是相同的。一个Executor上有多少个CPU core，就可以并行执行多少个task。而第一批并行执行的每个task都会创建一个shuffleFileGroup，并将数据写入对应的磁盘文件内。

consolidate机制允许不同的task复用同一批磁盘文件，这样就可以有效将多个task的磁盘文件进行一定程度上的合并，从而大幅度减少磁盘文件的数量，进而提升shuffle write的性能。

经过优化之后，每个Executor创建的磁盘文件的数量的计算公式为：CPU core的数量 * 下一个stage的task数量。



### SortShuffleManager



SortShuffleManager的运行机制主要分成两种，一种是普通运行机制，另一种是bypass运行机制。当shuffle read task的数量小于等于spark.shuffle.sort.bypassMergeThreshold参数的值时（默认为200），就会启用bypass机制。

#### 普通运行机制

原理图如下：

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\sortShuffleManager.jpg)



在该模式下，数据会先写入一个内存数据结构中(默认5M)，此时根据不同的shuffle算子，可能选用不同的数据结构。如果是reduceByKey这种聚合类的shuffle算子，那么会选用Map数据结构，一边通过Map进行聚合，一边写入内存;如果是join这种普通的shuffle算子，那么会选用Array数据结构，直接写入内存。接着，每写一条数据进入内存数据结构之后，就会判断一下，是否达到了某个临界阈值。如果达到临界阈值的话，那么就会尝试将内存数据结构中的数据溢写到磁盘，然后清空内存数据结构。



在溢写到磁盘文件之前，会先根据key对内存数据结构中已有的数据进行排序。排序过后，会分批将数据写入磁盘文件。默认的batch数量是10000条，也就是说，排序好的数据，会以每批1万条数据的形式分批写入磁盘文件。写入磁盘文件是通过Java的BufferedOutputStream实现的。BufferedOutputStream是Java的缓冲输出流，首先会将数据缓冲在内存中，当内存缓冲满溢之后再一次写入磁盘文件中，这样可以减少磁盘IO次数，提升性能。

一个task将所有数据写入内存数据结构的过程中，会发生多次磁盘溢写操作，也就会产生多个临时文件。最后会将之前所有的临时磁盘文件都进行合并，这就是merge过程，此时会将之前所有临时磁盘文件中的数据读取出来，然后依次写入最终的磁盘文件之中。此外，由于一个task就只对应一个磁盘文件，也就意味着该task为下游stage的task准备的数据都在这一个文件中，因此还会单独写一份索引文件，其中标识了下游各个task的数据在文件中的start offset与end offset。

SortShuffleManager由于有一个磁盘文件merge的过程，因此大大减少了文件数量。比如第一个stage有50个task，总共有10个Executor，每个Executor执行5个task，而第二个stage有100个task。由于每个task最终只有一个磁盘文件，因此此时每个Executor上只有5个磁盘文件，所有Executor只有50个磁盘文件。

#### bypass运行机制

下图说明了bypass SortShuffleManager的原理。bypass运行机制的触发条件如下：

- shuffle map task数量小于spark.shuffle.sort.bypassMergeThreshold参数的值。
- 不是聚合类的shuffle算子（比如reduceByKey）。



![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\sortShuffleManager2.jpg)



此时task会为每个下游task都创建一个临时磁盘文件，并将数据按key进行hash然后根据key的hash值，将key写入对应的磁盘文件之中。当然，写入磁盘文件时也是先写入内存缓冲，缓冲写满之后再溢写到磁盘文件的。最后，同样会将所有临时磁盘文件都合并成一个磁盘文件，并创建一个单独的索引文件。

该过程的磁盘写机制其实跟未经优化的HashShuffleManager是一模一样的，因为都要创建数量惊人的磁盘文件，只是在最后会做一个磁盘文件的合并而已。因此少量的最终磁盘文件，也让该机制相对未经优化的HashShuffleManager来说，shuffle read的性能会更好。

而该机制与普通SortShuffleManager运行机制的不同在于：第一，磁盘写机制不同；第二，不会进行排序。也就是说，启用该机制的最大好处在于，shuffle write过程中，不需要进行数据的排序操作，也就节省掉了这部分的性能开销。



### shuffle相关参数调优

##### spark.shuffle.file.buffer

- 默认值：32k
- 参数说明：该参数用于设置shuffle write task的BufferedOutputStream的buffer缓冲大小。将数据写到磁盘文件之前，会先写入buffer缓冲中，待缓冲写满之后，才会溢写到磁盘。
- 调优建议：如果作业可用的内存资源较为充足的话，可以适当增加这个参数的大小（比如64k），从而减少shuffle write过程中溢写磁盘文件的次数，也就可以减少磁盘IO次数，进而提升性能。在实践中发现，合理调节该参数，性能会有1%~5%的提升。

##### spark.reducer.maxSizeInFlight

- 默认值：48m
- 参数说明：该参数用于设置shuffle read task的buffer缓冲大小，而这个buffer缓冲决定了每次能够拉取多少数据。
- 调优建议：如果作业可用的内存资源较为充足的话，可以适当增加这个参数的大小（比如96m），从而减少拉取数据的次数，也就可以减少网络传输的次数，进而提升性能。在实践中发现，合理调节该参数，性能会有1%~5%的提升。

##### spark.shuffle.io.maxRetries

- 默认值：3
- 参数说明：shuffle read task从shuffle write task所在节点拉取属于自己的数据时，如果因为网络异常导致拉取失败，是会自动进行重试的。该参数就代表了可以重试的最大次数。如果在指定次数之内拉取还是没有成功，就可能会导致作业执行失败。
- 调优建议：对于那些包含了特别耗时的shuffle操作的作业，建议增加重试最大次数（比如60次），以避免由于JVM的full gc或者网络不稳定等因素导致的数据拉取失败。在实践中发现，对于针对超大数据量（数十亿~上百亿）的shuffle过程，调节该参数可以大幅度提升稳定性。

##### spark.shuffle.io.retryWait

- 默认值：5s
- 参数说明：具体解释同上，该参数代表了每次重试拉取数据的等待间隔，默认是5s。
- 调优建议：建议加大间隔时长（比如60s），以增加shuffle操作的稳定性。

##### spark.shuffle.memoryFraction

- 默认值：0.2
- 参数说明：该参数代表了Executor内存中，分配给shuffle read task进行聚合操作的内存比例，默认是20%。
- 调优建议：在资源参数调优中讲解过这个参数。如果内存充足，而且很少使用持久化操作，建议调高这个比例，给shuffle read的聚合操作更多内存，以避免由于内存不足导致聚合过程中频繁读写磁盘。在实践中发现，合理调节该参数可以将性能提升10%左右。

##### spark.shuffle.manager

- 默认值：sort
- 参数说明：该参数用于设置ShuffleManager的类型。Spark 1.5以后，有三个可选项：hash、sort和tungsten-sort。HashShuffleManager是Spark 1.2以前的默认选项，但是Spark 1.2以及之后的版本默认都是SortShuffleManager了。tungsten-sort与sort类似，但是使用了tungsten计划中的堆外内存管理机制，内存使用效率更高。
- 调优建议：由于SortShuffleManager默认会对数据进行排序，因此如果你的业务逻辑中需要该排序机制的话，则使用默认的SortShuffleManager就可以；而如果你的业务逻辑不需要对数据进行排序，那么建议参考后面的几个参数调优，通过bypass机制或优化的HashShuffleManager来避免排序操作，同时提供较好的磁盘读写性能。这里要注意的是，tungsten-sort要慎用，因为之前发现了一些相应的bug。

##### spark.shuffle.sort.bypassMergeThreshold

- 默认值：200
- 参数说明：当ShuffleManager为SortShuffleManager时，如果shuffle read task的数量小于这个阈值（默认是200），则shuffle write过程中不会进行排序操作，而是直接按照未经优化的HashShuffleManager的方式去写数据，但是最后会将每个task产生的所有临时磁盘文件都合并成一个文件，并会创建单独的索引文件。
- 调优建议：当你使用SortShuffleManager时，如果的确不需要排序操作，那么建议将这个参数调大一些，大于shuffle read task的数量。那么此时就会自动启用bypass机制，map-side就不会进行排序了，减少了排序的性能开销。但是这种方式下，依然会产生大量的磁盘文件，因此shuffle write性能有待提高。

##### spark.shuffle.consolidateFiles

### spark.shuffle.consolidateFiles

- 默认值：false
- 参数说明：如果使用HashShuffleManager，该参数有效。如果设置为true，那么就会开启consolidate机制，会大幅度合并shuffle write的输出文件，对于shuffle read task数量特别多的情况下，这种方法可以极大地减少磁盘IO开销，提升性能。
- 调优建议：如果的确不需要SortShuffleManager的排序机制，那么除了使用bypass机制，还可以尝试将spark.shffle.manager参数手动指定为hash，使用HashShuffleManager，同时开启consolidate机制。在实践中尝试过，发现其性能比开启了bypass机制的SortShuffleManager要高出10%~30%。







```


set hive.exec.dynamic.partition=true; ##--动态分区
set hive.exec.dynamic.partition.mode=nonstrict; ##--动态分区
set hive.auto.convert.join=true; ##-- 自动判断大表和小表

##-- hive并行
set hive.exec.parallel=true;
set hive.merge.mapredfiles=true;

##-- 内存能力
set spark.driver.memory=8G; 
set spark.executor.memory=2G; 

##-- 并发度
set spark.dynamicAllocation.enabled=true;
set spark.dynamicAllocation.maxExecutors=50;
set spark.executor.cores=2;

##-- shuffle
set spark.sql.shuffle.partitions=100; -- 默认的partition数，及shuffle的reader数
set spark.sql.adaptive.enabled=true; -- 启用自动设置 Shuffle Reducer 的特性，动态设置Shuffle Reducer个数（Adaptive Execution 的自动设置 Reducer 是由 ExchangeCoordinator 根据 Shuffle Write 统计信息决定）
set spark.sql.adaptive.join.enabled=true; -- 开启 Adaptive Execution 的动态调整 Join 功能 (根据前面stage的shuffle write信息操作来动态调整是使用sortMergeJoin还是broadcastJoin)
set spark.sql.adaptiveBroadcastJoinThreshold=268435456; -- 64M ,设置 SortMergeJoin 转 BroadcastJoin 的阈值，默认与spark.sql.autoBroadcastJoinThreshold相同
set spark.sql.adaptive.shuffle.targetPostShuffleInputSize=134217728; -- shuffle时每个reducer读取的数据量大小，Adaptive Execution就是根据这个值动态设置Shuffle reader的数量
set spark.sql.adaptive.allowAdditionalShuffle=true; -- 是否允许为了优化 Join 而增加 Shuffle,默认为false
set spark.shuffle.service.enabled=true; 


##-- orc
set spark.sql.orc.filterPushdown=true;
set spark.sql.orc.splits.include.file.footer=true;
set spark.sql.orc.cache.stripe.details.size=10000;
set hive.exec.orc.split.strategy=ETL -- ETL：会切分文件,多个stripe组成一个split，BI：按文件进行切分，HYBRID：平均文件大小大于hadoop最大split值使用ETL,否则BI
set spark.hadoop.mapreduce.input.fileinputformat.split.maxsize=134217728; -- 128M 读ORC时，设置一个split的最大值，超出后会进行文件切分
set spark.hadoop.mapreduce.input.fileinputformat.split.minsize=67108864; -- 64M 读ORC时，设置小文件合并的阈值

##-- 其他
set spark.sql.hive.metastorePartitionPruning=true;

##-- 广播表
set spark.sql.autoBroadcastJoinThreshold=268435456; -- 256M

##-- 小文件
set spark.sql.mergeSmallFileSize=10485760; -- 10M -- 小文件合并的阈值
set spark.hadoopRDD.targetBytesInPartition=67108864; -- 64M 设置stage 输入端的map（不涉及shuffle的时候）合并文件大小
set spark.sql.targetBytesInPartitionWhenMerge=67108864; --64M 设置额外的（非最开始）合并job的map端输入size


```

