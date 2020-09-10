## CheckPoint

Flink提供了Exactly once特性，是依赖于带有barrier的分布式快照+可部分重发的数据源功能实现的。而分布式快照中，就保存了operator的状态信息。

Flink的失败恢复依赖于 检查点机制 + 可部分重发的数据源。

检查点机制机制：checkpoint定期触发，产生快照，快照中记录了：
 - 当前检查点开始时数据源（例如Kafka）中消息的offset。
 - 记录了所有有状态的operator当前的状态信息（例如sum中的数值）。
 
** 可部分重发的数据源：** Flink选择最近完成的检查点K，然后系统重放整个分布式的数据流，然后给予每个operator他们在检查点k快照中的状态。数据源被设置为从位置Sk开始重新读取流。例如在Apache Kafka中，那意味着告诉消费者从偏移量Sk开始重新消费。

### Checkpoint 实现

Checkpoint是Flink实现容错机制最核心的功能，它能够根据配置周期性地基于Stream中各个Operator/task的状态来生成快照，从而将这些状态数据定期持久化存储下来，当Flink程序一旦意外崩溃时，重新运行程序时可以有选择地从这些快照进行恢复，从而修正因为故障带来的程序数据异常。快照的核心概念之一是barrier。 这些barrier被注入数据流并与记录一起作为数据流的一部分向下流动。 barriers永远不会超过记录，数据流严格有序，barrier将数据流中的记录隔离成一系列的记录集合，并将一些集合中的数据加入到当前的快照中，而另一些数据加入到下一个快照中。
每个barrier都带有快照的ID，并且barrier之前的记录都进入了该快照。 barriers不会中断流处理，非常轻量级。 来自不同快照的多个barrier可以同时在流中出现，这意味着多个快照可能并发地发生。

![IMAGE](resources/7FFBA879DA361FEDE4348433757CF2FA.jpg =633x236)

barrier在数据流源处被注入并行数据流中。快照n的barriers被插入的位置（记之为Sn）是快照所包含的数据在数据源中最大位置。例如，在Apache Kafka中，此位置将是分区中最后一条记录的偏移量。 将该位置Sn报告给checkpoint协调器（Flink的JobManager）。然后barriers向下游流动。当一个中间操作算子从其所有输入流中收到快照n的barriers时，它会为快照n发出barriers进入其所有输出流中。 一旦sink操作算子（流式DAG的末端）从其所有输入流接收到barriers n，它就向checkpoint协调器确认快照n完成。在所有sink确认快照后，意味快照着已完成。一旦完成快照n，job将永远不再向数据源请求Sn之前的记录，因为此时这些记录（及其后续记录）将已经通过整个数据流拓扑，也即是已经被处理结束。

**多流的barrier：**
![IMAGE](resources/F762287B983CAE207686E9CAA6D1FEFC.jpg =710x164)

接收多个输入流的运算符需要基于快照barriers上对齐(align)输入流。
 - 一旦操作算子从一个输入流接收到快照barriers n，它就不能处理来自该流的任何记录，直到它从其他输入接收到barriers n为止。 否则，它会搞混属于快照n的记录和属于快照n + 1的记录。
 - barriers n所属的流暂时会被搁置。 从这些流接收的记录不会被处理，而是放入输入缓冲区。可以看到1,2,3会一直放在Input buffer，直到另一个输入流的快照到达Operator。
 - 一旦从最后一个流接收到barriers n，操作算子就会发出所有挂起的向后传送的记录，然后自己发出快照n的barriers。
 - 之后，它恢复处理来自所有输入流的记录，在处理来自流的记录之前优先处理来自输入缓冲区的记录.
 

### state

state一般指一个具体的task/operator的状态。Flink中包含两种基础的状态：Keyed State和Operator State。

  Keyed State，就是基于KeyedStream上的状态。这个状态是跟特定的key绑定的，对KeyedStream流上的每一个key，可能都对应一个state。
  Operator State与Keyed State不同，Operator State跟一个特定operator的一个并发实例绑定，整个operator只对应一个state。相比较而言，在一个operator上，可能会有很多个key，从而对应多个keyed state。
  举例来说，Flink中的Kafka Connector，就使用了operator state。它会在每个connector实例中，保存该实例中消费topic的所有(partition, offset)映射。
  Keyed State和Operator State，可以以两种形式存在：原始状态和托管状态(Raw and Managed State)。
  托管状态是由Flink框架管理的状态，如ValueState, ListState, MapState等。而raw state即原始状态，由用户自行管理状态具体的数据结构，框架在做checkpoint的时候，使用byte[]来读写状态内容。通常在DataStream上的状态推荐使用托管的状态，当实现一个用户自定义的operator时，会使用到原始状态。

  这里重点说说State-Keyed State，基于key/value的状态接口，这些状态只能用于keyedStream之上。keyedStream上的operator操作可以包含window或者map等算子操作。这个状态是跟特定的key绑定的，对KeyedStream流上的每一个key，都对应一个state。
  
  key/value下可用的状态接口：
  - ValueState: 状态保存的是一个值，可以通过update()来更新，value()获取。
  - ListState: 状态保存的是一个列表，通过add()添加数据，通过get()方法返回一个Iterable来遍历状态值。
  - ReducingState: 这种状态通过用户传入的reduceFunction，每次调用add方法添加值的时候，会调用reduceFunction，最后合并到一个单一的状态值。
  - MapState：即状态值为一个map。用户通过put或putAll方法添加元素。
  
  以上所述的State对象，仅仅用于与状态进行交互（更新、删除、清空等），而真正的状态值，有可能是存在内存、磁盘、或者其他分布式存储系统中。实际上，这些状态有三种存储方式: HeapStateBackend、MemoryStateBackend、FsStateBackend、RockDBStateBackend。
  - MemoryStateBackend: state数据保存在java堆内存中，执行checkpoint的时候，会把state的快照数据保存到jobmanager的内存中。
  - FsStateBackend: state数据保存在taskmanager的内存中，执行checkpoint的时候，会把state的快照数据保存到配置的文件系统中，可以使用hdfs等分布式文件系统。
  - RocksDBStateBackend: RocksDB跟上面的都略有不同，它会在本地文件系统中维护状态，state会直接写入本地rocksdb中。同时RocksDB需要配置一个远端的filesystem。RocksDB克服了state受内存限制的缺点，同时又能够持久化到远端文件系统中，比较适合在生产中使用。
  
  通过创建一个StateDescriptor，可以得到一个包含特定名称的状态句柄，可以分别创建ValueStateDescriptor、 ListStateDescriptor或ReducingStateDescriptor状态句柄。状态是通过RuntimeContext来访问的，因此只能在RichFunction中访问状态。这就要求UDF时要继承Rich函数，例如RichMapFunction、RichFlatMapFunction等。

