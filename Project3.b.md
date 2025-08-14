# Project 3.b External Merge Sort 解决之路二（邪修实现）

前一篇文章中，我们是了解外部排序原理，然后想了想 SortPage 应该怎么用，设计了结构和函数。并且完成第一步操作：读取 Tuple 后存入 SortPage，之后写入磁盘。现在我们有了一堆 `page_id`，这里面每个都存的 SortPage，而且是排好序的。下面我们该开始外部排序的第二步了。

前一篇文章中说我用 SortPage 本质就是 `char` 数组，没有定义头部。这已经是邪修做法了，之后我在这条道路上越走越偏，彻底疯魔，最后我甚至没用到 `MegerSortRun` 这个类...

## 开始执行前的准备

开始实现。主要还是要对外部排序第二步要深刻，外部排序第二步每一轮都叫做一个 PASS，我在想每个 PASS 之前我应该有嵌套存储 `page_id` 的变量。PASS 执行前，结构如下，每个 `{...}` 都是排好序，即顺序的页。 

```
{ {...}, {...}, {...} ...}
```

PASS 执行后，其实还是这样的结构，只不过 `{...}` 里面的数量的增多，因为多个 `{...}` 合成了一个嘛。

所以我们要有这个逻辑，而且我在想，用 `queue` 比 `vector` 更方便，因为每次我们都是顺序操作的。一开始时，每个 `{...}` 只有一个元素。

我们第一步中得到了如下结构：
```cpp
// 原来：std::vector<page_id_t> page_ids;
// 现在我们需要记录好每一页中实际存储了多少个 Tuples
// 因为 SortPage 本身没有这个信息，这就是过分紧凑的代价
std::vector<std::pair<page_id_t, size_t>> page_infos;
```

我们转成这个上面提到的这个两重 `queue` 结构：
```cpp
using SortPageInfo = std::pair<page_id_t, size_t>
// 存储 SortPage 对于的 ID，以及对应的大小
auto page_ids = std::queue<std::queue<SortPageInfo>>();

for (auto& now_page_id: all_page_ids) {
    // 写入到磁盘
    page_ids.push(std::queue<SortPageInfo>());
    page_ids.back().push(now_page_id);
}
```

## 基本框架

第二步的基本框架如下：

```cpp
// 这里是要记录 SortEntry 从第几个中读取的，这样才知道哪个 SortPage 应该读取下一个 Tuple
using SortEntryInfo = std::pair<SortEntry, int>;

// 第二步：开始 PASS 阶段
while (page_ids.size() > 1) {
    // 这一轮(Pass)最终生成后的结果
    auto new_page_ids = std::queue<std::queue<SortPageInfo>>();

    // 计算 one_pass_count: 首先一轮(Pass) 需要几次读取；略
    for (size_t i = 0; i < onepass_count; i++) {
        auto input_pages = std::vector<std::queue<SortPageInfo>>();
        auto output_pages = std::queue<SortPageInfo>();

        // 读取 K 个 {...}
        for (size_t j = 0; j < K; j++) {
            // ...
        }

        // 实际可能不足 K 个（最后一轮）
        auto now_k = input_pages.size();

        // 输入：K 个 SortPage 和 SortPage 读到第几个 Tuple、SortPage 大小
        auto input_sort_pages = std::vector<SortPage>();
        auto input_sort_pages_idx = std::vector<size_t>(now_k, 0);
        auto input_sort_pages_size = std::vector<size_t>(now_k, 0);

        // 输出：SortPage 和对应数量
        auto output_sort_page = SortPage();
        auto output_sort_page_count = 0;

        // 每个 {...} 中的第一个 page_id，从磁盘中读取，然后转成 SortPage，对应上面的 input_sort_pages
        for (size_t j = 0; j < now_k; j++) {
            // ...
        }

        // 维护一个堆，每次从中选出最小的 Tuple，然后写入到 output_sort_page 中，索引信息也要记录
        auto heap = std::priority_queue<SortEntryInfo, std::vector<SortEntryInfo>, TupleComparator>{cmp_};
        for (size_t i = 0; i < now_k; i++) {
            // ...
        }

        // 开始遍历，每次从 heap 中取出最小的 Tuple，然后写入到 output_sort_page 中
        while (!heap.empty()) {
            // 取一个元素：now_sort_entry，和 now_idx，后者用来表示这个 SortEntry 是第几个 {...} 

            // 由于是最小值，那么放入 output_sort_page 中
            output_sort_page.PutTuple(output_sort_page_count, now_sort_entry.second, tuple_size);
            output_sort_page_count++;

            // 如果当前输出的 SortPage 满了，那么就写入到磁盘，把写入的 page_id 存到 output_pages 中
            // 做完之后，清空 SortPage
            if (output_sort_page_count == BUSTUB_PAGE_SIZE / tuple_size) {
                // ...
            }

            if (input_sort_pages_idx[now_idx] < input_sort_pages_size[now_idx]) {
                // 如果这次挑选的 SortPage 还有数据，那么就继续获取其下一个 Tuple，加入到 Heap 中
                // ...
            } else {
                // 如果当前输入的 SortPage 到底了，那么就从对应 {...} 读取下一个 Page
                if (input_pages[now_idx].empty()) {
                    // 如果发现当前的 page 列表已经空了，那么就跳过去
                    continue;
                }

                // 读取 Page，然后转成 SortPage
                // ...

                // 读取第一个 Tuple，存到 Heap 中
                // ...
            }
        }

        // while 循环结束，说明排序完毕了，如果还有数据，那么处理一下
        if (output_sort_page_count > 0) {
            // 同样把当前输出的 SortPage 写入磁盘，并且更新 output_pages
        }

        // 本轮结束，将 output_pages 加入到 new_pages_queue 中
        new_page_ids.push(std::move(output_pages));
    }

    // PASS 结束
    page_ids = std::move(new_page_ids);
}
```

其中始终维护一个堆，堆的元素是 `SortEntryInfo`，它是我自己定义的：
```cpp
// 这里是要记录 SortEntry 从第几个中读取的，这样才知道要从哪个 SortPage 继续读取下一个 Tuple
// 这就是 SortPage 过于紧凑的代价，如果 SortPage 可以设计一个 header，里面有 page_id 信息，就不用这么麻烦了
using SortEntryInfo = std::pair<SortEntry, int>;
```

所以也要添加这个 `SortEntryInfo` 的比较方法，具体代码不谈了。

按照这个逻辑，就通过了... 就没用到 `MergeSortRun` 这个类。感觉这部分像是做了一个中等偏上难度的算法题... 只调试了几次，周五下班后晚上实现加上周六午饭之后两个小时调试，很顺利地就过了..

Next 方法这里不详细讲了，还有一个注意点就是最好加一个析构函数，如果发现还有 SortPage 未释放的话，需要删掉