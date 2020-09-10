###  数据本地化
 * PROCESS_LOCAL: 该级别由参数spark.locality.wait.process指定
 * NODE_LOCAL: 该级别由参数spark.locality.wait.node指定
 * RACK_LOCAL: 该级别由参数spark.locality.wait.rack指定

上面三个参数, 如果不指定值的话, 则默认取值于参数spark.locality.wait的值, 该值默认为3s
Spark在task的调度过程中, 会优化把task分配给持有数据/缓存的同一个executor进程, 如果executor进程正在执行别的任务, 则会等待spark.locality.wait.process秒, 如果等待了这么多秒之后该executor还是不可用, 则数据本地化程度降低一级, 选择分配到统一节点的其他executor进程, 如果还是不可用, 则选择同一个集群的其他executor.
假如通过spark ui的executor页发现某个executor的数据input量特别大, 则极有可能会发生task倾斜.

spark.locality.wait.process=200ms, 