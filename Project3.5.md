# Project3.5 Aggreation 解决

开始实现 Task2，首先就是 `AggreationExecutor`，现在来看实现一点也不难，但就是理解起来需要花好多时间。理解之后实现真的很顺畅很爽。

## AggreationExecutor 是什么

形式就是下面的形式：
```
EXPLAIN SELECT colA, MIN(colB) FROM __mock_table_1 GROUP BY colA;
EXPLAIN SELECT COUNT(colA), min(colB) FROM __mock_table_1;
```

主要是两个新的东西：第一个是 `group by`，第二个是 `count(cola)`，只要有一个都会执行 AggreationExecutor。


## 不考虑 group by 时的实现

### 理解 AggreationExecutor 结构

还是延续我们逐步击破的思路，首先想想没有 group by 时候，即 `select count(colA) ..` 是怎么实现的？？

首先还是先看 `plan` 的构造，我们打印，假设语句时 `select min(v1+v2), count(*), count(v1) ..`，输出是：
```
Agg {
    types=["min", "count_start", "count"], 
    aggreates=["(#0.0+#0.1)", "1", "1"],
    groupby=[]
}
```

非常好理解！而且我们追踪，发现 `plan` 有两个方法：`GetAggregates` 和 `GetAggregateTypes`，即用来获取 `types` 和 `aggreates`，观察他们的返回值，知道了 `types` 有如下的类型：
```cpp
enum class AggregationType { CountStarAggregate, CountAggregate, SumAggregate, MinAggregate, MaxAggregate };
```

### 搭建初步框架

那么我们可以很自然想到，如果要计算结果，我们只需要从底层（如SeqScan）不断获取数据，然后遍历 `types` 和 `aggreates` 即可：

```cpp
while (child_executor_->Next(&tuple, &rid)) {
    for (auto& now_type: plan_->GetAggregateTypes()) {
      switch (now_type) {
        case AggregationType::CountStarAggregate:
            // ...
        case AggregationType::CountAggregate:
            // ...
        case AggregationType::SumAggregate:
            // ...
        // ...
      }
    }
}
```

### 理解 SimpleAggreationHashTable

然后官网给了说明，我们需要去看 `SimpleAggreationHashTable` 这个类，去看它时，发现它有一个方法 `CombineAggreagateValues` 基本就是这个形式，那就明白了，我们要用这个类去辅助实现，即上面的代码变成：
```cpp
while (child_executor_->Next(&tuple, &rid)) {
    hash_table_->CombineAggreateValues(/*?*/);
}
```

很显然，这个类要知道 `types` 和 `aggregates`，而这个类构造函数代码都已经给我们弄好了，我们都不用改，直接在 `AggreationExecutor` 中构造函数调用就行：

```cpp
AggregationExecutor::AggregationExecutor(/*....*/) {
  // ...
  auto &agg_exprs = plan_->GetAggregates();
  auto &agg_types = plan_->GetAggregateTypes();
  hash_table_ = std::make_unique<SimpleAggregationHashTable>(agg_exprs, agg_types);
}
```

那么就去实现这个 `CombineAggreateValues` 呗，这个方法有两个参数 `(AggregateValue *result, const AggregateValue &input)`，而 `AggreateValue` 就是:

```cpp
struct AggregateValue {
  /** The aggregate values */
  std::vector<Value> aggregates_;
};
```

所以其实 `result` 就是当前的结果，然后 `input` 就是传入的当前行，我们处理即可：
```cpp
  void CombineAggregateValues(AggregateValue *result, const AggregateValue &input) {
    for (uint32_t i = 0; i < agg_exprs_.size(); i++) {
      auto &now_result = result->aggregates_[i];
      auto &now_input = input.aggregates_[i];

      // 这里其实有点注意点，可以看文章最后的易错点一
      switch (agg_types_[i]) {
        // count(*) 直接加 1
        case AggregationType::CountStarAggregate:
          now_result = now_result.Add(ValueFactory::GetIntegerValue(1));
          break;
        // count(x) 需要判断是否为 NULL
        case AggregationType::CountAggregate:
          if (!now_input.IsNull()) {
            now_result = now_result.Add(ValueFactory::GetIntegerValue(1));
          }
          break;
        // sum(x)
        // min(x)...
        // max(x)...
      }
    }
  }
```

### 使用 SimpleAggreationHashTable

那么 `AggreateExecutor` 如何调用这个 `CombineAggregateValue` 呢？也就是来了 `tuple` 之后，我该怎么处理？？
```cpp
while (child_executor_->Next(&tuple, &rid)) {
    // tuple 和 rid 怎么转成参数，以便 CombineAggreateValues 使用？
    // result 是处理之前行的结果，input 是当前行，都是 AggreateValue 类型
    hash_table_->CombineAggreateValues(result, input);
}
```

