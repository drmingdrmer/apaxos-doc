example-Raft
# Raft

将其转变成 Raft 也很简单, 时间T定义为`((term, node_id), index)`, 前一部分是Candidate和Leader的全局唯一id, index为Log index; Event定义为一条log. MonoHistory的Order是全序, 复用时间T的Order.

但是注意Raft巧妙的使用了这个设计上的特点进行了一些简化:
- 在实现write的step-0的read阶段, Raft的做法不是把已有MonoHistory(即全部log)都读过来, 而是发现如果需要远程读log(即本地日志少于对端日志),直接放弃选举, 只让不需要跨节点读日志的节点(本地日志大于其他节点)来完成选举.
- Raft实际上在选举阶段(对应prepare_write), 对多个MonoHistory批量执行了prepare_write, 即所有相同`(term, node_id)` 的日志, 不论index是多少, 都一次性执行了prepare_write, 所以Raft的Election阶段没有index信息出现.
- `(term, node_id)`的比较关系是偏序而不是全序的, 即term相同, node_id不同的2个值被认为不可比较; 由此带来的优化是MonoHistory,也就是Raft Log中不需要记录node_id, 因为对一个相同的term,最多只有一个Leader选出(对应完成Abstract-paxos中的prepare_write阶段).

write()的实现上: Raft的实现中, 将MonoHistory的复制做的工程上的细化, 例如逐段传输和snapshot方式的传输, 都对应abstract-paxos的write()的step-3的实现.

同样, commit需要完成MonoHistory的写入到一个`write_quorum`, Raft规定只有当当前leader的term的一个log成功复制到多数派时才认为commit, 这也是因为abstract-paxos中的read()时只取最大的MonoHistory,而MonoHistory的大小定义是由它的最大时间T定义的, 最大时间T是最后一条log的`((term, node_id), index)`.
#TODO

