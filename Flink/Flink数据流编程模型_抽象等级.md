# 抽象等级
Flink提供不同等级的抽象，开发流式、批处理程序。
![](media/levels_of_abstraction.svg)

* 最低等级的抽象，提供有状态的数据流。它通过Process Function，嵌入在DataStream API中。
它允许用户自由的处理，从一个或多个流中过来的事件，并使用一致的容错机制。另外，用户可以注册事件时间和处理时间的回调，允许程序实现复杂的计算。

* 在生产环境中，大多数程序不会用到上面的低阶抽象，而是会使用Core APIs，像DataStream API (有界/无界流) 、DataSet API (有界数据集)。这些APIs，为数据处理，提供通用的构建块的方法，例如，各种类型的transformations, joins, aggregations, windows, state等。这些APIs中的数据类型处理，表示为相应的编程语言中的类。
* Table API是一个以表为中心的声明性DSL，可以动态的改变表(当处理流的时候)。Table API 遵循下面的关系模型: 表有一个附加的schema信息(就像关系型数据库中的表一样) 。API 提供可比较的操作，例如：select, project, join, group-by, aggregate等。Table API程序定义应该进行的操作，而不是操作的具体实现。Table API 可以被用户自定义的各种函数扩展，它的表现力不如 Core APIs, 但是用起来更简洁 (写更少的代码)。Table API程序，在执行前，同样也会根据优化器的规则进行优化。

用户可以无缝地在tables 和 DataStream/DataSet之间进行转换, 允许程序混合Table API、DataStream、DataSet APIs。

* Flink提供的最高等级的抽象是SQL。在语义上和表现力上，这个抽象和Table API类似, 但是以SQL查询表达式代替程序。SQL抽象和Table API可以密切交互,SQL查询，可以运行在Table API定义的表上。


