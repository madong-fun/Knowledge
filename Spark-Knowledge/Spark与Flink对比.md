



# Spark 与 Flink 对比

## 理念

**Flink** 是基于流的数据计算引擎，认为数据是无界的，批只是流的一种特殊情况。

**Spark**是基础批处理的数据计算引擎，数据是有界的，流处理看作是批处理的一种特殊形式,每次接收到一个时间间隔的数据才会去处理,所以天生很难在实时性上有所提升。

## Flink

### 编程模型

![img](../big-data/resouces/bounded-unbounded.png)



在Flink中，流也被分成两类：无界流和有界限，分别对应着Flink中的流处理场景和批处理场景。

无界流：有开始无结束的数据流；

有界流：有开始也有结束的数据流，批处理被抽象成有界流；

### 窗口类型

Flink中提供了三种类型的窗口：滚动窗口，滑动窗口和session窗口。

![Window assigners](..\big-data\resouces\window-assigners.svg)

### 时间语义

Flink提供了三种时间语义，分别是事件时间、注入时间和处理时间。

![img](..\big-data\resouces\v2-84490706dfccef0f48e71ef4947d385a_720w.jpg)

### watermark

Flink在事件时间应用程序中使用水印来判断时间。水印也是一种灵活的机制，以权衡结果的延迟和完整性。



## Spark

###   Spark-Streaming模型简介

![Spark Streaming](..\big-data\resouces\streaming-flow.png)

#### 编程模型

在Spark Streaming内部，将接收到数据流按照一定的时间间隔进行切分，然后交给Spark引擎处理，最终得到一个个微批的处理结果。

#### 数据抽象

![Spark Streaming](..\big-data\resouces\streaming-dstream.png)

离散数据流或者数据流是Spark Streaming提供的基本抽象。它可以是从数据源不断流入的，也可以是从一个数据流转换而来的。本质上就是一系列的RDD。每个流中的RDD包含了一个特定时间间隔内的数据集合，如上图所示。

#### 窗口操作

Spark Streaming提供了滑动窗口接口，滑动窗口的两个重要的参数是窗口大小，滑动步长。它允许在数据的滑动窗口上应用转换。如下图所示，每当窗口在源Dstream上滑动时，位于窗口内的源RDDs就会被合并操作，来生成窗口化的Dstream的RDDs。

![Spark Streaming](..\big-data\resouces\streaming-dstream-window.png)

### Structured Streaming



从Spark 2.0开始引入了Structured Streaming， 将微批次处理从高级 API 中解耦出去，简化了 API 的使用，API 不再负责进行微批次处理；开发者可以将流看成是一个没有边界的表，并基于这些“表”运行查询。 Structured Streaming的默认引擎基于微批处理引擎，并且可以达到最低100ms的延迟和数据处理的exactly-once保证。

从Spark 2.3开始，Structured Streaming继续向更快、更易用、更智能的目标迈进，引入了低延迟的持续流处理模式，这时候已经不再采用批处理引擎，而是一种类似Flink机制的持续处理引擎，可以达到端到端最低1ms的延迟和数据处理的at-least-once的保证

### 编程模型

![Stream as a Table](..\big-data\resouces\structured-streaming-stream-as-a-table.png)

Structured Streaming将数据流看作是一张无界表，每个流的数据源从逻辑上来说看做一个不断增长的动态表，从数据源不断流入的每个数据项可以看作为新的一行数据追加到动态表中。用户可以通过静态结构化数据的批处理查询方式（SQL查询），对数据进行实时查询。



### 触发类型

Structured Streaming通过不同的触发模式来实现不同的延迟级别和一致性语义。主要提供了以下四种触发模式：

单次触发：顾名思义就是只触发一次执行，类似于Flink的批处理；

周期性触发：查询以微批处理模式执行，微批执行将以用户指定的时间间隔来进行；

默认触发：一个批次执行结束立即执行下个批次；

连续处理：是Structured Streaming从2.3开始提出的新的模式，对标的就是Flink的流处理模式，该模式支持传入一个参数，传入参数为checkpoint间隔，也就是连续处理引擎每隔多久记录查询的进度；



### 写入模式

为了满足不同操作的结果需求，还提供了三种写入模式：

Complete：当trigger触发时，输出整个更新后的结果表到外部存储，存储连接器决定如何处理整个表的写入

Append：只有最后一次触发的追加到结果表中的数据行会被写入到外部存储，这只适用于已存在的数据项没有被更新的情况

Update：之后结果表中被更新的数据行会被写出到外部存储

### 窗口类型

在窗口类型方面，Structured Streaming继续支持滑动窗口，跟spark Streaming类似，但是Spark Streaming是基于处理时间语义的，Structured Streaming还可以基于事件时间语义进行处理。

### 时间语义

时间语义上，Structured Streaming也是根据当前的需要，支持了事件时间和处理时间，一步步向Flink靠近。

### watermark

在进行流处理的时候，不能无限保留中间状态结果，因此它也通过watermark来丢弃迟到数据。因为Flink和Structured Streaming都是支持事件时间语义，因此都支持watermark机制。

## 内存管理

### java对象存储

![](/data/workspace-github/Knowledge/Spark-Knowledge/resources/SouthEast.png)



