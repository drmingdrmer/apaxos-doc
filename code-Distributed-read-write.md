# 分布式多副本中的读写

分布式系统中的读写操作是基于单机 Node 的 API 实现的。
都是简单的代理实现, 对写操作, 只需要将一个 history 写入到每个节点即可。
对分布式读操作: 从每个副本节点读取 history,合并读取的结果返回给调用者.

下面是分布式系统中两个基础函数的实现：`write_to_nodes()` 和 `read_from_nodes()`。
这两个函数将请求转发到单机 Node 的读写操作。由于 `Node::read()` 返回 `Vec<History>`，
所以读操作需要将所有节点返回的 `History` 集合取并集。

```rust
impl Distributed {
    fn read_from_nodes(targets: &[NodeId]) -> Vec<History> {
        let mut histories = Vec::new();
        for node in nodes {
            histories.extend(node.read());
        }
        histories
    }

    fn write_to_nodes(&mut self, targets: &[NodeId], h: MonoHistory) {
        for node in targets {
            node.write(h);
        }
    }
}
```

