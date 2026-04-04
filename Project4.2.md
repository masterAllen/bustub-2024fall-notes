# Project4.2 

## 时间戳的理解

**User**

When a transaction begins, it is assigned a read timestamp that is equal to the commit timestamp of the most recently committed transaction. At a high level, you can think of this as marking down / recording the timestamp of the latest atomic write into the database. The read timestamp determines what data can be safely and correctly read by a transaction. Or in other words, the read timestamp determines the latest version of a tuple that the current transaction can see.

When a transaction commits, it will be assigned a monotonically-increasing commit timestamp. The commit timestamp determines the serialization order of the transactions. Since these commit timestamps are unique, we can also uniquely identify committed transactions by their commit timestamp.

请理解这句话

---

**Cursor**

...

### 读时间戳（read timestamp）

- **什么时候给？** 事务**一开始**就拿到一个读时间戳。
- **它等于什么？** 等于**当时数据库里“已提交事务”里最新的那个提交时间戳**——可以想象成：记下“到现在为止，数据库最后一次被原子提交写入”的那一时刻。
- **干什么用？** 它划定：**这个事务只能读到不晚于这个时间点的、已经稳定下来的数据版本**。  
  换句话说：**读时间戳决定了当前事务“最多能看到多新”的元组版本**；比它更新的未提交或不应见的版本，就不该被读到（否则可能读到不一致或未提交的结果）。

### 提交时间戳（commit timestamp）

- **什么时候给？** 事务**提交成功**时才分配。
- **性质：** **严格单调递增**，所以不同提交事务的提交时间戳**互不相同**。
- **干什么用？** 用来定义这些事务在**串行化顺序**里的先后：提交时间戳小的，可以理解为在“等价单线程执行顺序”里更早。

### 合在一起怎么想

- **读时间戳**：回答“我启动这一刻，世界定格在哪一版数据上？”  
- **提交时间戳**：回答“我这次提交在全局顺序里排在谁前谁后？”  

因为提交时间戳唯一，**一个已经提交的事务**也可以**用它的提交时间戳当作唯一 ID** 来指代。

若你接下来要结合 BusTub 里具体类名（例如 `read_ts_`、`commit_ts_`）对照实现，可以说一下文件或接口名，我可以按代码路径帮你对齐这段话。

---

## 实现时间戳更新

具体理解如下：

`Begin` 要做的事：
- 新事务启动时，把它的 `read_ts_` 设成当前系统里“最近一次已提交事务”的提交时间戳。
- 在代码里，这个值就是 `last_commit_ts_`。
- 含义是：这个事务从启动那一刻起，只能看到 `commit_ts <= read_ts_` 的已提交版本，也就是它的快照。

`Commit` 要做的事：
- 事务提交时，给它分配一个新的、严格递增的 `commit_ts_`。
- 这个时间戳必须比之前所有已提交事务都大。
- 然后再更新 `last_commit_ts_`，表示“系统最新提交版本”推进到了这个新值。

所以最直白地说：

- `Begin`：`txn->read_ts_ = last_commit_ts_`
- `Commit`：给 `txn->commit_ts_` 分配一个新的更大的时间戳，并把 `last_commit_ts_` 更新为它

另外，这段话还在提醒你：

- `Abort` 的 starter code 已经给好了
- **Task #1 不需要改 `Abort`**
- 只要把 `Begin` 和 `Commit` 的时间戳逻辑做对，就能覆盖这部分要求

`Begin` 具体代码如下：

```cpp
txn->read_ts_ = last_commit_ts_.load();
```

同理，`Commit` 里也要这样处理。比如一个常见写法是：

```cpp
auto commit_ts = last_commit_ts_.load() + 1;
txn->commit_ts_.store(commit_ts);
last_commit_ts_.store(commit_ts);
```

因为之前已经拿了 `commit_mutex_`，所以这里不会有多个事务同时分配提交时间戳。


## 实现 watermark

`watermark` 本质上就是一个，当前系统中**活着的**事务里面，最小的 `read_ts_`。

它的作用是告诉系统：

**还有没有事务可能需要看到某些旧版本。**

这表示：

- 还有一个很老的事务 `T1` 仍然活着
- 所以系统必须保留那些它可能读到的旧版本
- 不能因为有更新版本了，就把老版本立刻清掉

### 它究竟用来干什么

最核心用途是：

**垃圾回收（GC, garbage collection）旧版本。**


### 为什么取“最小 read_ts”

因为最危险的是**最老的那个还没结束的事务**。

例如：

- 一个新事务 `read_ts = 100`
- 一个老事务 `read_ts = 20`

虽然大部分事务都很新，但只要这个 `read_ts = 20` 的事务还活着，系统就必须保留能让它看到一致快照的那些旧版本。

所以必须看所有活跃事务里的最小值，而不是最大值或平均值。一句话总结：

**`watermark` 用来告诉系统：最老的还没结束的事务停留在哪个快照，因此哪些旧版本还必须保留、哪些才可以安全回收。**


### 实现

```cpp
/* transaction_manager.h */
  /** Stores all the read_ts of running txns so as to facilitate garbage collection. */
  Watermark running_txns_{0};

/* watermark.h */
  // watermark_: 当前维护的最小 read_ts_
  timestamp_t watermark_;
  // current_reads_: 某个 read_ts_ 出现了几次
  std::unordered_map<timestamp_t, int> current_reads_;
```


我先把 `watermark.cpp` 读一下，对照接口说明解释 `AddTxn` / `RemoveTxn` 在生命周期里的职责。

这里的 `AddTxn` 和 `RemoveTxn`，本质上是在维护一句话：**增加/删除某个事务后，它们最小的 `read_ts` 分别是什么？**

#### `AddTxn(read_ts)`

```cpp
current_reads_[read_ts] += 1;
```

#### `RemoveTxn(read_ts)`

这里是 `O(1)` 方法：

```cpp
  auto iter = current_reads_.find(read_ts);
  if (iter == current_reads_.end()) {
    throw Exception("read ts not found");
  }

  iter->second -= 1;
  if (iter->second == 0) {
    current_reads_.erase(iter);
  }

  if (current_reads_.empty()) {
    watermark_ = commit_ts_;
    return;
  }

  if (read_ts != watermark_ || current_reads_.count(read_ts) != 0) {
    return;
  }

  // O(1)
  while (watermark_ < commit_ts_ && current_reads_.count(watermark_) == 0) {
    watermark_ += 1;
  }
```
