# Apache Kafka 



## kafka入门

Kafka 是一个分布式消息引擎与流处理平台，经常用做企业的消息总线、实时数据管道，有的还把它当做存储系统来使用。早期 Kafka 的定位是一个高吞吐的分布式消息系统，目前则演变成了一个成熟的分布式消息引擎，以及流处理平台。

### Kafka系统架构

Kafka 的设计遵循生产者消费者模式，生产者发送消息到 broker 中某一个 topic 的具体分区里，消费者从一个或多个分区中拉取数据进行消费。

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\v2-3a32db47da79e4f7f251727d95b8920a_720w.jpg)





目前，Kafka 依靠 Zookeeper 做分布式协调服务，负责存储和管理 Kafka 集群中的元数据信息，包括集群中的 broker 信息、topic 信息、topic 的分区与副本信息等。

### Kafka术语

- producer：生产者，消息生产和发送。
- Broker ：Kafka实例，多个broker组成一个kafka集群，通常一台机器部署一个 Kafka 实例，一个实例挂了不影响其他实例。
- Consumer：消费者。
- Topic：主题，服务端消息的逻辑存储单元。一个 topic 通常包含若干个 Partition 分区。
- Partition：Topic的分区，分布式存储在各个 broker 中， 实现发布与订阅的负载均衡。若干个分区可以被若干个 Consumer 同时消费，达到消费者高吞吐量。一个分区拥有多个副本（Replica），这是Kafka在可靠性和可用性方面的设计。
- message：消息，或称日志消息，是 Kafka 服务端实际存储的数据，每一条消息都由一个 key、一个 value 以及消息时间戳 timestamp 组成。
- offset：偏移量，分区中的消息位置，由 Kafka 自身维护，Consumer 消费时也要保存一份 offset 以维护消费过的消息位置。

### Kafka特点

Kafka 主要起到削峰填谷（缓冲）、系统解构以及冗余的作用，主要特点有：

- 高吞吐、低延时：这是 Kafka 显著的特点，Kafka 能够达到百万级的消息吞吐量，延迟可达毫秒级；
- 持久化存储：Kafka 的消息最终持久化保存在磁盘之上，提供了顺序读写以保证性能，并且通过 Kafka 的副本机制提高了数据可靠性。
- 分布式可扩展：Kafka 的数据是分布式存储在不同 broker 节点的，以 topic 组织数据并且按 partition 进行分布式存储，整体的扩展性都非常好。
- 高容错性：集群中任意一个 broker 节点宕机，Kafka 仍能对外提供服务。

### Kafka消息发送机制

Kafka 生产端发送消息的机制非常重要，这也是 Kafka 高吞吐的基础，生产端的基本流程如下图所示：

![preview](D:\workspace\workspace-github\Knowledge\big-data\resouces\v2-70f10b05688336a616b3366740628e68_r.jpg)



#### 异步发送

生产端构建的 ProducerRecord 先是经过 keySerializer、valueSerializer 序列化后，再是经过 Partition 分区器处理，决定消息落到 topic 具体某个分区中，最后把消息发送到客户端的消息缓冲池 accumulator 中，交由一个叫作 Sender 的线程发送到 broker 端。

这里缓冲池 accumulator 的最大大小由参数 buffer.memory 控制，默认是 32M，当生产消息的速度过快导致 buffer 满了的时候，将阻塞 max.block.ms 时间，超时抛异常，所以 buffer 的大小可以根据实际的业务情况进行适当调整。



#### 批量发送

发送到缓冲 buffer 中消息将会被分为一个一个的 batch，分批次的发送到 broker 端，批次大小由参数 batch.size 控制，默认16KB。这就意味着正常情况下消息会攒够 16KB 时才会批量发送到 broker 端，所以一般减小 batch 大小有利于降低消息延时，增加 batch 大小有利于提升吞吐量。

