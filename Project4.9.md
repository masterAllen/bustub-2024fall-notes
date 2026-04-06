# Project4.9 Primary Key 含义大更新

前一篇完成 Task4.1 的要求：对 InsertExecutor 的修改，很简单。但是从 Task4.2 开始，有相当大的更新，**而且开始有并发问题**，这也难怪官网上进行了提醒：

![](images/20260331140358.png)

## Primary Key 大更新

**从 Task 4.2 开始，Primary Key 和 RID 的关系固定了，所以你不能再把删除后重新插入实现成分配一个全新的 RID。**

### 为什么要这样

还是因为 MVCC，旧事务可能还需要通过索引访问历史版本。如果你在 delete 时把索引 entry 真删掉了，那么：

- 新事务当然看不到了
- 但老事务也没法沿这个 key 找到旧 RID，再去读历史版本

举例而言，如果按照旧方法：

```
tuple = (10, 100) --> rid = r1, primary_key_map = {10: r1}
T1, start
T2, start
T2, delete (10, 100) --> primary_key_map = {}
T2, insert (10, 200) --> rid = r2, primary_key_map = {10: r2}
T1, select * from table where col0 = 10
```

按照 MVCC 语义，T1 当时启动的时候，数据库中快照能看到 `(10, 100)`，结果现在，它根据 primary_key 查找，得到了 `(10, 200), rid=r2`，而 `rid=r2` 这个 tuple 是没有 undo log 的，所以 T1 会返回 `(10, 200)` 这个错误的结果。

如果按照新的方法：

```
tuple = (10, 100, not_deleted) --> rid = r1, primary_key_map = {10: r1}
T1, start
T2, start
T2, delete (10, 100, deleted=true) --> primary_key_map = {10: r1}, undo_log = {(10, 100), ts=T2}
T2, insert (10, 200, deleted=false) --> rid = **r1**, primary_key_map = {10: r1}, undo_log = {(10, 100), ts=T2}
T1, select * from table where col0 = 10
```

此时 T1 根据 primary_key 查找，得到了 `(10, 100), rid=r1`，而 `rid=r1` 这个 tuple 有 undo_log，即 `{(10, 100), ts=T2}`，所以 T1 会恢复到 `(0, 100)` 这个正确结果。


### 这个带来什么改变

主要有四件事：

1. 修改 `InsertExecutor`
如果某个主键对应的 tuple 只是被 delete 标记了，而索引 entry 还在，那么后续再插入同样主键时**复活**这个RID，而不是以前的新建。

2. 修改 `DeleteExecutor`
delete 时不能再简单把索引 entry 删掉，而是标记为 deleted，这样后续才能有机会复活

3. 修改 `UpdateExecutor`
如果 update 改了 Primary Key 的列，比如 `(0,100)->(1,100)`，那么逻辑会变得更加复杂，具体的细节看对应的章节。


### 并发的问题
如下是例子：

```
tuple = (0, 100, RID=r5)
T2: update tuple to (1, 100)
T1: select * from table
``` 

就是这么简单的例子，但是如果 T1 和 T2 是多线程，那么可能：

#### 时刻 1

T2 插进来完成更新，但**还没有更新 UndoLink**，现在：

- tuple = (1, 100), ts=T2

#### 时刻 2

T1 开始执行：
- tuple = (1, 100), ts=T2
- undo_link = NULL

所以：T1 结束，它得到 `(1, 100)`

#### 时刻 3

T2 最后再更新 UndoLink: 

```cpp
txn_mgr->UpdateUndoLink(r5, undo_link, NULL);
```

结果：
现在 T1 手里拿到的是一个不一致的组合，是 `(1, 100)` 而不是 `(0, 100)`！


所以之后我们要用 `GetTupleAndUndoLink(...)` 来读取，用 `UpdateTupleAndUndoLink(...)` 来写入，这是使用锁确保原子操作的函数，记住**之前的 Task 中相关的代码**都要改成用这两个，因为我们已经正式进入并发的世界了，比如 `SeqScanExecutor` 和 `IndexScanExecutor` 都要改成用这两个。


## InsertExecutor 的修改

把 `InsertExecutor` 分成两条路径：

- `primary key` 没命中：正常新插入 + 插索引，这个保持不变
- `primary key` 命中一个已删除的 RID：复用旧 RID，原地“复活”tuple，不再新建索引项

所以只看后面的情况，整体上很复杂，但是其实每一步都是考验细节，难度倒不是很大：

