# 基于Receiver 方式
这个receiver是基于 Kafka high-level consumer API实现的。像其它的receivers一样，接收到的数据会放到spark的executor里面，然后sparkstreaming程序启动任务处理数据。

# 直接方法，没有receiver

这个方法是spark1.3引进的，现在都是spark2.0版本了，看样会一直延续下去了。这个的引入是为了保证端对端的可靠性保证。

这个方法定期的从 kafka的每个topic+partition里面查询最新offset的数据，然后相应的，定义每个batch里面处理的数据offset范围。
当处理数据的job启动的时候，Kafka的simple consumer api读取确定好的kafka中的offset范围（就是从文件系统中读文件）

这种方法，与第一种方法相比的优势：
* （一）简单的并行度
sparkstreaming会创建和kafka分区一样多的RDD分区来消费，这样的话，所有的节点都会从kafka并行的读取数据。
Kafka分区和RDD分区是一对一的关系，这样更便于理解和优化。
* （二）高效
第一种方法，为了达到零数据丢失，要写WAL，这样数据存了两份，一份在kafka，一份在WAL中，这是不高效的。第二种方法，不需要receiver，因此不需要WAL，
只要kafka中有足够的消息，消息就能从kafka中恢复处理。
* （三）Exactly Once语义
第一种方法，使用kafka的high level api，将消费的offset存储在zookeeper中。这种方法可以保证数据不会丢失，但是数据可能会被重复消费，因为spark真正接收到的数据可能和zookeeper中的offset不一致。
第二种方法，使用simple kafka api，不使用zookeeper，offset是通过sparkstreaming的checkpoint来记录的，这样就消除了sparkstreaming和kafka（或者zookeeper）之间的不一致性，此时，sparkstreaming不会丢失数据。如何保证 Exactly Once呢
（1）幂等操作
（2）业务代码添加事物操作
如果业务上不能幂等操作，只能通过业务自身的逻辑代码来解决了。

官方方案：

```scala
dstream.foreachRDD { (rdd, time) =>
  rdd.foreachPartition { partitionIterator =>
    val partitionId = TaskContext.get.partitionId()
    val uniqueId = generateUniqueId(time.milliseconds, partitionId)
    // use this uniqueId to transactionally commit the data in partitionIterator
  }
}
```
使用batch time和partition index来创建一个id，使用这个id来确保数据的唯一性
启动事务并使用这个id来更新外部系统数据，如果这个id不存在则提交更新，如果这个id已经存在那么则放弃更新。

注意，第二种方法，不会把offset写到zookeeper中，所以，基于zookeeper的kafka监控工具都失效了，然而，你可以在每个batch中访问offset，更新到zookeeper中。

**第二种方法使用checkpoint保存数据，当程序代码改变时，会产生问题，解决方法是offset保存在zookeeper中。
**
当你启动程序消费kafka的数据的时候，默认情况下，第二种方法会从最近的offset开始消费，你也可以通过auto.offeset.reset设置成	smallest，这样它就会从最小的offset开始消费。
与第二种方法相关的配置是saprk.streaming.kafka.*
重要的一个配置是，spark.streaming.kafka.maxRatePerPartition，表示，在每个kafka 分区中，被direct api读取的数据的最大速度。
在spark-submit的时候 –conf 指定这个配置。

