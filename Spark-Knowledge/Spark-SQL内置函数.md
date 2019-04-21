## 时间函数

1. 获取当前时间

* current_date获取当前日期
* current_timestamp/now()获取当前时间

2. 从日期时间中提取字段

* year,month,day/dayofmonth,hour,minute,second
* dayofweek (1 = Sunday, 2 = Monday, ..., 7 = Saturday),dayofyear
* weekofyear
* trunc截取某部分的日期，参数 ["year", "yyyy", "yy", "mon", "month", "mm"]
* date_format时间格式化

3.日期时间转换

* unix_timestamp返回当前时间的unix时间戳
* from_unixtime将时间戳换算成当前时间，to_unix_timestamp将时间转化为时间戳
* to_date 将字符串转化为日期格式，to_timestamp
* quarter 将1年4等分(range 1 to 4)

4.日期、时间计算

* months_between(timestamp1, timestamp2) 两个日期之间的月数
* add_months返回日期后n个月后的日期
* last_day(date)  input "2015-07-27" returns "2015-07-31"
* next_day(start_date, day_of_week) 参数["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"]
* date_add(start_date, num_days),date_sub(start_date, num_days)
* datediff（两个日期间的天数）



## 字符串函数

1. concat对于字符串进行拼接
2. concat_ws在拼接的字符串中间添加分隔符
3. decode转码
4. encode设置编码格式
5. format_string 格式化字符
6. initcap将每个单词的首字母变为大写，其他字母小写; lower全部转为小写，upper大写
7. length返回字符串的长度
8. levenshtein编辑距离（将一个字符串变为另一个字符串的距离）
9. lpad返回固定长度的字符串，如果长度不够，用某种字符补全，rpad右补全
10. ltrim去除空格或去除开头的某些字符,rtrim右去除，trim两边同时去除
11. regexp_extract 正则提取某些字符串，regexp_replace正则替换
12. repeat复制给的字符串n次
13. instr返回截取字符串的位置/locate
14. space 在字符串前面加n个空格
15. split以某些字符拆分字符串
16. substr截取字符串，substring_index
17. translate 替换某些字符串为
18. get_json_object
19. to_json
20. unhex
21. reverse(str: Column): 将str反转
22. soundex(e: Column): 计算桑迪克斯代码（soundex code）PS:用于按英语发音来索引姓名,发音相同但拼写不同的单词，会映射成同一个码
23. lower(e: Column): 转小写
24. upper(e: Column): 转大写
25. instr(str: Column, substring: String): substring在str中第一次出现的位置

## 窗口函数

cume_dist(): cumulative distribution of values within a window partition

currentRow(): returns the special frame boundary that represents the current row in the window partition.

rank():排名，返回数据项在分组中的排名，排名相等会在名次中留下空位 1,2,2,4。

dense_rank(): 排名，返回数据项在分组中的排名，排名相等会在名次中不会留下空位 1,2,2,3。

row_number():行号，为每条记录返回一个数字 1,2,3,4

percent_rank():returns the relative rank (i.e. percentile) of rows within a window partition.

lag(e: Column, offset: Int, defaultValue: Any): offset rows before the current row

lead(e: Column, offset: Int, defaultValue: Any): returns the value that is offset rows after the current row

ntile(n: Int): returns the ntile group id (from 1 to n inclusive) in an ordered window partition.

unboundedFollowing():returns the special frame boundary that represents the last row in the window partition.

## 数学函数
cos,sin,tan 计算角度的余弦，正弦。。。

sinh,tanh,cosh 计算双曲正弦，正切，。。

acos,asin,atan,atan2 计算余弦/正弦值对应的角度

bin 将long类型转为对应二进制数值的字符串For example, bin("12") returns "1100".

bround 舍入，使用Decimal的HALF_EVEN模式，v>0.5向上舍入，v< 0.5向下舍入，v0.5向最近的偶数舍入。

round(e: Column, scale: Int) HALF_UP模式舍入到scale为小数点。v>=0.5向上舍入，v< 0.5向下舍入,即四舍五入。

ceil 向上舍入

floor 向下舍入

cbrt Computes the cube-root of the given value.

conv(num:Column, fromBase: Int, toBase: Int) 转换数值（字符串）的进制

log(base: Double, a: Column):$log_{base}(a)$

log(a: Column):$log_e(a)$

log10(a: Column):$log_{10}(a)$

log2(a: Column):$log_{2}(a)$

log1p(a: Column):$log_{e}(a+1)$

pmod(dividend: Column, divisor: Column):Returns the positive value of dividend mod divisor.

pow(l: Double, r: Column):$r^l$ 注意r是列

pow(l: Column, r: Double):$r^l$ 注意l是列

pow(l: Column, r: Column):$r^l$ 注意r,l都是列

radians(e: Column):角度转弧度

