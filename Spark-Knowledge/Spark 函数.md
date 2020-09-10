## Action 操作
* collect() ,返回值是一个数组，返回dataframe集合所有的行
* collectAsList() 返回值是一个java类型的数组，返回dataframe集合所有的行
* count() 返回一个number类型的，返回dataframe集合的行数
* describe(cols: String*) 返回一个通过数学计算的类表值(count, mean, stddev, min, and max)，如果有字段为空，那么不参与运算，只这对数值类型的字段。
* first() 返回第一行 ，类型是row类型
* head(n:Int)返回n行 ，类型是row 类型
* show(n:Int)返回n行，，返回值类型是unit
* table(n:Int) 返回n行 ，类型是row 类型

## dataframe的基本操作
* cache()同步数据的内存
* columns 返回一个string类型的数组，返回值是所有列的名字
* dtypes返回一个string类型的二维数组，返回值是所有列的名字以及类型
* explan()打印执行计划 物理的
* explain(n:Boolean) 输入值为 false 或者true ，返回值是unit 默认是false ，如果输入true 将会打印 逻辑的和物理的
* isLocal 返回值是Boolean类型，如果允许模式是local返回true 否则返回false
* persist(newlevel:StorageLevel) 返回一个dataframe.this.type 输入存储模型类型
* printSchema() 打印出字段名称和类型 按照树状结构来打印
* registerTempTable(tablename:String)，将df的对象只放在一张表里面，这个表随着对象的删除而删除了
* schema 返回structType 类型，将字段名称和类型按照结构体类型返回
* toDF(colnames：String*)将参数中的几个字段返回一个新的dataframe类型的，
* unpersist() 返回dataframe.this.type 类型，移除缓存和硬盘上的所有blocks

## 计算与查询

* agg(expers:column*) 返回dataframe类型 ，同数学计算求值
```
df.agg(max("age"), avg("salary"))
df.groupBy().agg(max("age"), avg("salary"))
```
* agg(exprs: Map[String, String]) 返回dataframe类型，同数学计算求值

```
df.agg(Map("age" -> "max", "salary" -> "avg"))
df.groupBy().agg(Map("age" -> "max", "salary" -> "avg"))
```

* agg(aggExpr: (String, String), aggExprs: (String, String)*)
*  agg(aggExpr: (String, String), aggExprs: (String, String)*) 返回dataframe类型 ，同数学计算求值
```
df.agg(Map("age" -> "max", "salary" -> "avg"))
df.groupBy().agg(Map("age" -> "max", "salary" -> "avg"))
```
* apply(colName: String) 返回column类型，捕获输入进去列的对象
* as(alias: String) 返回一个新的dataframe类型，就是原来的一个别名
* col(colName: String) 返回column类型，捕获输入进去列的对象
* cube(col1: String, cols: String*) 返回一个GroupedData类型，根据某些字段来汇总
* distinct 去重 返回一个dataframe类型
* drop(col: Column) 删除某列 返回dataframe类型
* dropDuplicates(colNames: Array[String]) 删除相同的列 返回一个dataframe
* except(other: DataFrame) 返回一个dataframe，返回在当前集合存在的在其他集合不存在的
* filter(conditionExpr: String): 刷选部分数据，返回dataframe类型 
* groupBy(col1: String, cols: String*) 根据某写字段来汇总返回groupedate类型
* intersect(other: DataFrame) 返回一个dataframe，在2个dataframe都存在的元素
* join(right: DataFrame, joinExprs: Column, joinType: String) 关联的类型：inner, outer, left_outer, right_outer, leftsemi
* limit(n: Int) 返回dataframe类型 去n 条数据出来
* na: DataFrameNaFunctions，可以调用dataframenafunctions的功能区做过滤
```
df.na.drop().show() 
df.na.fill(0) //空值填充0
```
* orderBy(sortExprs: Column*) 做alise排序
* select(cols:string*) dataframe 做字段的筛选 
```
df.select($"colA", $"colB" + 1)
```
* selectExpr(exprs: String*) 做字段的筛选
```
df.selectExpr("name","name as names","upper(name)","age+1").show();
```
* sort(sortExprs: Column*) 排序 df.sort(df("age").desc).show(); 默认是asc
* unionAll(other:Dataframe) 合并 df.unionAll(ds).show();
* withColumnRenamed(existingName: String, newName: String) 修改列表 
```
df.withColumnRenamed("name","names").show();
```
* withColumn(colName: String, col: Column) 增加一列
```
df.withColumn("aa",df("name")).show();
```