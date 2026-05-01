## 多重宇宙合并为单一历史

尽管我们认识到多重历史的存在,但是我们还只是活在单时间线中的低等文明, 我们只能有一个时间线。
这意味着 read() 方法不能返回 `Vec<MonoHistory>`,而只能返回一个 History,它是由一组相互 Compatible 的 MonoHistory 合并而成。

## read() 合并 History 的职责

但是, 因为并不是所有的 MonoHistory 都是 Compatible 的, 不能直接将所有读到的 MonoHistory合并.
所以 `read()` 要负责: **在读到的所有 MonoHistory 中, 任意 2 个不 Compatible 的 MonoHistory中, 选择其中一个, 舍弃另一个**,
且选择必须是唯一确定的, 不能是随机的. 
最后把所有互相 Compatible 的 MonoHistory 合并, 作为最终读到的 History.

因此任意 2 个不 Compatible 的 MonoHistory, read() 总是选择其中一个而抛弃另一个.
这表示, read()函数为 **任意 2 个 MonoHistory, 定义了一个大小关系**, 即 MonoHistory 的 Order:

- 不 Compatible 的 2 个 MonoHistory必须可以比较大小;
- 无环(否则无法保证 read 选择的确定性: 不同的比较顺序会得出不同的结果).
