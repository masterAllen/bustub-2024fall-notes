# Project3.6 NestedLoopJoin 和 NestedIndexJoin

开始实现 NestedLoopJoin，它负责如下的语句：

```
EXPLAIN SELECT * FROM __mock_table_1 INNER JOIN __mock_table_3 ON colA = colE;
EXPLAIN SELECT * FROM __mock_table_1 LEFT OUTER JOIN __mock_table_3 ON colA = colE;
```

其中 INNER_JOIN 和 LEFT OUTER JOIN 的区别直接问 AI，这个是 SQL 那边的知识点。

两张表需要进行对比，朴素的是直接双重循环，复杂度是 `O(m*n)`。除非上哈希表之类，但并不需要这样，现在就用两重循环就可以了。后面有个 HashJoin 就干这个事的。

两张表直接就是构造函数传进去了，不需要我们去主动获取：
```cpp
NestedLoopJoinExecutor::NestedLoopJoinExecutor(/*..*/, std::unique_ptr<AbstractExecutor> &&left_executor, std::unique_ptr<AbstractExecutor> &&right_executor)
```

官网也给我们了说明，如果比较两个 Tuple 怎么办？直接用 `plan_->Predicate()` 中的 `EvaluateJoin` 就行，具体传入的参数看代码即可。

## 实现，版本一

最直观实现就是两张表都存起来，然后 Next 中去对比。我这里为了节省空间，就只存储右表了，左表的话就记录 `current_tuple`，之后在 Next 中进行更新就行，很简单，这里就直接写一点 Next 代码：

```cpp
// 并不能直接使用，隐藏了一些语句
auto NestedLoopJoinExecutor::Next(Tuple *tuple, RID *rid) -> bool { 
  // 左右两个 executor，每个都要进行比对是否符合 predict，即 O(m*n)
  auto predicate = plan_->Predicate();

  // left_tuple: 当前坐标遍历的节点，使用 std::optional<Tuple> 来判断

  while (true) {
    // 首先看看左边当前的 Tuple 是否为空
    if (!left_tuple_.has_value()) {
      if (!left_executor_->Next(&tmp_tuple, &tmp_rid)) {
        return false;
      }
      left_tuple_ = std::make_optional<Tuple>(tmp_tuple);
    }

    // 然后就开始比较，看看左边是否能匹配上右边
    while (right_tuple_iter_ != right_tuples_.end()) {
      auto join_result = predicate->EvaluateJoin(&left_tuple, left_schema, right_tuple, right_schema);

      ++right_tuple_iter_;

      // Value 类有一个 GetAs<class T> 方法，可以转成想要的类型，我们就转成 bool 类型看看是否是 true
      if (/* join_result is True */) {
        std::vector<Value> values;
        // 参考 Tuple::ToString 方法，获取得到每一列的值
        for (uint32_t column_itr = 0; column_itr < left_schema.GetColumnCount(); column_itr++) {
            values.push_back(left_tuple.GetValue(&left_schema, column_itr));
        }
        for (uint32_t column_itr = 0; column_itr < right_schema.GetColumnCount(); column_itr++) {
            values.push_back(right_tuple->GetValue(&right_schema, column_itr));
        }

        *tuple = Tuple(values, &plan_->OutputSchema());
        return true;
      }
    }

    // 到这一步还没有，那么就清空相关数据
    left_tuple_.reset();
    right_tuple_iter_ = right_tuples_.begin();
  }

  std::cout << "End" << std::endl;
  return false;
}
```

没啥困难的，有一个地方是匹配上之后，如何获取两张表的值，这个直接参考 `ToString()` 方法就行，其实也简单。实现地很快。


## 实现，版本二

我知道还没判断 INNER JOIN 和 LEFT JOIN，但当时还是想跑一遍试试看，结果好像第一句就出错了。后来才知道课程要求是必须要双重遍历，然后右表直接用 Init 就可以回到最开头，这样还更方便了。不需要事先存储右表的 RIDs 了。

```cpp
  while (true) {
    // 首先看看左边当前的 Tuple 是否为空，如果不为空，说明它之前和右边的某条已经匹配过了
    if (!left_tuple.has_value()) {
      // ....
      if (!left_executor_->Next(&tmp_tuple, &tmp_rid)) {
        return false;
      }
      left_tuple_ = std::make_optional<Tuple>(tmp_tuple);
    }

    auto left_tuple = left_tuple_.value();

    Tuple right_tuple;
    RID right_rid;
    while (right_executor_->Next(&right_tuple, &right_rid)) {
      auto join_result = predicate->EvaluateJoin(&left_tuple, left_schema, &right_tuple, right_schema);

      // make tuple and return ture
    }

    // 清空相关数据，直接用 Init 回到开头，这样更方便了
    left_tuple_.reset();
    right_executor_->Init();
  }
```

执行之后发现能 PASS 几条语句，然后就是 INNER JOIN 和 LEFT JOIN 的不同处理了。其实很直观，LEFT JOIN 就是在清空相关数据那里，加一个输出 NULL 即可。

