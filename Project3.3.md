# Porject3.3 IndexScan 的解决之路一：解决相关 Optimizer

## IndexScan 是什么？

面对 IndexScan，当时真的一头雾水，完全不知道如何开码。现在回头看，其实是因为我 Insert、Update 这些偷懒，没有去考虑索引的事情，所以我完全不知道索引是什么。

所以一定要先去尝试理解索引是什么，如上一篇文章所述，我觉得 [AI 回答的很好](./references/001.md)。所以 IndexScan 和 SeqScan 区别就是：如 `SELECT * FROM t1 where v1=1`，如果 `v1` 是有索引的，那么就会用 IndexScan。

GPT 回答 IndexScan 和 SeqSCan 也很好，令人感叹：

| 特性     | `SeqScanExecutor`   | `IndexScanExecutor`               |
| ------ | ------------------- | --------------------------------- |
| 数据来源   | 全表扫描（TableHeap）     | 先通过索引找到 RID，再从 TableHeap 拿数据      |
| 依赖索引   | ❌ 不依赖索引             | ✅ 依赖已有索引（B+Tree等）                 |
| 查询效率   | 慢（O(n)）             | 快（O(log n)，仅扫部分）                  |
| 使用场景   | 表没有索引，或者查询无法使用索引    | 查询使用了被索引的列（如 WHERE id = ...）      |
| 谓词过滤   | **全靠 executor 自己做** | 可以部分靠索引裁剪，少量依赖谓词二次过滤              |
| 内部数据结构 | `TableIterator` 顺序读 | `BPlusTree::ScanKey(...)` 找到 RIDs |


## SeqScan 的解决更新

在实现 IndexScan 的时候，我发现了之前实现 SeqScan 的方式有点不规范，当时是 Init 提前遍历好，存入一个 RID 数组，在 Next 的时候进行遍历即可。

```cpp
/** Initialize the sequential scan */
void SeqScanExecutor::Init() {
  // 获取 TableHeap，难点在于理解这些结构；我的方法是根据那张结构图，逐渐地进各个类里面去找
  // ...

  // 需要声明： std::vector<RID> rids_; std::vector<RID>::iterator rid_iter_;
  // 遍历 TableHeap，并且保存 RIDs
  auto iter = table_heap_->MakeIterator();
  while (!iter.IsEnd()) {
    rids_.push_back(iter.GetRID());
    ++iter;
  }
  rid_iter_ = rids_.begin();
}
```

但代码阅读的更多之后，才明白过来：原来有专门的 Iterator 方法... 所以使用下面的语句其实更规范..
```cpp
void SeqScanExecutor::Init() {
  // 获取 TableHeap，难点在于理解这些结构；我的方法是根据那张结构图，逐渐地进各个类里面去找
  // ...

  table_iter_ = std::make_unique<TableIterator>(table_heap_->MakeIterator());
}
```

## 尝试实现 IndexScan 后理解了 Optimizer

好的，那我们开始实现 IndexScan，我们先在构造方法里打印出来（这个很有效果，推荐每次实现都先这样做）：

```cpp
IndexScanExecutor::IndexScanExecutor(ExecutorContext *exec_ctx, const IndexScanPlanNode *plan)
    : AbstractExecutor(exec_ctx) {
  plan_ = plan;
  std::cout << "IndexScanExecutor" << std::endl;
  std::cout << plan_->ToString() << std::endl;
}
```

编译，执行 `p03.05-index-scan-btree.slt`，嗯？？怎么前几句的 `select * from t1 order by v1` 就错了？而且没有打印 IndexScan，而是打印了 SeqScan？？

这就是比较坑的地方，我们需要继续阅读官网的说明：
> You will need to finish the optimizer rule in the next section to transform a SeqScan into an IndexScan. It may make more sense to implement the optimizer rule before implementing IndexScan to understand the kind of queries IndexScanExecutor will need to support.

我们需要先实现 `Optimize SeqScan To IndexScan`，而从这我们就能理解 Optimizer 这个优化器，它的一部分作用是什么了。还是看我们最开始那张图：

![](./images/Project3_01.svg)

Optimizer 其中一部分作用就是将效率低的方式转成效率高的方式，比如这里：一开始都是无脑 SeqScan，进入到 Optimzier 之后，他会看看如果是 SeqScan，如果用到的列有索引，那就尝试转成 IndexScan，这样之后执行会快一些。

## Optimize: SeqScan --> IndexScan 整体逻辑实现

所以我们打开 `optimzer` 里面的 `seqscan_as_indexscan.cpp`，去实现它，看到这个类，非常经典的茫然。

```cpp
auto Optimizer::OptimizeSeqScanAsIndexScan(const bustub::AbstractPlanNodeRef &plan) -> AbstractPlanNodeRef {
  // TODO(P3): implement seq scan with predicate -> index scan optimizer rule
  // The Filter Predicate Pushdown has been enabled for you in optimizer.cpp when forcing starter rule
  return plan;
}
```

### 初步理解

