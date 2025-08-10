# Project3.8 HashJoin 实现

下面讲讲如何实现 HashJoin，这个难度就是：难者不会，会者不难。

思路是 Init 时遍历右表，把各个条目加入到 HashTable 中；Next 时遍历左表，每次去在 HashTable 中找即可。

## 实现 HashTable

那就先实现一个 HashTable，本来我是打算直接 `std::unordered_map<Tuple, std::vector<Tuple>>` 的，但是 Tuple 这个类没有定义 `std::hash<Tuple>`，所以不能直接作为 Key 值。

其实这是一个 C++ 语法题了，即自定义 HashTable 的 Key，现在这种东西问 AI 立马就出来了，不得不说 AI 真是入门的神器。

```cpp
// 本想 HashTable 是 <Tuple, vector<Tuple>> 的形式，但是 Tuple 没有定义 std::hash，所以不能作 Key
class HashJoinKey {
  public:
  HashJoinKey(const std::vector<Value> &key_values) : key_values_(key_values) {}
  bool operator==(const HashJoinKey &other) const { 
    if (key_values_.size() != other.key_values_.size()) {
      return false;
    }

    for (uint32_t i = 0; i < key_values_.size(); ++i) {
      if (key_values_[i].CompareEquals(other.key_values_[i]) != CmpBool::CmpTrue) {
        return false;
      }
    }
    return true;
  }

  std::vector<Value> key_values_;
};

// 要用 unordered_map，因为 Key 是 HashJoinKey，所以需要进行如下操作，此时这个 HashJoinKey 才能作为 Key
class HashJoinKeyHash {
  public:
  auto operator()(const HashJoinKey &key) const -> size_t {
    size_t hash_result = 0;
    for (const auto &value : key.key_values_) {
      auto now_value_hash = bustub::HashUtil::HashValue(&value);
      hash_result = bustub::HashUtil::CombineHashes(hash_result, now_value_hash);
    }
    return hash_result;
  }
};

// 最终使用方式：
std::unordered_map<HashJoinKey, std::vector<Tuple>, HashJoinKeyHash> ht_{};
```

## 给 HashTable 再包装一层

官网说的是我们可以模仿 AggreationExecutor 中的 SimpleAggregationHashTable，它除了有一个 HashTable，还有其他的辅助函数，那我们也写一个呗。

想想主要就是两个函数，插入和查找。以插入为例，想想我们需要传入什么？传入 Tuple 肯定要，还需要传入 `schema` 以及对应的表达式（其实就是第几列），这个是用来从 Tuple 中抽取出用来作为 Key 的列。

这里我一开始以为用的是 Tuple 的 `KeyFromTuple`，但是后来发现其实里面有些参数还是需要处理一下才能传进去，那其实可以直接用表达式自带的 `Evaluate` 方法就能提取出对应的值，代码如下：

```cpp
class SimpleHashJoinHashTable {
 public:
  SimpleHashJoinHashTable() = default;

  /**
   * 右表遍历到某个 Tuple 后，就会调用这个方法；这个方法就会从 Tuple 中提取出需要的列，作为 Key 值，然后保存
   */
  void InsertTuple(const Tuple &tuple, const Schema &schema, const std::vector<AbstractExpressionRef> &exprs) {
    // 根据传入的列的 Schema、Exprs，提取出 Key 值
    std::vector<Value> key_values;
    for (const auto &expr : exprs) {
      auto now_value = expr->Evaluate(&tuple, schema);
      key_values.push_back(now_value);
    }
    auto now_key = HashJoinKey(key_values);

    // 提取出 Key 之后，插入相关信息，略过 ...
    // ...
  }

  // 查找同理，和插入基本一样
  auto FindTuple(const Tuple &tuple, const Schema &schema, const std::vector<AbstractExpressionRef> &exprs) -> std::vector<Tuple> {
    // ...
  }
}
```

## 实现 HashJoin

最后剩下的就是 HashJoin 的实现，这里没有什么特别的，Init 就是遍历右表加入，Next 就是获取左表然后查找，这里就直接给 Next 的一些代码：

```cpp
auto HashJoinExecutor::Next(Tuple *tuple, RID *rid) -> bool { 
  auto left_schema = left_child_->GetOutputSchema();
  auto right_schema = right_child_->GetOutputSchema();

  while (true) {
    auto is_matched_before = left_tuple_.has_value();

    // 如果 left_tuple_ 有值，说明之前已经匹配过至少一次，继续遍历剩下的值
    if (is_matched_before && right_tuple_iter_ != right_tuples_.end()) {
        // 返回 LEFT + RIGHT，和 NestedLoopJoin 一样
    }

    // 如果发现是匹配的结束了，要清空相关的数据结构
    if (is_matched_before && right_tuple_iter_ == right_tuples_.end()) {
      // 这里算是一个之前出错过的点，如果是匹配到结束，那和未匹配是一样的处理，所以这里要把这个标志置为 False
      is_matched_before = false;
      left_tuple_.reset();
      right_tuples_.clear();
      right_tuple_iter_ = right_tuples_.begin();
    }

    // 那就开始在左表中拿出一个，并且进行寻找
    // 这里和 NestedLoopJoin 一样，不详细给出 ...

    // 在 HashTable 中寻找
    right_tuples_ = ht_->FindTuple(left_tuple, left_schema, plan_->LeftJoinKeyExpressions());
    right_tuple_iter_ = right_tuples_.begin();

    // 同样地，如果没有匹配到，并且是 LEFT JOIN，那么就输出 NULL
    if (!is_matched_before && right_tuples_.empty() && plan_->GetJoinType() == JoinType::LEFT) {
        // left_tuple + NULL，这个和 NestedLoopJoin 一样
    }
  }

  return false;
}
```

这样 Task3 就完成了，Task3 算是最顺利最简单的了。

## 特别致谢
感谢本文的博主，让我无从下笔的时候能有参考：https://blog.csdn.net/qq_40878302/article/details/137741785