rint(e: Column):Returns the double value that is closest in value to the argument and is equal to a mathematical integer.

shiftLeft(e: Column, numBits: Int):向左位移

shiftRight(e: Column, numBits: Int):向右位移

shiftRightUnsigned(e: Column, numBits: Int):向右位移（无符号位）

signum(e: Column):返回数值正负符号

sqrt(e: Column):平方根

hex(column: Column):转十六进制

unhex(column: Column):逆转十六进制

## 聚合函数
approx_count_distinct
count_distinct近似值

avg 平均值

collect_list 聚合指定字段的值到list

collect_set 聚合指定字段的值到set

corr 计算两列的Pearson相关系数

count 计数

countDistinct 去重计数 SQL中用法 select count(distinct class)

covar_pop 总体协方差（population covariance）

covar_samp 样本协方差（sample covariance）

first 分组第一个元素

last 分组最后一个元素

grouping

grouping_id

kurtosis 计算峰态(kurtosis)值

skewness 计算偏度(skewness)

max 最大值

min 最小值

mean 平均值

stddev 即stddev_samp

stddev_samp 样本标准偏差（sample standard deviation）

stddev_pop 总体标准偏差（population standard deviation）

sum 求和

sumDistinct 非重复值求和 SQL中用法 同select sum(distinct class)

var_pop 总体方差（population variance）

var_samp 样本无偏方差（unbiased variance）

variance 即var_samp

## 集合函数
array_contains(column,value) 检查array类型字段是否包含指定元素

explode 展开array或map为多行

explode_outer 同explode，但当array或map为空或null时，会展开为null。

posexplode 同explode，带位置索引。

posexplode_outer 同explode_outer，带位置索引。

from_json 解析JSON字符串为StructType or ArrayType，有多种参数形式，详见文档。

to_json 转为json字符串，支持StructType, ArrayType of StructTypes, a MapType or ArrayType of MapTypes。

get_json_object(column,path) 获取指定json路径的json对象字符串。select get_json_object('{"a"1,"b":2}','$.a');

json_tuple(column,fields) 获取json中指定字段值。select json_tuple('{"a":1,"b":2}','a','b');

map_keys 返回map的键组成的array

map_values 返回map的值组成的array

size array or map的长度

sort_array(e: Column, asc: Boolean) 将array中元素排序（自然排序），默认asc。

## 混杂(misc)函数
crc32(e: Column):计算CRC32,返回bigint

hash(cols: Column*):计算 hash code，返回int

md5(e: Column):计算MD5摘要，返回32位，16进制字符串

sha1(e: Column):计算SHA-1摘要，返回40位，16进制字符串

sha2(e: Column, numBits: Int):计算SHA-1摘要，返回numBits位，16进制字符串。numBits支持224, 256, 384, or 512.

## 其他

abs(e: Column)
绝对值

array(cols: Column*)
多列合并为array，cols必须为同类型

map(cols: Column*):
将多列组织为map，输入列必须为（key,value)形式，各列的key/value分别为同一类型。

bitwiseNOT(e: Column):
Computes bitwise NOT.

broadcast[T](df: Dataset[T]): Dataset[T]:
将df变量广播，用于实现broadcast join。如left.join(broadcast(right), "joinKey")

coalesce(e: Column*):
返回第一个非空值

col(colName: String):
返回colName对应的Column

column(colName: String):
col函数的别名

expr(expr: String):
解析expr表达式，将返回值存于Column，并返回这个Column。

greatest(exprs: Column*):
返回多列中的最大值，跳过Null

least(exprs: Column*):
返回多列中的最小值，跳过Null

input_file_name():返
回当前任务的文件名 ？？

isnan(e: Column):
检查是否NaN（非数值）

isnull(e: Column):
检查是否为Null

lit(literal: Any):
将字面量(literal)创建一个Column

typedLit[T](literal: T)(implicit arg0: scala.reflect.api.JavaUniverse.TypeTag[T]):
将字面量(literal)创建一个Column，literal支持 scala types e.g.: List, Seq and Map.

monotonically_increasing_id():
返回单调递增唯一ID，但不同分区的ID不连续。ID为64位整型。

nanvl(col1: Column, col2: Column):
col1为NaN则返回col2

negate(e: Column):
负数，同df.select( -df("amount") )

not(e: Column):
取反，同df.filter( !df("isActive") )

rand():
随机数[0.0, 1.0]

rand(seed: Long):
随机数[0.0, 1.0]，使用seed种子

randn():
随机数，从正态分布取

randn(seed: Long):
同上

spark_partition_id():
返回partition ID

struct(cols: Column*):
多列组合成新的struct column ？？

when(condition: Column, value: Any):
当condition为true返回value，如
`people.select(when(people("gender") === "male", 0).when(people("gender") === "female", 1).otherwise(2))`
如果没有otherwise且condition全部没命中，则返回null.