在HotSpot虚拟机中，对象在内存中存储的布局可以分为3块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding），上图分别对应普通对象实例与数组对象实例的数据结构。

HotSpot虚拟机的对象头包括两部分信息：

1. Markword: 用于存储对象自身运行时数据，如hashcode，GC分代年龄，锁状态标志，线程持有的锁，偏向线程ID，偏向时间戳等。这部分数据的长度在32位和64位的虚拟机（未开启压缩指针）中分别为32bit和64bit。
2. klass：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例.
3. 数组长度：对象是一个数组, 那在对象头中还必须有一块数据用于记录数组长度.

**padding**：padding不是必须存在的，也没有特殊含义，仅仅是起到占位符的作用。由于HotSpot VM的自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说，就是对象的大小必须是8字节的整数倍。而对象头部分正好是8字节的倍数（1倍或者2倍），因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。

**对象大小计算**：

1. 在32位系统下，存放Class指针的空间大小是4字节，MarkWord是4字节，对象头为8字节。
2. 在64位系统下，存放Class指针的空间大小是8字节，MarkWord是8字节，对象头为16字节。



**所以**，基于JVM的大数据引擎（Hadoop、Spark、Flink等）需要将大量的数据加载至内存当中，这就不得不面对JVM存在的几个问题：

1. java对象的存储密度低。一个只包含 boolean 属性的对象占用了16个字节内存：对象头占了8个，boolean 属性占了1个，对齐填充占了7个。
2. Full GC会极大的影响性能，尤其是为了处理大数据而开了很大内存空间的JVM来说，GC会达到秒级甚至分钟级别。
3. OOM影响稳定性。OutOfMemoryError是分布式计算框架经常会遇到的问题，当JVM中所有对象大小超过分配给JVM的内存大小时，就会发生OutOfMemoryError错误，导致JVM崩溃，分布式框架的健壮性和性能都会受到影响。

所以目前，新生代的大数据框架都开始自己管理JVM内存了，如Spark的Tungsten，为的就是获得像 C 一样的性能以及避免 OOM 的发生。

### Spark内存管理



执行Spark程序时，Spark集群会启动Driver和Executor两种JVM进程，前者为主控进程，负责创建 Spark 上下文，提交 Spark 作业（Job），并将作业转化为计算任务（Task），在各个 Executor 进程间协调任务的调度。者负责在工作节点上执行具体的计算任务，并将结果返回给 Driver，同时为需要持久化的 RDD 提供存储功能。由于 Driver 的内存管理相对来说较为简单，我们主要对 Executor 的内存管理进行分析，下文中的 Spark 内存均特指 Executor 的内存。

#### 内存规划



![alt](./resources/soark-work-momery.png)



如果上图所示，Executor 的内存管理建立在 JVM 的内存管理之上，Spark 对 JVM 的堆内（On-heap）空间进行了更为详细的分配，以充分利用内存。同时，Spark 引入了堆外（Off-heap）内存，使之可以直接在工作节点的系统内存中开辟空间，进一步优化了内存的使用。

堆内内存的大小，由 Spark 应用程序启动时的 –executor-memory 或 spark.executor.memory 参数配置。Executor 内运行的并发任务共享 JVM 堆内内存，这些任务在缓存 RDD 数据和广播（Broadcast）数据时占用的内存被规划为存储（Storage）内存，而这些任务在执行 Shuffle 时占用的内存被规划为执行（Execution）内存，剩余的部分不做特殊规划，那些 Spark 内部的对象实例，或者用户定义的 Spark 应用程序中的对象实例，均占用剩余的空间.

为了进一步优化内存的使用以及提高 Shuffle 时排序的效率，Spark 引入了堆外（Off-heap）内存，使之可以直接在工作节点的系统内存中开辟空间，存储经过序列化的二进制数据。利用 JDK Unsafe API,Spark可以直接操作堆外内存,减少不必要的内存开销，以及频繁的GC扫描和回收，提升处理性能。

堆外内存通过 spark.memory.offHeap.enabled=true 参数启用，并由 spark.memory.offHeap.size 参数设定堆外空间的大小。



#### 内存空间分配

**spark内存分配-堆内**

![Spark内存分配-堆内](./resources/spark-memory-heap.png)



**spark内存分配-堆外**

![alt](/data/workspace-github/Knowledge/Spark-Knowledge/resources/spark-memory-offheap.png)



Spark在1.6 之后引入统一内存管理，相比于静态内存管理，统一内存管理存储内存和执行内存共享同一块空间，可以动态占用对方的空闲区域。

其中最重要的优化在于动态占用机制，其规则如下：

- 设定基本的存储内存和执行内存区域（spark.storage.storageFraction 参数），该设定确定了双方各自拥有的空间的范围
- 双方的空间都不足时，则存储到硬盘；若己方空间不足而对方空余时，可借用对方的空间;（存储空间不足是指不足以放下一个完整的 Block）
- 执行内存的空间被对方占用后，可让对方将占用的部分转存到硬盘，然后”归还”借用的空间
- 存储内存的空间被对方占用后，无法让对方”归还”，因为需要考虑 Shuffle 过程中的很多因素，实现起来较为复杂

















