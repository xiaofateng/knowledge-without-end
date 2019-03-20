## 程序和数据流
Flink程序的基本构成块是streams（流）和transformations。(注意，Flink的DataSet API 使用的DataSets在内部也是streams) 在概念上，一个stream就是一个数据流, transformation是一个操作，该操作把一个或多个stream（流）作为输入，产生一个或多个stream（流）作为输出结果。

当执行的时候，Flink程序由streams和transformation操作组成。
每个数据流（dataflow）以一个或者多个sources作为开始，以一个或者多个sinks作为结束。数据流组成DAGs。 尽管通过迭代，允许特殊形式的循环，但为了简单起见，我们在大多数情况下对其进行掩饰。
![](media/program_dataflow.svg)
## 并行数据流
Flink程序，在内部是分布式并行的。在执行时，一个stream有一个或者多个stream partitions。
Stream的并行度，总是由生成它的操作符决定的。
![](media/parallel_dataflow.svg)

Streams可以在两个操作符之间传输数据，传输数据可以，以one-to-one(或者forwarding) 方式, 或者redistributing方式:

* One-to-one streams，像map操作
* Redistributing streams，像keyBy操作，window操作，sink操作等。这些操作改变stream的partitions



