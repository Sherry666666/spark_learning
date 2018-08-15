# **Machine Learning Library (MLlib)**
-----
[官方文档](http://spark.apache.org/docs/latest/ml-guide.html)

[参考文档](https://blog.csdn.net/liulingyuan6/article/details/53582300)

[Piplines参考文档](https://blog.csdn.net/yhao2014/article/details/52235729)

[scikit-learn](http://scikit-learn.org/stable/)

[基于DataFrame的MLlib之API](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.ml.package)

[安装eclipse for scala](http://scala-ide.org/resources/images/sdk-4.0.0.png)


----
MLlib是Spark的机器学习库。它提供的工具如下：
* ML 算法：公共学习算法，例如分类、回归、聚类和协同过滤。
* 特征工程（Featurization）:特征提取、转换、降维、特征选择。
* 管道（Pipelines）:构造、评估和调整的管道的工具。
* 存储（Persistence）：保存和加载算法，模型及管道。
* 实用工具：线性代数，统计，数据处理等。
#### 其中，基于DataFrom的API为主要的API。基于MLlib rdd的API现在处于维护模式。
## **Basic Statistics(基础统计)**
### **相关性（Correlation）**
MLlib提供了多向量之间的相关性的计算。目前支持的相关性的计算包括皮尔逊相关系数（Pearson）和斯皮尔曼等级相关系数（Spearman）
```scala
import org.apache.spark.ml.linalg.{Matrix, Vectors}
import org.apache.spark.ml.stat.Correlation
import org.apache.spark.sql.Row

//建立列表
val data = Seq(
  Vectors.sparse(4, Seq((0, 1.0), (3, -2.0))),
  Vectors.dense(4.0, 5.0, 0.0, 3.0),
  Vectors.dense(6.0, 7.0, 0.0, 8.0),
  Vectors.sparse(4, Seq((0, 9.0), (3, 1.0)))
)

val df = data.map(Tuple1.apply).toDF("features")
\\默认为皮尔逊相关系数
val Row(coeff1: Matrix) = Correlation.corr(df, "features").head
println(s"Pearson correlation matrix:\n $coeff1")

\\指定为斯皮尔曼等级相关系数
val Row(coeff2: Matrix) = Correlation.corr(df, "features", "spearman").head
println(s"Spearman correlation matrix:\n $coeff2")
```
### **假设检验（Hypothesis testing）**
假设检验是统计中判断结果是否具有统计学意义的有力工具。目前MLlib中支持使用皮尔逊的ChiSquareTest。
要求测试数据必须具有label和features
```scala
import org.apache.spark.ml.linalg.{Vector, Vectors}
import org.apache.spark.ml.stat.ChiSquareTest
//第一项为标签，第二项为特征
val data = Seq(
  (0.0, Vectors.dense(0.5, 10.0)),
  (0.0, Vectors.dense(1.5, 20.0)),
  (1.0, Vectors.dense(1.5, 30.0)),
  (0.0, Vectors.dense(3.5, 30.0)),
  (0.0, Vectors.dense(3.5, 40.0)),
  (1.0, Vectors.dense(3.5, 40.0))
)

val df = data.toDF("label", "features")
val chi = ChiSquareTest.test(df, "features", "label").head
println(s"pValues = ${chi.getAs[Vector](0)}")
println(s"degreesOfFreedom ${chi.getSeq[Int](1).mkString("[", ",", "]")}")
println(s"statistics ${chi.getAs[Vector](2)}")
```
## **管道（ML Pipelines）**
ML提供了建立在DataFrames基础上的API，帮助用户创建和优化实用的机器学习管道。
### **pipelines中的主要概念**
MLlib对机器学习算法的api进行了标准化。这可以方便用户构建和调试机器学习流水线。
* DataFrame是来自Spark SQL的ML DataSet 可以存储一系列的数据类型。这个ML API使用来自Spark SQL的DataFrame作为ML数据集，它可以容纳各种数据类型。
* Transformer：Transformer是一种能够把一个DataFrame转换为另一个DataFrame的算法。例如，一个ML模型就是一个可以将一个包含特征的DataFrame转换为包含预测值的DataFrame的Transformer。
* Estimator：**Estimator是**能够使用在DataFrame上用以产生一个Transformer的**算法**。例如，一个学习算法就是一个能够在DataFrame上进行训练并产生模型的Estimator。
* Pipline：Pipline可以将多个Transformer和Estimator链接到一起形成一个ML工作流。
* Parameter：所有的Transformer和Estimator共享一个通用的指定参数的API。

#### Pipline与Estimator关系例子：LogisticRegression这样的学习算法就是一个Estimator，它调用fit()类训练一个LogisticRegressionModel，LogisticRegressionModel是一个模型，因此也是一个Transformer。
#### **Estimator, Transformer, and Param**
```scala
import org.apache.spark.ml.classification.LogisticRegression
import org.apache.spark.ml.linalg.{Vector, Vectors}
import org.apache.spark.ml.param.ParamMap
import org.apache.spark.sql.Row

// Prepare training data from a list of (label, features) tuples.
val training = spark.createDataFrame(Seq(
  (1.0, Vectors.dense(0.0, 1.1, 0.1)),
  (0.0, Vectors.dense(2.0, 1.0, -1.0)),//Creates a dense vector from its values
  (0.0, Vectors.dense(2.0, 1.3, 1.0)),
  (1.0, Vectors.dense(0.0, 1.2, -0.5))
)).toDF("label", "features")

// Create a LogisticRegression instance. This instance is an Estimator.
val lr = new LogisticRegression()
// Print out the parameters, documentation, and any default values.
println(s"LogisticRegression parameters:\n ${lr.explainParams()}\n")

// We may set parameters using setter methods.
lr.setMaxIter(10)//Set the maximum number of iterations
  .setRegParam(0.01)//Set the regularization parameter

// Learn a LogisticRegression model. This uses the parameters stored in lr.
val model1 = lr.fit(training)
// Since model1 is a Model (i.e., a Transformer produced by an Estimator),
// we can view the parameters it used during fit().
// This prints the parameter (name: value) pairs, where names are unique IDs for this
// LogisticRegression instance.
println(s"Model 1 was fit using parameters: ${model1.parent.extractParamMap}")

// We may alternatively specify parameters using a ParamMap,
// which supports several methods for specifying parameters.
val paramMap = ParamMap(lr.maxIter -> 20)
  .put(lr.maxIter, 30)  // Specify 1 Param. This overwrites the original maxIter.
  .put(lr.regParam -> 0.1, lr.threshold -> 0.55)  // Specify multiple Params.

// One can also combine ParamMaps.
val paramMap2 = ParamMap(lr.probabilityCol -> "myProbability")  // Change output column name.
val paramMapCombined = paramMap ++ paramMap2

// Now learn a new model using the paramMapCombined parameters.
// paramMapCombined overrides all parameters set earlier via lr.set* methods.
val model2 = lr.fit(training, paramMapCombined)
println(s"Model 2 was fit using parameters: ${model2.parent.extractParamMap}")

// Prepare test data.
val test = spark.createDataFrame(Seq(
  (1.0, Vectors.dense(-1.0, 1.5, 1.3)),
  (0.0, Vectors.dense(3.0, 2.0, -0.1)),
  (1.0, Vectors.dense(0.0, 2.2, -1.5))
)).toDF("label", "features")

// Make predictions on test data using the Transformer.transform() method.
// LogisticRegression.transform will only use the 'features' column.
// Note that model2.transform() outputs a 'myProbability' column instead of the usual
// 'probability' column since we renamed the lr.probabilityCol parameter previously.
model2.transform(test)
  .select("features", "label", "myProbability", "prediction")
  .collect()
  .foreach { case Row(features: Vector, label: Double, prob: Vector, prediction: Double) =>
    println(s"($features, $label) -> prob=$prob, prediction=$prediction")
  }
  ```
  内部使用到的API详解：

  [import org.apache.spark.ml.linalg.Vector](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.ml.linalg.Vector) : 表示一个索引为整形，数值为double型的数字向量。

  [import org.apache.spark.ml.linalg.Vectors](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.ml.linalg.Vectors$)是import org.apache.spark.ml.linalg.Vector的[Factory Method](https://blog.csdn.net/u010373266/article/details/53764737)

  [org.apache.spark.ml.classification.LogisticRegression](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.ml.classification.LogisticRegression):支持多分类问题（softmax）和二分类问题。

  

[org.apache.spark.ml.param.ParamMap](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.ml.param.ParamMap):一个从参数到值的映射

[org.apache.spark.sql.Row](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Row):表示关系运算符的一行输出




## **提取、转换与特征选择**
## **分类与回归**
## **聚类**
## **协同过滤**
关于协同过滤的原理部分可参照此链接：
[协同过滤原理](https://blog.csdn.net/yimingsilence/article/details/54934302)

## **频繁模式挖掘**
## **模型选择和调优**