
# Write 约束

在多重宇宙环境中, 所有 write 到`write_quorum`的 MonoHistory 都可以被读到, 因为 read()不会舍舍弃任何结果.

但是当把多重宇宙压成单一结果时, read()就要根据 MonoHistory的 Order 关系舍弃到较小的不 Compatible 的 MonoHistory, 因此简单的 write 一个 MonoHistory 并不能保证它后续能被读到了, 只有足够大的 MonoHistory 在写入后才能被读到.

但是,当 write() 一个比较大的 MonoHistory 时, 可能会覆盖已 commit 的其他 MonoHistory, 导致其他写入者的写入丢失, 因此 write()之前还要保证不覆盖可能已经 commit 的 MonoHistory: 具体做法就是执行一个 read() 操作, 因为 read() 保证能读到已经 committed 的 MonoHistory: `h`, 写入者在`h`的基础上增加新的时间点 T 和 Event, 再写回到一个`write_quorum`, 就可以保证是基于最新看到的 MonoHistory进行写入的.

但是这个过程并不是原子的, 读到 committed MonoHistory后, 可能有其他新的`h'` 又写入了, 我们要写入的 h1 可能还会覆盖已有 committed 的值.

所有这里要引入一个两阶段的写, 类似于 2-phase-commit, 但是我们的算法是一个活锁, 以下是 write 完整的过程, 包括最初的 read(), 一共需要 3 次 RPC:

- 写入者进行一次 read()得到 committed h0;
- 写入者基于 h0 增加 Event 得到 h1;
- 写入者向一个`write_quorum`发一个`pre_write(h1)`,它告诉每个收到`pre_write`的节点, 下一个要写的内容, 节点则:
  - 对比 h1 和自己本地存储的已接受的 History 和 pre_write 的 History:
  - 如果 h1 小于任何一个 incompatible 的 MonoHistory, 则返回 pre_write 失败;
  - 否则节点删除本地所有 h1-incompatible 的 MonoHistory, 将 h1 添加到本地的 pre_write History 中, 返回 pre_write 成功.
    这一步的目的是阻止系统中任何`h: h < h1` 都不可以 commit, 保证`h1`的写入不会意外覆盖已 committed 的值.
- 写入者向一个`write_quorum`发一个 `write(h1)` , 收到的节点则:
  - 检查本地已接收的 History 或 pre_write 的 History, 是否包含 h: h ≭ h1 且 h > h1, 如果有, 说明有其他写入者在阻止这次写入, 返回写入失败.
  - 否则, 删除所有与 h: h ≭ h1, 这是因为, 完成了 pre_write 阶段后, 已经没有更小的 h 可以被 commit 了, 而已经 commit 的一定已经在 step-0 读到了并包含在这次写入中, 所以删除并不会造成已经 committed 丢失. 然后记录 h1 到 Accepted 的 History 集合, 返回成功.

至此, 单宇宙的共识就描述完了, 下面是一个完整的执行 write 的例子,这个例子中使用一个简单的有4个vertex的时间系统:

- 初始状态: 1->3 完成了 prepare, 但是 write 只写到 N2,未完成 commit;
- step-0: read(N1,N3): 得到一个 MonoHistory, `1`, 它可能是已经 committed, 在此基础上, 添加要写入的 Event 在 T=4;
- step-1: prepare_write(N1,N3), 都成功,其中 N3 节点上删掉了 Incompatible 的 prepared History: 1->3;
- step-2: write(N2,N3): 都成功, 其中 N2 节点上删除了 Incompatible 的 1->3; 写入完成 commit.

![](write.x.svg)