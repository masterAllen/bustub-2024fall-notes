# Project4.3  UndoLogs 初步理解

## 理解 Task2

**User**

...

这段话说明了什么

---

**Cursor**

我先把这些提示对应到代码里的关键函数和测试上，确认接下来 Task 2 具体要补哪几块，再和你一起逐步实现。

我已经定位到 Task 2 的几个空函数了，再补一下头文件里的辅助注释，这样可以把“版本链怎么走、怎么回滚”讲成可直接落代码的步骤。

这段提示其实是在告诉你：**BusTub 的版本不是直接一份份完整存起来的，而是“最新版本 + 回滚链”**。

具体可以这样理解。

### 它说明了什么

一条 tuple 的信息分散在 3 个地方：

- `TableHeap`：放的是**当前最新版本**的 `tuple` 和 `TupleMeta`
- `TransactionManager`：放的是这条 tuple 对应的**最新 undo log 指针**，也就是版本链表头 `UndoLink`
- `Transaction`：放的是这个事务自己生成的所有 `UndoLog`

所以系统并不是“直接存每个历史版本的完整 tuple”，而是：

- 先把**最新 tuple**放在 `TableHeap`
- 再用一串 `UndoLog` 记录“它之前是怎么一步步变成现在这样的”

也就是说，历史版本需要**反推**出来。

举例而言，比如：

```
T1, start, tuple = (0, 100)
T2, start, tuple = (0, 100)
T2, update (0, 100) -> (0, 200)
T1, select * where col0==0
```

此时最后一句，T1 得到的是 (0, 200)，因为 `TableHeap` 始终是最新版本。但是 T1 不应该看到 (0, 200)，因为它启动的时候，那个时候数据库中是 (0, 100)，所以他就会找 `UndoLog`。

第三句，当 T2 修改了 tuple 的时候，就会留下这一个 UndoLog，所以 T1 就可以用这个 UndoLog 把 (0, 200) 恢复成 (0, 100)。

### 实际运行的逻辑

如果当前事务的 `read_ts = T`，而表里拿到的始终是最新版本，所以要往回推：

1. 找出所有 **时间戳晚于 `T`** 的修改（`CollectUndoLog`）
2. 从最新版本开始，按顺序把这些修改“撤销”掉（`ReconstructTuple`）

这样就能恢复出 `T` 时刻这个事务应该看到的版本。

## 实现 ReconstructTuple

### 这函数要做的事

输入是：

- `base_tuple` / `base_meta`：`TableHeap` 里现在的条目，即最新版本
- `undo_logs`：要往回撤销的一串修改

输出是：

- 目标历史时刻的 tuple
- 如果回滚后该 tuple 处于删除态，返回 `std::nullopt`

### 自然语言解释

每条 `UndoLog` 表示的是：

**“把当前版本撤回一层之后，旧版本长什么样”**

> 注意这里是旧版本应该是什么样，比如 (0, 100, 2) 变为 (1, 100, 2)，那么它是 (col=0, value=0)

它的结构如下：

- `is_deleted`：是否应该删除
- `modified_fields`：哪些列被修改了，如上面的场景是 `(True, False, False)`，标识 0th column 修改
- `tuple_`：被修改的值组成的 tuple，注意只是**被修改的值**，比如上面的场景就是 `Tuple(value=[0])`，而不是 `value=[0, 100, 2]`

所以应用一条 undo log 时：

- 先看 `log.is_deleted_`，判断旧版本是 deleted
- 再用 `log.modified_fields_` 指定的列，把这些列恢复成 `log.tuple_` 里的值
- 防御性编程：如果旧版本是 deleted，结果这次 undo log 还进行了一些操作，出错

### 代码骨架

