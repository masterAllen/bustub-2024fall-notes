# Project3.4 IndexScan 的解决之路二：实现 IndexScan 和完善 Optimizer

上一篇总算了解索引是干什么的，而且也写了 `SeqScan` 转向 `IndexScan` 的 Optimizer，现在执行 `p3.05` 就可以发现终于走进了 `IndexScanExecutor`，那么开始实现吧。

## 实现 IndexScan

官网上说其实有两种方式最终会执行 IndexScan：

1. Point Lookup: `SELECT FROM <table> WHERE v1 = 1`. 官网要求使用 ScanKey 方法
2. Ordered Scan: `SELECT FROM <table> ORDER BY v1.` 官网要求使用 IndexIterator 方法

这两个方法等会再说，我完成这个实验只用了第二种方法，也就是我没管是 Point Lookup 还是 Ordered Scan，下面开始实现。

思路其实很简单，找到索引，然后用索引去把所有的节点找到就行。找到索引不谈，这个实现了 InsertExecutor 以及 SeqScan 转 IndexScan 之后，肯定直到怎么做了。那么关键就是怎么把所有节点都找到，官网也给了说明，转成 `BPluseTreeIndexForTwoIntegerColumn` 即可，然后我们进入这个类去看它的方法，很容易找到对应的方法。

```cpp
  // ...

  // 根据索引，寻找所有符合条件的 RID；需要先转成 BTree
  auto tree = dynamic_cast<BPlusTreeIndexForTwoIntegerColumn *>(now_index);
  auto iter = tree->GetBeginIterator();
  while (iter != tree->GetEndIterator()) {
    rids_.push_back((*iter).second);
    ++iter;
  }
  rid_iter_ = rids_.begin();
```

然后 `Next` 就不详解了，都有 RIDs 了，遍历呗。

**说明：本文之后，实现本地可以通过，但在网站上会过不了，具体原因就是上面的代码有问题，会在 [3.d](./Project3.d.md) 进行说明，我感觉是一个很经典和微妙的知识点。**


## 填补 B+ 树的坑

测试本地程序，会不能通过。幸好我之前用了 `assert`，很快地就直到原因了，在 `b_plus_tree.cpp` 中获取索引，我当时以为不可能会对一个空树进行查找的，但实际上对一个空表进行查找确实会发生这种情况，修改对应的方法就好：

```cpp
// b_plus_tree.cpp
auto BPLUSTREE_TYPE::Begin() -> INDEXITERATOR_TYPE {
  auto root_page_id = GetRootPageId();
  // 本来以为不会对空树查找，但实际会的（对空表进行查找）
  // assert(root_page_id != INVALID_PAGE_ID);
  if (root_page_id == INVALID_PAGE_ID) {
    return INDEXITERATOR_TYPE(INVALID_PAGE_ID, 0, bpm_);
  }

  // ...
}

// End() 同理 ...

// index_iterator.cpp
INDEX_TEMPLATE_ARGUMENTS
auto INDEXITERATOR_TYPE::IsEnd() -> bool {
  // 需要加上这句
  if (page_id_ == INVALID_PAGE_ID) {
    return true;
  }
  // ...
}

```

## 完善 Optimizer 1

现在应该没问题了吧。但...还是过不了，原因是 `SeqScan --> IndexScan` 需要递归处理！

如语句 `update t1 set v3 = 645, v1 = 8, v2 = -20 where v1 = 2;`，它的执行流程如下是：

```
Update { table_oid=30, target_exprs=["8", "-20", "645"] } | (__bustub_internal.update_rows:INTEGER)
  Filter { predicate=(#0.0=2) } | (t1.v1:INTEGER, t1.v2:INTEGER, t1.v3:INTEGER)
    SeqScan { table=t1 } | (t1.v1:INTEGER, t1.v2:INTEGER, t1.v3:INTEGER)
```

