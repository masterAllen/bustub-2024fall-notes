# Project3.9 Limit 实现

开始实现 Task4，有两个任务：External Merge Sort 和 Limit，这两个难度完全一个天一个地。前者我觉得是这个实验中最难的部分，后者两分钟写完。

这里就直接简单记录一下 LimitExecutor，它就是负责这样的语句：`select * from t1 limit 10`，太简单了：

```cpp
auto LimitExecutor::Next(Tuple *tuple, RID *rid) -> bool {
  if (idx_ >= plan_->GetLimit()) {
    return false;
  }
  if (!child_executor_->Next(tuple, rid)) {
    return false;
  }
  idx_++;
  return true;
}
```

**最好先实现 Limit**，因为本地测试语句里面第一句就带 limit，我当时实现 external merge sort 之后，测试总是出问题，搞了半天才发现要把 limit 实现了...

## 特别致谢
感谢本文的博主，让我无从下笔的时候能有参考：https://blog.csdn.net/qq_40878302/article/details/137741785