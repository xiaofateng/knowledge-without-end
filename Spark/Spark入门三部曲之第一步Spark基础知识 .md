#Spark入门三部曲之第一步Spark基础知识 
Spark运行环境

Spark 是Scala写的, 运行在JVM上。所以运行环境是Java6或者以上。
如果想要使用 Python API，需要安装Python 解释器2.6版本或者以上。
目前Spark(1.2.0版本) 与Python 3不兼容。
Spark下载

下载地址：http://spark.apache.org/downloads.html，选择Pre-built for Hadoop 2.4 and later 这个包，点击直接下载，这会下载一个spark-1.2.0-bin-hadoop2.4.tgz的压缩包
搭建Spark不需要Hadoop，如果你有hadoop集群或者hdfs，你可以下载相应的版本。
解压：tar -zxvf spark-1.2.0-bin-hadoop2.4.tgz
Spark的Shells

Spark的shell使你能够处理分布在集群上的数据(这些数据可以是分布在硬盘上或者内存中)。
Spark可以把数据加载到工作节点的内存中，因此，许多分布式处理(甚至是分布式的1T数据的处理)都可以在几秒内完成。
上面的特性，使迭代式计算，实时查询、分析一般能够在shells中完成。Spark提供了Python shells和 Scala shells。
打开Spark的Scala Shell：

到Spark目录bin/pysparkbin/spark-shell打开Scala版本的shell


例子：

```scala
scala> val lines = sc.textFile(“../../testfile/helloSpark”) // 创建一个叫lines的RDD
lines: org.apache.spark.rdd.RDD[String] = ../../testfile/helloSpark MappedRDD[1] at textFile at <console>:12
scala> lines.count() // 对这个RDD中的行数进行计数
        res0: Long = 2
scala> lines.first() // 文件中的第一行
        res1: String = hello spark
```

修改日志级别：conf/log4j.properties  log4j.rootCategory=WARN, console

 

Spark的核心概念

Driver program：

包含程序的main()方法，RDDs的定义和操作。(在上面的例子中，driver program就是Spark Shell它本身了)
它管理很多节点，我们称作executors。
count()操作解释(每个executor计算文件的一部分，最后合并)。
SparkContext:
Driver programs 通过一个 SparkContext 对象访问 Spark，SparkContext 对象代表和一个集群的连接。
在Shell中SparkContext 自动创建好了，就是sc，


RDDs：
在Spark中，我们通过分布式集合(distributed collections，也就是RDDs)来进行计算，这些分布式集合，并行的分布在整个集群中。
RDDs 是 Spark分发数据和计算的基础抽象类。
用SparkContext创建RDDs
上面例子中使用sc.textFile()创建了一个RDD，叫lines，它是从我们的本机文本文件中创建的，这个RDD代表了一个文本文件的每一行。我们可以在RDD上面进行各种并行化的操作，例如计算数据集中元素的个数或者打印出第一行。

Spark PPT 下载地址：http://pan.baidu.com/s/1i3IkdoD


