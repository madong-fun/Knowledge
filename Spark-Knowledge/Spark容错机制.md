## 前言
   一般来说，分布式数据集的容错性有两种方式：**数据检查点**和**记录数据的更新**。 
面向大规模数据分析，数据检查点操作成本很高，需要通过数据中心的网络连接在机器之间复制庞大的数据集，而网络带宽往往比内存带宽低得多，同时还需要消耗更多的存储资源。 
因此，Spark选择记录更新的方式。但是，如果更新粒度太细太多，那么记录更新成本也不低。因此，RDD只支持粗粒度转换，即只记录单个块上执行的单个操作，然后将创建RDD的一系列变换序列（每个RDD都包含了他是如何由其他RDD变换过来的以及如何重建某一块数据的信息。因此RDD的容错机制又称“血统(Lineage)”容错）记录下来，以便恢复丢失的分区。 
Lineage本质上很类似于数据库中的重做日志（Redo Log），只不过这个重做日志粒度很大，是对全局数据做同样的重做进而恢复数据。

## Lineage机制
### Lineage简介
   相比其他系统的细颗粒度的内存数据更新级别的备份或者LOG机制，RDD的Lineage记录的是粗颗粒度的特定数据Transformation操作（如filter、map、join等）行为。当这个RDD的部分分区数据丢失时，它可以通过Lineage获取足够的信息来重新运算和恢复丢失的数据分区。因为这种粗颗粒的数据模型，限制了Spark的运用场合，所以Spark并不适用于所有高性能要求的场景，但同时相比细颗粒度的数据模型，也带来了性能的提升。
   
### 两种依赖关系
   RDD在Lineage依赖方面分为两种：窄依赖(Narrow Dependencies)与宽依赖(Wide Dependencies,源码中称为Shuffle Dependencies)，用来解决数据容错的高效性。
   - 窄依赖是指父RDD的每一个分区最多被一个子RDD的分区所用，表现为一个父RDD的分区对应于一个子RDD的分区 
或多个父RDD的分区对应于一个子RDD的分区，也就是说一个父RDD的一个分区不可能对应一个子RDD的多个分区。 
1个父RDD分区对应1个子RDD分区，这其中又分两种情况：1个子RDD分区对应1个父RDD分区（如map、filter等算子），1个子RDD分区对应N个父RDD分区（如co-paritioned（协同划分）过的Join）。
   - 宽依赖是指子RDD的分区依赖于父RDD的多个分区或所有分区，即存在一个父RDD的一个分区对应一个子RDD的多个分区。 
1个父RDD分区对应多个子RDD分区，这其中又分两种情况：1个父RDD对应所有子RDD分区（未经协同划分的Join）或者1个父RDD对应非全部的多个RDD分区（如groupByKey）。


根据父RDD分区是对应1个还是多个子RDD分区来区分窄依赖（父分区对应一个子分区）和宽依赖（父分区对应多个子分 
区）。如果对应多个，则当容错重算分区时，因为父分区数据只有一部分是需要重算子分区的，其余数据重算就造成了冗余计算。

对于宽依赖，Stage计算的输入和输出在不同的节点上，对于输入节点完好，而输出节点死机的情况，通过重新计算恢复数据这种情况下，这种方法容错是有效的，否则无效，因为无法重试，需要向上追溯其祖先看是否可以重试（这就是lineage，血统的意思），窄依赖对于数据的重算开销要远小于宽依赖的数据重算开销。

窄依赖和宽依赖的概念主要用在两个地方：一个是容错中相当于Redo日志的功能；另一个是在调度中构建DAG作为不同Stage的划分点。

### 依赖关系的特性
第一，窄依赖可以在某个计算节点上直接通过计算父RDD的某块数据计算得到子RDD对应的某块数据；宽依赖则要等到父RDD所有数据都计算完成之后，并且父RDD的计算结果进行hash并传到对应节点上之后才能计算子RDD。 
第二，数据丢失时，对于窄依赖只需要重新计算丢失的那一块数据来恢复；对于宽依赖则要将祖先RDD中的所有数据块全部重新计算来恢复。所以在长“血统”链特别是有宽依赖的时候，需要在适当的时机设置数据检查点。也是这两个特性要求对于不同依赖关系要采取不同的任务调度机制和容错恢复机制。

