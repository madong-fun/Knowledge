1. spark.sql.shuffle.partitions and spark.default.parallelism
   `spark.sql.shuffle.partitions configures the number of partitions that are used when shuffling data for joins or aggregations`
    `spark.default.parallelism, which determines the 'default number of partitions in RDDs returned by transformations like join, reduceByKey, and parallelize when not set by user', however this seems to be ignored by Spark SQL and only relevant when working on plain RDDs.`
```
sqlContext.setConf("spark.sql.shuffle.partitions", "300")
sqlContext.setConf("spark.default.parallelism", "300")
```
```
spark-submit --conf spark.sql.shuffle.partitions=300 --conf spark.default.parallelism=300
```

## 参数
1） executor_cores*num_executors 不宜太小或太大！一般不超过总队列 cores 的 25%，比如队列总 cores 400，最大不要超过100，最小不建议低于 40，除非日志量很小。

2） executor_cores 不宜为1！否则 work 进程中线程数过少，一般 2~4 为宜。

3） executor_memory 一般 6~10g 为宜，最大不超过 20G，否则会导致 GC 代价过高，或资源浪费严重。

4） spark_parallelism 一般为 executor_cores*num_executors 的 1~4 倍，系统默认值 64，不设置的话会导致 task 很多的时候被分批串行执行，或大量 cores 空闲，资源浪费严重。

5） driver-memory 早前有同学设置 20G，其实 driver 不做任何计算和存储，只是下发任务与yarn资源管理器和task交互，除非你是 spark-shell，否则一般 1-2g 就够了。

Spark Memory Manager：

6）spark.shuffle.memoryFraction(默认 0.2) ，也叫 ExecutionMemory。这片内存区域是为了解决 shuffles,joins, sorts and aggregations 过程中为了避免频繁IO需要的buffer。如果你的程序有大量这类操作可以适当调高。

7）spark.storage.memoryFraction(默认0.6)，也叫 StorageMemory。这片内存区域是为了解决 block cache(就是你显示调用dd.cache, rdd.persist等方法), 还有就是broadcasts,以及task results的存储。可以通过参数，如果你大量调用了持久化操作或广播变量，那可以适当调高它。

8）OtherMemory，给系统预留的，因为程序本身运行也是需要内存的， (默认为0.2)。Other memory在1.6也做了调整，保证至少有300m可用。你也可以手动设置 spark.testing.reservedMemory . 然后把实际可用内存减去这个reservedMemory得到 usableMemory。 ExecutionMemory 和 StorageMemory 会共享usableMemory * 0.75的内存。0.75可以通过 新参数 spark.memory.fraction 设置。目前spark.memory.storageFraction 默认值是0.5,所以ExecutionMemory，StorageMemory默认情况是均分上面提到的可用内存的。


## 读取ORC 配置
```
–conf spark.sql.orc.impl=native
–conf spark.sql.orc.enableVectorizedReader=true
–conf spark.sql.hive.convertMetastoreOrc=true

```


## Adaptive Execution
开启与调优该特性的方法如下：

将spark.sql.adaptive.skewedJoin.enabled设置为 true 即可自动处理 Join 时数据倾斜；

spark.sql.adaptive.skewedPartitionMaxSplits控制处理一个倾斜 Partition 的 Task 个数上限，默认值为 5；

spark.sql.adaptive.skewedPartitionRowCountThreshold设置了一个 Partition 被视为倾斜 Partition 的行数下限，也即行数低于该值的 Partition 不会被当作倾斜 Partition 处理。其默认值为 10L * 1000 * 1000 即一千万；

spark.sql.adaptive.skewedPartitionSizeThreshold设置了一个 Partition 被视为倾斜 Partition 的大小下限，也即大小小于该值的 Partition 不会被视作倾斜 Partition。其默认值为 64 * 1024 * 1024 也即 64MB；

spark.sql.adaptive.skewedPartitionFactor该参数设置了倾斜因子。如果一个 Partition 的大小大于spark.sql.adaptive.skewedPartitionSizeThreshold的同时大于各 Partition 大小中位数与该因子的乘积，或者行数大于spark.sql.adaptive.skewedPartitionRowCountThreshold的同时大于各 Partition 行数中位数与该因子的乘积，则它会被视为倾斜的 Partition。