## 1. 数据倾斜问题

### 1.1 数据倾斜的原因

1. 上游生产数据分布不均衡
2. 数据过滤
3. key by

### 1.2 问题解决办法

Rescale、Rebalance


注：

```
Shuffle	: 随机选择发往下游的channel
Rebalance	: round-robin，将输入流中的事件以轮流方式均匀分配给后继任务
Rescale	: rescale也会以轮流的方式对数据进行分发，但分发目标仅限于部分后继任务，是一种轻量级的负载均衡方案
Broadcast	: 广播
```