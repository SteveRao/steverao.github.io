---
layout:     post
title:      "SparkSql技术轮廓"
subtitle:   "SparkSql大数据计算框架"
date:       2018-10-26
author:     "ZihaoRao"
catalog: true
header-img: "img/post-bg-bigdata.jpg"
tags: 大数据
---


### **总体概述**
---
> This article is written in Chinese. If necessary, please consider using [Google Translate](http://translate.google.com/translate?hl=en&sl=auto&tl=en&u=https://steverao.github.io/2018/10/26/SparkSql-Technology-Profile/)
>
> Apache Spark 是一个快速、多用途的集群计算系统。它提供了Scala、Java、Python和R语言等高级语言的API。本节主要介绍Spark中国处理结构化数据的工具——`SparkSql`，它的定位就是在内存中对结构化数据进行复杂的逻辑处理操作，它不仅支持多种数据源（Hive,Json,CSV和Parquet）的读取访问，还在兼备传统Sql语句规范同时，提供了`Dataframe`数据抽象来提供更多复杂数据处理操作，本文正是对SparkSql技术轮廓的总结。





### **SparkSql依赖配置**
---
- 在使用SparkSql技术前，需要在maven项目的`pom.xml`文件中添加如下依赖：

  ```xml
  <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-sql_2.11</artifactId>
      <version>2.1.0</version>
  </dependency> 
  ```


### **初始化SparkSql**
---

- `SparkSession`类作为SparkSql程序的入口类(SparkSession是对Spark早期版本中的SqlContext和HiveContext的组合，继承了它们所具有的功能并进行了一些扩展)，描述了程序的相关基本信息。编写一个SparkSql程序首先要创建一个SparkSession对象。

- `SparkSql`初始化代码：
```java
SparkSession spark = SparkSession.builder()
.appName("Test")
.master("local")
.getOrCreate();
```

- 其中appName参数是一个在集群UI上展现应用程序的名称，master描述的是一个集群的URL(cluster URL)。它有几种设置模式，由于我们这里是在本地搭建的一个测试小应用，所以`appName="Test"`，`master="local"`。





### **DataFrame分布式数据集**
---

- **基本定义**：`DataFrame`是SparkSql中专门为处理大数据而提供的数据处理抽象单元——弹性分布式数据集。其中，***弹性：***指该数据集具有容错性和可恢复性，***分布式：***指该数据集中的操作计算任务可以被分成若干个切片并分配给集群中的机器进行单独计算，最后再合并结果，为大数据的处理提供了强有力地计算支持。



- **结构描述**：其结构就像传统关系型数据库中的表，以列的形式构成，包含列名和列数据以及列结构信息等（详情见下图，其是一个DataFrame打印出的结构）但与此同行，它在数据读取等操作上进行了很多的优化，例如，可以按列读取字段。相比于传统关系数据库中的表在数据处理上的性能有很大提升。
```java
//DataFrame打印结果
+---+------+-----+
|age|height| name|
+---+------+-----+
| 17|   180|steve|
| 18|   175| mike|
| 19|   165|james|
+---+------+-----+
```



### **DataFrame的构建**
---

- 在Spark中主要有两种方式构造DataFrame：从现有**数据源中导入**和通过**`RDD`和`Schema`构造**

- **现有数据源导入：**主要是指通过`SparkSession`类从外部的Hive表、CSV和Json文件中读取源数据，然后自动识别数据中的结构从而构造`DataFrame`。（详细介绍见最后一节，与外部数据源的连接）

- **RDD和Schema构造：**其中的`RDD`（*Resilient Distributed Datasets*）分布式弹性数据集指的是Spark低版本中所提供的数据处理抽象，相比于DataFrame而言，其缺少列结构信息。所以通过添加`Schema`列结构信息就可以由SparkSession类的createDataFrame()方法构造出对应的DataFrame（**特此说明**，在Java版本中Spark默认使用`Dataset<Row>`指代DataFrame）

- [***相关案例***](http://spark.apachecn.org/docs/cn/2.2.0/sql-programming-guide.html#rdd%E7%9A%84%E4%BA%92%E6%93%8D%E4%BD%9C%E6%80%A7)



### **DataFrame上的操作**
---

- DataFrame主要具有的操作类型：**转化操作**和**行动操作**

- **背景介绍：**其实DataFrame中的两类操作方式都是来源于Spark中的RDD，但由于DataFrame

&emsp;&emsp;是在RDD基础上新添加的，所以继承了RDD中的两种操作，在性能和可读性等方面提供了更好的数据处理效果。

- **转化操作：**“转化”两字突出了转化操作是从一个DataFrame转化成另外一个DataFrame。

- **行动操作：**而行动操作仅仅只是对DataFrame的进行实际的计算。

- **两者区别：**编译器在处理转化操作时会出现“惰性运算”，即当RDD执行转化操作时，计算不会立即执行，只有当RDD执行行动操作时计算才会提交并执行。这也是在为大数据处理提供了计算性能上的保证。

- [***DataFrame操作API文档介绍***](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html#transformations%E8%BD%AC%E6%8D%A2)（以RDD中的API进行展示，DataFrame中都有相关方法）



### **用户自定义聚集函数**
---

- 用户自定义的聚集函数有：***UDF***（*User-Defined Functions*）、***UDAF***（*User-Defined Aggregate Functions*）和***UDTF***（*User-Defined Table-Generating Functions*）三类。

- 三类函数分别对应的使用场景：

   ***UDF***:主要解决的是一些基本的、逻辑简单的问题，如：使用Java1.8中的lamdba表达式就可直接完成一个类似于isNull()的用户自定义函数。

   ***UDAF***:主要用来编写一些内置聚集函数以外的用户自定义聚集函数，类似于sum、max和avg等类似的功能。通过继承`UserDefinedAggregateFunction`基类，并实现其中的特定方法来实现。

   ***UDTF***:主要用来编写完成对表的某列进行拆分等生成表的操作。通过继承`GenericUDTF`基类并实现其中的特定方法实现需要的逻辑功能。 

- **UDF示例**

   ```java
   //通过SparkSession类对象调用udf()方法创建isNull（）函数
   spark.udf().register("isNull",    //函数名，和下行函数逻辑代码
   (String field, String defaultValue) -> field==null? defaultValue : field,
   DataTypes.StringType);

   //调用代码
   Dataset<Row> result = spark.sql("select a,isNull(b,'null') as b,c from table1");
   ```

   ***[其他示例](http://spark.apachecn.org/docs/cn/2.2.0/sql-programming-guide.html#aggregations)***



### **连接外部数据源**
---

- **读取文件数据**
```java
//调用SparkSession对象的read()方法读取json文件构造DataFrame
Dataset<Row> in =spark.read().json("**/user.json");
in.show();
//结果显示：
+---+------+-----+
|age|height| name|
+---+------+-----+
| 17|   180|steve|
| 18|   175| mike|
| 19|   165|james|
+---+------+-----+
```
```java
//读取csv文件
Dataset<Row> in = spark.read().format("csv").csv("**/user.csv");
in.show();
+---+------+-----+
|age|height| name|
+---+------+-----+
| 17|   180|steve|
| 18|   175| mike|
| 19|   165|james|
+---+------+-----+
```



- **数据库数据**
```java
SparkConf conf = new SparkConf(true);
SparkContext sc = new SparkContext("local", "Test", conf);
SparkSession spark = SparkSession.builder()
.config("spark.cassandra.connection.host","127.0.0.1")//连接本地单节点
.appName("Test")
.master("local")//本地单节点
.getOrCreate();
Dataset<Row> in=spark.read().format("org.apache.spark.sql.cassandra")
.option("table", "student")//数据库表名
.option("keyspace", "demo1").load();//数据库keyspace名
in.show();
//结果显示
+---+------------------+-----+-----------+
| id|           address| name|nationality|
+---+------------------+-----+-----------+
|  1|                []|james|    America|
|  2|[America, NewYork]| mike|    America|
+---+------------------+-----+-----------+
```

### 小结
---
&emsp;&emsp;本文仅是对最近学习SparkSql部分知识轮廓的一个总结（并未展示太多细节，当中所使用的API是Java），其中还有一些细节和重要的点需要后续继续攻关，如其中最为重要的DataFrame操作部分！


### **参考资料**
---
- [Spark大数据之DataFrame和Dataset](https://zhuanlan.zhihu.com/p/29830732)
- [Spark编程指南](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html)
- [SparkRDD中转化操作和行动操作](https://blog.csdn.net/YQlakers/article/details/76056413)
- [SparkSql,DataFrames and Datasets Guide](http://spark.apachecn.org/docs/cn/2.2.0/sql-programming-guide.html#spark-sql-dataframes-and-datasets-guide)
- [IBM专家深入浅出降解Spark2](http://www.10tiao.com/html/157/201607/2653159975/1.html)

