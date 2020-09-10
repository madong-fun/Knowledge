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