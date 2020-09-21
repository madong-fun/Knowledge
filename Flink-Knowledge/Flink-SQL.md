# FLink-SQL



## 窗口函数

窗口函数Flink SQL支持基于无限大窗口的聚合（无需在SQL Query中，显式定义任何窗口）以及对一个特定的窗口的聚合。例如，需要统计在过去的1分钟内有多少用户点击了某个的网页，可以通过定义一个窗口来收集最近1分钟内的数据，并对这个窗口内的数据进行计算。

Flink SQL支持的窗口聚合主要是两种：Window聚合和Over聚合。Window聚合支持Event Time （Rowtime）和Processing Time （PROCTIME）两种时间属性定义窗口。每种时间属性类型支持三种窗口类型：滚动窗口（TUMBLE）、滑动窗口（HOP）和会话窗口（SESSION）。



Rowtime列在经过窗口操作后，其Event Time属性将丢失。使用辅助函数`TUMBLE_ROWTIME`、`HOP_ROWTIME`或`SESSION_ROWTIME`，获取窗口中的Rowtime列的最大值`max(rowtime)`作为时间窗口的Rowtime，其类型是具有Rowtime属性的TIMESTAMP，取值为 `window_end - 1` 。 例如`[00:00, 00:15) `的窗口，返回值为`00:14:59.999` 。

### 滚动窗口

| 窗口标识                                    | 返回类型                | 描述                                                         |
| :------------------------------------------ | ----------------------- | ------------------------------------------------------------ |
| `TUMBLE_START(time-attr, size-interval)`    | TIMESTAMP               | 返回窗口的起始时间（包含边界）。例如`[00:10, 00:15) `窗口，返回`00:10`。 |
| `TUMBLE_END(time-attr, size-interval)`      | TIMESTAMP               | 返回窗口的结束时间（包含边界）。例如`[00:00, 00:15]`窗口，返回`00:15`。 |
| `TUMBLE_ROWTIME(time-attr, size-interval)`  | TIMESTAMP(rowtime-attr) | 返回窗口的结束时间（不包含边界）。例如`[00:00, 00:15]`窗口，返回`00:14:59.999`。返回值是一个rowtime attribute，即可以基于该字段做时间属性的操作，例如，级联窗口只能用在基于Event Time的Window上 |
| `TUMBLE_PROCTIME(time-attr, size-interval)` | TIMESTAMP(rowtime-attr) | 返回窗口的结束时间（不包含边界）。例如`[00:00, 00:15]`窗口，返回`00:14:59.999`。返回值是一个proctime attribute，即可以基于该字段做时间属性的操作，例如，级联窗口只能用在基于Processing Time的Window上 |
|                                             |                         |                                                              |



```java
        EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);


        StreamTableEnvironment tEnv = StreamTableEnvironment.create(env,settings);

       DataStream<Tuple3<String,Integer,Long>> dataStream = env.fromCollection(Arrays.asList(
                new Tuple3<String,Integer,Long>("zhangSan",100,1599752307000L),
                new Tuple3<>("zhangSan",300,1599752367000L),
                new Tuple3<>("wangwu",500,1599752427000L),
                new Tuple3<>("zhaoqiansun",50,1599752487000L)));


       SingleOutputStreamOperator<Tuple3<String,Integer,Long>> singleOutputStreamOperator = dataStream.assignTimestampsAndWatermarks(WatermarkStrategy.<Tuple3<String,Integer,Long>>forBoundedOutOfOrderness(Duration.ofSeconds(10))
               .withTimestampAssigner(new SerializableTimestampAssigner<Tuple3<String,Integer,Long>>() {
                   @Override
                   public long extractTimestamp(Tuple3<String, Integer, Long> element, long recordTimestamp) {
                       return element.f2;
                   }

               }));


      Table user = tEnv.fromDataStream(singleOutputStreamOperator,$("user"),$("money"),$("t").rowtime());

      Table result = tEnv.sqlQuery("select TUMBLE_START(t, interval '5' MINUTE ) AS window_start , TUMBLE_END(t, INTERVAL '5' MINUTE) as window_end, count(user) from "
      + user + " group by TUMBLE(t, INTERVAL '5' MINUTE), user "
      );

      tEnv.toAppendStream(result, TypeInformation.of(new TypeHint<Tuple3<Timestamp,Timestamp,Long>>() {}))
      .print();


      env.execute();
```



### 滑动窗口

滑动窗口（HOP），也被称作Sliding Window。不同于滚动窗口，滑动窗口的窗口可以重叠。

滑动窗口有两个参数：**slide**和**size**。**slide**为每次滑动的步长，**size**为窗口的大小。

- **slide < size**，则窗口会重叠，每个元素会被分配到多个窗口。
- **slide = size**，则等同于滚动窗口（TUMBLE）。
- **slide > size**，则为跳跃窗口，窗口之间不重叠且有间隙。