我们在进行 `SeqScan --> IndexScan` 处理时的 `plan` 只能是最上层的，即这里的 `Update`，所以之前判断当前 `plan` 是否有问题就没法处理这个。

所以需要遍历所有节点，我一开始并不想要进行递归处理，而是希望找到所有节点，如果是 `SeqScan` 那么就放入一个 vector 里面，然后遍历处理。

但是这样有一个问题就是，需要存储对应的父节点和在父节点的位置，这样还要调用父节点的某些函数来修改对应的 Child 是我们改完后的。这种修改特定 Child 的好像没有方法，那就更麻烦了。

所以使用递归了，因为有一个好处是 `plan` 有一个 `CloneWithChildren` 方法，传入一个 vector，他使用这里面的元素作为 `plan` 的孩子，所以我们这样处理就可以：

```cpp
  if (plan->GetType() != PlanType::SeqScan) {
    // 如果不是 SeqScan 节点，那么就递归去对子节点进行操作，以此防止万一子节点可能是 SeqScan
    std::vector<bustub::AbstractPlanNodeRef> new_children;
    for (const auto &child : plan->GetChildren()) {
      new_children.emplace_back(OptimizeSeqScanAsIndexScan(child));
    }
    // 递归转化其所有的子节点
    auto now_plan = plan->CloneWithChildren(std::move(new_children));
    return now_plan;
  }

  // ... 和之前一样的处理
```

## 完善 Optimizer 2

继续执行测试，发现还有一个问题没有考虑，果然真正要实现数据库这种东西，真的要考虑太多 case 了，一个学校实验就有这么多，可想而知工业级真的不容易。

出错的地方是因为我们解析 Filter 的时候是这样的：

```cpp
  auto children = std::vector<AbstractExpressionRef>();
  // 检查 Filter 是否是 LogicExpression；不可以说明就是一个表达式；可以说明是多个表达式
  auto logic_expr = std::dynamic_pointer_cast<LogicExpression>(filter);
  if (logic_expr == nullptr) {
    children.push_back(filter);
  } else {
    children = logic_expr->GetChildren();
  }
```

但 Filter 有可能是多层嵌套的，比如: `where (v1=10 or v1=20) or v1=30`，如果按照上面解析其实就只能获取 `(v1=10)or(v1=20)` 和 `v1=30`，所以需要查找到底：

```cpp
auto children = std::vector<AbstractExpressionRef>();
// 必须要一直查找到底才行，比如 ( (#0.1=10) or (#0.1=20) ) or (#0.1=30)
auto queue_children = std::queue<AbstractExpressionRef>();
queue_children.push(logic_expr);
while (!queue_children.empty()) {
    auto now_child = queue_children.front();
    queue_children.pop();
    auto now_child_logic_expr = std::dynamic_pointer_cast<LogicExpression>(now_child);
    if (now_child_logic_expr != nullptr) {
    for (auto &child : now_child_logic_expr->GetChildren()) {
        queue_children.push(child);
    }
    } else {
    children.push_back(now_child);
    }
}
```

## 一个小坑点

Init 可能会多次被调用，如：`select * from t2, (select count(*) as cnt from t1);`，在 `(select count(*) as cnt from t1)` 中，会调用多次 `IndexScanExecutor::Init`，t2 有多少行就调用多少次。

所以我们要在 Init 中先清空 `rids_`，否则其会累加，导致输出结果不正确。


## 结语

本文结束，p3.06 及其之前的文件可以顺利在本地执行。Task1 总结而言其实还是花了许多时间的，因为确实无从入手，发不出力的感觉。后面几个 Task，除了 Task4 也曾有过这种感觉，其他虽然难度比 Task1，花费时间其实很短，而且在等高铁时、在高铁地铁上也都能很顺的去完成。

感觉就像写作文一样，高中时我有好几次作文课上交白卷，这种无从下笔的感觉很不喜欢。

## 特别致谢
感谢本文的博主，让我无从下笔的时候能有参考：https://blog.csdn.net/qq_40878302/article/details/137741785