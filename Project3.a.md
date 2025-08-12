# Project3.a External Merge Sort 解决之路一（邪修实现）

开始实现本实验我认为难度最大的部分：External Merge Sort，也就是外部排序。

## 了解原理
一开始我也是跟着官网的介绍想着实现方式，但回过头感觉还是应该先抛弃掉这些类，比如 `SortPage` 等等，先不管。我们就先解决一个问题：外部排序，它究竟是什么？ 


1. 一个[可视化网站](https://valeriodiste.github.io/ExternalMergeSortVisualizer/External%20Merge%20Sort%20Visualizer/index.html)，左下角设置参数。

2. 先想着二路排序是什么，[这个链接](https://zhuanlan.zhihu.com/p/592127871)还可以。可视化网站也是这样，就看 2-way 时候的可视化。

3. 然后就是多路排序，这里直接看可视化网站中的 k-way，参数设置 B=10, M=4, File records=30, Max=3

一个问题：后期的排序中，每次排序的元素都很大啊，加载不进内存，那怎么办？就拿二路排序来说，后面每次要对两个排序合并成一个，这两个本身就非常大了，加载不进内存。

回答：实际上这就是外部排序的作用：
1. 问题所说的对于两个元素排序合并成一个，这两个元素本身就是由很多页构成的。
2. 所以内存中是由两个输入区，一个输出区。每次这两个输入区分别是从各自元素读取一页，然后逐步比较。
3. 输出区满了，那么就赶紧把输出去写入到某一页中，再清空。某个输入区到底了，就把那个元素下一页替换进来，继续比较。

## 开始实现: SortPage 是什么？

开始实现，第一个遇到的问题就是，SortPage 是啥，按理来说我们应该只对这些页进行排序呀？那我问两个问题：哪些 Page？如何从 Page 中读取出 Tuple 来？

这就是 SortPage 的作用。我们从表中读取 Tuple，然后组成一个 SortPage，写入到磁盘中，并记录写在哪页中，此时我们知道了要对哪些页排序。而 SortPage 又是我们自己定义的结构，我们知道如何从中取出 Tuple。

所以逻辑应该是这样的：

```cpp
void ExternalMergeSortExecutor<K>::Init() {
  child_executor_->Init();

  size_t tuple_count = 0, tuple_size = 0;

  using SortPageInfo = std::pair<page_id_t, size_t>;

  // 第一步：先从 Child 中读取数据，读取到 Tuple 后，存入到 SortPage 中
  auto sort_page = std::make_unique<SortPage>();
  auto page_ids = std::vector<page_id_t>();
  while (true) {
    if (!child_executor_->Next(&tuple, &rid)) {
      break;
    }
    sort_page->PutTuple(tuple_count);

    // 如果满了，那么就排序，然后写入到磁盘，然后清空 SortPage，然后继续读取
    if (sort_page->IsFull()) {
      sort_page->Sort();

      // 写入到磁盘
      auto new_id = sort_page->WriteToDisk();
      page_ids.push_back(new_id);

      // 清空 SortPage 和相关变量
      sort_page = std::make_unique<SortPage>();
    }
  }
  // 如果 SortPage 中还有数据，那么就说明这是剩余的部分，同样排序后写入到磁盘
  if (!sort_page->IsEmpty()) {
      sort_page->Sort();

      // 写入到磁盘
      auto new_id = sort_page->WriteToDisk();
      page_ids.push_back(new_id);
  }
}
```

我们看到其中涉及了多个方法：`PutTuple` 放入 Tuple；`WriteToDisk` 写入磁盘；`Sort` 排序等等。


## 设计 SortPage 结构

下面我们开始设计 SortPage，一个原则是尽可能地紧凑。还是先来一个问题：上面说的 `PutTuple` 等等方法，是不是很占空间啊，那怎么知道它所占空间大小？

回答：一点也不占！构造函数或者成员函数都是不占空间的（排除虚函数）！因为它们是编译生成在代码段里面，不占每个对象的空间的。

所以放心了，那我们想想除了 Tuple，SortPage 还应该放入什么？放入 tuple_count，tuple_size 等等，但是我的实现有点极限，我是什么都没有存，就存 Tuple。

甚至由于我看到 Tuple 的构造，它也是就是一个 RID 加上 char 数据：
```cpp
class Tuple {
    // ...
 private:
  auto GetDataPtr(const Schema *schema, uint32_t column_idx) const -> const char *;

  RID rid_{};  // if pointing to the table heap, the rid is valid
  std::vector<char> data_;
};
```

而我们又不需要 RID，所以我直接存储 char 就好了啊，所以我的 SortPage 就是这样：

```cpp
class SortPage {
 public:
  // 成员函数本身在运行时 不会占用每个对象的空间，它的代码只在编译期生成一次，在代码段中共用，不会复制到每个对象里。
  SortPage() = default;

  SortPage(const char *data, size_t size) { memcpy(data_, data, size); }

 private:
  char data_[BUSTUB_PAGE_SIZE];
};
```

没错，就是朴实无华的 data 数组。我之所以敢这样设计，就是因为本课程实验中确保每次 Tuple 的大小都是一样的。不信的话可以检查：

```cpp
auto tuple_size = 0;
while (!child_executor_->Next(&tuple, &rid)) {
    if (tuple_size == 0) {
      tuple_size = tuple.GetLength();
    }
    assert(tuple_size == tuple.GetLength());
}
```

这个好处在设计 SortPage 函数方便很多。**所以我在标题写了邪修，因为我感觉肯定加一些 Header 字段更规范，我这里是讨巧了。**

## 设计 SortPage 函数

前面提到 SortPage 需要多个方法，首先是 `PutTuple` 放入 Tuple，而 SortPage 只是 char 数组，我们直接把这个 Tuple 里面的 `vector<char> data` 放进去即可：

```cpp
  // i 表示第几个，至于 tuple_size，其实也可以直接用 tuple.GetLength() 来获取
  void PutTuple(size_t i, const Tuple &tuple, size_t tuple_size) {
    auto tuple_data = tuple.GetData();
    memcpy(data_ + i * tuple_size, tuple_data, tuple_size);
  }
```

`PutTuple` 的反面 `GetTuple`，也是同样的道理。我们取出对应的 `char` 数组，需要自己构造一个 Tuple 返回，返回的 RID 直接用默认的即可：
```cpp
  auto GetTuple(size_t i, size_t tuple_size) -> Tuple {
    return {RID(), data_ + i * tuple_size, static_cast<uint32_t>(tuple_size)};
  }
```

`IsEmpty()` 和 `IsFull()` 方法的实现，就需要我们传入参数 `tuple_count` 了，因为 SortPage 本身不记录存入多少个 Tuple，不过后来我都没实现这个方法，因为我们都已经记录了 `tuple_count` 了，那直接比较某个数就行了...
```cpp
// 是否满了
tuple_count == BUSTUB_PAGE_SIZE / tuple_size;
// 是否为空
tuple_count == 0;
```

`WriteToDisk()` 需要传入 `bpm(bufferpoolmanager)` 和 `tuple_count`，我直接在 `ExternalMergeSortExecutor` 上定义的，其实定义在 `SortPage` 也可以的：
```cpp
template <size_t K>
auto ExternalMergeSortExecutor<K>::WriteToDisk(SortPage &sort_page) -> page_id_t {
  // 自行实现

  return page_id;
}
```

`ReadFromDisk()` 即从磁盘中读取数据，从哪读，读多少。同样的，我也是在 ExternalMergeSortExecutor 上定义的。注意下面的代码中**一定要释放掉 `ReadPageGuard`**，否则不会被删除！
```cpp
template <size_t K>
auto ExternalMergeSortExecutor<K>::ReadFromDisk(page_id_t page_id, size_t tuple_count, size_t tuple_size) -> SortPage {
  // 自行实现

  return now_sort_page;
}
```

最后就是 `Sort` 了，这个就比较麻烦一点，需要单开一个章节。

## 设计 Sort

很明显，sort 需要用到 TupleCompator，进入到它的语句：

```cpp
/** The SortKey defines a list of values that sort is based on */
using SortKey = std::vector<Value>;
/** The SortEntry defines a key-value pairs for sorting tuples and corresponding RIDs */
using SortEntry = std::pair<SortKey, Tuple>;

/** The Tuple Comparator provides a comparison function for SortEntry */
class TupleComparator {
 public:
  explicit TupleComparator(std::vector<OrderBy> order_bys);
  auto operator()(const SortEntry &entry_a, const SortEntry &entry_b) const -> bool;

 private:
  std::vector<OrderBy> order_bys_;
};
```

它是针对 SortEntry 的，所以我们要从 Tuple 转到 SortEntry，查阅发现有一个 `GenerateSortKey` 方法，这是需要我们实现的：

```cpp
// bound_order_by.h
enum class OrderByType : uint8_t {
  INVALID = 0, /**< Invalid order by type. */
  DEFAULT = 1, /**< Default order by type. */
  ASC = 2,     /**< Ascending order by type. */
  DESC = 3,    /**< Descending order by type. */
};

using OrderBy = std::pair<OrderByType, AbstractExpressionRef>;

// execution_common.cpp
auto GenerateSortKey(const Tuple &tuple, const std::vector<OrderBy> &order_bys, const Schema &schema) -> SortKey {
}
```

很简单，OrderBy 第二个是 `AbstractExpressionRef`，也就是表达式，在这里应该就是 `#0.1` 这种表示哪一列的形式，我们使用 `Evaluate` 获取对应的值。
```cpp
auto GenerateSortKey(const Tuple &tuple, const std::vector<OrderBy> &order_bys, const Schema &schema) -> SortKey {
  // 这个简单，就遍历 order_bys，每个 second 就是如 (#0.1) 这种表达式，调用 Evaluate 就行
  SortKey sort_key;
  for (const auto &order_by : order_bys) {
    auto expr = order_by.second;
    auto now_key = expr->Evaluate(&tuple, schema);
    sort_key.push_back(now_key);
  }
  return sort_key;
}
```

那我们就从 Tuple 转到 SortEntry，专门定义一个 `TupleToSortEntry` 方法：
```cpp
auto ExternalMergeSortExecutor<K>::TupleToSortEntry(Tuple &tuple) -> SortEntry {
  auto key = GenerateSortKey(tuple, plan_->GetOrderBy(), plan_->OutputSchema());
  return {key, tuple};
}
```

那我们实际使用的 TupleCompator 就可以这样用：
```cpp
void ExternalMergeSortExecutor<K>::Sort(SortPage &sort_page, size_t tuple_size, size_t tuple_count) {
  // sort_page 的各个 Tuple --> sort_entries，代码略...
  std::sort(sort_entries.begin(), sort_entries.end(), cmp_);
  // sort_entries --> 依次放入 sort_page 中，代码略...
}
```

但还有一个地方要实现，即 `cmp_` 的 `operator()`，即 `TupleComparator` 的 `operator()`，这是一个小于比较器：
```cpp
/** TODO(P3): Implement the comparison method */
auto TupleComparator::operator()(const SortEntry &entry_a, const SortEntry &entry_b) const -> bool {
  assert(entry_a.first.size() == entry_b.first.size());
  assert(entry_a.first.size() == order_bys_.size());
  for (size_t i = 0; i < order_bys_.size(); i++) {
    auto order_by = order_bys_[i];
    if (order_by.first == OrderByType::INVALID) {
      continue;
    }
    auto key_a = entry_a.first[i];
    auto key_b = entry_b.first[i];
    // ...
  }
  return false;
}

```

## 第一步的整体框架

设计完这些函数之后，我们重新整理第一个代码块的框架。即不断读取 Tuple 塞入 SortPage，满了的话就 Sort 然后写入磁盘。

这里就不贴出代码了，总之逻辑就是文章第一个代码的逻辑，无非就是要记录 `tuple_count` 和 `tuple_size`，然后有些函数进行微调。

有一个地方需要修改：我们必须要记录 `SortPage` 实际存储了多少个 Tuples，所以我们之前的 vector 有修改：
```cpp
// 原来：std::vector<page_id_t> page_ids;
// 现在我们需要记录好每一页中实际存储了多少个 Tuples
// 因为 SortPage 本身没有这个信息，这就是过分紧凑的代价
std::vector<std::pair<page_id_t, size_t>> page_infos;
```