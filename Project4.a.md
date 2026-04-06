# Project4.a UpdatePrimaryKeyTest 的注意点

处理到这里，其实基本已经完成了。于是进行了测试，发现 UpdatePrimaryKeyTest 错了好几次，下面进行总结。


## 批量处理后统一更新 key

之前的实现逻辑中，UpdateExecutor 的实现是一行一行处理，每次处理完之后更新 Key，但是，如果出现：

```sql
UPDATE maintable SET col1 = col1 + 1;
```

假设表里原来有：

- `(1, 0)`
- `(2, 0)`
- `(3, 0)`
- `(4, 0)`

更新后应该变成：

- `(2, 0)`
- `(3, 0)`
- `(4, 0)`
- `(5, 0)`

如果你“处理一条就立刻插新 key”：

1. 先处理 `(1,0) -> (2,0)`
2. 你想往主键索引里插入 key `2`
3. 但此时旧的 `(2,0)` 还没处理，索引里本来就已经有 key `2`
4. B+Tree 主键索引看到重复 key，会认为冲突，`InsertEntry(2)` 失败
5. 于是你错误地把这次合法更新当成了冲突

但实际上这不是冲突，因为原来的 key `2` 那一行，马上也会被更新成 key `3`。所以正确做法是：

1. 先把这一批要更新的 tuple 的旧主键都从主键索引删掉
   - 删 `1`
   - 删 `2`
   - 删 `3`
   - 删 `4`
2. 再统一插入新主键
   - 插 `2`
   - 插 `3`
   - 插 `4`
   - 插 `5`

这样就不会被“这条语句自己尚未完成的旧 key”挡住。


## 批量处理后 IndexScan 发生错误

在经过上面的操作之后，结果还是发生错误。原因是查找 Key 的方法需要更新，举个最直接的例子。

初始表：

- `RID r1 -> (1, 0)`
- `RID r2 -> (2, 0)`
- `RID r3 -> (3, 0)`
- `RID r4 -> (4, 0)`

事务 `T1` 先开始，记住它的 `read_ts = 4`。  
这时它看到的是：

- `1, 2, 3, 4`

然后另一个事务 `T2` 执行：

```sql
UPDATE maintable SET col1 = col1 + 1;
```

提交后，当前最新版本变成：

- `RID r1 -> (2, 0)`
- `RID r2 -> (3, 0)`
- `RID r3 -> (4, 0)`
- `RID r4 -> (5, 0)`

注意，**对旧事务 `T1` 来说，它仍然应该看到旧世界**：

- `RID r1` 应该还原成 `(1, 0)`
- `RID r2` 应该还原成 `(2, 0)`
- `RID r3` 应该还原成 `(3, 0)`
- `RID r4` 应该还原成 `(4, 0)`

所以这时如果 `T1` 做：

```sql
SELECT * FROM maintable WHERE col1 = 2
```

正确答案应该是：

- 返回旧版本 `(2, 0)`，也就是从 `RID r2` 回退出来的结果

问题来了：

如果 `IndexScanExecutor` 还是直接做

```cpp
real_index->ScanKey(key=2, &rids_, txn);
```

那它查的是**当前索引**。

而当前索引里 key=2 指向的是：

- `RID r1`，因为最新版本里 `r1` 已经从 `1` 变成 `2`

于是后面你沿 `r1` 的 undo log 回退，旧事务 `T1` 看到的是：

- `r1` 的旧版本 `(1, 0)`

这显然不满足 `WHERE col1 = 2`，所以会被 filter 掉。

更糟的是，如果你的写路径还把 old key 从索引里删掉了，那当前索引里可能**根本没有任何 RID 能让你还原出旧的 `(2,0)`**，于是就会出现：

- `expect col1 = 2 to have 1 row, found 0 rows`

这就是隐藏测试打到的点。

所以为什么我说不能只靠 `ScanKey`？

因为 `ScanKey(key=2)` 查到的是：

- “**当前最新版本**里 key=2 的 RID”

但旧事务真正需要的是：

- “**某个能还原出历史版本 key=2** 的 RID”

这两个不一定是同一个 RID。

### 解决代码

原来的代码，里面用的 `ScanKey` 方法，此时就可能出现上面所说的，`ScanKey(2)` 找到的是现在的 `(2, 0)`：

```cpp
  if (!plan_->pred_keys_.empty()) {
    // point lookup: 用 ScanKey 去查 key_tuple 对应的 RID，key_tuple 就是主键构建的内容
    std::vector<Value> key_values;
    key_values.reserve(plan_->pred_keys_.size());
    for (const auto &expr : plan_->pred_keys_) {
      key_values.push_back(expr->Evaluate(nullptr, *table_schema_));
    }
    auto key_tuple = Tuple(key_values, &index_info->key_schema_);
    real_index->ScanKey(key_tuple, &rids_, exec_ctx_->GetTransaction());
  }
```

更新的代码，获取表中**所有 RID**，后续循环中每个 tuple 都会进行回退，然后再去判断是否满足 filter 条件：

```cpp
  if (!plan_->pred_keys_.empty()) {
    // 对 MVCC 来说，当前索引只反映“最新版本”的 key -> RID。
    // 如果事务在读旧快照，直接用 ScanKey 做 point lookup 可能拿不到历史上应该可见的那条 RID。
    // 这里保守地退化成“遍历整张表的所有 RID”，再配合后面的 MVCC 重建 + filter predicate 做过滤。
    for (auto iter = table_heap_->MakeEagerIterator(); !iter.IsEnd(); ++iter) {
      rids_.push_back(iter.GetRID());
    }
  }
```
