#TODO 是否需要添加History演进的定义?

# Compatible History的定义

虽然History是一个DAG, 可以按照DAG的定义进行合并, 但合并后的History可能会给某些顶点引入新的边.
这些新引入的边可能违反History演进的定义, 即可能改变某个时间顶点之前的Event集合.

## Compatible History

两个History可合并的定义是: 合并后的History中, 对于每个时间顶点, 其可达的顶点集合保持不变.
满足这个条件的History称为Compatible History: `h1 ≍ h2`

2个History h1 h2可合并的定义以下定义是对等的:
- 只在h1中的顶点和只在h2中的顶点都没有大小关系, 同时在h1和h2的节点必须有相同的入向边;

显然, 一个History的所有ancestor都是Compatible的.
同样, 一个History的所有successor也都是Compatible的.

## 示例

考虑下图所示的时间DAG:
- ⊥表示系统初始状态
- T=1和T=2是两个无先后顺序的时间顶点
- T=3和T=4都在T=1和T=2之后
- T=3和T=4之间没有先后顺序

假设有两个History:
- h1: `⊥->1->3`
- h2: `⊥->2->4`

这两个History是不Compatible的, 原因如下:
- 合并后的History为`⊥->{1,2}->{3,4}`
- T=3在合并前看不到T=2, 合并后却可以看到T=2, 这改变了T=3的历史
- T=4在合并前看不到T=1, 合并后却可以看到T=1, 这改变了T=4的历史

或者说: 合并产生了新边`1->4`和`2->3`, 这违反了History演进的定义

![](history-compatible.x.svg)

从History的演进图中也可以看出, `⊥->1->3`不在`⊥->{1,2}->{3,4}`的任何演进路径上:

![](history-12-34-evolve.x.svg)

另外:
- `⊥->1` 和以下History有Compatible关系: `⊥->2`, `⊥->{1,2}`, `⊥->1->3`.
- `⊥->1` 和以下History没有Compatible关系: `⊥->2->3`, `⊥->2->4`, 因为合并后T=3和T=4的入向边会改变.