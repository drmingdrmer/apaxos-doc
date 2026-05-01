## MonoHistory Order

对于 MonoHistory Order,需要满足以下性质:
给定 `h1: MonoHistory(t1)` 和 `h2: MonoHistory(t2)`,若 `t1 < t2`,则 `h1 < h2`。

这个性质确保了 History 可以增长, 即我们常说的一个共识算法的(liveness) - client 可以通过使用更大的时间 T 构造更大的 MonoHistory 来完成写入。
这样后续的读请求就能读取并选择这个新写入的 MonoHistory。如果 client 无法构造更大的 MonoHistory,
其写入将无法完成 Commit,造成整个系统无法向前推进。

值得注意的是,History 的比较关系**不要求满足传递性**。例如对于下面三个 MonoHistory `h1`,`h2`,`h3`,
假设时间 `a,b,c` 之间没有大小关系,那么:
- 不 Compatible 的是: `h1 ≭ h2`, `h2 ≭ h3`
- Compatible 的是: `h1 ≍ h3`
- 虽然 `h1 < h2 < h3`,但 `h1` 和 `h3` 是 Compatible 的,允许它们不可比较:

```
h1: x -> a
h2:      a -> b -> c
h3: x -----------> c
```

当 read() 读取到这三个 MonoHistory 时,会选择 `h1` 和 `h3` 的并集作为返回值:
```
 x -> a
  `------> c
```

> 实践中,大多数 MonoHistory Order 都满足传递性,构成偏序关系,更常见的是全序关系。
>
> 以下是一些具体系统中 read 操作选择 MonoHistory 的例子:
>
> - Paxos: 仅允许提交单个值,其 MonoHistory 形式为 `(value_ballot_num, value)`,通过 `value_ballot_num` 
>   比较大小。在 Phase-1 中,Proposer 遇到多个现有值时,选择最大 `value_ballot_num` 对应的值。
>
> - Raft: MonoHistory 是一组 log,大小关系由最大 log id `(last_log_term, last_log_index)` 决定。
>   这体现在 election 过程中,具有最大 last_log_id 的 candidate 才能当选。
>
> 本文讨论的 MonoHistory 是一个抽象概念,仅需满足:
> 是 DAG 结构,且 Order 中可比较的关系由 MonoHistory 中的最大时间 T 决定。