```cpp
    // Task 4.2: 如果主键索引命中一个 deleted tuple，则应当复用原来的 RID，而不是创建新的索引项。
    // primary_hits: 也是上面通过 ScanKey 查找得到的
    if (primary_index != nullptr && !primary_hits.empty()) {
      auto target_rid = primary_hits.front();

      // 原子化读取 tuple 和 undo_link
      auto tuple_and_undo = GetTupleAndUndoLink(txn_mgr, table_heap, target_rid);
      auto base_meta = std::get<0>(tuple_and_undo);
      auto base_tuple = std::get<1>(tuple_and_undo);
      auto undo_link = std::get<2>(tuple_and_undo);

      // 是否有 Write-Write Conflict
      if (HasWriteWriteConflict(base_meta, txn)) {
        MarkTaintedAndThrow(txn);
      }

      // 根据 UndoLog 还原出当前这个事务 txn 应该看到的版本
      auto undo_logs = CollectUndoLogs(target_rid, base_meta, base_tuple, undo_link, txn, txn_mgr);
      auto visible_tuple =
          undo_logs.has_value() ? ReconstructTuple(&table_schema, base_tuple, base_meta, *undo_logs) : std::nullopt;
        
      // 如果这个 tuple 存在，说明不应该（这里是 insert，不能重复插入）
      if (visible_tuple.has_value()) {
        MarkTaintedAndThrow(txn);
      }

      // 到了这里，说明：能找到这个 tuple，但是这个 tuple 其实已经是删除的状态
      // 应该做的事情：复活它，并且把值修改成本次 insert 想要的值

      // 是否是 self-modified，是否是当前这个事务删除掉它的
      const bool self_modified = base_meta.ts_ == txn->GetTransactionTempTs();
      std::optional<UndoLink> new_undo_link = undo_link;
      if (self_modified) {
        // 如果是当前这个事务删掉的，那么就用 GenerateUpdatedUndoLog 去更新维护的 undolog
        // 这里要判断 undo_link 是否有值，原因：insert 再 delete 再 insert，undo_log 是空的，但是 tuple 上面的 ts_ 确实变成了自己
        if (undo_link.has_value() && undo_link->prev_txn_ == txn->GetTransactionId()) {
          auto old_log = txn->GetUndoLog(undo_link->prev_log_idx_);
          auto new_log = GenerateUpdatedUndoLog(&table_schema, &base_tuple, tuple, old_log);
          // 因为是更新 txn 的 undo log，所以也要把 txn 自己的 undo_log buffers 对应的更新好
          txn->ModifyUndoLog(undo_link->prev_log_idx_, std::move(new_log));
          // 此时 undo_link 这个链表头其实还是自己，这句话有点多此一举，但放这里清晰一些
          new_undo_link = undo_link;
        }
      } else {
        // 如果是别的事务删掉的，说明自己肯定没有碰过，那么就用 GenerateNewUndoLog 去生成一个新的 undolog
        auto old_undo_link = undo_link.has_value() ? *undo_link : UndoLink{};
        auto new_log = GenerateNewUndoLog(&table_schema, nullptr, tuple, base_meta.ts_, old_undo_link);
        // 因为是新的 undolog，所以新加入到 txn 维护的 undo_logs 中
        // 并且更新 Undo_link 这个链表头，更新成我们这个 txn 对应的 undo_log
        new_undo_link = txn->AppendUndoLog(std::move(new_log));
      }

      // 复活
      auto new_meta = TupleMeta{txn->GetTransactionTempTs(), false};
      auto updated = UpdateTupleAndUndoLink(
          txn_mgr, target_rid, new_undo_link, table_heap, txn, new_meta, *tuple,
          [&](const TupleMeta &meta, const Tuple &table_tuple, RID now_rid, std::optional<UndoLink> now_undo_link) {
            return now_rid == target_rid && meta == base_meta && IsTupleContentEqual(table_tuple, base_tuple) &&
                   now_undo_link == undo_link;
          });
      if (!updated) {
        MarkTaintedAndThrow(txn);
      }

      txn->AppendWriteSet(table_id, target_rid);
      inserted_rows++;
      continue;
    }
```

这里 `self_modified` 那里有一个微妙的逻辑，如果是 False，那么自己肯定没有这个 tuple 的 undo log，否则一定会触发 Write-Write Conflict。具体见 [references/007.md](./references/007.md)

还有一处，是 `AppendUndoLog` 和 `ModifyUndoLog` 的区分，简单说：

1. `AppendUndoLog` 作用是： **新增一条 undo log 到当前事务的 undo log 数组末尾。**
2. `ModifyUndoLog` 作用是： **修改当前事务已经存在的某一条 undo log。**


## DeleteExecutor 的修改

Delete 的话其实主要修改索引那里就可以了。因为删除 tuple 我们之前就是用 `is_deleted` 标志位，而不是直接擦除。索引这里其实就是判断如果是 primary key，那么就跳过：

```cpp
    // Task 4.2: 主键索引项不能删除；它需要始终指向同一个 RID，以支持历史版本访问和 tombstone 复用。
    for (auto &index : indexes_) {
      if (index->is_primary_key_) {
        continue;
      }
      auto real_index = (index->index_).get();
      auto tuple_key = tuple->KeyFromTuple(table_info_->schema_, index->key_schema_, real_index->GetKeyAttrs());
      real_index->DeleteEntry(tuple_key, *rid, txn);
    }
```