那么生成端消息是不是必须要达到一个 batch 大小时，才会批量发送到服务端呢？答案是否定的，Kafka 生产端提供了另一个重要参数 linger.ms，该参数控制了 batch 最大的空闲时间，超过该时间的 batch 也会被发送到 broker 端。



#### 消息重试

Kafka 生产端支持重试机制，对于某些原因导致消息发送失败的，比如网络抖动，开启重试后 Producer 会尝试再次发送消息。该功能由参数 retries 控制，参数含义代表重试次数，默认值为 0 表示不重试，建议设置大于 0 比如 3。



#### 消息交付语义

Kafka可以提供的消息交付语义有以下几种：

- at most once  消息可能丢失但绝对不会重传
- at least once   消息可以重传但是绝对不会丢失
- Exactly once   每条消息只被传送一次

发布消息时，我们会有一个消息的概念被“committed”到 log 中。 一旦消息被提交，只要有一个 broker 备份了该消息写入的 partition，并且保持“alive”状态，该消息就不会丢失。 



#### Acks确认机制

acks参数指定了必须要有多少个分区副本收到消息，生产者才认为该消息是写入成功的，这个参数对于消息是否丢失起着重要作用，该参数的配置具体如下：

- ack =0， 表示生产者在成功写入消息之前不会等待任何来自服务器的响应。一旦出现了问题导致服务器没有收到消息，那么生产者就无从得知，消息也就丢失了. 改配置由于不需要等到服务器的响应，所以可以以网络支持的最大速度发送消息，从而达到很高的吞吐量。

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\ack=0.jpg)



- ack=1，表示只要集群的leader分区副本接收到了消息，就会向生产者发送一个成功响应的ack，此时生产者接收到ack之后就可以认为该消息是写入成功的. 一旦消息无法写入leader分区副本(比如网络原因、leader节点崩溃),生产者会收到一个错误响应，当生产者接收到该错误响应之后，为了避免数据丢失，会重新发送数据.这种方式的吞吐量取决于使用的是异步发送还是同步发送.

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\ack=1.png)

- ack=all，表示只有所有参与复制的节点(ISR列表的副本)全部收到消息时，生产者才会接收到来自服务器的响应. 这种模式是最高级别的，也是最安全的，可以确保不止一个Broker接收到了消息. 该模式的延迟会很高.

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\ack=all.png)



##### 最小同步副本

当acks=all时，需要所有的副本都同步了才会发送成功响应到生产者. 其实这里面存在一个问题：如果Leader副本是唯一的同步副本时会发生什么呢？此时相当于acks=1.所以是不安全的.

Kafka的Broker端提供了一个参数`min.insync.replicas`,该参数控制的是消息至少被写入到多少个副本才算是"真正写入",该值默认值为1，生产环境设定为一个大于1的值可以提升消息的持久性. 因为如果同步副本的数量低于该配置值，则生产者会收到错误响应，从而确保消息不丢失.



##### Case 1

当min.insync.replicas=2且acks=all时，如果此时ISR列表只有[1,2],3被踢出ISR列表，只需要保证两个副本同步了，生产者就会收到成功响应.

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\replicas=2.png)



##### Case 2

，当min.insync.replicas=2，如果此时ISR列表只有[1],2和3被踢出ISR列表，那么当acks=all时，则不能成功写入数；当acks=0或者acks=1可以成功写入数据.

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\ISR=1.png)



##### Case 3

这种情况是很容易引起误解的，如果acks=all且min.insync.replicas=2，此时ISR列表为[1,2,3],那么还是会等到所有的同步副本都同步了消息，才会向生产者发送成功响应的ack.因为min.insync.replicas=2只是一个最低限制，即同步副本少于该配置值，则会抛异常，而acks=all，是需要保证所有的ISR列表的副本都同步了才可以发送成功响应. 如下图所示：

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\ISR=3.png)



acks=0，生产者在成功写入消息之前不会等待任何来自服务器的响应.