我在这里就卡住了，我看到可以使用 `MakeAggreateValue` 方法可以用：
```cpp
  auto MakeAggregateValue(const Tuple *tuple) -> AggregateValue {
    std::vector<Value> vals;
    for (const auto &expr : plan_->GetAggregates()) {
      vals.emplace_back(expr->Evaluate(tuple, child_executor_->GetOutputSchema()));
    }
    return {vals};
  }
```

那么 `CombineAggreateValues` 中的 `input` 参数应该就调用这了。但是 `result` 参数呢？我们一开始应该要初始化构建一个？看到了 `GenerateInitalAggreateValue` 函数：
```cpp
  auto GenerateInitialAggregateValue() -> AggregateValue {
    std::vector<Value> values{};
    for (const auto &agg_type : agg_types_) {
      switch (agg_type) {
        case AggregationType::CountStarAggregate:
          // Count start starts at zero.
          values.emplace_back(ValueFactory::GetIntegerValue(0));
          break;
        case AggregationType::CountAggregate:
        case AggregationType::SumAggregate:
        case AggregationType::MinAggregate:
        case AggregationType::MaxAggregate:
          // Others starts at null.
          values.emplace_back(ValueFactory::GetNullValueByType(TypeId::INTEGER));
          break;
      }
    }
    return {values};
  }
```

于是就想到应该先调用这个函数，初始化之后，然后每次 `tuple` 来了，就调用 `CombineAggreateValues` 就行。

## 考虑 group by

然后开始考虑 group by 了

### 朴素实现

如果让我们实现应该怎么做？使用 HashTable！来了一个 `Tuple`，如果没有 `group by` 我们是怎么做的？处理它把结果累加到之前处理的结果上去，比如 `count(*)` 就加一。

如果有 `group by` 呢？就多一步查找步骤呗！我们挑出 `groupby` 的列，然后看看 HashTable 是否有这个记录：如果有，那么拿出对应的结果，把这一行累加上去；如果没有，就创建一个空结果呗，同样把这一行累加上去。

举例：`select count(*) from t1 group by t1.v2`，来了 `(Allen, Friday)`，那么就看看 HashTable 有没有记录过 `Friday`：
1. 如果有就把之前的结果拿出来，即 `HashTable[Friday]`，由于这里是 `count(*)`，所以直接加一；
2. 如果没有记录过，那么从空上面累加当前行处理结果，即结果是 1，那么就 `HashTable[Friday]=[1]` 即可。

### 使用 SimpleAggreationHashTable

实际上，这就是 `SimpleAggreationHashTable` 的作用！它有 `InsertCombine` 方法：
```cpp
  void InsertCombine(const AggregateKey &agg_key, const AggregateValue &agg_val) {
    // ht_ 格式：std::unordered_map<AggregateKey, AggregateValue> ht_{};
    if (ht_.count(agg_key) == 0) {
      ht_.insert({agg_key, GenerateInitialAggregateValue()});
    }
    CombineAggregateValues(&ht_[agg_key], agg_val);
  }
```

而 `AggreateKey` 又是什么？进入一看，发现它是一个 group by 组合：
```cpp
/** AggregateKey represents a key in an aggregation operation */
struct AggregateKey {
  /** The group-by values */
  std::vector<Value> group_bys_;
  // ...
}
```

AggreationExecutor 又有和 `MakeAggreateValue` 类似的 `MakeAggreateKey` 方法：
```cpp
  auto MakeAggregateKey(const Tuple *tuple) -> AggregateKey {
    std::vector<Value> keys;
    for (const auto &expr : plan_->GetGroupBys()) {
      keys.emplace_back(expr->Evaluate(tuple, child_executor_->GetOutputSchema()));
    }
    return {keys};
  }
```

那么就很显然了，我们把之前的 `CombineAggreateValues` 改成 `InsertCombine` 就行了，so easy.

```cpp
while (child_executor_->Next(&tuple, &rid)) {
    // 之前的处理：
    // result 在一开始使用 GenerateInitialAggreateValue 初始化过
    // auto input = MakeAggreateValues(&tuple);
    // hash_table_->CombineAggreateValues(result, input);

    // 现在的处理
    auto agg_key = MakeAggregateKey(&tuple);
    auto agg_val = MakeAggregateValue(&tuple);
    hash_table_->InsertCombine(agg_key, agg_val);
}
```

## 分清楚 Init 和 Next

现在有一个问题：上面的 while 循环应该放在 Init 和 Next 中？？

答案：应该在 Init 中！Init 中我们要把所有结果得到，Next 的作用？Next 需要遍历 group_bys，逐步返回。比如下面的执行语句：

```
select office_hour, count(*) from __mock_table_tas_2025_spring group by office_hour;
----
Tuesday 2
Friday 2
Monday 2
Wednesday 1
Thursday 1
```

### 实现 Init
Init 在上面已经介绍如何写了，即不断调用 Child 的 Next 方法，然后使用 hashtable 进行计算和插入。


### 实现 Next

