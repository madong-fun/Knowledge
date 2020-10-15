# Spark CBO

CBO，基于代价优化，原理是计算所有可能的物理计划的代价，并挑选出代价最小的物理计划。其核心在于评估一个给定的物理计划的代价。

物理执行计划是一个树状图，其代价等于每个执行节点的代价总和，如下图所示：

![CBO 总代价](./resources/spark_sql_cost_model.png)



而每个执行节点的代价，可以分为两部分：

- 该执行节点对数据集的影响，或者说该节点输出数据集的大小和分布
- 该执行节点操作算子的代价



每个操作算子的代价相对固定，可以用规则描述，而执行节点输出数据的大小与分布，分为两个部分：

1. 初始数据集，其数据集的大小和分布可以通过统计得到

2. 中间节点输出数据集大小和分布可以由输入数据集的信息以及操作本身特点进行推算。

   

   所以，最终需要解决两个问题：

   - 如何获取原始数据的统计信息
   - 如何根据输入数据集估算特定算子的输出数据集

   

   ## Statistics收集

   

   通过如下 SQL 语句，可计算出整个表的记录总数以及总大小

   `````sql
   ANALYZE TABLE table_name COMPUTE STATISTICS;
   
   `````

   

   通过以下SQL语句，可以统计特定列的信息

   ```sql
   ANALYZE TABLE table_name COMPUTE STATISTICS FOR COLUMNS [column1] [,column2] [,column3] [,column4] ... [,columnn];
   ```

   

   除上述示例中的统计信息外，Spark CBO 还直接等高直方图 spark.sql.statistics.histogram.enabled 默认值为 false，也即 ANALYZE 时默认不计算及存储 histogram。通过 `SET spark.sql.statistics.histogram.enabled=true;` 启用 histogram 后，可以完整的统计信息。

   

   ```
   spark-sql> ANALYZE TABLE customer COMPUTE STATISTICS FOR COLUMNS c_customer_sk,c_customer_id,c_current_cdemo_sk,c_current_hdemo_sk,c_current_addr_sk,c_first_shipto_date_sk,c_first_sales_date_sk,c_salutation,c_first_name,c_last_name,c_preferred_cust_flag,c_birth_day,c_birth_month,c_birth_year,c_birth_country,c_login,c_email_address,c_last_review_date;
   Time taken: 125.624 seconds
   
   spark-sql> desc extended customer c_customer_sk;
   col_name       c_customer_sk
   data_type      bigint
   comment NULL
   min     1
   max     280000
   num_nulls      0
   distinct_count 274368
   avg_col_len    8
   max_col_len    8
   histogram       height: 1102.3622047244094, num_of_bins: 254
   bin_0   lower_bound: 1.0, upper_bound: 1090.0, distinct_count: 1089
   bin_1   lower_bound: 1090.0, upper_bound: 2206.0, distinct_count: 1161
   bin_2   lower_bound: 2206.0, upper_bound: 3286.0, distinct_count: 1124
   
   ...
   
   bin_251 lower_bound: 276665.0, upper_bound: 277768.0, distinct_count: 1041
   bin_252 lower_bound: 277768.0, upper_bound: 278870.0, distinct_count: 1098
   bin_253 lower_bound: 278870.0, upper_bound: 280000.0, distinct_count: 1106
   ```

   

   从上面的数据可见，生成的 histogram 为 equal-height histogram，且高度为 1102.36，bin 数为 254。其中 bin 个数可由 spark.sql.statistics.histogram.numBins 配置。对于每个 bin，匀记录其最小值，最大值，以及 distinct count。

这里的discount值并不是精确值，而是通过HyperLogLog计算出的近似值。



## 算子对数据集的影响估计

![Spark SQL CBO Operator Estimation](./resources/spark_cbo_operator_estimation.png)





以常见的Filter为例，对应常见的Column A < value B Filter, 可以通过如下方式估算输出中间结果的统计信息

- 若B  < A.min, 	则无数据被选中，则输出结果为空
- 若B > A.max, 则全部数据都被选中，输出结果与 A 相同，且统计信息不变
- 若A.min < B < A.max,则被选中的数据占比为 (B.value - A.min) / (A.max - A.min)，A.min 不变，A.max 更新为 B.value，A.ndv = A.ndv * (B.value - A.min) / (A.max - A.min)

![Spark SQL CBO Filter Estimation](./resources/spark_cbo_filter_estimation.png)



