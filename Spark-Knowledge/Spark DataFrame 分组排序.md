```
val window = Window.partitionBy("user_log_acct").orderBy($"probability".desc)
df.withColumn("rn",row_number().over(window))
```
## topN
```
df.withColumn("rn",row_number().over(window)).where($"rn" <= n)
```