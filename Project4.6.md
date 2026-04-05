# Project4.6 实现 UpdateExecutor 和 DeleteExecutor


现在来实现 UpdateExecutor，在这里我们需要想一下我们的骨架。主要是和 Project3 有哪些不同：  

##  骨架

### 1. 从 child executor 拿到要修改的 RID

也就是先知道“要改哪条记录”。这个和 P3 是一样的。

### 2. 检查 write-write conflict

这是这次任务新增的关键点。

你要看这条记录当前 `TupleMeta.ts_`，判断它能不能被当前事务修改。

需要拦住两类情况：

- **别的未提交事务已经改了它**
  - 当前事务不能再改
- **这条记录已经被一个“对我来说太新”的已提交事务改掉/删掉**
  - 比如别的事务在我开始之后删掉了它，我再去删/改它，也算冲突

举例而言：

```
T1: start, tuple = (0, 100)
T2: start, tuple = (0, 100)
T2: update (0, 100) -> (0, 200)
T1: update (0, 100) -> (0, 300) --> conflict
```

如果有冲突，让 SQL 语句失败，事务进入 tainted 状态。
- `txn->SetTainted()`
- `throw ExecutionException`

### 3. 生成新 tuple

这里没有什么不同，更新呗。

### 4. 生成 UndoLog 相关结构

这一步很重要。需要先判断是不是 self-modification，即这个 tuple 是不是被自己改过，原因就是项目要求：

**同一个事务对同一个 RID 最多只有一条 undo log。**

如果这条记录当前的 `meta.ts_` 就是：

```cpp
txn->GetTransactionTempTs()
```

说明：

**这条记录已经被当前事务自己改过一次了。**

此时不能再追加一条新的 undo log 了，而是在原来 undo log 基础上更新。

比如：

```
T1, start, tuple = (0, 100)
T1, update (0, 100) -> (1, 100)
T1, update (1, 100) -> (1, 200)
```

第二句之后 `undo_log = (col=0, value=0)`，第三句之后更新为 `undo_log = (col=1, value=100, col=0, value=0)`。

**所以这里为什么给了我们两个函数**：`GenerateNewUndoLog(...)` 和 `GenerateUpdatedUndoLog(...)`

生成或者更新 UndoLog 之后，也别忘记把链表头，也就是 UndoLink 更新一下。


### 5. 更新 tuple

这里没什么好说的，和 P3 一样，更新呗。而且现在直接给了我们 `UpdateTupleInPlace` 这个 API，直接用就行。

### 6. 把这条 RID 记进 write set

这样提交时 `Commit()` 才能把这些 tuple 的 `meta.ts_` 从临时事务时间戳刷成正式 `commit_ts`。

你前面已经在 `InsertExecutor` 里做了，  
现在 `UpdateExecutor` 和 `DeleteExecutor` 也要做。


## UpdateUndoLog

上面相当关键、变化也很大的第四步，里面涉及了两个函数：`GenerateNewUndoLog(...)` 和 `GenerateUpdatedUndoLog(...)`。

那就实现他们吧，先来说 `GenerateNewUndoLog(...)`，这里直接贴出代码。

```cpp
auto GenerateNewUndoLog(const Schema *schema, const Tuple *base_tuple, const Tuple *target_tuple, timestamp_t ts,
                        UndoLink prev_version) -> UndoLog {
  const auto column_count = schema->GetColumnCount();
  std::vector<bool> modified_fields(column_count, false);
  std::vector<Value> undo_values;
  bool is_deleted = false;

  // TODO: 为什么这里会出现 base_tuple 和 target_tuple 都为空的情况？
  if (base_tuple != nullptr && target_tuple != nullptr) {
    for (uint32_t i = 0; i < column_count; i++) {
      auto base_val = base_tuple->GetValue(schema, i);
      auto target_val = target_tuple->GetValue(schema, i);
      if (!base_val.CompareExactlyEquals(target_val)) {
        modified_fields[i] = true;
        undo_values.push_back(base_val);
      }
    }
    is_deleted = false;
  } else if (base_tuple != nullptr && target_tuple == nullptr) {
    modified_fields.assign(column_count, true);
    undo_values = GetTupleValues(*base_tuple, schema);
    is_deleted = false;
  } else if (base_tuple == nullptr && target_tuple != nullptr) {
    is_deleted = true;
  } else {
    is_deleted = true;
  }

  auto undo_schema = GetUndoLogSchema(schema, modified_fields);
  auto undo_tuple = Tuple(undo_values, &undo_schema);
  return UndoLog{is_deleted, modified_fields, undo_tuple, ts, prev_version};
}
```

一个问题，为什么会出现 base_tuple 和 target_tuple 都为空的情况。简单来说，防御性编程，以防万一...

再来谈 `GenerateUpdatedUndoLog(...)`，这里就简单提一下，其实没啥好说的，就获取旧的，比较一下，把差异放在 undo log 中就行。注意点有以下几个

1. 要保留旧 undo log 中已经记录过的列，并把这次新引入的列追加进去，绝不删除原有列。也就是说：后面即使又被改回原值，也不能从 undo log 里移除。而是保留着，同时这列的值是原值。

```
T1, start, tuple = (0, 100)
T1, update (0, 100) -> (0, 200)
T1, update (0, 200) -> (0, 100)
```

第二句之后 `undo_log = (col=0, value=0)`，第三句之后更新为 `undo_log = (col=0, value=0, col=1, value=100)`。

所以这里要小心处理，一开始就是直接比对最原始的 tuple 和最新的 tuple，这样有的样例无法通过。

