# Project4.8 Primary Key

## Primary Key 初步认知

下面开始 Task4，首先就是要知道 Primary Key 的含义，Primary Key 是用来唯一标识一个元组的，所以不能重复。

比如：

```sql
CREATE TABLE maintable(col1 int PRIMARY KEY, col2 int);
```

如果有记录：

```
(0, 100)
(1, 200)
```

不能再插入 `(0, 200)`，会报错，因为 `primary_key=0` 已经有记录了。

## 修改 InsertExecutor

那我们是否要调整一些函数，显然，Insert 需要检查是否插入成功。然而其实 P3 中我们已经做了这件事情：

```cpp
auto is_ok = real_index->InsertEntry(tuple_key, new_rid, exec_ctx_->GetTransaction());
```

但这里其实还有更改，因为我们引入了事务这个概念：**既然插入失败了，那这个事务就应该设置为 Tainted 状态**：

```cpp
auto is_ok = real_index->InsertEntry(tuple_key, new_rid, exec_ctx_->GetTransaction());
if (index->is_primary_key_ && !is_ok) {
    // 自己实现的函数，其实就是设置 tainted，然后抛出异常
    MarkTaintedAndThrow(txn); 
}
```

注意这里只考虑 Primary Key 的情况，其他 Key 不考虑。

但除了这个以外，AI 写的代码在插入之前就进行一遍检查：

```cpp
for (auto &index : indexes) {
    if (index->is_primary_key_) {
        // ...
        real_index->ScanKey(tuple_key, &existing_rids, txn);
        if (!existing_rids.empty()) {
            MarkTaintedAndThrow(txn);
        }
    }
}
```

对此的解释，一句话概括就是提高效率、尽早发现：

**提前发现已存在的主键，避免先插 heap 再失败，从而减少不必要的脏 tuple；第二段则负责处理并发竞态下的最终兜底。**

具体的回答：[为什么 InsertExecutor 要做两段检查](./references/006.md)


## 修改 IndexScanExecutor

然后再来修改 IndexScanExecutor，其实这个和 SeqScanExecutor 很像。就是因为引入了事务，现在 IndexScan 拿到 tuple 之后，也要回溯就行。这个可以直接参考 SeqScanExecutor 的实现。