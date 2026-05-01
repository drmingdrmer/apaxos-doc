#TODO 
Universe::read 实现
read() 的实现:

```rust
trait Universe: Distributed {
    fn read(&self) -> History {
        let read_quorum = self.read_quorum();
        let mut candidate_histories = Self::read_from_nodes(read_quorum);

        let mut selected_histories = vec![];
        while !candidate_histories.is_empty() {
            for candidate in candidate_histories {

                // 检查h1是否是最大的, 即不小于任何其他MonoHistory
                let mut is_maximal = true;
                for other in candidate_histories {
                    if !candidate.is_compatible_with(other) {
                        is_maximal &= candidate > other;
                    }
                }

                if is_maximal {
                    selected_histories.push(candidate);
                    for history in candidate_histories {
                        if !candidate.is_compatible_with(history) {
                            candidate_histories.remove(history);
                        }
                    }
                }
            }
        }
        // 合并所有兼容的MonoHistory
        History::merge_compatible(selected_histories)
    }
}