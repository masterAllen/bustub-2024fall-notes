# Project4.7 完成事务的几个函数

下面进行事务的两个函数实现，分别是 Commit 和 Abort 和 GC


## Commit

`Commit` 也就是事务提交后，系统应该做什么？最最关键的就是：

- 将该事务修改的所有 tuple 从临时时间(temp_ts) 改成正式的 commit_ts。

而这根据 `WriteSet` 来遍历即可，
```cpp
  for (const auto &[table_oid, write_set] : txn->GetWriteSets()) {
    auto table_info = catalog_->GetTable(table_oid);
    auto table_heap = table_info->table_.get();
    for (const auto &rid : write_set) {
      auto tuple_meta = table_heap->GetTupleMeta(rid);
      tuple_meta.ts_ = commit_ts;
      table_heap->UpdateTupleMeta(tuple_meta, rid);
    }
  }
```

然后还有一个坑：不要过早提前更新时间，既不要立刻修改 `last_commit_ts_`，否则这会让并发 `Begin()` 看到一个**尚未真正提交完成**的时间戳。

```cpp

// 先更新对应的 tuple 的 meta.ts_.........
// 最后再更新 last_commit_ts_ = commit_ts ......
```

## Abort

`Abort` 也就是事务回滚后，要做的事情，思路也非常直观：根据 `WriteSet` 来遍历，找到修改的 tuple，然后根据 undo logs 进行回滚。

这里给出回滚操作，其实没啥，还是那老一套的 `GetUndoLog` 和 `ReconstructTuple` 来操作即可。
```cpp
      Tuple restored_tuple = base_tuple;
      TupleMeta restored_meta{0, true};
      std::optional<UndoLink> restored_link = std::nullopt;

      if (undo_link.has_value() && undo_link->prev_txn_ == txn->GetTransactionId()) {
        auto undo_log = txn->GetUndoLog(undo_link->prev_log_idx_);
        auto restored_tuple_opt = ReconstructTuple(schema, base_tuple, base_meta, {undo_log});
        if (restored_tuple_opt.has_value()) {
          restored_tuple = *restored_tuple_opt;
          restored_meta = TupleMeta{undo_log.ts_, false};
        } else {
          restored_meta = TupleMeta{undo_log.ts_, true};
        }
        restored_link = undo_log.prev_version_.IsValid() ? std::make_optional(undo_log.prev_version_) : std::nullopt;
      }
```

这个任务其实是 Bonus 任务，即附加项，但是放在这里也是合适的，不难。

## GC

GC 其实就是统计一下系统中所有的事务，有没有已经可以删除的。

### 几个小问题

#### 问题一

也就是说后面我们会主动删除某些事务？因为看起来，假如事务不会被删除的话，那么 watermark 始终为最小的 read_ts，此时 GC 结果中永远不会有无用的事务。

> 回答：是的，如果不删除，永远不会有无用的事务

#### 问题二

实现这个首先要回答好一个问题：watermak 是所有事务最小的 read_ts，这里面事务包含所有事务吗，还是没有 aborted 的事务？

> 回答：不是所有事务，不包括 ABORTED 和 COMMITED 的事务。具体的实现是在 `Begin(), Commit(), Abort()` 等方法中，他们调用了 `AddTxn(), RemoveTxn()` 方法。这样会删除增加事务时更新 watermak


### 一个大问题

了解完上面的问题，就明白 GC 做了什么。有些已经 ABORTED 或者 COMMITED 的事务没必要存在了。比如最开始只有一个事务做了很多操作，后来它提交了；接着才有许多其他事务开启，在系统中来说，这个最早的事务已经没有用了，因为其他的事务在它之后才产生，不会看它的修改。

OK，那实现似乎很直观：直接找出所有 `commit_ts` 或者 `abort_ts` 小于 `watermark` 的事务，然后删除？

这是很关键的问题，并不是，有两个原因：

1. 现在的框架中没有 `abort_ts`，所以对于 ABORTED 的事务，不太好处理。这个倒好办，加一个变量就行。
2. 更关键的，上面的只是充要条件，即：如果 `commit_ts` 大于 `watermark`，其实也有可能删除。

第 2 点而言，insert 是一个很好的反例，假设有一条记录的版本链是这样：

```
table = None
1. T1, start
2. T2, insert tuple (10, 100)
3. T2, commit
4. T3, start
```

很明显按照上面的逻辑，这些事务都不应该删除。但是我们看 T2 真的还有用吗？对于 T1 而言，它无所谓，因为它开始的时候表就是空的；对于 T3 而言，它也无所谓，因为它启动的时候，表就是 `(10, 100)`，不需要回滚。

所以此时 T2 是可以删除的，因此说上面的逻辑只是充分条件。

**Project4 中唯一一处 AI 搞半天没明白的部分，代码写出来了但是它没弄清楚这个问题，人类阵营还没输😭**

### 新的思路和实现

那如何去做？一个新的思路：遍历所有的 table 中的所有 tuple，然后去不断回溯，回溯直到 watermark 结束。每次回溯都涉及一个 UndoLog，根据 UndoLog 找到对应的事务，这个事务就不能被删除。


```cpp
  // 遍历所有的表
  for (const auto &table_name : catalog_->GetTableNames()) {
    // 遍历当前表中的 所有 tuple
    for (...) {
      // base_meta, base_tuple, rid

      // tuple 被已经提交的事务修改过，且修改时间小于 watermark
      // 如果这样，那说明这个 tuple 对任何事务都是可见的
      const bool base_visible = base_meta.ts_ < TXN_START_ID && base_meta.ts_ <= watermark;
      if (base_visible) {
        continue;
      }

      // 否则就不断去回溯，回溯直到 watermark 结束
      auto undo_link = GetUndoLink(rid);
      auto cur_link = undo_link;
      while (cur_link.has_value()) {
        auto undo_log = GetUndoLogOptional(*cur_link);
        if (!undo_log.has_value()) {
          break;
        }

        // 每次获取 undolog 对应的 txn，说明这个 txn 不能被删除
        reachable_txns.insert(cur_link->prev_txn_);
        if (undo_log->ts_ <= watermark) {
          break;
        }

        cur_link = undo_log->prev_version_.IsValid() ? std::make_optional(undo_log->prev_version_) : std::nullopt;
      }
    }
  }
```

非常巧妙地思路，可以仔细体会。后续就是把那些没遍历到的事务，删除掉即可。