2. 要处理 insert 的情况，此时 `original` 是空，这里我们还是不插入任何 UndoLog：

```cpp
const auto column_count = schema->GetColumnCount();
auto current_tuple = base_tuple == nullptr ? MakeNullTupleForSchema(schema) : *base_tuple;
auto original_tuple = ReconstructTuple(schema, current_tuple, TupleMeta{0, base_tuple == nullptr}, {log});

// 如果最初状态根本不存在（典型场景：insert 后继续 update / delete），继续保持“回滚后不存在”的语义。
if (!original_tuple.has_value()) {
return log;
}
```

3. 处理 delte 的情况，也就是 `target_tuple == nullptr`，此时等价于把整行都放入 undolog 中。


## 实现

检查是否有 `Write-Write Conflict`，这里不讲了，其实就是之前的 `IsVersionVisible` 的逻辑，通过 `ts` 来判断。

```cpp
auto *txn = exec_ctx_->GetTransaction();
auto *txn_mgr = exec_ctx_->GetTransactionManager();
auto table_oid = plan_->GetTableOid();
int return_count = 0;

// Pipeline breaker: 先把 child 输出全部缓存在本地，再统一执行更新。
std::vector<std::pair<Tuple, RID>> buffered_tuples;
while (child_executor_->Next(tuple, rid)) {
    buffered_tuples.emplace_back(*tuple, *rid);
}

for (const auto &[old_visible_tuple, tuple_rid] : buffered_tuples) {
    auto [base_meta, base_tuple] = table_info_->table_->GetTuple(tuple_rid);
    auto undo_link = txn_mgr->GetUndoLink(tuple_rid);

    // 检查 Write-Write Conflict，如果存在，那么 transaction 进入 tainted 状态，然后抛出异常
    if (HasWriteWriteConflict(base_meta, txn)) {
        MarkTaintedAndThrow(txn);
    }

    // 计算新的 tuple 值
    // ...
    auto new_tuple = Tuple(new_values, &table_info_->schema_);
    auto new_meta = TupleMeta{txn->GetTransactionTempTs(), false};

    // self-modified: 是否当前这个 tuple 已经被自己修改过了；
    const bool self_modified = base_meta.ts_ == txn->GetTransactionTempTs();
    if (self_modified) {
        // 1. 如果是，那由于我们对每个 tuple 维护一个 undolog，所以这里应该用 GenerateUpdatedUndoLog 去更新维护的 undolog
        // 这里为什么多加一个判断：见下一章节
        if (undo_link.has_value() && undo_link->prev_txn_ == txn->GetTransactionId()) {
            auto old_log = txn->GetUndoLog(undo_link->prev_log_idx_);
            auto new_log = GenerateUpdatedUndoLog(&table_info_->schema_, &base_tuple, &new_tuple, old_log);
            txn->ModifyUndoLog(undo_link->prev_log_idx_, std::move(new_log));
        }
    } else {
        // 2. 如果不是，这个 tuple 我们之前没有碰过（否则就会有 Write-Write Conflict），此时用 GenerateNewUndoLog 去生成一个新的 undolog
        auto old_undo_link = undo_link.has_value() ? *undo_link : UndoLink{};
        auto new_log = GenerateNewUndoLog(&table_info_->schema_, &base_tuple, &new_tuple, base_meta.ts_, old_undo_link);
        auto new_undo_link = txn->AppendUndoLog(std::move(new_log));
        txn_mgr->UpdateUndoLink(tuple_rid, new_undo_link, nullptr);
    }

    // 更新 Tuple，直接使用 UpdateTupleInPlace 去更新
    table_info_->table_->UpdateTupleInPlace(new_meta, new_tuple, tuple_rid, nullptr);
    // 不要忘记把事务的 write set 也更新一下，最后事务 commit 的时候会用到
    txn->AppendWriteSet(table_oid, tuple_rid);

    // 更新索引：删除旧的，插入新的
    for (auto &index : indexes_) {
        auto real_index = index->index_.get();
        auto old_key = old_visible_tuple.KeyFromTuple(table_info_->schema_, index->key_schema_, real_index->GetKeyAttrs());
        auto new_key = new_tuple.KeyFromTuple(table_info_->schema_, index->key_schema_, real_index->GetKeyAttrs());
        real_index->DeleteEntry(old_key, tuple_rid, txn);
        real_index->InsertEntry(new_key, tuple_rid, txn);
    }

    return_count++;
}
```

## 一个问题

这里有一个问题，如果弄懂，对理解有帮助：

> 一个事务首先 insert (0, 100)，当他想要更新为 (0, 200) 是，此时他是怎么判断出 old_tuple == null 的？

这时候系统并不会判断出 `old_tuple == null`，问题就在于如下语句：

```cpp
if (self_modified) {
    if (undo_link.has_value() && undo_link->prev_txn_ == txn->GetTransactionId()) {
        //...
    }
}
```

这里他会判断出 `self_modified`，但我们 insert 是不插入 undolog 的，所以此时 `undo_link` 为空，于是不进入 `if` 语句。

所以这里**依然没有生成 UndoLog**，只是普通进行了更新 tuple，这是合理的，非常精妙的逻辑。因为以后再更新，同样 UndoLog 还是为空，还是继续正常的逻辑。


## DeleteExecutor

删除也是类似的，这里不细讲。总之都是上面的逻辑，判断是否是 `self-modified`，如果是，则用 `GenerateUpdatedUndoLog` 去更新维护的 undolog，否则用 `GenerateNewUndoLog` 去生成一个新的 undolog。