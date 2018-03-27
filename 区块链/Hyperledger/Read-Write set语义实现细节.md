# Read-Write set语义
本文讨论了关于Read-Write set语义的当前实现的细节。



## Transaction 模拟和 read-write set

**在endorser上模拟transaction时，为transaction准备了read-write set**。 `read set`包含在模拟期间，transaction读取的唯一keys及其提交版本（version）的列表。 `write set`包含唯一keys列表（尽管可能与read set中的keys存在重叠）以及transaction写入的新值。 如果transaction执行的更新是删除key，则为该key设置delete marker（删除标记）（代替新值）。

 另外，如果transaction为一个key多次写入一个值，则只保留最后一次写入的值。 此外，如果transaction读取某个key的值，则即使transaction在发出读取之前就更新了key的值，也会返回committed state的值（即，读取的是ledger中的值，而不是模拟执行的结果）。 换句话说，不支持Read-your-writes语义。

如前所述，key的版本只记录在read set中; write set 只包含由transaction设置的唯一key列表及其最新值。

可能有各种实现版本（version）的方案。版本控制方案的最低要求是为给定key生成非重复标识符。例如，对版本（version）使用单调递增数字的方案。

在当前实现中，我们使用基于blockchain（区块链）高度的版本控制方案，其中提交transaction的高度，用作transaction修改的所有key的最新版本。在这种方案中，transaction的高度由一个元组表示（`txNumber` 是块内transaction的高度）。与增量数字方案相比，此方案具有许多优点 - 主要是，它可以实现其他组件，如statedb，transaction模拟和验证，以便进行高效的设计选择。

以下是通过模拟假设transaction，准备的`read-write set`的说明。为了简单起见，在插图中，我们使用增量数字来表示版本。

```c
<TxReadWriteSet>
  <NsReadWriteSet name="chaincode1">
    <read-set>
      <read key="K1", version="1">
      <read key="K2", version="1">
    </read-set>
    <write-set>
      <write key="K1", value="V1"
      <write key="K3", value="V2"
      <write key="K4", isDelete="true"
    </write-set>
  </NsReadWriteSet>
<TxReadWriteSet>
```

此外，如果transaction在模拟过程中执行范围查询，范围查询及其结果将作为query-info添加到read-write set中。

## Transaction 有效性检查 和 使用read-write set 更新 world state 

committer使用read-write set中的`read set`部分来检查transaction的有效性，并使用`write set`部分来更新受影响的key的版本和值。

在验证阶段，如果transaction的 `read set` 中存在的每个key的版本与世界状态（world state）下的相同key的版本相匹配，则认为该transaction有效 - **假定所有先前的有效transaction（包括相同 block 中前述的transaction）都被提交了（提交状态）**。 如果read-write set还包含一个或多个query-info，则执行额外的验证。

这种额外的验证，应确保在query-info(s)中获取的结果的超范围（super range，即，范围的联合）中没有key被插入/删除/更新。换句话说，如果我们在committed-state验证期间，重新执行任何范围查询（模拟期间执行的transaction），它应该产生与模拟transaction观察到的结果相同的结果。该检查可确保，如果transaction在提交期间观察到幻象结果（假的，不符合的结果），则该transaction应被标记为无效。

请注意，该幻象保护仅限于范围查询（即，Chaincode中的`GetStateByRange`函数）并且尚未针对其他查询（即，链式代码中的`GetQueryResult`函数）应用。其他查询处于幻影风险中，因此其它查询只能用于未提交排序的只读transaction，除非应用程序可以保证，结果集在 模拟和验证/提交 之间的稳定性。

如果一个transaction通过有效性检查，committer使用write set来更新world state（世界状态）。 在更新阶段，对于write set的每个key，同一个key的world state值设置为write set 中指定的值。 此外，world state中的key的版本被改变，以反映是最新版本。


## 模拟执行和验证的例子

本节通过示例场景帮助理解语义。 在世界状态中存在一个密钥`k`,由一个元组`（k，ver，val）`表示，其中`ver`是以`val`为值的密钥`k`的最新版本。

现在，考虑一组五个transactions`T1`，`T2`，`T3`，`T4`和`T5`，全部在世界状态的相同快照上模拟。 以下显示了模拟transactions的世界状态的快照,以及每个transactions执行的读写活动的顺序。

```c
World state: (k1,1,v1), (k2,1,v2), (k3,1,v3), (k4,1,v4), (k5,1,v5)
T1 -> Write(k1, v1'), Write(k2, v2')
T2 -> Read(k1), Write(k3, v3')
T3 -> Write(k2, v2'')
T4 -> Write(k2, v2'''), read(k2)
T5 -> Write(k6, v6'), read(k5)
```
现在，假定这些transactions按照T1，...，T5的顺序排序（可以包含在单个block或不同的block中）

1. `T1` 通过了有效验证，因为它不进行任何read操作。世界状态中的 `k1` 和 `k2` 更新成了 `(k1,2,v1')`, `(k2,2,v2')`
2. `T2` 没通过有效验证， 因为它读了一个key, `k1`, 而这个`k1`,在前面的transaction `T1` 中修改了。
3. `T3` 通过了有效验证，因为它不进行任何read操作。世界状态中的 `k2` 更新成了 `(k2,3,v2'')`
4. `T4` 没通过有效验证， 因为它读取了一个key, `k2`, 而这个`k1`,在前面的transaction `T1` 中修改了。
5. `T5` 通过了有效验证，因为读取的key, `k5`, `k5`没有在前面的 transactions 中修改过。

注意: 具有多个 read-write sets的 Transactions 尚不支持。

欢迎加入区块链技术交流QQ群 694125199

更多区块链知识：
https://github.com/xiaofateng/knowledge-without-end/tree/master/区块链/Hyperledger

本文参考
http://hyperledger-fabric.readthedocs.io/en/latest/readwrite.html



