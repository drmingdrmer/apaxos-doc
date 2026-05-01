# 线性宇宙中的read

虽然不知道MonoHistory 的Order是什么, 但可以先假定它已经存在了,
那么现在分布式共识中, read() 就要负责从多重宇宙中选择一个或几个互相兼容的MonoHistory 返回给调用者.

read() 选择MonoHistory 的算法描述:

- 在MonoHistory的初始集合S中, 找出最大的, 即那些不小于任何其他MonoHistory的MonoHistory.
  这些MonoHistory 需要保留的, 放入 "已选择" 的集合中, 同时从S中移除与其不Compatible的所有MonoHistory.
- 重复上面的过程, 直到S为空.

