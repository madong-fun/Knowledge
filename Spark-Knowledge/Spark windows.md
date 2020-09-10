## 求移动平均

```
val rankSpec = Window.partitionBy("seller_id").orderBy(orders("order_date")).rowsBetween(-1, 0)
val shopOrderRank = orders.withColumn("avg sum", avg("price").over(rankSpec))
```

## 分组排序

```
val window = Window.partitionBy("user_log_acct").orderBy($"probability".desc)
df.withColumn("rn",row_number().over(window))
```
## topN
```
df.withColumn("rn",row_number().over(window)).where($"rn" <= n)
```