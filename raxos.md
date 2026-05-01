TODO: remove concept History quorum

[[def-Time]]

[[def-Time-Unique]]

[[desc-Time-in-Distributed]]

[[example-T-in-single-threaded]]

[[def-API-read-write]]

[[def-RW-Necessity]]

[[example-RW-single-threaded]]

[[def-Past-Future]]

[[example-Past-Future-single-threaded]]

[[example-Past-Future-distributed]]

[[def-Distributed-HA]]

[[def-Apply]]

[[def-Distributed-Copies]]

[[code-Distributed-read-write]]

[[def-Mergeable-History-2]]

[[prop-Merge-Read]]

[[prop-Discard-Smaller]]

[[prop-Write-Forbid-Smaller]]

[[desc-History Read Set]]

[[def-Read-Quorum-Set]]

[[example-Quorum-Set]]

[[def-Write-Quorum-Set]]

[[example-Quorum-Set-xy]]

[[def-Observable-History]]

[[def-T-Committed]]

[[def-Committed]]

[[desc-Availability]]

[[def-multiverse]]

[[def-multiverse-consensus]]

接下来进入本文的第二部分

[[def-multiverse-collapse]]

## Commit 在线性 History 约束中的定义

这里 Commit 的定义没有变, 即对一个 History 永远能读到
但是我们现在用 read_linear() 替换了 read(),
它会抛弃一些 History 分支,
所以要保证 Commit, 一次写入不仅要保证 read() 总是可见,
还要保证 read_linear()总是可见.
对一个 History, 如果

- 1 读操作 read_linear()可以通过系统中`read_quorum_set`中任意一个 read_quorum 读到它,
- 2 且总是能被选择, 即它有最大的 T, 或是一个最大 T 的 History 的一部分(这样它也能被选择).

它就是 Committed.


protocol-write-prepare

## Write Prepare

现在 write 保证了不覆盖已有的 History, 且写到了`write_quorum_set`,
能保证`read()` 一定读到它, 但还没能保证 read_linear()一定选择它.

所以可能产生一个问题, 写入完成时, 可能因为系统中有更大的 History,
导致 read_linear()忽略了自己写`write_quorum_set`的 History:

例如可能有 2 个 Writer W8 和 W9,

- W8 写了 node-1,node-2,
- W9 写了 node-3.

我们假设系统的`read_quorum_set`是多数派模式,
那么 W8 写的 History{..,E3} 虽然能被任意一个 read()读到,但不一定被 read_linear()
读到:

- 如果 read_linear()操作选择了 read_quorum {1,2}, 那么它会选择 W8 写入的 History{..,E3}, 得到预期的结果
- 但是如果 read_linear()操作选择了 read_quorum {2,3}, 那么它会选择 W9 写入的更大的 History{..,E4},

这违背了 Commit 的原则, 没有达到**总是能被读到**的要求.

![](history-dirty-write.x.png)

[[protocol-Write-Forbid-Smaller]]

[[protocol-Write-After-Read]]

[[protocol-Write-P2]]

[[protocol-All]]

[[example-Classic-Paxos]]

[[example-Raft]]

[[2d-consensus]]

[[example-2d-consensus]]

## 二维 Time 的应用

有一个常见的 Raft 使用常见是用 Raft 做一个授时服务, 集群选出 Leader, Leader 处理申请一个时间戳的请求, Leader 提交一个日志记录已经分配的时间, 然后返回这个时间戳. 这里我们假设本地时间不会回退.

但是当 Leader 切换时, 新的 Leader 可能本地时间略小于上一个 Leader, 导致新分配的时间回退.

这个问题的解决并不困难, 在生成时间戳的逻辑中判断禁止生成回退的时间.

但问题的根本原因是, Raft 本身在使用一个虚拟时间(`term`)来实现全局的虚拟时间永远递增, 但是它没有考虑墙上时钟, 分布式本身就是为了建立一个单调递增的系统, 所以最根本的解决方法是把墙上时钟的值也包括到 Raft 的虚拟时间中:

现在我们用Abstract-paxos来实现授时服务,
定义 `Time: [(term, voted_for), timestamp]`, 其中term和voted_for还是沿用Raft里的概念, term是递增整数, voted_for是节点Id, 表示一个candidate 或 leader的node id.
其中`(term, voted_for)` 的Ordering 也沿用Raft中的偏序设计, 即:
- term较大的整体较大;
- term相同时, 只有voted_for一样时,整体相等, 否则认为不可比较; 这就继承了Raft的一个term中只能选出一个Leader的设计.

而Time的的偏序关系按照字典序定义, 即先比较 `(term, voted_for)` 部分, 如果相等则比较timestamp. 其中timestamp 是 wall clock 时间, 它是全序的.

History中的Event都定义为空`()`,因为授时服务中不需要它.


#TODO 这里增加到其他章节,描述的是从多重宇宙到单一宇宙的演进: 
一个存储节点Node可以选择: 存储多个互相Compatible的MonoHistory, 或只存储一个MonoHistory.
选择存储多个则允许multi leader, 否则只允许单一leader
#TODO 这里要选择节点只存储一个MonoHistory
考虑如果raft的节点允许存储多MonoHistory,那样`(term, voted_for)` 的偏序关系会让raft在一个term里产生2个独立的leader,他们互相不影响, 但互相看不到彼此的数据, 只有新的leader才能同时看到他们俩的数据.


整个系统依照Abstract-paxos的算法运行, 


上面的时间定义也反应出了这个系统的需要:


然后

- 把 term 替换为`Time: [term, timestamp]`, 这个二维的向量是使用我们上面定义
- 再把`LogId = (term, index)` 替换为 `([term, timestamp], index)`
  来构建一个分布式系统.

这样我们只改了数据结构, 协议还是跟原来一样的. 这也说明 Raft 只是偏序时间的一致性算法的一个特例.

这时我们会得到一个改进版的 Raft: 在每个成员有时钟飘逸的情况下,
新的 Leader 也总是有更大的 unix-timestamp, 如果一个 Candidate 的本地时间较小, 那么在 RequestVote 阶段会失败:
**注意,这里 RequestVote 要比较的是最新的时间戳...**

这个分布式系统可以用来构建分布式授时服务, 优雅的解决 Leader 切换时的时间可能回退问题.

这个例子说明, Time 的每个维度之间的类型可以是不同的类型, 而虚拟时间也有把互相关联的变量统一起来的作用.

[[example-crdt]]

[[example-2d-non-transitive-time]]

[[exmaple-crdt-define]]

[[example-crdt-implementation]]

例如

如果 T 不是**传递的**, 那么第一阶段必须在一个 read-quorum-set 上阻止要写入的 History 上所有的 T, 而不是只有最大的一个 T.因为不能通过 T 来防止之前的顶点被写入.
而在第二阶段, 也必须逐个判断要写入的 History 的每个 T 是否小于已经被阻止的 T, 而不仅仅是要写入的 History 的 Maximal T.

[[def-2d-consensus-Apply]]

[Hasse diagram]: https://xxx
[Maximal]: foo
[Time, Clocks and the Ordering of Events in a Distributed System]: https://lamport.azurewebsites.net/pubs/time-clocks.pdf
[Paxos Made Simple]: https://lamport.azurewebsites.net/pubs/paxos-simple.pdf
[Raft]: https://raft.github.io/raft.pdf
[Generalized Consensus and Paxos]: https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-2005-33.pdf
[CRDT]: https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type
[CURP]: https://www.usenix.org/system/files/nsdi19-park.pdf