这是什么？？所以依旧先打印，这才理解了：这 `plan` 的形式不就是传入 `SeqScanExecutor` 这些 `executor` 的参数吗，所以就多看代码，看有哪些方法和成员变量，之后很容易能写出下面的代码了：

```cpp
auto Optimizer::OptimizeSeqScanAsIndexScan(const bustub::AbstractPlanNodeRef &plan) -> AbstractPlanNodeRef {
  std::cout << "OptimizeSeqScanAsIndexScan" << std::endl;
  std::cout << plan->ToString() << std::endl;

  // 只有 SELECT FROM <t1> WHERE <index column> = <val> 才可以
  // 注意 WHERE v1 = 1 or v1 = 4 可以；WHERE v1 = 1 AND v2 = 2 不可以
  // 检查是否是 SeqScan 节点
  if (plan->GetType() != PlanType::SeqScan) {
    std::cout << "plan->GetType() != PlanType::SeqScan" << std::endl;
    return plan;
  }

  // 下面就应该是要把 SeqScanNode 转成 IndexScanNode 了
  // ...
}
```

所以我们如何把 `SeqScanNode` 转成 `IndexScanNode`？？看代码发现他们都是 `AbstractPlanNode` 的继承，这里就要使用 `std::dynamic_pointer_cast<>` 这种语法了，具体请自行搜索，总之它的作用如下：

```cpp
// 如果 plan 不是只想 SeqScanPlanNode，那么结果就是 NULL
// 如果是，那么就转成功，之后就可以用 SeqScanPlanNode 的方法了
  auto seq_scan_plan = std::dynamic_pointer_cast<const SeqScanPlanNode>(plan);
```

转成 `IndexScan` 的代码如下，回头看确实简单，但当时写还是感觉很困难的，一定要多打印去理解，不要怕出错。在这过程，别人分享的代码也帮助了我很多，所以我这里也分享一下吧，虽然课程要求不要这样做，但想想自己从头写的话，确实会很难受。

### 获取 filter

首先是先获取 `filter`，即获取 `((#0.1=1) OR (#0.1=3))`：
```cpp
// 获取 Filter 表达式
auto seq_scan_plan = std::dynamic_pointer_cast<const SeqScanPlanNode>(plan);

auto filter = seq_scan_plan->filter_predicate_;
// 检查 Filter 是否为空
if (filter == nullptr) {
  return plan;
}

std::cout << "filter = " << filter->ToString() << std::endl;
```

### 拆分 filter

