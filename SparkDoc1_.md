# Spark SQL, DataFrames and Datasets Guide
[对应网址](http://spark.apache.org/docs/latest/sql-programming-guide.html#datasets-and-dataframes)

[Sqlcontext](http://spark.apache.org/docs/1.4.0/api/scala/index.html#org.apache.spark.sql.SQLContext)

[Dataset](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset)

[functions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$)

------

# **综述，Overview**
 Spark SQL是用于结构化数据处理的Spark模块。SQL和DATAsetSQL可以与Spark SQL实现交互。当计算结果时，使用相同的执行引擎，而不依赖于您正在使用的API/语言来表示计算。

## SQL
 Sparksql可读取中的数据，最终返回Dataset或者DataFrame格式数据
## Datasets and DataFrames
 Dataset是数据的分布式集合。Dataset是Spark 1.6中添加的一个新接口，它提供了RDDs(强类型，使用强大的lambda函数的能力)的优点和Spark SQL优化执行引擎的优点。Dataset API在Scala和Java中可用。
 DataSet是带有类型的（typed），例：DataSet<Persono>。取得每条数据某个值时，使用类似person.getName()这样的API，可以保证类型安全。而Dataframe是无类型的。
 

 DataFrame是组织到指定列中的数据集。可以将其看成关系型数据库中的table或者python中的DataFrame类型。dataframe可以从大量的数据源构建，例如:结构化数据文件、Hive中的表、外部数据库或现有的RDDs。DataFrame API可以在Scala、Java、Python和R中使用。**在Scala和Java中，DataFrame由一行数据集表示。在Scala API中，DataFrame只是Dataset[Row]的别名。而在Java API中，用户需要使用Dataset来表示一个DataFrame。**DataFrame带有schema,schema定义了每行数据的“数据结构”，就像关系型数据库中的“列”，schema指定了某个DataFrame有多少列。

# **DataFrom API**
### **def read: DataFrameReader**
返回一个DataFrameReader，它可以作为DataFrame读取数据。包括*spark.read.json     spark.read.parquet*

### **def printSchema(): Unit**
树格式打印

### **def select(col: String, cols: String\*): DataFrame**
使用列名选择某一列
```scala
df.select("name")   
df.select($"name",$"Age"+1)
```
### **def filter(condition: Column): DataFrame**
使用给出的条件过滤记录
```scala
// The following are equivalent:
peopleDf.filter($"age" > 15)
peopleDf.where($"age" > 15)
peopleDf($"age" > 15)
```
### **def groupBy(cols: Column\*): GroupedData**
使用指定列进行分组聚合
```scala
// Compute the average for all numeric columns grouped by department.
df.groupBy($"department").avg()

// Compute the max age and average salary, grouped by department and gender.
df.groupBy($"department", $"gender").agg(Map(
  "salary" -> "avg",
  "age" -> "max"
))
```
# **临时视图，TempView**
 Spark SQL中的临时视图是会话范围的，如果创建它的会话终止，临时视图就会消失。如果想拥有一个在所有会话之间共享的临时视图，并且在Spark应用程序终止之前保持活跃，那么可以创建一个全局临时视图。全局临时视图绑定到系统保留的数据库global_temp，我们必须使用限定名引用它
 ## 临时视图创建例子
 ```scala
df.createOrReplaceTempView("people")

val sqlDF = spark.sql("SELECT * FROM people")
sqlDF.show()
```
## 全局视图创建例子
```scala
// Register the DataFrame as a global temporary view
df.createGlobalTempView("people")

// Global temporary view is tied to a system preserved database `global_temp`
spark.sql("SELECT * FROM global_temp.people").show()

// Global temporary view is cross-session
spark.newSession().sql("SELECT * FROM global_temp.people").show()
```

# **创建Datasets**
```scala
case class Person(name: String, age: Long)
//case,表示自动创建类

// Encoders are created for case classes
val caseClassDS = Seq(Person("Andy", 32)).toDS()
caseClassDS.show()
// +----+---+
// |name|age|
// +----+---+
// |Andy| 32|
// +----+---+

// Encoders for most common types are automatically provided by importing spark.implicits._
val primitiveDS = Seq(1, 2, 3).toDS()
primitiveDS.map(_ + 1).collect() // Returns: Array(2, 3, 4)

// DataFrames can be converted to a Dataset by providing a class. Mapping will be done by name
val path = "examples/src/main/resources/people.json"
val peopleDS = spark.read.json(path).as[Person]
peopleDS.show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

# **RDD与Dataset的转换**
Spark SQL有两种不同的方法将已存在的RDDs转换成DataSets。
1、使用反射来获取包含特定对象类型的RDD内的schema。
当已知schema的时，使用基于反射的方法会使得代码更简洁，效果更好。
2、通过编程接口创建schema，并将其应用到已有的RDD上。
当运行时才知道列和列类型的情况下，允许你创建DataSets。


