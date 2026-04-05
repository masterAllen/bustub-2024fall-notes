# Project4.5 实现 SeqScanExecutor 和 InsertExecutor

## 实现 SeqScanExecutor

实际上和 P3 的唯一区别就是，我们拿到 tuple 之后不能立刻当成正确结果返回，而是要执行前面的 UndoLog 回溯操作，找到这个事务真正应该看到的版本。如果前面理解了，这里会很顺畅。

```cpp
auto SeqScanExecutor::Next(Tuple *tuple, RID *rid) -> bool {
  // ...

  auto *txn = exec_ctx_->GetTransaction();
  auto *txn_mgr = exec_ctx_->GetTransactionManager();

  // 遍历 rids，每一次上层会调用 Next，每次调用就把当前的 rid 对应结果放在 tuple 中，让上层用
  while (true) {
    if (table_iter_->IsEnd()) {
      return false;
    }
    auto now_rid = table_iter_->GetRID();
    ++(*table_iter_);

    // TableHeap 里永远是最新版本；要按当前事务的 read_ts 回退到它真正可见的版本，所以我们要根据 undo_logs 逐步恢复
    // undo_logs 如何获取，首先要从 txn_mgr 获取 undo_link，这是最新的 undo_log 链接，即相当于链表头部；然后我们就可以逐步获取了
    auto [base_meta, base_tuple] = table_heap_->GetTuple(now_rid);

    // 获取 undo_link，从 txn_mgr 中获取，这相当于是 undo_logs 的链表头部
    auto undo_link = txn_mgr->GetUndoLink(now_rid);

    // 先收集“为了回到当前事务可见版本，需要撤销哪些更新”。
    auto undo_logs = CollectUndoLogs(now_rid, base_meta, base_tuple, undo_link, txn, txn_mgr);
    if (!undo_logs.has_value()) {
      continue;
    }

    // 再把这些 undo logs 应用到最新版本上，恢复出历史可见版本。
    auto visible_tuple = ReconstructTuple(table_schema_, base_tuple, base_meta, *undo_logs);
    if (!visible_tuple.has_value()) {
      continue;
    }

    // 过滤条件也必须基于重建后的可见版本，而不是 table heap 里的最新版本。
    auto is_ok = true;
    if (plan_->filter_predicate_ != nullptr) {
      auto filter_predicate = plan_->filter_predicate_;
      is_ok = filter_predicate->Evaluate(&visible_tuple.value(), *table_schema_).GetAs<bool>();
    }

    if (is_ok) {
        // ...
    }
  }
  return false;
}
```

## 实现 InsertExecutor

其实 InsertExecutor 更简单，我们可以在这里暂停，想一想有了事务之后，对于 Insert 来说究竟有什么要改动的？

首先就是插入的时候，不再像以前插入固定的 `0`，而是插入当前事务的 `temp_ts`。
```cpp
// auto tuple_meta = TupleMeta{0, false};
auto tuple_meta = TupleMeta{txn->GetTransactionTempTs(), false};
```

然后就是写入事务的 `writeset`，这个其实也是对事务的数据结构了解一下就知道了。这里的 `WriteSet` 干啥用的，其实相当有用。比如事务 Commit 之后，这些被我改过的 tuple 中的 `ts` 都不能是 `temp_ts` 了，而是要变成 `commit_ts`，这个步骤就需要用到 `WriteSet`。再比如事务 Abort 的时候，同理需要找到所有 tuple，也需要用到 `WriteSet`。
```cpp
txn->AppendWriteSet(table_id, new_rid);
```

最后就是一个关键的问题，我需要添加 `UndoLog` 吗？直觉上，我应该留下 `UndoLog`，这样后面别人或者自己才能用这个 `UndoLog` 去还原，但其实不需要：

**理解这个问题相当关键，请务必理解**

还是举例说明。初始时，表里只有一条记录：

```text
T1, start, NULL
T2, start, NULL
T2, insert tuple = (0, 100)
T2, commit
T1, select * from t
```

现在看 T1 去查表时会怎样，就简单的 `select * from t` 查找所有。因为 `T1.read_ts = 0`，而：

```
tuple.ts_ = 3 > 0
```

所以对 T1 来说，这条 tuple 不可见，在 `CollectUndoLogs` 中一开始就会检查是否可见，不可见返回了 `std::nullopt`。而我们的 `SeqScan` 是这样的：

```cpp
// 获取 undo_link，从 txn_mgr 中获取，这相当于是 undo_logs 的链表头部
auto undo_link = txn_mgr->GetUndoLink(now_rid);

// 先收集“为了回到当前事务可见版本，需要撤销哪些更新”。
auto undo_logs = CollectUndoLogs(now_rid, base_meta, base_tuple, undo_link, txn, txn_mgr);
if (!undo_logs.has_value()) {
    continue;
}
```

可以看到，我们进行了 `undo_logs.has_value()` 判断，如果为 `false`，则说明这条 tuple 不可见，需要继续找下一个。