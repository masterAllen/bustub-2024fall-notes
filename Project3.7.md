# Project3.7 NestedLoopJoin to HashJoin

进入 Task3 之后，一开始就是事先 Hash Join，其处理的语句是：

```
EXPLAIN SELECT * FROM __mock_table_1 INNER JOIN __mock_table_3 ON colA = colE;
EXPLAIN SELECT * FROM __mock_table_1 LEFT OUTER JOIN __mock_table_3 ON colA = colE;
```

一看，这和 Task2 的没区别啊，那为什么这里是 HashJoin？其实这就是 Task3 第二个任务，需要实现 NestedLoopJoin 转成 HashJoin 的 Optimizer。这两个 Join 的区别就是方法不一样，NestedLoopJoin 我们用的双层循环，而 HashJoin 则是用的哈希表。

所以其实我们应该先完成第二个任务，即 NestedLoopJoin 转成 HashJoin，其实完成了 SeqScan 转 IndexScan 的那个任务后，这里并不是很难。

## 初步实现

照着之前任务的代码写就行，首先判断 PlanType 是不是想要的，然后去解析 Predict，即类似于 `(#0.1=#1.0)` 这种表达式：

```cpp
  if (plan->GetType() == PlanType::NestedLoopJoin) {
    // 遍历 plan 的 predicate
    auto nlj_plan = std::dynamic_pointer_cast<const NestedLoopJoinPlanNode>(plan);
    auto predicate = nlj_plan->Predicate();

    auto children = std::vector<AbstractExpressionRef>();
    // 参考 seqscan_as_indexscan.cpp 的写法，先转成 LogicExpr，如果转不了，说明就是单个表达式
    auto queue_children = std::queue<AbstractExpressionRef>();
    queue_children.push(predicate);
    while (!queue_children.empty()) {
      auto now_child = queue_children.front();
      queue_children.pop();
      auto now_child_logic_expr = std::dynamic_pointer_cast<LogicExpression>(now_child);
      if (now_child_logic_expr != nullptr) {
        // 这里需要判断是 OR 还是 AND，要求必须是 AND，否则就转不了
        if (now_child_logic_expr->logic_type_ != LogicType::And) {
          return plan;
        }
        for (auto &child : now_child_logic_expr->GetChildren()) {
          queue_children.push(child);
        }
      } else {
        children.push_back(now_child);
      }
    }

    // ...
  }
```

可以看到，基本可以说是一模一样。**唯一不同的是这里的转换要求必须全部都是 AND 才可以**。

下面就有所不同了，检查 HashJoinPlanNode 的构造函数，我们来看看需要那些：
```cpp
HashJoinPlanNode(schema, LeftPlanNode, RightPlanNode, vector<>LeftExpressions, vector<>RightExpressions, JoinType)
```

除了 `LeftExpressions` 和 `RightExpressions` 之外，其他都是一样的。那就处理这个就行了，这两个的意思就是左右表的哪几列，比如 `(#0.1=#1.5) and (#1.2=#0.3)`，最后得出的就是 `[#0.1, #0.3]` 和 `[#1.5, #1.4]`。

在 SeqScan 转 IndexScan 中也遇到，只不过那儿表达式是一个常数和一个列，即 `#0.1=2` 这种的，但也基本类似了。区别就是那里拆分之后是判断是常数还是列；这里我们拆分完之后要判断是左列还是右列。

代码我不详细写了，使用的是 `GetTupleIdx`，它就是从 `#0.1` 这种提取出 `#0`。那思路就很明显，拆分完左边使用这个函数，看看是 0 还是 1：如果是 0，那肯定有一句 `left_expressions.push_back(expr->GetChildAt(0))`，其他请自行完成。


## 完善 NetsedLoopJoin 转 HashJoin

但上面的实现有问题，还是犯了 SeqScan 转 IndexScan 一样的问题，即 optimizer 只会处理最上层的元素，必须要递归去处理所有元素才行。比如下面的语句：
```
select t1.colA from temp_1 t1 inner join temp_4 t4 on t1.colA = t4.colA and t1.colB = t4.colB and t1.colC = t4.colC and t1.colD = t4.colD;

Projection { exprs=["#0.0"] } | (t1.cola:INTEGER)
  NestedLoopJoin { type=Inner, predicate=((((#0.0=#1.0)and(#0.1=#1.1))and(#0.2=#1.2))and(#0.3=#1.3)) }
    SeqScan { table=temp_1 } | (t1.cola, t1.colb, t1.colc, t1.cold)
    SeqScan { table=temp_4 } | (t4.cola, t4.colb, t4.colc, t4.cold)
```

那处理方式也是一样而已：

```cpp
  if (plan->GetType() != PlanType::NestedLoopJoin) {
    // 递归去对子节点进行操作
    std::vector<bustub::AbstractPlanNodeRef> new_children;
    for (const auto &child : plan->GetChildren()) {
      new_children.emplace_back(OptimizeNLJAsHashJoin(child));
    }
    // 递归转化其所有的子节点
    auto now_plan = plan->CloneWithChildren(std::move(new_children));
    return now_plan;
  }

  // 剩下的操作一样 ...
```

## 继续完善

完成这个之后，我就心满意足的去实现 HashJoin 了，但后来实现 HashJoin 之后才发现这里还是不对。为了统一，我这里就干脆继续写后面的修正了。虽然本来想的是尽量全部按照时间线来记录的，但这个就破例吧。

上面的处理有一个问题。如果子节点也是 NestedLoopJoin，那么就不会对子节点进行转换了。但这个现象是存在的：

```
NestedLoopJoin { type=Inner, predicate=(#0.0=#1.0) } | (a.v1:INTEGER, b.v1:INTEGER, c.v1:INTEGER)
    NestedLoopJoin { type=Inner, predicate=(#0.0=#1.0) } | (a.v1:INTEGER, b.v1:INTEGER)
        SeqScan { table=t1 } | (a.v1:INTEGER)
        SeqScan { table=t1 } | (b.v1:INTEGER)
    SeqScan { table=t1 } | (c.v1:INTEGER)
```

那我们就强制肯定会处理子节点呗，即类似于 DFS 的过程：

```cpp
  // 强制处理所有子节点
  std::vector<bustub::AbstractPlanNodeRef> new_children;
  for (const auto &child : plan->GetChildren()) {
    new_children.emplace_back(OptimizeNLJAsHashJoin(child));
  }
  auto now_plan_node = plan->CloneWithChildren(std::move(new_children));
  auto now_plan = std::shared_ptr<const AbstractPlanNode>(std::move(now_plan_node));

  // 后续都是一样了，只不过这里是对 now_plan 进行处理，而不是原始的 plan
  if (now_plan->GetType() == PlanType::NestedLoopJoin) {
    // ....
  }

```

结果上面的修改，最后问题解决。

## 特别致谢
感谢本文的博主，让我无从下笔的时候能有参考：https://blog.csdn.net/qq_40878302/article/details/137741785