acks=1,只要集群的leader分区副本接收到了消息，就会向生产者发送一个成功响应的ack.

acks=all,表示只有所有参与复制的节点(ISR列表的副本)全部收到消息时，生产者才会接收到来自服务器的响应，此时如果ISR同步副本的个数小于`min.insync.replicas`的值，消息不会被写入.

#### 副本机制

Kafka 允许 topic 的 partition 拥有若干副本，你可以在server端配置partition 的副本数量。当集群中的节点出现故障时，能自动进行故障转移，保证数据的可用性。

#### Kafka 副本作用

Kafka 默认只会给分区设置一个副本，由 broker 端参数 default.replication.factor 控制，默认值为 1，通常我们会修改该默认值，或者命令行创建 topic 时指定 replication-factor 参数，生产建议设置 3 副本。副本作用主要有两方面：

- 消息冗余存储，提高 Kafka 数据的可靠性；
- 提高 Kafka 服务的可用性，follower 副本能够在 leader 副本挂掉或者 broker 宕机的时候参与 leader 选举，继续对外提供读写服务。

#### **关于读写分离**

Kafka 并不支持读写分区，生产消费端所有的读写请求都是由 leader 副本处理的，follower 副本的主要工作就是从 leader 副本处异步拉取消息，进行消息数据的同步，并不对外提供读写服务。

Kafka 之所以这样设计，主要是为了保证读写一致性，因为副本同步是一个异步的过程，如果当 follower 副本还没完全和 leader 同步时，从 follower 副本读取数据可能会读不到最新的消息。



#### ISR 副本集合

Kafka 为了维护分区副本的同步，引入 ISR（In-Sync Replicas）副本集合的概念，ISR 是分区中正在与 leader 副本进行同步的 replica 列表，且必定包含 leader 副本。

ISR 列表是持久化在 Zookeeper 中的，任何在 ISR 列表中的副本都有资格参与 leader 选举。

副本被包含在 ISR 列表中的条件是由参数 replica.lag.time.max.ms 控制的，参数含义是副本同步落后于 leader 的最大时间间隔，默认10s，意思就是说如果某一 follower 副本中的消息比 leader 延时超过10s，就会被从 ISR 中排除。Kafka 之所以这样设计，主要是为了减少消息丢失，只有与 leader 副本进行实时同步的 follower 副本才有资格参与 leader 选举，这里指相对实时。



### Rebalance



**Rebalance 概念**

就 Kafka 消费端而言，有一个难以避免的问题就是消费者的重平衡即 Rebalance。Rebalance是让一个消费组的所有消费者就如何消费订阅 topic 的所有分区达成共识的过程，在 Rebalance 过程中，所有 Consumer 实例都会停止消费，等待 Rebalance 的完成。因为要停止消费等待重平衡完成，因此 Rebalance 会严重影响消费端的 TPS，是应当尽量避免的。



**Rebalance 发生条件**

关于何时会发生 Rebalance，总结起来有三种情况：

- 消费组的消费者成员数量发生变化
- 消费主题的数量发生变化
- 消费主题的分区数量发生变化

其中后两种情况一般是计划内的，比如为了提高消息吞吐量增加 topic 分区数，这些情况一般是不可避免的。



#### kafka协调器

在介绍如何避免 Rebalance 问题之前，先来认识下 Kafka 的协调器 Coordinator，和之前 Kafka 控制器类似，Coordinator 也是 Kafka 的核心组件。

主要有两类 Kafka 协调器：

- 组协调器（Group Coordinator）
- 消费者协调器（Consumer Coordinator）

Kafka为了更好的实现消费成员管理、位移管理，以及Rebalance等，broker服务端引入了组协调器

### Kafka生产者事务和幂等

幂等性引入目的：

- 生产者重复生产消息。生产者进行retry会产生重试时，会重复产生消息。有了幂等性之后，在进行retry重试时，只会生成一个消息。

  

  

  

