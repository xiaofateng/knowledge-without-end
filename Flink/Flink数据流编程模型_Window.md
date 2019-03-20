## Windows

streams上的聚合操作，（例如counts, sums等）, 是被windows界定的, 例如 “count过去5分钟的数据”, “sum上100个元素”。
Windows可以是time driven (例如: 每30秒) 或者 data driven (例如: 每100个元素)。一般会区分不同类型的window, 例如tumbling windows (没有重叠), sliding windows (有重叠), session windows (被不活跃的空隙所打断)。
## Time
当在streaming程序中，提到time，可能指的是不同的time：

* Event Time，是event的创建时间。通常由events中的一个timestamp描述, 例如，由传感器附加的, 或者产生event的服务器。
* Ingestion time，是在进行source操作时，event进入Flink数据流的时间。
* Processing Time，是每个操作的本地时间。

## Stateful操作
stateful操作的state（状态）维护在一个内置的key/value存储上。state被分区，并被分布式地和streams在一起。因此, 访问key/value state只能在keyed streams上,在keyBy()函数后面,并且，被限制在和当前event的key关联的values。对齐streams的keys和state，确保了所有的state更新都是本地操作，在没有额外开销的前提下，保证一致性。这种对齐，也使Flink redistribute state和透明地调整stream分区。
![](media/state_partitioning.svg)