通常，大部分元素符合多个窗口情景，窗口是重叠的。因此，滑动窗口在计算移动平均数（moving averages）时很实用。例如，计算过去5分钟数据的平均值，每10秒钟更新一次，可以设置**slide**为10秒，**size**为5分钟。下图为您展示间隔为30秒，窗口大小为1分钟的滑动窗口。

| 窗口标识                                                     | 返回类型                  | 描述                                                         |
| ------------------------------------------------------------ | ------------------------- | ------------------------------------------------------------ |
| `HOP_START（<time-attr>, <slide-interval>, <size-interval>）` | TIMESTAMP                 | 返回窗口的起始时间（包含边界）。例如`[00:10, 00:15) `窗口，返回`00:10 `。 |
| `HOP_END（<time-attr>, <slide-interval>, <size-interval>）`  | TIMESTAMP                 | 返回窗口的结束时间（包含边界）。例如`[00:00, 00:15) `窗口，返回`00:15`。 |
| `HOP_ROWTIME（<time-attr>, <slide-interval>, <size-interval>）` | TIMESTAMP（rowtime-attr） | 返回窗口的结束时间（不包含边界）。例如`[00:00, 00:15) `窗口，返回`00:14:59.999`。返回值是一个rowtime attribute，即可以基于该字段做时间类型的操作，例如[级联窗口](https://help.aliyun.com/document_detail/62510.html#section-cwf-1kt-jhb)，只能用在基于event time的window上。 |
| `HOP_PROCTIME（<time-attr>, <slide-interval>, <size-interval>）` | TIMESTAMP（rowtime-attr） | 返回窗口的结束时间（不包含边界）。例如`[00:00, 00:15) `窗口，返回`00:14:59.999 `。返回值是一个proctime attribute，即可以基于该字段做时间类型的操作，例如[级联窗口](https://help.aliyun.com/document_detail/62510.html#section-cwf-1kt-jhb)，只能用在基于processing time的window上。 |
|                                                              |                           |                                                              |

**测试语句：**

```java
CREATE TABLE user_clicks (
    username VARCHAR,
    click_url VARCHAR,
    ts TIMESTAMP,
    WATERMARK wk FOR ts AS WITHOFFSET (ts, 2000)--为rowtime定义Watermark。
) WITH ( TYPE = 'datahub',
        ...);
CREATE TABLE hop_output (
    window_start TIMESTAMP,
    window_end TIMESTAMP,
    username VARCHAR,
    clicks BIGINT
) WITH (TYPE = 'rds',
        ...);
INSERT INTO
    hop_output
SELECT
    HOP_START (ts, INTERVAL '30' SECOND, INTERVAL '1' MINUTE),
    HOP_END (ts, INTERVAL '30' SECOND, INTERVAL '1' MINUTE),
    username,
    COUNT (click_url)
FROM
    user_clicks
GROUP BY
    HOP (ts, INTERVAL '30' SECOND, INTERVAL '1' MINUTE),
    username    
```

### **会话窗口**



会话窗口（SESSION）通过Session活动来对元素进行分组。会话窗口与滚动窗口和滑动窗口相比，没有窗口重叠，没有固定窗口大小。相反，当它在一个固定的时间周期内不再收到元素，即会话断开时，这个窗口就会关闭。

会话窗口通过一个间隔时间（Gap）来配置，这个间隔定义了非活跃周期的长度。例如，一个表示鼠标点击活动的数据流可能具有长时间的空闲时间，并在两段空闲之间散布着高浓度的点击。 如果数据在指定的间隔（Gap）之后到达，则会开始一个新的窗口。



### OVER窗口



OVER窗口（OVER Window）是传统数据库的标准开窗，不同于Group By Window，OVER窗口中每1个元素都对应1个窗口。OVER窗口可以按照实际元素的行或实际的元素值（时间戳值）确定窗口，因此流数据元素可能分布在多个窗口中。

在应用OVER窗口的流式数据中，每1个元素都对应1个OVER窗口。每1个元素都触发1次数据计算，每个触发计算的元素所确定的行，都是该元素所在窗口的最后1行。在实时计算的底层实现中，OVER窗口的数据进行全局统一管理（数据只存储1份），逻辑上为每1个元素维护1个OVER窗口，为每1个元素进行窗口计算，完成计算后会清除过期的数据。



#### 类型

Flink SQL中对OVER窗口的定义遵循标准SQL的定义语法，传统OVER窗口没有对其进行更细粒度的窗口类型命名划分。按照计算行的定义方式，OVER Window可以分为以下两类：

- ROWS OVER Window：每1行元素都被视为新的计算行，即每1行都是一个新的窗口。
- RANGE OVER Window：具有相同时间值的所有元素行视为同一计算行，即具有相同时间值的所有行都是同一个窗口。

https://help.aliyun.com/document_detail/62514.html?spm=a2c4g.11186623.6.715.5a331f40RSZUto