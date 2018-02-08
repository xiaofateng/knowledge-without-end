# 
编写wordcount程序

手动导入包：import org.apache.spark.SparkContext._

```
val conf = new SparkConf().setAppName(“wordCount”)// 创建一个Spark Context.
val sc = new SparkContext(conf)
val input = sc.textFile(“/home/spark/testfile/helloSpark”)// 加载数据
val words = input.flatMap(line => line.split(” “))// 把每一行分割成单词
val counts = words.map(word => (word, 1)).reduceByKey{case (x, y) => x + y}//转换成pairs 并且计数
counts.saveAsTextFile(“/home/spark/testfileResult/wordCountRes”)// 保存动作。
```


打包：

build->build artifacts->build

打成jar包，将jar包上传至spark集群上。

启动集群：


```
启动master
./sbin/start-master.sh
启动worker
./bin/spark-class org.apache.spark.deploy.worker.Worker spark://ubuntu:7077
提交作业
./bin/spark-submit –master spark://ubuntu:7077 –class HelloSpark /home/spark/testjar/hellosbt.jar
```

提交后，可以在下面的ui上看作业的运行。

Spar job UI        http://localhost:4040/
master的UI    http://localhost:8080/

如果，有不清楚的地方，可以看我录制的spark入门视频，完全免费，

视频地址：https://pan.baidu.com/s/1c4ba8dq