这样就保证了 primary key 的索引项不会被删除。后续查找到这个 tuple，他会根据 tuple 的 `is_delted` 标志位再去判断是否应该使用这个 tuple。


## UpdateExecutor 的修改

Update 这里就有相当多的修改了，如果这次修改不涉及主键，那还是保持原样，即比如 `(0, 100)->(0, 200)` 这种改变，和之前是一样的。但一旦涉及到 Primary Key 的修改，那就相当麻烦。

1. 首先要检查这次 Update 合法吗？比如 table 中已经有了 `(1, 200)`，现在我们修改 `(0, 100)->(1, 100)`，这是不允许的；而且这里需要用 UndoLog 回溯为正确版本再去判断。
2. 需要更新 Primary Key，旧键映射不用去删除；新键映射时，需要去判断是否已经有这条 RID 了，比如 `update0; update1; update0` 这种操作，由于我们不删除旧的映射，所以最后会是 `0->rid, 1->rid` 这种情况，最后一个 `update 0` 时新键映射会发现已经有这条 RID 了，这是合理的。

第一个检测有如下代码：

```cpp
// old_key: old primary value; new_key: new primary value
std::vector<RID> existing_rids;
real_index->ScanKey(new_key, &existing_rids, txn);
for (const auto &existing_rid : existing_rids) {
    // 主键索引可能仍然保存着“当前这条 tuple 自己”的旧映射；这不构成冲突。
    if (existing_rid == tuple_rid) {
        continue;
    }

    // 索引命中的另一个 RID 只有在当前事务视角下仍然可见时，才说明新主键真的被占用了。
    // 如果它对当前事务已经是 deleted / 不可见，那么这次主键更新仍然允许继续。
    auto visible_tuple =
        GetVisibleTupleForTxn(existing_rid, &table_info_->schema_, table_info_->table_.get(), txn, txn_mgr);
    if (visible_tuple.has_value()) {
        MarkTaintedAndThrow(txn);
    }
}
```

上面的 `existing_rids` 就是通过主键找到的所有 RID，按照正常理解这个 RID 只能一个，但我们现在不进行删除，所以可以有多个，比如现在:

```
insert (1, 200, rid=R1)
update (1, 200, rid=R1) -> (0, 200, rid=R1)
insert (0, 100, rid=R2)
update (0, 100, rid=R2) -> (1, 100, rid=R2)
insert (2, 300, rid=R3)
update (2, 300, rid=R3) -> (1, 300, rid=R3)
```

最后一句中，`existing_rids` 能够得到 `(R1, R2)`，此时是允许的，只不过 `R1` 结果检测是不可见的，只有 `R2` 是可见的。但我们不应该有可见的 tuple，所以会触发 `MarkTaintedAndThrow(txn);`


对于第二点，更新 Primary Key 的逻辑如下：

```cpp
      // 只判断主键索引，其他不判断。因为主键索引要求是唯一性，即一个 key 只能对应出一个 value
      if (index->is_primary_key_) {
        // 判断这次 update 是否涉及到了主键值的改变
        if (!IsTupleContentEqual(old_key, new_key)) {

          // 新主键 new_key 下面，索引里是不是已经有一条 (new_key -> 当前这个 RID) 的映射了？
          // 比如 update 1, update 2, update 1，最后一个 update 的时候会发现 1 已经和这个 RID 映射了，因为 old_key 我们不删除
          std::vector<RID> existing_rids;
          real_index->ScanKey(new_key, &existing_rids, txn);
          bool has_same_rid = false;
          for (const auto &existing_rid : existing_rids) {
            if (existing_rid == tuple_rid) {
              has_same_rid = true;
              break;
            }
          }

          // 为什么 old_key 不删除，如下：
          // CREATE TABLE t(a int PRIMARY KEY, b int); 有记录 (1, 100)，其中 RID = r7
          // 事务 T_old 很早开始，read_ts 停留在旧快照。后来又有一个事务 `T_new` 更新 (2, 100)
          // 如果删除 old_key，此时 1->None, 2->r7
          // 当 T_old 再次执行 SELECT * FROM t WHERE a = 1;` 时，会返回 None，而不是 r7
          // 如果不删除 old_key，此时 1->r7, 2->r7，此时 T_old 执行上述语句，得到 r7，然后：
          // 拿到 base_tuple(此时是 (2, 100))，再沿着 undo log 还原出旧版本

          // 如果没有 new_key -> RID，那么可以直接插进 index 中，需要判断是否失败
          if (!has_same_rid) {
            auto inserted = real_index->InsertEntry(new_key, tuple_rid, txn);
            if (!inserted) {
              MarkTaintedAndThrow(txn);
            }
          }
        }
      } else {
        // 如果是非主键，那么按照原来的方式进行修改
        real_index->DeleteEntry(old_key, tuple_rid, txn);
        real_index->InsertEntry(new_key, tuple_rid, txn);
      }
```

具体更细节的探讨：[references/008.md](./references/008.md)