### 容错原理
在容错机制中，如果一个节点死机了，而且运算窄依赖，则只要把丢失的父RDD分区重算即可，不依赖于其他节点。而宽依赖需要父RDD的所有分区都存在，重算就很昂贵了。可以这样理解开销的经济与否：在窄依赖中，在子RDD的分区丢失、重算父RDD分区时，父RDD相应分区的所有数据都是子RDD分区的数据，并不存在冗余计算。在宽依赖情况下，丢失一个子RDD分区重算的每个父RDD的每个分区的所有数据并不是都给丢失的子RDD分区用的，会有一部分数据相当于对应的是未丢失的子RDD分区中需要的数据，这样就会产生冗余计算开销，这也是宽依赖开销更大的原因。因此如果使用Checkpoint算子来做检查点，不仅要考虑Lineage是否足够长，也要考虑是否有宽依赖，对宽依赖加Checkpoint是最物有所值的。

## Checkpoint机制
通过上述分析可以看出在以下两种情况下，RDD需要加检查点。

DAG中的Lineage过长，如果重算，则开销太大（如在PageRank中）。
在宽依赖上做Checkpoint获得的收益更大。
由于RDD是只读的，所以Spark的RDD计算中一致性不是主要关心的内容，内存相对容易管理，这也是设计者很有远见的地方，这样减少了框架的复杂性，提升了性能和可扩展性，为以后上层框架的丰富奠定了强有力的基础。 
在RDD计算中，通过检查点机制进行容错，传统做检查点有两种方式：通过冗余数据和日志记录更新操作。在RDD中的doCheckPoint方法相当于通过冗余数据来缓存数据，而之前介绍的血统就是通过相当粗粒度的记录更新操作来实现容错的。

**检查点（本质是通过将RDD写入Disk做检查点）是为了通过lineage做容错的辅助，lineage过长会造成容错成本过高，这样就不如在中间阶段做检查点容错，如果之后有节点出现问题而丢失分区，从做检查点的RDD开始重做Lineage，就会减少开销。**

一个 Streaming Application 往往需要7*24不间断的跑，所以需要有抵御意外的能力（比如机器或者系统挂掉，JVM crash等）。为了让这成为可能，Spark Streaming需要 checkpoint 足够多信息至一个具有容错设计的存储系统才能让 Application 从失败中恢复。Spark Streaming 会 checkpoint 两种类型的数据。
 - Metadata（元数据） checkpointing - 保存定义了 Streaming 计算逻辑至类似 HDFS 的支持容错的存储系统。用来恢复 driver，元数据包括：
   - 配置 - 用于创建该 streaming application 的所有配置
   - DStream 操作 - DStream 一些列的操作
   - 未完成的 batches - 那些提交了 job 但尚未执行或未完成的 batches
 -  Data checkpointing - 保存已生成的RDDs至可靠的存储。这在某些 stateful 转换中是需要的，在这种转换中，生成 RDD 需要依赖前面的 batches，会导致依赖链随着时间而变长。为了避免这种没有尽头的变长，要定期将中间生成的 RDDs 保存到可靠存储来切断依赖链。
 具体来说，metadata checkpointing主要还是从drvier失败中恢复，而Data Checkpoing用于对有状态的transformation操作进行checkpointing

### 什么时候需要启用 checkpoint
- 使用了 stateful 转换 - 如果 application 中使用了updateStateByKey或reduceByKeyAndWindow等 stateful 操作，必须提供 checkpoint 目录来允许定时的 RDD checkpoint
- 希望能从意外中恢复 driver
### checkpoint 时机
   在 Spark Streaming 中，JobGenerator 用于生成每个 batch 对应的 jobs，它有一个定时器，定时器的周期即初始化 StreamingContext 时设置的 batchDuration。这个周期一到，JobGenerator 将调用generateJobs方法来生成并提交 jobs，这之后调用 doCheckpoint 方法来进行 checkpoint。doCheckpoint 方法中，会判断当前时间与 streaming application start 的时间之差是否是 checkpoint duration 的倍数，只有在是的情况下才进行 checkpoint。
### Checkpoint 的局限
Spark Streaming 的 checkpoint 机制看起来很美好，却有一个硬伤。上文提到最终刷到外部存储的是类 Checkpoint 对象序列化后的数据。那么在 Spark Streaming application 重新编译后，再去反序列化 checkpoint 数据就会失败。这个时候就必须新建 StreamingContext。
