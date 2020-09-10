## 示例数据抽样
```
{"time":1534996820000,"msg":"pin=lhx13572524232;datetime=1534996805438;refer=2;skuIdNumList=29263311633_0;mqType=2 ","host":"9b90691c-10be-495b-b785-288214d389ff","ip":"11.18.10.17","app-name":"cart.sdk","file":"cartmarket.log","microsecond":1534996820848095}
 ","host":"e9dc80ab-001b-48c6-afbd-ea424da6824b","ip":"11.18.15.203","app-name":"cart.sdk","file":"cartmarket.log","microsecond":1534996820511030}
{"time":1534996820000,"msg":"price=28922953907_14.80;addr=2-2813-51976-0;pin=shen30215;datetime=1534996819716;refer=3;skuIdNumList=28922953907_1;mqType=1 ","host":"1ed0bbe3-59fc-4fed-8f95-67b105bbf0e8","ip":"11.17.109.48","app-name":"cart.sdk","file":"cartmarket.log","microsecond":1534996820305650}
{"time":1534996820000,"msg":"price=2655649_11.90;addr=18-1586-1589-0;pin=jd_4d3108c5727ed;datetime=1534996819257;refer=2;skuIdNumList=2655649_2;mqType=1 ","host":"2179a880-f69e-4ae5-9b29-77069953b9bb","ip":"10.173.97.5","app-name":"cart.sdk","file":"cartmarket.log","microsecond":1534996820689113}
{"time":1534996821000,"msg":"pin=441581ZMD666;datetime=1534996807264;refer=1;skuIdNumList=4928604_0;mqType=2 ","host":"436ef1e8-4edb-4109-9b45-cf1611319b0c","ip":"11.26.131.44","app-name":"cart.sdk","file":"cartmarket.log","microsecond":1534996821177021}
{"time":1534996820000,"msg":"pin=jd_4086ff855f82b;datetime=1534996818865;refer=2;skuIdNumList=3479262_3;mqType=3 ","host":"1432fcd0-3f25-4d9e-ac7d-91a9ae084b03","ip":"10.173.65.107","app-name":"cart.sdk","file":"cartmarket.log","microsecond":1534996820248729}
{"time":1534996820000,"msg":"pin=jd_5ccc45f664e53;datetime=1534996820248;refer=2;skuIdNumList=26025462771_1,30135887477_1,31089916577_1;mqType=5 ","host":"1432fcd0-3f25-4d9e-ac7d-91a9ae084b03","ip":"10.173.65.107","app-name":"cart.sdk","file":"cartmarket.log","microsecond":1534996820943240}
{"time":1534996820000,"msg":"pin=wdurmkgfWnfIDe;datetime=1534996818498;refer=5;skuIdNumList=25730603113_1;mqType=5 ","host":"13b8fb97-4ae9-4ef1-8590-77f7fea8d03f","ip":"11.26.131.224","app-name":"cart.sdk","file":"cartmarket.log","microsecond":1534996820354629}
```

## get_json_object
方法始于spark1.6版本，从一个json 字符串中根据指定的json 路径抽取一个json 对象。也可以从dataset中取出部分数据，然后抽取部分字段组装成新的json 对象

```
  /**
   * Extracts json object from a json string based on json path specified, and returns json string
   * of the extracted json object. It will return null if the input json string is invalid.
   *
   * @group collection_funcs
   * @since 1.6.0
   */
  def get_json_object(e: Column, path: String): Column = withExpr {
    GetJsonObject(e.expr, lit(path).expr)
  }
```
执行代码:
```
data.select(get_json_object($"value","$.msg").alias("msg"),get_json_object($"value","$.ip").alias("ip")).show()
```
结果如下：
```
+--------------------+--------------+
|                 msg|            ip|
+--------------------+--------------+
|pin=220947922-676...| 10.185.174.62|
|price=15505010577...| 10.190.50.104|
|pin=jd_44f18eff85...|10.186.219.169|
|price=13261996959...| 10.187.239.26|
|pin=jd_427dfe6e4e...| 10.186.48.116|
|price=24685339852...| 10.191.94.162|
|pin=嘲男002;datetim...|  11.26.163.10|
|price=7567797_18....| 10.191.94.194|
|pin=jd_71e896587a...| 10.181.53.209|
|price=12318902396...|  11.26.68.246|
+--------------------+--------------+
```

## from_json
与get_json_object不同的是该方法，使用schema去抽取单独列。在dataset的api select中使用from_json()方法，我可以从一个json 字符串中按照指定的schema格式抽取出来作为DataFrame的列

```
  /**
      * json schema
      */
    val schema = new StructType().add("host", StringType).add("time",LongType)
        .add("ip",StringType).add("msg",StringType)
        .add("app-name",StringType).add("file",StringType).add("microsecond",LongType)
    /**
      * 解析json
      */
    val dataFrame = data.select(from_json($"value",schema)as "json").select($"json.host",$"json.ip",$"json.msg")
    
```
结果如下：

```
+--------------------+--------------+--------------------+
|                host|            ip|                 msg|
+--------------------+--------------+--------------------+
|  host-10-185-174-62| 10.185.174.62|pin=220947922-676...|
|  host-10-190-50-104| 10.190.50.104|price=15505010577...|
| host-10-186-219-169|10.186.219.169|pin=jd_44f18eff85...|
|  host-10-187-239-26| 10.187.239.26|price=13261996959...|
|  host-10-186-48-116| 10.186.48.116|pin=jd_427dfe6e4e...|
|  host-10-191-94-162| 10.191.94.162|price=24685339852...|
|bb5afa51-e5a6-4a0...|  11.26.163.10|pin=嘲男002;datetim...|
|  host-10-191-94-194| 10.191.94.194|price=7567797_18....|
|  host-10-181-53-209| 10.181.53.209|pin=jd_71e896587a...|
|3ac5f93f-1163-4bf...|  11.26.68.246|price=12318902396...|
+--------------------+--------------+--------------------+
```

## to_json
to_json()将获取的数据转化为json格式
接上面的结果继续

```
    /**
      * 设置Index
      */
    val dataX = dataFrame.map(row =>(row.getAs[String]("host"),row.getAs[String]("ip"),row.getAs[String]("msg")))
      .rdd.zipWithUniqueId().map(x => (x._2,x._1._1,x._1._2,x._1._3)).toDF("index","ip","host","msg")

    dataX.select(to_json(struct($"*"))).toDF("message")
```