上述估算的前提是，字段 A 数据均匀分布。但很多时候，数据分布并不均匀，且当数据倾斜严重是，上述估算误差较大。此时，可充分利用 histogram 进行更精确的估算.

![Spark SQL CBO Filter Estimation with Histogram](./resources/spark_cbo_equal_height_histogram.png)



启用 Historgram 后，Filter `Column A < value B`的估算方法为

- 若 B < A.min，则无数据被选中，输出结果为空
- 若 B > A.max，则全部数据被选中，输出结果与 A 相同，且统计信息不变
- 若 A.min < B < A.max，则被选中的数据占比为 height(<B) / height(All)，A.min 不变，A.max = B.value，A.ndv = ndv(<B)

在上图中，B.value = 15，A.min = 0，A.max = 32，bin 个数为 10。Filter 后 A.ndv = ndv(<B.value) = ndv(<15)。该值可根据 A < 15 的 5 个 bin 的 ndv 通过 HyperLogLog 合并而得，无须重新计算所有 A < 15 的数据。



## 算子代价估计

SQL 中常见的操作有 Selection（由 select 语句表示），Filter（由 where 语句表示）以及笛卡尔乘积（由 join 语句表示）。其中代价最高的是 join。

Spark SQL 的 CBO 通过如下方法估算 join 的代价:

```sql
Cost = rows * weight + size * (1 - weight)
Cost = CostCPU * weight + CostIO * (1 - weight)
```



其中 rows 即记录行数代表了 CPU 代价，size 代表了 IO 代价。weight 由 spark.sql.cbo.joinReorder.card.weight 决定，其默认值为 0.7。



### Build侧选择

对于两表Hash Join，一般选择小表作为build size，构建哈希表，另一边作为 probe side。未开启 CBO 时，根据表原始数据大小选择 t2 作为build side

![Spark SQL build side without CBO](./resources/spark_cbo_build_side.png)



而开启 CBO 后，基于估计的代价选择 t1 作为 build side。



![Spark SQL build side with CBO](./resources/spark_cbo_build_side_selection.png)



### 优化Join类型

在 Spark SQL 中，Join 可分为 Shuffle based Join 和 BroadcastJoin。Shuffle based Join 需要引入 Shuffle，代价相对较高。BroadcastJoin 无须 Join，但要求至少有一张表足够小，能通过 Spark 的 Broadcast 机制广播到每个 Executor 中。

在不开启 CBO 中，Spark SQL 通过 spark.sql.autoBroadcastJoinThreshold 判断是否启用 BroadcastJoin。其默认值为 10485760 即 10 MB。

并且该判断基于参与 Join 的表的原始大小。



在下图示例中，Table 1 大小为 1 TB，Table 2 大小为 20 GB，因此在对二者进行 join 时，由于二者都远大于自动 BroatcastJoin 的阈值，因此 Spark SQL 在未开启 CBO 时选用 SortMergeJoin 对二者进行 Join。

而开启 CBO 后，由于 Table 1 经过 Filter 1 后结果集大小为 500 GB，Table 2 经过 Filter 2 后结果集大小为 10 MB 低于自动 BroatcastJoin 阈值，因此 Spark SQL 选用 BroadcastJoin。

![Spark SQL join type selection with CBO](./resources/spark_cbo_join_type.png)



### 优化多表Join顺序

未开启 CBO 时，Spark SQL 按 SQL 中 join 顺序进行 Join。极端情况下，整个 Join 可能是 left-deep tree。在下图所示 TPC-DS Q25 中，多路 Join 存在如下问题，因此耗时 241 秒。

- left-deep tree，因此所有后续 Join 都依赖于前面的 Join 结果，各 Join 间无法并行进行
- 前面的两次 Join 输入输出数据量均非常大，属于大 Join，执行时间较长

![Spark SQL multi join](./resources/spark_cbo_linear_join.png)



开启 CBO 后， Spark SQL 将执行计划优化如下

![Spark SQL multi join reorder with CBO](./resources/spark_cbo_join_reorder.png)



优化后的 Join 有如下优势，因此执行时间降至 71 秒

- Join 树不再是 left-deep tree，因此 Join 3 与 Join 4 可并行进行，Join 5 与 Join 6 可并行进行
- 最大的 Join 5 输出数据只有两百万条结果，Join 6 有 1.49 亿条结果，Join 7相当于小 Join





