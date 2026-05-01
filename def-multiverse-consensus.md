## 多重宇宙里的分布式共识

现在我们的节点读写接口是支持存储多个History的, 在这个模型上共识是trivial的.
因为它暗示了我干我的,你说你的, 互不干扰.

这时分布式HA只需提供quorum-rw即可:
将一个History写到某个`write_quorum`中,
那么一定可以被一个`read_quorum` 读到, 那么这次写入就可以认为是 Committed.

```rust
trait Multiverse: Distributed {
    fn write(&self, history: History) {
        self.write_to_nodes(self.write_quorum(), history);
    }

    fn read(&self) -> Vec<History> {
        self.read_from_nodes(self.read_quorum())
    }
}
```



多重宇宙的共识比单宇宙的共识更简单, 因为每个分支都是独立的, 
单宇宙里(我们的现实),最终只能看到唯一的时间线, 就引入如何选择的问题.
这是分布式共识中的核心问题.

举例来说, 我们还是假设系统中的时间T是如下结构的(仅有4个时间点):

![](time-12-34.x.svg)

例如: 下图中, `read()` 会读到 2 个 History, `write(⊥→1→3)` 写到 N2,N3 2 个节点, 其中N2保存了2个History, N3保存了1个History.


![](multiverse-rw.x.png)


因为没有覆盖的问题, 在多重宇宙里, 我们只需要quorum-read-write就够了.
