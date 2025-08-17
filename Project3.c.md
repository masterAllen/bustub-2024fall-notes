# Project3.a External Merge Sort 解决之路三（邪修实现）

其实已经完成了，这里就是简单说一下后续的一个更新。

第一步中读取 Tuples，组成 SortPage 并且排序，逻辑是这样的：

```cpp
auto sort_page = std::make_unique<SortPage>();

while (child_executor_->Next(&tuple, &rid)) {
  // ...

  sort_page->PutTuple(tuple_count, tuple, tuple_size);
  tuple_count++;

  // 如果满了，那么就排序，然后写入到磁盘，然后清空 SortPage，然后继续读取
  if (tuple_count == BUSTUB_PAGE_SIZE / tuple_size) {
    // 排序
    Sort(*sort_page, tuple_size, tuple_count);

    // 写入到磁盘 ...

    // 清空 SortPage 和相关变量...
  }
}
// 如果 SortPage 中还有数据，那么就写入到磁盘
if (tuple_count > 0) {
  Sort(*sort_page, tuple_size, tuple_count);
  page_ids.push_back(std::make_pair(WriteToDisk(*sort_page), tuple_count));
}
```

这里是每次都把 Tuples 放进 sortpage 中，但感觉没必要，这样排序的时候还要重新从 sortpage 中解析，所以直接放在 vector 中就好了，满了就排序，排序完再统一放在 sortpage 中写入磁盘即可。

而且好处就是不需要记录 `tuple_count` 这个变量了，每次用 `vector.size` 就可以了，`Sort` 方法也不那么奇怪了：

```cpp
// 原来的
void ExternalMergeSortExecutor<K>::Sort(SortPage &sort_page, size_t tuple_size, size_t tuple_count) {
  std::vector<SortEntry> sort_entries;
  for (size_t i = 0; i < tuple_count; i++) {
    auto now_tuple = sort_page.GetTuple(i, tuple_size);
    sort_entries.push_back(std::move(TupleToSortEntry(now_tuple)));
  }
  std::sort(sort_entries.begin(), sort_entries.end(), cmp_);
  for (size_t i = 0; i < tuple_count; i++) {
    auto now_tuple = sort_entries[i].second;
    sort_page.PutTuple(i, now_tuple, tuple_size);
  }
}

// 现在的
void ExternalMergeSortExecutor<K>::Sort(std::vector<Tuple> &tuples) {
  std::vector<SortEntry> sort_entries;
  for (auto& tuple : tuples) {
    sort_entries.push_back(TupleToSortEntry(tuple));
  }
  std::sort(sort_entries.begin(), sort_entries.end(), cmp_);
  for (size_t i = 0; i < tuples.size(); i++) {
    auto now_tuple = sort_entries[i].second;
    tuples[i] = now_tuple;
  }
}
```