# Spark 与 Flink 对比

## 理念

**Flink** 是基于流的数据计算引擎，认为数据是无界的，批只是流的一种特殊情况。

**Spark**是基础批处理的数据计算引擎，数据是有界的，流处理看作是批处理的一种特殊形式,每次接收到一个时间间隔的数据才会去处理,所以天生很难在实时性上有所提升。

## Flink

### 编程模型

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\bounded-unbounded.png)



在Flink中，流也被分成两类：无界流和有界限，分别对应着Flink中的流处理场景和批处理场景。

无界流：有开始无结束的数据流；

有界流：有开始也有结束的数据流，批处理被抽象成有界流；

### 窗口类型

Flink中提供了三种类型的窗口：滚动窗口，滑动窗口和session窗口。

![Window assigners](D:\workspace\workspace-github\Knowledge\big-data\resouces\window-assigners.svg)

### 时间语义

Flink提供了三种时间语义，分别是事件时间、注入时间和处理时间。

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\v2-84490706dfccef0f48e71ef4947d385a_720w.jpg)

### watermark

Flink在事件时间应用程序中使用水印来判断时间。水印也是一种灵活的机制，以权衡结果的延迟和完整性。



### 内存模型

### 序列化

## Spark

###   Spark-Streaming模型简介

![Spark Streaming](D:\workspace\workspace-github\Knowledge\big-data\resouces\streaming-flow.png)

#### 编程模型

在Spark Streaming内部，将接收到数据流按照一定的时间间隔进行切分，然后交给Spark引擎处理，最终得到一个个微批的处理结果。

#### 数据抽象

![Spark Streaming](D:\workspace\workspace-github\Knowledge\big-data\resouces\streaming-dstream.png)

离散数据流或者数据流是Spark Streaming提供的基本抽象。它可以是从数据源不断流入的，也可以是从一个数据流转换而来的。本质上就是一系列的RDD。每个流中的RDD包含了一个特定时间间隔内的数据集合，如上图所示。

#### 窗口操作

Spark Streaming提供了滑动窗口接口，滑动窗口的两个重要的参数是窗口大小，滑动步长。它允许在数据的滑动窗口上应用转换。如下图所示，每当窗口在源Dstream上滑动时，位于窗口内的源RDDs就会被合并操作，来生成窗口化的Dstream的RDDs。

![Spark Streaming](D:\workspace\workspace-github\Knowledge\big-data\resouces\streaming-dstream-window.png)

### Structured Streaming



从Spark 2.0开始引入了Structured Streaming， 将微批次处理从高级 API 中解耦出去，简化了 API 的使用，API 不再负责进行微批次处理；开发者可以将流看成是一个没有边界的表，并基于这些“表”运行查询。 Structured Streaming的默认引擎基于微批处理引擎，并且可以达到最低100ms的延迟和数据处理的exactly-once保证。

从Spark 2.3开始，Structured Streaming继续向更快、更易用、更智能的目标迈进，引入了低延迟的持续流处理模式，这时候已经不再采用批处理引擎，而是一种类似Flink机制的持续处理引擎，可以达到端到端最低1ms的延迟和数据处理的at-least-once的保证

### 编程模型

![Stream as a Table](D:\workspace\workspace-github\Knowledge\big-data\resouces\structured-streaming-stream-as-a-table.png)

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



