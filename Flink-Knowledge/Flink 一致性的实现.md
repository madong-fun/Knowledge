## Flink+kafka 端到端一致性实现

若要 sink 支持仅一次语义，必须以事务的方式写数据到 Kafka，这样当提交事务时两次 checkpoint 间的所有写入操作作为一个事务被提交。这确保了出现故障或崩溃时这些写入操作能够被回滚。
在一个分布式且含有多个并发执行 sink 的应用中，仅仅执行单次提交或回滚是不够的，因为所有组件都必须对这些提交或回滚达成共识，这样才能保证得到一致性的结果。Flink在1.4版本后，使用两阶段提交协议以及预提交(pre-commit)阶段来解决这个问题。

![IMAGE](resources/E4D3991F282E75041C34E514D5E32E11.jpg =865x447)

如上图所示：
 - 一个source，从Kafka中读取数据（即KafkaConsumer）
 - 一个时间窗口化的聚会操作
 - 一个sink，将结果写回到Kafka（即KafkaProducer）
 

![IMAGE](resources/C02C652C7724ED15AECB6C0C21854E6D.jpg =865x416)

flink 的两段提交思路：如上图，Flink checkpointing 开始时便进入到 pre-commit 阶段。具体来说，一旦 checkpoint 开始，Flink 的 JobManager 向输入流中写入一个 checkpoint barrier ，将流中所有消息分割成属于本次 checkpoint 的消息以及属于下次 checkpoint 的，barrier 也会在操作算子间流转。对于每个 operator 来说，该 barrier 会触发 operator 状态后端为该 operator 状态打快照。data source 保存了 Kafka 的 offset，之后把 checkpoint barrier 传递到后续的 operator。

这种方式仅适用于 operator 仅有它的内部状态。内部状态是指 Flink state backends 保存和管理的内容（如第二个 operator 中 window 聚合算出来的 sum）。

当一个进程仅有它的内部状态的时候，除了在 checkpoint 之前将需要将数据更改写入到 state backend，不需要在预提交阶段做其他的动作。在 checkpoint 成功的时候，Flink 会正确的提交这些写入，在 checkpoint 失败的时候会终止提交。过程如下图：

![IMAGE](resources/8AA94579BF6113FE599E700018D9D994.jpg =865x468)

当结合外部系统的时候，外部系统必须要支持可与两阶段提交协议捆绑使用的事务。显然本例中的 sink 由于引入了 kafka sink，因此在预提交阶段 data sink 必须预提交外部事务。如下图：

![IMAGE](resources/89A250B963302F9C3F3017CF3DF9DECB.jpg =865x446)

当 barrier 在所有的算子中传递一遍，并且触发的快照写入完成，预提交阶段完成。所有的触发状态快照都被视为 checkpoint 的一部分，也可以说 checkpoint 是整个应用程序的状态快照，包括预提交外部状态。出现故障可以从 checkpoint 恢复。下一步就是通知所有的操作算子 checkpoint 成功。该阶段 jobmanager 会为每个 operator 发起 checkpoint 已完成的回调逻辑。

本例中 data source 和窗口操作无外部状态，因此该阶段，这两个算子无需执行任何逻辑，但是 data sink 是有外部状态的，因此，此时我们必须提交外部事务，如下图：

![IMAGE](resources/62231023AC28EB38CBB00643F91F466A.jpg =865x399)

汇总以上所有信息，总结一下：

1. 一旦所有operator完成各自的pre-commit，它们会发起一个commit操作

2. 倘若有一个pre-commit失败，所有其他的pre-commit必须被终止，并且Flink会回滚到最近成功完成decheckpoint

3. 一旦pre-commit完成，必须要确保commit也要成功——operator和外部系统都需要对此进行保证。倘若commit失败(比如网络故障等)，Flink应用就会崩溃，然后根据用户重启策略执行重启逻辑，之后再次重试commit。这个过程至关重要，因为倘若commit无法顺利执行，就可能出现数据丢失的情况

因此，所有opeartor必须对checkpoint最终结果达成共识：即所有operator都必须认定数据提交要么成功执行，要么被终止然后回滚。

## Flink中实现两阶段提交

这种operator的管理有些复杂，这也是为什么Flink提取了公共逻辑并封装进TwoPhaseCommitSinkFunction抽象类的原因。

下面讨论一下如何扩展TwoPhaseCommitSinkFunction类来实现一个简单的基于文件的sink。若要实现支持exactly-once semantics的文件sink，我们需要实现以下4个方法：

1. **beginTransaction**：开启一个事务，在临时目录下创建一个临时文件，之后，写入数据到该文件中

2. **preCommit**：在pre-commit阶段，flush缓存数据块到磁盘，然后关闭该文件，确保再不写入新数据到该文件。同时开启一个新事务执行属于下一个checkpoint的写入操作
3. **commit**：在commit阶段，我们以原子性的方式将上一阶段的文件写入真正的文件目录下。注意：这会增加输出数据可见性的延时。通俗说就是用户想要看到最终数据需要等会，不是实时的。
4. **abort**：一旦终止事务，我们自己删除临时文件

当出现崩溃时，Flink会恢复最新已完成快照中应用状态。需要注意的是在某些极偶然的场景下，pre-commit阶段已成功完成而commit尚未开始（也就是operator尚未来得及被告知要开启commit），此时倘若发生崩溃Flink会将opeartor状态恢复到已完成pre-commit但尚未commit的状态。

在一个checkpoint状态中，对于已完成pre-commit的事务状态，我们必须保存足够多的信息，这样才能确保在重启后要么重新发起commit亦或是终止掉事务。本例中这部分信息就是临时文件所在的路径以及目标目录。

TwoPhaseCommitSinkFunction考虑了这种场景，因此当应用从checkpoint恢复之后TwoPhaseCommitSinkFunction总是会发起一个抢占式的commit。这种commit必须是幂等性的，虽然大部分情况下这都不是问题。本例中对应的这种场景就是：临时文件不在临时目录下，而是已经被移动到目标目录下。