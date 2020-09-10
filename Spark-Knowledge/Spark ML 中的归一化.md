org.apache.spark.ml.feature包中包含了4种不同的归一化方法：
* Normalizer
* StandardScaler
* MinMaxScaler
* MaxAbsScaler

## Normalizer
Normalizer的作用范围是每一行，使每一个行向量的范数变换为一个单位范数

```
// 正则化每个向量到1阶范数
val normalizer = new Normalizer()
  .setInputCol("features")
  .setOutputCol("normFeatures")
  .setP(1.0)

val l1NormData = normalizer.transform(dataFrame)

// 将每一行的规整为1阶范数为1的向量，1阶范数即所有值绝对值之和。
+---+--------------+------------------+
| id|      features|      normFeatures|
+---+--------------+------------------+
|  0|[1.0,0.5,-1.0]|    [0.4,0.2,-0.4]|
|  1| [2.0,1.0,1.0]|   [0.5,0.25,0.25]|
|  2|[4.0,10.0,2.0]|[0.25,0.625,0.125]|
+---+--------------+------------------+

// 正则化每个向量到无穷阶范数
val lInfNormData = normalizer.transform(dataFrame, normalizer.p -> Double.PositiveInfinity)

// 向量的无穷阶范数即向量中所有值中的最大值
+---+--------------+--------------+
| id|      features|  normFeatures|
+---+--------------+--------------+
|  0|[1.0,0.5,-1.0]|[1.0,0.5,-1.0]|
|  1| [2.0,1.0,1.0]| [1.0,0.5,0.5]|
|  2|[4.0,10.0,2.0]| [0.4,1.0,0.2]|
+---+--------------+--------------+

```
## StandardScaler

StandardScaler处理的对象是每一列，也就是每一维特征，将特征标准化为单位标准差或是0均值，或是0均值单位标准差。
主要有两个参数可以设置：

- withStd: 默认为true。将数据标准化到单位标准差。
- withMean: 默认为false。是否变换为0均值。

StandardScaler需要fit数据，获取每一维的均值和标准差，来缩放每一维特征。
```
import org.apache.spark.ml.feature.StandardScaler

val scaler = new StandardScaler()
  .setInputCol("features")
  .setOutputCol("scaledFeatures")
  .setWithStd(true)
  .setWithMean(false)

// Compute summary statistics by fitting the StandardScaler.
val scalerModel = scaler.fit(dataFrame)

// Normalize each feature to have unit standard deviation.
val scaledData = scalerModel.transform(dataFrame)

// 将每一列的标准差缩放到1。
+---+--------------+------------------------------------------------------------+
|id |features      |scaledFeatures                                              |
+---+--------------+------------------------------------------------------------+
|0  |[1.0,0.5,-1.0]|[0.6546536707079772,0.09352195295828244,-0.6546536707079771]|
|1  |[2.0,1.0,1.0] |[1.3093073414159544,0.1870439059165649,0.6546536707079771]  |
|2  |[4.0,10.0,2.0]|[2.618614682831909,1.870439059165649,1.3093073414159542]    |
+---+--------------+------------------------------------------------------------+
```

## MinMaxScaler

MinMaxScaler作用同样是每一列，即每一维特征。将每一维特征线性地映射到指定的区间，通常是[0, 1]。
它也有两个参数可以设置：

- min: 默认为0。指定区间的下限。
- max: 默认为1。指定区间的上限。

import org.apache.spark.ml.feature.MinMaxScaler

val scaler = new MinMaxScaler()
  .setInputCol("features")
  .setOutputCol("scaledFeatures")

// Compute summary statistics and generate MinMaxScalerModel
val scalerModel = scaler.fit(dataFrame)

// rescale each feature to range [min, max].
val scaledData = scalerModel.transform(dataFrame)
println(s"Features scaled to range: [${scaler.getMin}, ${scaler.getMax}]")
scaledData.select("features", "scaledFeatures").show

// 每维特征线性地映射，最小值映射到0，最大值映射到1。
+--------------+-----------------------------------------------------------+
|features      |scaledFeatures                                             |
+--------------+-----------------------------------------------------------+
|[1.0,0.5,-1.0]|[0.0,0.0,0.0]                                              |
|[2.0,1.0,1.0] |[0.3333333333333333,0.05263157894736842,0.6666666666666666]|
|[4.0,10.0,2.0]|[1.0,1.0,1.0]                                              |
+--------------+-----------------------------------------------------------+

## MaxAbsScaler

MaxAbsScaler将每一维的特征变换到[-1,1]闭区间上，通过除以每一维特征上的最大的绝对值，它不会平移整个分布，也不会破坏原来每一个特征向量的稀疏性。

```
import org.apache.spark.ml.feature.MaxAbsScaler

val scaler = new MaxAbsScaler()
  .setInputCol("features")
  .setOutputCol("scaledFeatures")

// Compute summary statistics and generate MaxAbsScalerModel
val scalerModel = scaler.fit(dataFrame)

// rescale each feature to range [-1, 1]
val scaledData = scalerModel.transform(dataFrame)
scaledData.select("features", "scaledFeatures").show()

// 每一维的绝对值的最大值为[4, 10, 2]
+--------------+----------------+                                               
|      features|  scaledFeatures|
+--------------+----------------+
|[1.0,0.5,-1.0]|[0.25,0.05,-0.5]|
| [2.0,1.0,1.0]|   [0.5,0.1,0.5]|
|[4.0,10.0,2.0]|   [1.0,1.0,1.0]|
+--------------+----------------+

```
## final
所有4种归一化方法都是线性的变换，当某一维特征上具有非线性的分布时，还需要配合其它的特征预处理方法。