下面实现 Next：
```cpp
  while (true) {
    if (*aht_iterator_ == aht_->End()) {
      return false;
    }

    auto agg_key = aht_iterator_->Key();
    auto agg_val = aht_iterator_->Val();

    auto values = std::vector<Value>();

    // 如 select office_hour, count(*) from __mock_table_tas_2025_spring group by office_hour;
    // 这里的 agg_key 和 agg_val 都是从 HashTable 中获取的，是已经处理后的值，可以放心使用
    // 返回结果需要先添加上 groupby 的内容
    for (auto &group_value : agg_key.group_bys_) {
      values.emplace_back(group_value);
    }
    for (auto &agg_value : agg_val.aggregates_) {
      values.emplace_back(agg_value);
    }

    *tuple = Tuple(values, &plan_->OutputSchema());
    *rid = RID();

    ++(*aht_iterator_);
    return true;
  }
```

上面返回结果需要先添加 group by 的内容，这是要求：
> The output schema consists of the group-by columns followed by the aggregation columns.

但是我看下面语句明明最后还是输出了三个呀？如果我们把 groupby 加上应该是四个。
```
select sum(v1), min(v2), count(*) from __mock_agg_input_big group by v5 + v4;
----
4500 9000 1000 
4500 8000 1000 
```

原因是因为上层又进行了一次处理，这个就和 AggreationExecutor 没有关系了，反正它只要每次返回都要无脑加上 group by 条目就行。


## 一些易错细节

### CountAggreate 和 CountStarAggreate

有一个坑的地方，CountAggreate 和 CounStarAggreate 默认值不一样，分别如下：

```
CountStarAggregate: ValueFactory::GetIntegerValue(0)
CountAggregate: ValueFactory::GetNullValueByType(TypeId::INTEGER);
```

CounStarAggreate 可以直接调用 `Add` 方法加一；而 CounAggreate 就要先转换才行：

```cpp
// count(*) 直接加 1
case AggregationType::CountStarAggregate:
    now_result = now_result.Add(ValueFactory::GetIntegerValue(1));
    break;
// count(x) 需要判断是否为 NULL
case AggregationType::CountAggregate:
    if (!now_input.IsNull()) {
    // ! 特别坑爹的一点，从上面 GenerateInitialAggregateValue 函数可以看到，除了 count(*) 其他默认都要是 NULL 而非 0
    // ! 而如果是 integer_NULL，它是不支持 Add 的，默认会保持不变，所以一开始用 Add 发现最后都是 integer_NULL，找了半天原因
    // ! 关键就是它要求必须初始为 integer_NULL，还不能改 GenerateInitalAggreateValue
    if (now_result.IsNull()) {
        now_result = ValueFactory::GetIntegerValue(0);
    }
    now_result = now_result.Add(ValueFactory::GetIntegerValue(1));
    }
    break;
```

### 空表处理

如果是空表，而且已经没有 group by 时候，也需要返回结果，即返回默认的 NULL：

```
select min(v1) from t1;
----
integer_null
```

所以对 `Init` 进行修改，如果满足条件，插入一个默认值进入：

```cpp
  // 如果 Begin == End，说明是空表，但是我们也要返回对应的值
  // 比如 select office_hour, count(*) from t1; 如果 t1 是空表，那么返回的值应该是 (NULL, 0)
  // 但如果 select office_hour, count(*) from t1 group by office_hour; 如果 t1 是空表，那么应该为空！！
  if (hash_table_->Begin() == hash_table_->End() && plan_->GetGroupBys().empty()) {
    hash_table_->InsertDefault();  // 新建一个方法，本质上就是 hashtable 中插入一个默认的记录
  }

  // SimpleAggreationHashTable 新建一个方法
  void InsertDefault() { ht_.insert({{}, GenerateInitialAggregateValue()}); }
```

### HashTable 初始化

Hashtable 初始化应该放在 Init 中，有些语句可能会调用 Init 多次，比如：`select * from t2, (select count(*) as cnt from t1);`，此时会调用多次 Init，t2 有几行就会调用几次 `AggreationExecutor::Init`，而构造函数只会在开始时调用一次。所以每次 Init 都要重新构造 `hashtable`，防止累加，比如上面的语句，如果 Init 不重新构造，会输出: n, 2n, 3n...

```cpp
void AggregationExecutor::Init() {
  child_executor_->Init();

  // 应该放在这里，因为 Init 会调用多次，构造函数只调用一次的情况
  // 如：select * from t2, (select count(*) as cnt from t1);
  // 此时会调用多次 Init，t2 有几行就会调用几次 AggreationExecutor::Init，而构造函数只会在开始时调用一次
  // 所以每次 Init 都要重新构造 aht_，防止累加，比如上面的语句，如果 Init 不重新构造，会输出: n, 2n, 3n...
  auto &agg_exprs = plan_->GetAggregates();
  auto &agg_types = plan_->GetAggregateTypes();
  aht_ = std::make_unique<SimpleAggregationHashTable>(agg_exprs, agg_types);

  // ...
}
```

## 特别致谢
感谢本文的博主，让我无从下笔的时候能有参考：https://blog.csdn.net/qq_40878302/article/details/137741785