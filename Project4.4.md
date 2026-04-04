# Project4.4 CollectUndoLogs 实现

## 关于事务的问与答

### 如果是单线程，是不是某个时刻只可能有一个事务？

不是。**单线程** 只表示“同一时刻只有一个执行流在跑代码”，不表示“系统里只能存在一个事务对象”。

在数据库里，即使单线程，也完全可以这样：

```cpp
auto txn1 = Begin();   // read_ts = 0
auto txn2 = Begin();   // read_ts = 0
Commit(txn2);          // commit_ts = 1
auto txn3 = Begin();   // read_ts = 1
Abort(txn3);
Commit(txn1);
```

### 事务中的 `txn_mgr->GetUndoLink` 是什么

一句话解释，`txn_mgr->GetUndoLink(rid)` 返回的是：**这条 `RID` 对应 tuple 的“版本链表头指针”**。

也就是：

**当前这条记录最新版本往前回溯时，第一条该看的 undo log 在哪里。**

整个结构像这样，获取这个链表头之后，就可以不断往回找，可以理解成链表头部指针：

```text
TableHeap latest tuple (V3)
   |
   v
GetUndoLink(rid) -> UndoLog(V2 info)
                      |
                      v
                 prev_version_ -> UndoLog(V1 info)
```

### 事务中的 `temp_ts` 是什么

`temp_ts` 本质上是事务的 `txn_id_`:
```cpp
/** @return the transaction state */
inline auto GetTransactionState() const -> TransactionState { return state_; }
```

事务的 `txn_id_` 又是什么，他是一个 `int64_t`，其中

- `0 ... TXN_START_ID - 1`：表示正常的 **commit timestamp**，即已经提交完毕的事务
- `TXN_START_ID ...`： 表示一个 **temporary transaction timestamp**，也就是某个尚未提交事务的**临时**时间戳，它提交后会改为上面的时间戳

其中 `TXN_START_ID` 用的是 `1 << 62`，之所以不用最高位，`int64_t` 会变成负数，普通比较大小会变麻烦。用次高位后，`temp_ts` 仍然是一个很大的正数，比较更方便。

所以你可以把它理解成：

- `commit_ts`：正式发布时间
- `temp_ts`：未提交作者标签


## 实现 CollectUndoLogs

`CollectUndoLogs` 的作用可以一句话概括成：

**给定“当前表里这条 tuple 的最新版本”和“当前事务的读时间戳”，找出为了还原出该事务可见版本所需要撤销的那一串 undo logs。**

然后这些 undo logs 再交给 `ReconstructTuple(...)` 去真正恢复 tuple。

### 它在整个流程里的位置

顺序扫描时，对每条 tuple 大概是这样：

1. 从 `TableHeap` 拿到最新的 tuple
2. 从 `TransactionManager` 拿到这条 tuple 的最新版本链头 `undo_link`
3. 根据链表头，调 `CollectUndoLogs(...)` 收集 undo logs
4. 再调 `ReconstructTuple(...)`，用这些 undo logs 从回滚到过去版本
5. 输出**这个事务真正应该看到的** tuple

### 实现

**CollectUndoLogs 的实现和上面的 temp_ts 高度相关，必须要理解 temp_ts**

#### IsVersionVisible

有一个辅助函数：

```cpp
auto IsVersionVisible(timestamp_t ts, Transaction *txn) -> bool {
  if (ts >= TXN_START_ID) {
    return ts == txn->GetTransactionTempTs();
  }
  return ts <= txn->GetReadTs();
}
```

它的作用？判断这个 tuple 现在对我这个事务是不是可见的。分为两种情况：

1. 这个 tuple 被某个未提交事务修改过
   1. 如果这个事务是我，也就是我刚改这个 tuple，但还没提交，那我肯定就要这一版就行了。
   ```
   txn1.execute("update (0, 1) -> (0, 2)")
   txn1.execute("select * where col1==2")
   // 应该得到 (0, 2) 结果，也就是 (0, 2) 这个最新 tuple 现在就是我想看见的
   ```
   2. 如果这个事务是其他人，那么我不能看见，我需要去找 undo log 去恢复它，**这个最后再讨论**。

2. 这个 tuple 被已经提交事务修改过。那最简单了，看看提交时间和我事务的启动时间，如果在我启动之前就提交的，那我看不到；如果在我启动之后修改的，那我需要回溯。


#### 代码骨架

有了这个辅助函数就方便很多了，相当于是一个 while 循环，每次：

1. 这个 tuple 对我是不是可见的，用 IsVersionVisible 判断，如果不可见那么返回
2. 如果可见，那么这个 undo log 在返回结果中，然后寻找链表下一个 undo log（其中链表头 `undo_link` 是参数传入进来了）

```cpp
auto CollectUndoLogs(RID rid, const TupleMeta &base_meta, const Tuple &base_tuple, std::optional<UndoLink> undo_link,
                     Transaction *txn, TransactionManager *txn_mgr) -> std::optional<std::vector<UndoLog>> {
  static_cast<void>(rid);
  static_cast<void>(base_tuple);

  if (IsVersionVisible(base_meta.ts_, txn)) {
    return std::vector<UndoLog>{};
  }

  std::vector<UndoLog> undo_logs;
  auto cur_link = undo_link;

  while (cur_link.has_value()) {
    auto undo_log_opt = txn_mgr->GetUndoLogOptional(*cur_link);
    if (!undo_log_opt.has_value()) {
      return std::nullopt;
    }

    undo_logs.push_back(*undo_log_opt);
    if (IsVersionVisible(undo_log_opt->ts_, txn)) {
      return undo_logs;
    }

    cur_link = undo_log_opt->prev_version_;
    if (!cur_link->IsValid()) {
      cur_link = std::nullopt;
    }
  }

  return std::nullopt;
}
```

#### tuple 被未提交时事务修改的情况

在 `IsVersionVisible` 中，留有一个问题，假设当前这个 `tuple.ts_` 是未提交事务修改的，为什么必须是我这个事务，才算能看见？

下面举出一个例子，其他场景也是类似的。

```
T1, read (0, 100)
T2, update (0, 100) -> (0, 200)
T1, select * where col0==0
```

我们是 `T1`，在第三句中，调用 `GetTuple()`，确实会拿到：`base_tuple = (1, 100)`。此时会进行 `CollectUndoLogs()`，由于当前 base tuple 的 ts 是：

- 别人的 temp_ts，不是我的 temp_ts

所以 CollectUndoLogs 会认为：这个最新版本对我不可见，**需要继续沿版本链**找更早版本。所以代码中就会继续找 undo logs，而不是立刻返回。
