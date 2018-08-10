# Spark SQL, DataFrames and Datasets Guide
[对应网址](http://spark.apache.org/docs/latest/sql-programming-guide.html#datasets-and-dataframes)

[Sqlcontext](http://spark.apache.org/docs/1.4.0/api/scala/index.html#org.apache.spark.sql.SQLContext)

[Dataset](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset)

[functions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$)

[部分官方文档翻译](https://blog.csdn.net/u014459326/article/details/53467946)


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
```scala
/ For implicit conversions from RDDs to DataFrames
import spark.implicits._

// Create an RDD of Person objects from a text file, convert it to a Dataframe
val peopleDF = spark.sparkContext
  .textFile("examples/src/main/resources/people.txt")
  .map(_.split(","))
  .map(attributes => Person(attributes(0), attributes(1).trim.toInt))
  .toDF()
// Register the DataFrame as a temporary view
peopleDF.createOrReplaceTempView("people")

// SQL statements can be run by using the sql methods provided by Spark
val teenagersDF = spark.sql("SELECT name, age FROM people WHERE age BETWEEN 13 AND 19")

// The columns of a row in the result can be accessed by field index
teenagersDF.map(teenager => "Name: " + teenager(0)).show()
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+

// or by field name
teenagersDF.map(teenager => "Name: " + teenager.getAs[String]("name")).show()
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+

// No pre-defined encoders for Dataset[Map[K,V]], define explicitly
implicit val mapEncoder = org.apache.spark.sql.Encoders.kryo[Map[String, Any]]
//将String用Any表示
// Primitive types and case classes can be also defined as
// implicit val stringIntMapEncoder: Encoder[Map[String, Any]] = ExpressionEncoder()

// row.getValuesMap[T] retrieves multiple columns at once into a Map[String, T]
teenagersDF.map(teenager => teenager.getValuesMap[Any](List("name", "age"))).collect()
// Array(Map("name" -> "Justin", "age" -> 19))
```

```scala
import org.apache.spark.sql.types._

// Create an RDD
val peopleRDD = spark.sparkContext.textFile("examples/src/main/resources/people.txt")

// The schema is encoded in a string
val schemaString = "name age"

// Generate the schema based on the string of schema
val fields = schemaString.split(" ")
  .map(fieldName => StructField(fieldName, StringType, nullable = true))
val schema = StructType(fields)

// Convert records of the RDD (people) to Rows
val rowRDD = peopleRDD
  .map(_.split(","))
  .map(attributes => Row(attributes(0), attributes(1).trim))

// Apply the schema to the RDD
val peopleDF = spark.createDataFrame(rowRDD, schema)

// Creates a temporary view using the DataFrame
peopleDF.createOrReplaceTempView("people")

// SQL can be run over a temporary view created using DataFrames
val results = spark.sql("SELECT name FROM people")

// The results of SQL queries are DataFrames and support all the normal RDD operations
// The columns of a row in the result can be accessed by field index or by field name
results.map(attributes => "Name: " + attributes(0)).show()
// +-------------+
// |        value|
// +-------------+
// |Name: Michael|
// |   Name: Andy|
// | Name: Justin|
// +-------------+
```

# **聚和(Aggregations)**
常用的聚合函数包括count(), countDistinct(), avg(), max(), min()，除此之外，用户也可自行设定函数。这些函数的本意是为DataFrames设计的。
## **无类型定义的聚合函数**
必须继承自UserDefinedAggregateFunction 类

# **数据源**
Spark SQL的DataFrame接口支持多种数据源的操作。一个DataFrame可以进行RDDs方式的操作，也可以被注册为临时表。把DataFrame注册为临时表之后，就可以对该DataFrame执行SQL查询。
## **载入保存方法**
Spark SQL的默认数据源为Parquet格式。数据源为Parquet文件时，Spark SQL可以方便的执行所有的操作。修改配置项spark.sql.sources.default，可修改默认数据源格式。
```scala
val df = sqlContext.read.load("examples/src/main/resources/users.parquet")
df.select("name", "favorite_color").write.save("namesAndFavColors.parquet")
```
## **手动指定数据源格式**
当数据源格式不是parquet格式文件时，需要手动指定数据源的格式。
如：json, parquet, jdbc, orc, libsvm, csv, text
```scala
//加载json数据的例子
val df = sqlContext.read.format("json").load("examples/src/main/resources/people.json")
df.select("name", "age").write.format("parquet").save("namesAndAges.parquet")
```
```scala
//加载csv数据的例子
val peopleDFCsv = spark.read.format("csv")
  .option("sep", ";")
  .option("inferSchema", "true")
  .option("header", "true")
  .load("examples/src/main/resources/people.csv")
  ```
除了上面load的方法外，也可以使用查询方式转成DataFrames格式
```scala
val sqlDF = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")
```
## **保存模式**
|Scala/Java| Any Language	| Meaning|
|:----------:|:----------:|:-------:|
|SaveMode.ErrorIfExists (default)| "error" or "errorifexists" (default)|当将DataFrame保存到数据源时，如果数据已经存在，就会抛出异常。
| SaveMode.Append|"append"|当将DataFrame保存到数据源时，如果数据/表已经存在，那么DataFrame的内容将追加到现有数据中。|
| SaveMode.Overwrite|"overwrite"|覆盖模式意味着在将DataFrame保存到数据源时，如果数据/表已经存在，那么现有的数据将被DataFrame的内容覆盖。|
|SaveMode.Ignore|"ignore"|忽略模式意味着，当将DataFrame保存到数据源时，如果数据已经存在，则save操作将不会保存DataFrame的内容，也不会更改现有数据。|

## **保存持久表**
还可以使用saveAsTable命令将dataframe作为持久化表保存到Hive metastore。注意，使用此特性不需要现有的Hive部署。Spark将为您创建一个默认的本地Hive metastore(使用Derby)。与createOrReplaceTempView命令不同，saveAsTable将实现DataFrame的内容，并创建一个指向Hive metastore中的数据的指针。只要您保持与同一个转移程序的连接，即使在您的Spark程序重新启动之后，持久表仍然存在。

# **JDBC与其他数据库的互动**
Spark SQL支持使用JDBC访问其他数据库。
在使用JDBC连接其他数据库时，需要在spark类路径中包含特定数据库的JDBC驱动程序。例如,与postgresql的连接，需要在启动spark时加入后面的程序
```bash
bin/spark-shell --driver-class-path postgresql-9.4.1207.jar --jars postgresql-9.4.1207.jar
```
```scala
//scala例子
// Note: JDBC loading and saving can be achieved via either the load/save or jdbc methods
// Loading data from a JDBC source
//方法一
val jdbcDF = spark.read
  .format("jdbc")
  .option("url", "jdbc:postgresql:dbserver")
  .option("dbtable", "schema.tablename")
  .option("user", "username")
  .option("password", "password")
  .load()
//方法二
val connectionProperties = new Properties()
connectionProperties.put("user", "username")
connectionProperties.put("password", "password")
val jdbcDF2 = spark.read
  .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties)
  
// Specifying the custom data types of the read schema
connectionProperties.put("customSchema", "id DECIMAL(38, 0), name STRING")
val jdbcDF3 = spark.read 
  .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties)

// Saving data to a JDBC source
jdbcDF.write
  .format("jdbc")
  .option("url", "jdbc:postgresql:dbserver")
  .option("dbtable", "schema.tablename")
  .option("user", "username")
  .option("password", "password")
  .save()

jdbcDF2.write
  .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties)

// Specifying create table column data types on write
jdbcDF.write
  .option("createTableColumnTypes", "name CHAR(64), comments VARCHAR(1024)")
  .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties)
```
