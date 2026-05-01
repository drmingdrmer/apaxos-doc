# 读写操作对时间的依赖

在实际软件编写中，我们习惯于不显式指定操作的时间，而是默认使用当前的 wall clock，且新的写操作会覆盖旧值。这些在现实世界中无需证明的特性，在基于非线性时间构建的系统中却需要特别关注。

考虑一个简单的代码示例：
```rust
x = 2;
x = 3;
println!("x = {}", x);
```

这段代码可以被解释为：
```rust
x.write(2);
x.write(3);
println!("x = {}", x.read());
```

这段代码会打印出 3，但这引发了一个值得思考的问题：为什么存储了两个值，最终只保留了一个?
read 函数为什么不返回所有写入的值？为什么较早时间写入的值被忽略了？
这是因为每个 `x.write()` 操作都实际上是时间相关的行为，它发生在特定时刻，而我们的常识中默认有一条规则:"后发生的写操作覆盖先发生的写操作"。

如果去掉这些默认的假设, 一个适用于分布式系统的存储系统应该：
1. 记录所有写入的值
2. 能够按照时间顺序返回所有写入的值
3. 将值的选择决策权交给 application 层，而不是在 storage 层做决定

## 完整的时间感知实现

我们将时间相关的行为暴露出来后, 对变量的读写操作的完整实现可以表示为：
```rust
struct Variable {
	storage: BTreeSet<(Time, i32)>,
}
impl Variable {
    pub fn write(&mut self, v: i32) {
    	self.write_at((Time::now(), v));
    }
    
    fn write_at(&mut self, time_value: (Time, i32)) {
    	self.storage.insert(time_value);
    }
    
    pub fn read(&self) -> i32 {
        self.read_all().last().unwrap_or_default().1
    }

    fn read_all(&self) -> Vec<(Time, i32)> {
    	self.storage.iter().map(|(t, v)| (*t, *v)).collect()
    }
}
```

这里的 `Time::now()` 获取当前的 wall clock 时间。由于我们处在持续流动的时间中，每次获取的时间都是递增的。

要准确的描述使用虚拟时间的分布式系统, 我们不再使用常识上的读写函数`write()`和`read()`, 我们将重新定义适用于分布式系统的读写函数。