### checkpoint 设置

默认情况下，checkpoint不会被保留，取消程序时即会删除它们，但是可以通过配置保留定期检查点。开启Checkpoint功能，有两种方式。其一是在conf/flink_conf.yaml中做系统设置；其二是针对任务再代码里灵活配置。

```

//获取flink的运行环境
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

//设置statebackend
env.setStateBackend(new MemoryStateBackend());

CheckpointConfig config = env.getCheckpointConfig();

// 任务流取消和故障时会保留Checkpoint数据，以便根据实际需要恢复到指定的Checkpoint
config.enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

// 设置checkpoint的周期, 每隔1000 ms进行启动一个检查点
// 设置模式为exactly-once
env.enableCheckpointing(每隔1000,CheckpointingMode.EXACTLY_ONCE)

// 确保检查点之间有至少500 ms的间隔【checkpoint最小间隔】
config.setMinPauseBetweenCheckpoints(500);
// 检查点必须在一分钟内完成，或者被丢弃【checkpoint的超时时间】
config.setCheckpointTimeout(60000);
// 同一时间只允许进行一个检查点
config.setMaxConcurrentCheckpoints(1);

```

上面调用enableExternalizedCheckpoints设置为ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION，表示一旦Flink处理程序被cancel后，会保留Checkpoint数据，以便根据实际需要恢复到指定的Checkpoint处理。

- ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION： 取消作业时保留检查点。请注意，在这种情况下，您必须在取消后手动清理检查点状态。
- ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION： 取消作业时删除检查点。只有在作业失败时，检查点状态才可用。

默认情况下，如果设置了Checkpoint选项，则Flink只保留最近成功生成的1个Checkpoint，而当Flink程序失败时，可以从最近的这个Checkpoint来进行恢复。但是，如果希望保留多个Checkpoint，并能够根据实际需要选择其中一个进行恢复，这样会更加灵活，比如，发现最近4个小时数据记录处理有问题，希望将整个状态还原到4小时之前。
Flink可以支持保留多个Checkpoint，需要在Flink的配置文件conf/flink-conf.yaml中，添加如下配置，指定最多需要保存Checkpoint的个数：

```
state.checkpoints.num-retained: 20
```
Flink checkpoint目录分别对应的是 jobId，flink提供了在启动之时通过设置 -s 参数指定checkpoint目录, 让新的jobId 读取该checkpoint元文件信息和状态信息，从而达到指定时间节点启动job。

## SavePoint

Savepoint是通过Flink的检查点机制创建的流作业执行状态的一致图像。可以使用Savepoints来停止和恢复，分叉或更新Flink作业。保存点由两部分组成：稳定存储（例如HDFS，S3，…）上的（通常是大的）二进制文件和（相对较小的）元数据文件的目录。稳定存储上的文件表示作业执行状态图像的净数据。Savepoint的元数据文件以（绝对路径）的形式包含（主要）指向作为Savepoint一部分的稳定存储上的所有文件的指针。

从概念上讲，Flink的Savepoints与Checkpoints的不同之处在于备份与传统数据库系统中的恢复日志不同。检查点的主要目的是在意外的作业失败时提供恢复机制。

Checkpoint的生命周期由Flink管理，即Flink创建，拥有和发布Checkpoint，无需用户交互。作为一种恢复和定期触发的方法，Checkpoint实现的两个主要设计目标是：i）being as lightweight to create （轻量级），ii）fast restore （快速恢复）。针对这些目标的优化可以利用某些属性，例如，JobCode在执行尝试之间不会改变。

与此相反，Savepoints由用户创建，拥有和删除。它们的用例是planned (计划) 的，manual backup( 手动备份 ) 和 resume（恢复）。例如，这可能是Flink版本的更新，更改Job graph ，更改 parallelism ，分配第二个作业，如红色/蓝色部署，等等。当然，Savepoints必须在终止工作后继续存在。从概念上讲，保存点的生成和恢复成本可能更高，并且更多地关注可移植性和对前面提到的作业更改的支持。

为了能够在将来升级程序，主要的必要更改是通过uid(String)方法手动指定operator ID 。这些ID用于确定每个运算符的状态。

如果未手动指定ID，则会自动生成这些ID。只要这些ID不变，就可以从保存点自动恢复。生成的ID取决于程序的结构，并且对程序更改很敏感。因此，强烈建议手动分配这些ID。
触发保存点时，会创建一个新的保存点目录，其中将存储数据和元数据。可以通过配置默认目标目录或使用触发器命令指定自定义目标目录来控制此目录的位置。