然后开始处理 `filter`，总结而言，有如下的注意点：
1. 只有一个比较，如 (#0.1=1)，它实际上是 ComparisonExpression
2. 有多个比较，如 ((#0.1=1) OR (#0.1=3))，它实际是 LogicExpression

所以这里有一个麻烦点，是我们需要判断是否是 ComparisonExpression 和 LogicExpression，然后有不同的处理方法。但其实不需要，我们可以统一用包装成一个 `vector`，即如果是 Comparison 那么就只有一个元素，若是 Logic 那么就是多个元素：

```cpp
  auto children = std::vector<AbstractExpressionRef>();
  // 检查 Filter 是否是 LogicExpression；不可以说明就是一个表达式；可以说明是多个表达式
  // LogicExpression 是类似 ((#0.0=1) OR (#0.0=3))，而只有一个 (#0.0=1) 就不是 LogicExpression
  auto logic_expr = std::dynamic_pointer_cast<LogicExpression>(filter);
  if (logic_expr == nullptr) {
    children.push_back(filter);
  } else {
    children = logic_expr->GetChildren();
  }
```

### 遍历得到的 vector

获取之后，开始遍历这个 `vector`，由于我们上面拆分了 filter，所以每次遍历都是处理如 `(#0.1=1)` 这种只有一个的表达式，即 ComparisonExpression，方法如下：

1. (#0.1=1) 中 #0.1 是 ColumnValueExpression，1 是 ConstantValueExpression
2. ColumnValueExpression 中 `GetColIdx()` 方法提取的是列的索引，如 #0.1 提取的是 1

此外，题目要求说 `v1=1 OR v1=2` 可以通过，但是 `v1=1 OR v2=1` 不可以，所以遍历的时候要判断一下是不是都是一个列，代码如下：
```cpp
  auto col_idx = std::optional<uint32_t>();
  auto constant_values = std::vector<AbstractExpressionRef>();

  // 开始遍历 Children，检查每个是否是 ComparisonExpression
  for (auto &child : children) {
    auto comparison_expr = std::dynamic_pointer_cast<ComparisonExpression>(child);
    if (comparison_expr == nullptr) {
      return plan;
    }
    // 必须要是等式
    if (comparison_expr->comp_type_ != ComparisonType::Equal) {
      return plan;
    }

    // 左边的表达式要求是 Column；右边是常数
    // 这里不看别人代码是真不会，原来还有这个常量表达式
    auto lhs = std::dynamic_pointer_cast<ColumnValueExpression>(comparison_expr->GetChildAt(0));
    auto rhs = std::dynamic_pointer_cast<ConstantValueExpression>(comparison_expr->GetChildAt(1));
    if (lhs == nullptr || rhs == nullptr) {
      lhs = std::dynamic_pointer_cast<ColumnValueExpression>(comparison_expr->GetChildAt(1));
      rhs = std::dynamic_pointer_cast<ConstantValueExpression>(comparison_expr->GetChildAt(0));
      if (lhs == nullptr || rhs == nullptr) {
        return plan;
      }
    }

    // 获取列的索引，要求各个表达式必须要都是一样的列
    auto now_col_idx = lhs->GetColIdx();
    if (!col_idx.has_value()) {
      col_idx = now_col_idx;
    } else if (col_idx.value() != now_col_idx) {
      return plan;
    }

    constant_values.push_back(rhs);
  }

  // 以 (#0.1=1) OR (#0.1=3) 为例
  // 进行到这里，constant_values 是各个常量，即 {1, 3}；col_idx 则是表中列的索引，即 #0.1 中的 1
  // ...
```

### 一个小问题

上面的代码似乎有问题，因为我并没有去判断 LogicExpression 是用的 OR 还是 AND？那后续我该怎么筛选呀？？这就涉及到我们最终要返回的 IndexScan 形式了：

```cpp
  IndexScanPlanNode(SchemaRef output, table_oid_t table_oid, index_oid_t index_oid,
                    AbstractExpressionRef filter_predicate = nullptr, std::vector<AbstractExpressionRef> pred_keys = {})
```

可以看到，其中有 `filter_predicate`，那也就是说，我最后把如 `((#0.1=1) OR (#0.1=3))` 这种 filter 直接作为参数进行构造就好了，是 OR 还是 AND 交给 `IndexScanPlanNode` 去处理，我们这里不去多管闲事。

### 构造返回值

如何构造？其实主要就一个：`index_oid`。即索引，文章最开始已经提到索引是什么了（问 AI），然后就可以看具体的数据结构了，这里虽然说看懂了就很简单，但实际花费的时间真不少，也幸好有好心人分享，帮我省了很多时间。

1. `plannode` 有 `table_name_` 属性，用来获取要处理表的名字，很显然这是个 string
2. `catalog` 中有 `GetTableIndexes(table_name)`，即根据表的名字获取和这个表所有关联的索引。很显然这是个 vector
3. 那么 vector 里面就是各个索引了？别急，vector 里面是一个 `IndexInfo`，这是用于在索引外面又包装了一层，可以看他的成员变量，有 `name`, `table_name_`, `index_oid_`；最关键的就是有一个 `index_`
4. 这个 `index_` 就是我们要找的索引了，他有 `GetKeyAttrs` 用于说明它是对在原表上的哪几列进行索引的。

所以，我们应该这样做：
1. 按照上面的 1-2 条目，找到这个表的所有 `IndexInfo`，即找到一个 vector
2. 遍历这个 vector，判断当前是否是我们想要的索引？方法：`IndexInfo` 中的 `index_` 是真正索引类，他有 `GetKeyAttrs` 返回索引处理的是表中哪几列，而我们在上面解析了表达式，直到表达式处理的是第几列，即 `col_idx`，二者进行对比。
3. 找到了对应的索引，那么我们就可以返回它的 `index_oid` 了，而索引的外包装类 `IndexInfo` 正好就记录了这个信息。


## 解决遗留问题：Insert 等操作需要更新索引

解决到这一步之后，有一个遗留问题。当时 Insert 等操作没有更新索引，当时是不太了解，现在懂了，所以加上去就行：

```cpp
    // ...

    // 这里是我们插入时得到新的 RID，我们要把这个 RID 和插入的 tuple 绑定起来
    auto new_rid_optial = table_heap->InsertTuple(tuple_meta, *tuple);
    auto new_rid = new_rid_optial.value();

    // ....

    // 这里的 index_infos 寻找过程类似于上文的说明
    for (auto &index : index_infos) {
        auto real_index = (index_infos->index_).get();

        // 本次插入的条目是 (*tuple)，但我们需要从中提取出对应列的值，然后才能使用 InsertEntry 插入到对应列对应的索引上
        // 这里 Schema 也是看别人的代码才知道的，然后自己去打印了解了结构，如果不是别人的代码，真的很难从零开始
        auto tuple_key = tuple->KeyFromTuple(table_schema, index_infos->key_schema_, real_index->GetKeyAttrs());

        // 可以配合打印去了解过程
        // std::cout << "real_index->GetName() = " << real_index->GetName() << std::endl;
        // std::cout << "table_schema = " << table_schema.ToString() << std::endl;

        auto is_ok = real_index->InsertEntry(tuple_key, new_rid, exec_ctx_->GetTransaction());
        std::cout << "is_ok = " << is_ok << std::endl;
    }
```

## 说明
现在的 Optimizer 还不是最终的解决方案。

## 特别致谢
感谢本文的博主，让我无从下笔的时候能有参考：https://blog.csdn.net/qq_40878302/article/details/137741785