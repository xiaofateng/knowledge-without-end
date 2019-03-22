## Tasks和Operator Chains
对于分布式运行，Flink中，每个操作符就是一个子任务，Flink把这些子任务链接成一个任务。每个任务都被一个线程执行。把操作符链接成任务，是一个很有用的优化，可以减少线程到线程切换的开销，和缓存的开销，在减少延迟的时候，增大吞吐量。
![](media/tasks_chains.svg)
## Job Managers, Task Managers, Clients
Flink运行环境中有两类进程:
JobManagers(也叫做masters) 协调分布式执行。协调tasks、checkpoints, 当任务失败时，协调恢复。
集群中至少有一个Job Manager，高可用环境下有多个JobManagers,一个是leader, 其它的是standby.
TaskManagers(也叫做workers) 执行数据流的tasks, 缓存和交换data streams。
![](media/processes.svg)
## Task Slots and Resources
每个worker(TaskManager)都是一个JVM进程, 可以在不同的线程中，执行一个或多个子任务（subtasks）。work有task slots概念，来控制一个worker可以接收多少个任务。
每个task slot代表TaskManager中一个固定的资源。有三个slots的TaskManager，会指定它管理的内存的1/3到每个slot。这样可以避免一个子任务和其它job的子任务竞争内存资源。注意，这里没有CPU隔离。

通过调整task slots的数量,用户可以决定子任务之间的隔离情况。每个TaskManager都有一个slot，意味着每个task group 都运行在一个独立的JVM中(例如，可以在一个单独的container中启动)。有多个slots，意味着，在同一个JVM中，有多个子任务。在同一个JVM中任务，共享TCP连接 (通过复用技术)和心跳消息。它们也可以共享数据集和数据结构，因此可以减少每个任务的开销。
![](media/tasks_slots.svg)

根据经验，task slots的默认值建议是CPU cores数目。
## state备份
state备份方式的选择，决定了key/values存储的数据结构。一种state备份方式，把数据存储在内存Hashmap中，另一个备份方式，使用RocksDB。state备份，除了定义数据结构，还实现了，获取key/value state快照的逻辑，和存储快照作为checkpoint的一部分。
![](media/checkpoints.svg)
## Savepoints
Data Stream API写的程序，可以从一个savepoint恢复执行。 Savepoints允许更新程序，并且让你的 Flink集群不丢失任何状态。

Savepoints是手工触发的checkpoints, 它将程序的快照，写进state备份。他们依赖于规则的checkpointing机制。在程序运行的时候，会定时的，在worker节点产生快照和checkpoints。对于恢复，只需要上一个，已经完成的checkpoint。之前的checkpoints都可以安全的删除。

Savepoints和这些定时的checkpoints很像，除了一点，Savepoints是用户触发的。Savepoints可以通过命令行创建，或者通过REST API取消一个job的时候。

参考：
https://ci.apache.org/projects/flink/flink-docs-release-1.7/concepts/runtime.html


