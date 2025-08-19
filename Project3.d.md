# Project3.d 一个小问题

在 [3.4](./Project3.4.md) 中，我们实现 IndexScan 中的 `Init` 有部分代码是这样的：

```cpp
  // 根据索引，寻找所有符合条件的 RID；需要先转成 BTree
  auto tree = dynamic_cast<BPlusTreeIndexForTwoIntegerColumn *>(now_index);
  auto iter = tree->GetBeginIterator();
  while (iter != tree->GetEndIterator()) {
    rids_.push_back((*iter).second);
    ++iter;
  }
  rid_iter_ = rids_.begin();
```

看起来很 OK，但就是因为这个，导致我本地过了，但是在测评网站中 P3.05 和 P3.06 始终过不了。没想到因为这个居然 DEBUG 了两三个小时，比实现许多子任务还要花的时间长！

本质还是 C++ 没学好，我很早就发现一个异常现象：

```cpp
  while (iter != tree->GetEndIterator()) {
    auto now_rid = (*iter).second;
    std::cout << now_rid.ToString() << std::endl;
    std::cout << (*iter).second.ToString() << std::endl;
    // ...
  }
```
上面的两行输出居然不一样！！现在想想，这都已经把答案告诉我了，那肯定是 `iter` 的 `operator*()` 有问题啊，来看代码：

```cpp
INDEX_TEMPLATE_ARGUMENTS
auto INDEXITERATOR_TYPE::operator*() -> std::pair<const KeyType &, const ValueType &> {
  auto page_guard = bpm_->ReadPage(page_id_);
  auto page = reinterpret_cast<const LeafPage *>(page_guard.GetData());
  return std::make_pair(page->KeyAt(index_), page->ValueAt(index_));
}

INDEX_TEMPLATE_ARGUMENTS
auto B_PLUS_TREE_LEAF_PAGE_TYPE::KeyAt(int index) const -> KeyType { return key_array_[index]; }

INDEX_TEMPLATE_ARGUMENTS
auto B_PLUS_TREE_LEAF_PAGE_TYPE::ValueAt(int index) const -> ValueType { return rid_array_[index]; }
```

这就是非常经典的**悬空引用**(dangling reference)，具体是什么，问 AI 吧，[AI回答备份](./references/005.md)：

--- 

这段代码存在一个**非常经典且隐蔽**的问题：**返回了悬空引用（dangling reference）**。

我们逐层分析它为什么错、错在哪、以及怎么修。

## 🚨 代码结构回顾

```cpp
auto INDEXITERATOR_TYPE::operator*() -> std::pair<const KeyType &, const ValueType &> {
  auto page_guard = bpm_->ReadPage(page_id_);
  auto page = reinterpret_cast<const LeafPage *>(page_guard.GetData());
  return std::make_pair(page->KeyAt(index_), page->ValueAt(index_));
}
```

## ❌ 问题来源

你返回的是：

```cpp
std::pair<const KeyType &, const ValueType &>
```

也就是说，你返回了位于 `LeafPage` 内部数组 `key_array_` 和 `rid_array_` 中的**引用**。

但注意，你是通过：

```cpp
auto page_guard = bpm_->ReadPage(page_id_);
```

来**获取页面内容的生命周期的**。这是通过 buffer pool manager 获得的某种 RAII 封装，当 `page_guard` 离开作用域时，它会：

* 自动解锁页面
* **释放页面引用计数**
* 可能将页面 evict 掉（从内存中删掉）

所以一旦 `operator*()` 返回后，`page_guard` 析构，`page` 也就**失效了**，此时你返回的 `KeyAt()` 和 `ValueAt()` 的引用将是指向无效内存的**悬空引用**。

## 🔥 为什么这很经典？

这是很多数据库系统（包括 Bustub）的实现中，**迭代器或 cursor 类设计中的典型陷阱**：

> **返回页面内部数据的引用，而没有保留页面的生命周期管理手段。**

这在调试时**可能不会立刻崩溃**，但一旦访问多了、buffer pool 替换了、或 multithreaded 下干扰，就会引起**野指针访问、莫名其妙的数据错乱、崩溃等问题**。

## ✅ 解决方案一：返回拷贝

最简单、安全的修复方式：

```cpp
auto INDEXITERATOR_TYPE::operator*() -> std::pair<KeyType, ValueType> {
  auto page_guard = bpm_->ReadPage(page_id_);
  auto page = reinterpret_cast<const LeafPage *>(page_guard.GetData());
  return std::make_pair(page->KeyAt(index_), page->ValueAt(index_));
}
```

这会将 `Key` 和 `Value` 拷贝出来，不再依赖于 page buffer 的生命周期。

代价：如果 `KeyType` 或 `ValueType` 很大，性能可能稍有影响。

---

## ✅ 解决方案二：保留 `page_guard` 的生命周期

如果你**必须返回引用**，那就需要确保返回对象（如 `BPlusTreeIterator`）**持有 `page_guard`**：

```cpp
class BPlusTreeIterator {
  ...
 private:
  BufferPoolManager *bpm_;
  ReadPageGuard page_guard_;
  ...
};
```

然后在 `operator*()` 中：

```cpp
auto INDEXITERATOR_TYPE::operator*() -> std::pair<const KeyType &, const ValueType &> {
  auto page = reinterpret_cast<const LeafPage *>(page_guard_.GetData());
  return {page->KeyAt(index_), page->ValueAt(index_)};
}
```

这样就可以确保 page 在迭代器析构前都存在。

## 📌 总结

| 问题          | 原因                  |
| ----------- | ------------------- |
| 返回了页面中数据的引用 | 但页面 guard 在函数退出时被销毁 |
| 引用指向已被释放的页面 | 悬空引用，行为未定义          |

| 解决方法                | 适用场景    |
| ------------------- | ------- |
| 返回值改为拷贝             | 简单、安全   |
| 迭代器内持有 `page_guard` | 若必须返回引用 |

如你在实现 Bustub 等系统，这是个**非常经典也常被考察的点**。如果你想深入，我们还可以讨论 const-correctness 或 move 语义优化等方面。

---

## 我的解决

但是上面 AI 其实没给出我的解决方式，其中它给的第二个方案有点不好，需要保持 `page_guard` 生命周期，很显然我们不想这样；第一个方案不错。

不过我是这样写的：
```cpp
  while (iter != tree->GetEndIterator()) {
    auto now_rid = (*iter).second;
    rids_.push_back(now_rid);
    ++iter;
  }
```

为什么可以？因为在 `auto now_rid = (*iter).second;` 这句时，会把内容拷贝给 `now_rid`，然后才会执行析构。即赋值语句结束之后才会右边的析构。所以 `now_rid` 就是有内容了，然后 `rids_` 也会赋值。

还有一个问题：那 vector push_back 了 `now_rid`，下一个循环他不就析构了嘛？不会有问题，这是 C++ 语法，push_back 也会执行一次拷贝。[AI 参考回答](./references/004.md)

## 最终

Project3 最终结果，顺利通过：

![pass](./images/Project3_04.png)


网上都说 Project3 很简单，但其实我花的时间最多，而且多不少，真的有许多需要注意的地方。

## 特别致谢
感谢本文的博主，让我无从下笔的时候能有参考：https://blog.csdn.net/qq_40878302/article/details/137741785