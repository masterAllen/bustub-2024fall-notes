# Project3.2 Insert、Delete 等操作初步解决

继续记录解决历程之路

## Insert 的整体架构

Insert 就不像 SeqScan 那样，处于最底层只要负责遍历就够了。Insert 是再上面一层，所以我们首先就要知道怎么从最底层获取数据？

其实可以参考已经实现的 `filter_executor.cpp`，就是利用 `child_executor`，在 `insert_executor` 中的 Next 操作调用子层的 Next 即可，这个之后给代码。

首先，还是是先搞定 Init，由于有 Seqscan 的实践，这些都轻车熟路了，差别就是多了个 `child_exectuor`:

```cpp
void InsertExecutor::Init() {
  child_executor_->Init();

  auto catalog = exec_ctx_->GetCatalog();
  auto table_id = plan_->GetTableOid();
  auto table_info = catalog->GetTable(table_id);
  table_heap_ = (table_info->table_).get();
}
```

### 插入功能实现

Next 操作即先利用 `child_executor`（即 `seqscan`）获取数据，然后使用 `InsertTuple` 来插入，这个具体进入函数查看怎么用的就行，比如参数有一个是 `tuple_meta`，这个用来表示这个条目是否被删除，我们插入就设置 `false` 就行。

```cpp
auto InsertExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) -> bool {
  while (true) {
    const auto status = child_executor_->Next(tuple, rid);
    if (!status) {
      break;
    }

    auto tuple_meta = TupleMeta{0, false};
    table_heap->InsertTuple(tuple_meta, *tuple);
  }
  return true;
}
```

### 返回值

好，基础的完成了，那么再来开始处理返回值的事情，官网的指导也说了，返回的是插入了多少行，所知道这一点剩下就没难的。这个可以我也是直接看别人的博客文章知道怎么写的：
```cpp
  // 返回插入的行数
  auto tuple_value = std::vector<Value>{Value(TypeId::INTEGER, inserted_rows)};
  *tuple = Tuple(tuple_value, &plan_->OutputSchema());
  return true;
```

### 索引

各个文章和官方指导也都说了，还需要更新索引。我觉得最关键的要知道什么是索引？？我之前以为是每个条目的顺序... 确实，这个 index 太容易让人误解了。具体地去问 AI 就知道是什么了，下面是[我问的 ChatGPT 的记录](./references/001.md)

**但其实我们可以暂时搁置，因为并不是一定就有索引，必须要用特定的语法，才能为某些列创建索引。**还是问 AI，[回答的很好](./references/002.md)。如果这个表没有索引，那就当然不需要更新了。

而事实上，测试的 SQL 文件中，插入、删除、更新都不会为哪怕一个表创建索引，所以这个坑，我决定后面在填。所以索引这一块我就直接忽略了。

### 只需执行一次 Next

在 SeqScan 中，使用的一个 while 循环，直到遍历结束就返回 false，而 Insert 这里的实现，是等 child_executor 的 Next 返回 false，然后他也就停止插入了，然后返回插入的行数并且返回 true，看起来很合理。

但想一想，Insert 再上一层调用 Insert 的 Next 后，发现是 true，就会再次调用 Insert 的 Next，发现是 true，再次调用... 死循环了！所以我们必须要给一个变量，告诉上层的节点，我 Insert 已经结束了，不要再来打扰我了！

```cpp
auto InsertExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) -> bool {
  // 这里 Insert 只需要执行一次 Next，所以需要一个标志位
  if (is_done_) {
    return false;
  }
  is_done_ = true;

  // ....
}
```

## Update 和 Delete

解决完 Insert 之后，Update 和 Delete 就很简单了。先谈 Update，既然是更新，那么就要知道两个：更新哪个条目（tuple），更新成什么？

更新哪个条目？这个就调用下一层的 Next 方法去获取。更新成什么？这个则是 Update 自己这一层负责的内容。那就是获取了旧的 Tuple，然后应该会有一个更新 Tuple 的 API？但实际上不是的，只有删除和插入的 API，所以这里就曲线救国：删除更新前的，插入更新后的...

```cpp
// 只展示核心部分
while (true) {
    // 删除当前的元组，通过更新元组中的 is_deleted 字段即可实现
    auto tuple_meta = TupleMeta{0, true};
    table_info_->table_->UpdateTupleMeta(tuple_meta, *rid);

    // 插入新的元组
    // auto new_tuple_meta = TupleMeta{0, false};
    // table_info_->table_->InsertTuple(new_tuple_meta, *tuple);
    std::vector<Value> new_values{};
    // Expressions 就是每一列的表达式，可以打印出来查看；所以我们要遍历他，获取得到每一列的值
    // 比如 update t1 set v3 = 645 where v1 >= 3; 它的 Expr 为 (#0.0, #0.1, 645)
    for (auto &expr : plan_->target_expressions_) {
      // Evaluate 则是根据 expr 和传入的 Schema 去计算出对应的值；关键还是了解 Schema 是什么，最简单直接地可以就当成列来看待
      auto new_value = expr->Evaluate(tuple, table_info_->schema_);
      new_values.push_back(new_value);
    }
    auto new_tuple = Tuple(new_values, &table_info_->schema_);
    auto new_tuple_meta = TupleMeta{0, false};
    table_info_->table_->InsertTuple(new_tuple_meta, new_tuple);
}
```

上面的代码现在我回头看，确实简单，但当时还是花了好长时间去思考怎么构建新的 Tuple，有一个关键是知道 Schema 是什么，还是问 AI，[回答的很好](./references/003.md)。而且我觉得多使用 ToString 打印，很有帮助！

实现了 Update 之后，Delete 那就太简单了。这里就不写了。