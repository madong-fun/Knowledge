#FLink 面试题

**1. Flink 中 exactly-once 语义是如何实现的，状态是如何存储的？**

Flink 依靠 checkpoint 机制来实现 exactly-once 语义，如果要实现端到端 的 exactly-once，还需要外部 source 和 sink 满足一定的条件。状态的存储通过状态 后端来管理，Flink 中可以配置不同的状态后端。

**2.什么是solts**

solts ：槽，slot 的数量通常与每个 TaskManager 的可用 CPU 内核数成比例。
一般情况下你的 slot 数是你每个 TaskManager 的 cpu 的核数。

**3.Flink的 BackPressure 背压机制，跟spark有什么区别？**
Flink是通过自下而上的背压检测从而 控制流量。如果下层的operation压力大那么上游的operation就会让它慢下来。Jobmanager会反复调用一个job的task运行所在线程的Thread.getStackTrace()，默认情况下，jobmanager会每个50ms触发对一个job的每个task依次进行100次堆栈跟踪调用，根据调用结果来确定backpressure，flink是通过计算得到一个比值radio来确定当前运行的job的backpressure状态。在web页面可以看到这个radio值，它表示在一个内部方法调用中阻塞的堆栈跟踪次数，例如radio=0.01表示100次中仅有一次方法调用阻塞。
OK: 0 <= Ratio <= 0.10
LOW: 0.10 < Ratio <= 0.5
HIGH: 0.5 < Ratio <= 1。
spark的背压检测和处理方式跟flink不同，在spark1.5之前需要自己配置接收速率来限速，所以这个值需要人为测试来决定，spark1.5版本之后只需要开启backpressure功能即可，spark会自己根据计算时间、延时时间等来确定是否限流。

