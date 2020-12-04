# FLink-SQL

## Flink SQL 流程

```
// NOTE : 执行顺序是从上至下, " -----> " 表示生成的实例类型
* 
*        +-----> "left outer JOIN" (SQL statement)
*        |   
*        |     
*     SqlParser.parseQuery // SQL 解析阶段，生成AST（抽象语法树），作用是SQL–>SqlNode      
*        |   
*        |      
*        +-----> SqlJoin (SqlNode)
*        |   
*        |     
*     SqlToRelConverter.convertQuery // 语义分析，生成逻辑计划，作用是SqlNode–>RelNode
*        |    
*        |     
*        +-----> LogicalProject (RelNode) // Abstract Syntax Tree，未优化的RelNode   
*        |      
*        |     
*    FlinkLogicalJoinConverter (RelOptRule) // Flink定制的优化rules      
*    VolcanoRuleCall.onMatch // 基于Flink定制的一些优化rules去优化 Logical Plan 
*        | 
*        |   
*        +-----> FlinkLogicalJoin (RelNode)  // Optimized Logical Plan，逻辑执行计划
*        |  
*        |    
*    StreamExecJoinRule (RelOptRule) // Rule that converts FlinkLogicalJoin without window bounds in join condition to StreamExecJoin     
*    VolcanoRuleCall.onMatch // 基于Flink rules将optimized LogicalPlan转成Flink物理执行计划
*        |       
*        |   
*        +-----> StreamExecJoin (FlinkRelNode) // Stream physical RelNode，物理执行计划
*        |      
*        |     
*    StreamExecJoin.translateToPlanInternal  // 作用是生成 StreamOperator, 即Flink算子  
*        |     
*        |     
*        +-----> StreamingJoinOperator (StreamOperator) // Streaming unbounded Join operator in StreamTask   
*        |     
*        |       
*    StreamTwoInputProcessor.processRecord1// 在TwoInputStreamTask调用StreamingJoinOperator，真实的执行 
*        |
*        |  

```



## Calcite



FLink 并没有像Spark 一样，自行实现SQL解析和优化工作，而是使用了开源项目Calcite。

### Calcite 处理流程

Sql 的执行过程一般可以分为四个阶段，Calcite 与这个很类似，但Calcite是分成五个阶段 ：

1. SQL 解析阶段，生成AST（抽象语法树）（SQL–>SqlNode）
2. SqlNode 验证（SqlNode–>SqlNode）
3. 语义分析，生成逻辑计划（Logical Plan）（SqlNode–>RelNode/RexNode）
4. 优化阶段，按照相应的规则（Rule）进行优化（RelNode–>RelNode）
5. 生成ExecutionPlan，生成物理执行计划（DataStream Plan）



## Flink SQL相关对象



### TableEnvironment

TableEnvironment对象是Table API和SQL集成的一个核心，支持以下场景：

- 注册一个Table。
- 将一个TableSource注册给TableEnvironment，这里的TableSource指的是将数据存储系统的作为Table，例如mysql,hbase,CSV,Kakfa,RabbitMQ等等。
- 注册一个外部的catalog，可以访问外部系统的数据或文件。
- 执行SQL查询。
- 注册一个用户自定义的function。
- 将DataStream或DataSet转成Table。



一个查询中只能绑定一个指定的TableEnvironment，TableEnvironment可以通过来配置TableConfig来配置，通过TableConfig可以自定义查询优化以及translation的进程。

TableEnvironment执行过程如下：

- TableEnvironment.sql()为调用入口；
- Flink实现了FlinkPlannerImpl，执行parse(sql)，validate(sqlNode)，rel(sqlNode)操作；
- 生成Table；



```
/**
  * The abstract base class for batch and stream TableEnvironments.
  *
  * @param config The configuration of the TableEnvironment
  */
abstract class TableEnvironment(
    private[flink] val execEnv: JavaStreamExecEnv,
    val config: TableConfig) extends AutoCloseable {

  protected val DEFAULT_JOB_NAME = "Flink Exec Table Job"

  protected val catalogManager: CatalogManager = new CatalogManager()
  private val currentSchema: SchemaPlus = catalogManager.getRootSchema

  private val typeFactory: FlinkTypeFactory = new FlinkTypeFactory(new FlinkTypeSystem)

  // Table API/SQL function catalog (built in, does not contain external functions)
  private val functionCatalog: FunctionCatalog = BuiltInFunctionCatalog.withBuiltIns()

  // Table API/SQL function catalog built in function catalog.
  private[flink] lazy val chainedFunctionCatalog: FunctionCatalog =
    new ChainedFunctionCatalog(Seq(functionCatalog))

  // the configuration to create a Calcite planner
  protected var frameworkConfig: FrameworkConfig = createFrameworkConfig

  // the builder for Calcite RelNodes, Calcite's representation of a relational expression tree.
  protected var relBuilder: FlinkRelBuilder = createRelBuilder

  // the planner instance used to optimize queries of this TableEnvironment
  private var planner: RelOptPlanner = createRelOptPlanner

  // reuse flink planner
  private var flinkPlanner: FlinkPlannerImpl = createFlinkPlanner

  // a counter for unique attribute names
  private[flink] val attrNameCntr: AtomicInteger = new AtomicInteger(0)

  // a counter for unique table names
  private[flink] val tableNameCntr: AtomicInteger = new AtomicInteger(0)

  private[flink] val tableNamePrefix = "_TempTable_"

  // sink nodes collection
  private[flink] val sinkNodes = new mutable.MutableList[SinkNode]

  private[flink] val transformations = new ArrayBuffer[StreamTransformation[_]]

  protected var userClassloader: ClassLoader = null

  // a manager for table service
  private[flink] val tableServiceManager = new FlinkTableServiceManager(this)
```



### CataLog

**Catalog** – 定义元数据和命名空间，包含 Schema（库），Table（表），RelDataType（类型信息）。

所有对数据库和表的元数据信息都存放在Flink CataLog内部目录结构中，其存放了Flink内部所有与Table相关的元数据信息，包括表结构信息/数据源信息等。



```
public class CatalogManager {

	// A list of named catalogs.
	private Map<String, ReadableCatalog> catalogs;
	
	
	
/**
 * This interface is responsible for reading database/table/views/UDFs from a registered catalog.
 * It connects a registered catalog and Flink's Table API.
 */
public interface ReadableCatalog extends Closeable {



/**
 * An in-memory catalog.
 */
public class FlinkInMemoryCatalog implements ReadableWritableCatalog {

	public static final String DEFAULT_DB = "default";

	private String defaultDatabaseName = DEFAULT_DB;

	private final String catalogName;
	private final Map<String, CatalogDatabase> databases;
	private final Map<ObjectPath, CatalogTable> tables;
	private final Map<ObjectPath, Map<CatalogPartition.PartitionSpec, CatalogPartition>> partitions;
	
```









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