```cpp
  while (true) {
    // 首先看看左边当前的 Tuple 是否为空，如果不为空，说明它之前和右边的某条已经匹配过了
    // 这里是 Left Join 和 Inner Join 最大的注意点，需要根据这个来判断遍历结束后是否输出 NULL
    auto is_matched_before = left_tuple_.has_value();
    if (!is_matched_before) {
        // 和上面一样，加载 left_tuple
    }

    // ... 中间是一模一样的，即遍历右表看看能否匹配

    // 到这一步还没有，如果是 Inner Join，就直接跳过了
    // 但如果是 Left Join，需要看看是否 is_matched_before，如果不是说明没有右表全部匹配不了，那么就输出 NULL
    if (plan_->GetJoinType() == JoinType::LEFT && !is_matched_before) {
      std::vector<Value> values;
      // 添加左表，这个略过了...
      // 添加右表，即添加空值
      for (uint32_t column_itr = 0; column_itr < right_schema.GetColumnCount(); column_itr++) {
        values.push_back(ValueFactory::GetNullValueByType(right_schema.GetColumn(column_itr).GetType()));
      }
      // ...
    }

    // 清空相关数据 ...
  }
```

## NestedInexJoin

NestedIndexJoin 是指如果发现右表的查找列有索引，那就可以不需要遍历右表，直接用索引去查找是否匹配就行了。这个也确实很直观，它的结构如下：

```
EXPLAIN SELECT * FROM t1 INNER JOIN t2 ON v1 = v3;
NestedIndexJoin { type=Inner, key_predicate=#0.0, index=t2v3, index_table=t2 } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
  SeqScan { table=t1 } | (t1.v1:INTEGER, t1.v2:INTEGER)
```
其中 SeqScan 负责的是左表，索引处理的是右表。

和 NestedLoopJoin 不同的是，构造函数只存一个左表，这里直接叫 child_executor；不存右表，右表是通过上面的 index_table 去找。

Init() 方法也没有特别好说的，仿照之前涉及索引的代码去找右表即可：

```cpp
  auto catalog = exec_ctx_->GetCatalog();

  // 获取索引
  auto index_id = plan_->index_oid_;
  auto index_info = catalog->GetIndex(index_id);
  inner_index_ = index_info->index_.get();
  inner_index_schema_ = &index_info->key_schema_;

  // 和 NestedLoopJoinExecutor 一样，需要保存 Left(Child) 的 Tuple，Right(Inner) 的当前遍历到哪个 Tuple
  child_tuple_.reset();
  inner_rids_.clear();
  inner_rid_iter_ = inner_rids_.begin();
```

Next() 方法也很简单直观：

```cpp
  auto key_predicate = plan_->Predicate();
  while (true) {
    // 如果 child_tuple_ 有值，说明之前已经匹配过至少一次，继续遍历剩下的值
    if (child_tuple_.has_value() && inner_rid_iter_ != inner_rids_.end()) {
      auto inner_rid = *inner_rid_iter_;
      ++inner_rid_iter_;

      // 获取右表的 Tuple
      auto inner_tuple = inner_table_heap_->GetTuple(inner_rid).second;

      // 构造 values，返回结果，和之前代码基本一样 ...
    }

    // 进行到这一步，说明需要从头开始了，即获取 Child 的下一条，然后索引寻找
    // 和之前代码一样，即更新 child_tuple_

    // 在 Child (Left, Outer) 中筛出对应的列的数据，然后利用 Key 构造一个 Tuple
    auto key_value = key_predicate->Evaluate(&child_tuple_.value(), child_schema);
    auto key_tuple = Tuple({key_value}, inner_index_schema_);

    // NestedIndexJoin 直接用索引去搜匹配
    // 具体是用 ScanKey 在右表查是否有对应的 Tuple，要注意每次都要清空，因为 ScanKey 是累加的...
    inner_rids_.clear();
    inner_index_->ScanKey(key_tuple, &inner_rids_, exec_ctx_->GetTransaction());
    inner_rid_iter_ = inner_rids_.begin();

    // 然后也是，如果这里为空的话，那就说明这个 Child 没有右表匹配上的条目，如果是 Left Join 要输出为 NULL
    if (plan_->GetJoinType() == JoinType::LEFT && inner_rids_.empty()) {
      // 构造左表 + NULL，这里和之前代码也一样 ...

      // 不要忘记 Reset 一下，不然会重复输出
      child_tuple_.reset();
      return true;
    }

    // 然后就是不为空，那么就要输出。但这里不用写了，因为 while 循环第一段（即判断是否有值）已经在做这种事情了
  }
  return false;
```

唯一注意点就是用 `ScanKey` 获取得到右表匹配的 Tuples，然后要注意需要先清空，因为 `ScanKey` 是累加的。

完成之后，P03.13 及之前都可以了。

## 特别致谢
感谢本文的博主，让我无从下笔的时候能有参考：https://blog.csdn.net/qq_40878302/article/details/137741785