```cpp
auto ReconstructTuple(const Schema *schema, const Tuple &base_tuple, const TupleMeta &base_meta,
                      const std::vector<UndoLog> &undo_logs) -> std::optional<Tuple> {
  // 最终要返回的结果
  std::vector<Value> values;
  auto column_count = schema->GetColumnCount();
  values.reserve(column_count);  // [i] 标识 tuple 第 i 列的值

  // 始终维护的变量：是否被删除
  bool is_deleted = base_meta.is_deleted_;

  // 先创立最新的 tuple 值，后面会根据 undo_logs 进行修改
  if (!is_deleted) {
    for (uint32_t i = 0; i < column_count; i++) {
      values.push_back(base_tuple.GetValue(schema, i));
    }
  } else {
    for (uint32_t i = 0; i < column_count; i++) {
      values.push_back(ValueFactory::GetNullValueByType(schema->GetColumn(i).GetType()));
    }
  }

  // 开始遍历 undo_logs，进行调整
  for (const auto &undo_log : undo_logs) {
    is_deleted = undo_log.is_deleted_;

    auto undo_schema = GetUndoLogSchema(schema, undo_log.modified_fields_);

    // undo_log 中 tuple_ 标识的是：**被修改列** 组成的值！，所以不能直接按照下面的方式去获取
    // for (uint32_t i = 0; i < column_count; i++) {
    //   values[i] = undo_log.tuple_.GetValue(&undo_schema, i);
    // }

    // 相反，我们利用 undo_log 中 modified_fields_ 标识哪些列被修改，提取出了对应的列，即 undo_schema
    uint32_t undo_column_idx = 0;
    for (uint32_t i = 0; i < column_count; i++) {
      if (undo_log.modified_fields_[i]) {
        values[i] = undo_log.tuple_.GetValue(&undo_schema, undo_column_idx);
        undo_column_idx += 1;
      }
    }
  }

  if (is_deleted) {
    return std::nullopt;
  }

  return Tuple(values, schema);
}
```

### GetUndoSchema

上面有 `GetUndoSchema` 自行实现，需要这个就是因为 `log.tuple_` 不是全部的值，而只是被修改的值组成的。做法很简单：

```cpp
auto GetUndoLogSchema(const Schema *base_schema, const std::vector<bool> &modified_fields) -> Schema {
  std::vector<Column> columns;
  columns.reserve(base_schema->GetColumnCount());
  for (uint32_t i = 0; i < base_schema->GetColumnCount(); i++) {
    if (modified_fields[i]) {
      columns.push_back(base_schema->GetColumn(i));
    }
  }
  return Schema(columns);
}
```

当时的疑惑：为什么不直接提取出 `tuple_.data`，然后按照列去切分不就好了：

```cpp
auto data = undo_log.tuple_.GetData();
// 然后切分成 data[0], data[1]...
```

但是看了一下才明白，tuple 的 `data` 是很序列化的数据，必须要根据各种类型然后进行拆分：

```cpp
auto Tuple::GetValue(const Schema *schema, const uint32_t column_idx) const -> Value {
  assert(schema);
  const TypeId column_type = schema->GetColumn(column_idx).GetType();
  const char *data_ptr = GetDataPtr(schema, column_idx);
  // the third parameter "is_inlined" is unused
  return Value::DeserializeFrom(data_ptr, column_type);
}

auto Tuple::GetDataPtr(const Schema *schema, const uint32_t column_idx) const -> const char * {
  assert(schema);
  const auto &col = schema->GetColumn(column_idx);
  bool is_inlined = col.IsInlined();
  // For inline type, data is stored where it is.
  if (is_inlined) {
    return (data_.data() + col.GetOffset());
  }
  // We read the relative offset from the tuple data.
  int32_t offset = *reinterpret_cast<const int32_t *>(data_.data() + col.GetOffset());
  // And return the beginning address of the real data for the VARCHAR type.
  return (data_.data() + offset);
}
```

如果真要拆分也能拆，无非就是上面的代码复用呗，但是何必呢。直接新建一个 schema 就好了，到时候就直接用，以后想要哪列就能得到哪列：

```cpp
undo_log.tuple_.GetValue(&undo_schema, i);
```
