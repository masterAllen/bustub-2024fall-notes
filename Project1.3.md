# Project1.3 BufferPoolManager 记录

开始实现 BufferPoolManager，先不考虑并行。

## NewPage

这里我真是绕了一大圈，这也是我感觉官网说明需要加强的地方，这里基本没提。我一开始觉得 `NewPage`，那就是要在磁盘中新申请一个 Page 咯。

很顺理成章地，我觉得应该调用成员变量 `disk_scheduler` 里面的函数，但是我们实现的 disk scheduler 并没有这个功能的函数。

然后又想了半天，想到 disk sheduler 里面有个 disk manager，是不是直接用这个？果然发现有个 `AllocatePage`，那似乎很显然了？应该申请一个 Page:
```cpp
  // 一开始我还这样做，以为用 disk_manager 里面的 AllocatePage
  // 但仔细看一下 AllocatePage 的实现，它不是我们这里的意图，它是申请一个空间。
  std::scoped_lock lock(*bpm_latch_);
  // 磁盘上申请一个 Page
  page_id_t page_id = disk_scheduler_->AllocatePage();
  if (page_id == INVALID_PAGE_ID) {
    return INVALID_PAGE_ID;
  }
  std::cout << "NewPage: " << page_id << std::endl;
  return page_id;
```

但实际上并不是这样，真正对的是... 直接成员变量 `next_page_id` 递增... 非常离谱。

原因其实是这次我们得到了一个 `page_id`，假如后续我们对其操作，比如 `Write(page_id)`，显然会触发 `disk_scheduler` 调用 `disk_manager` 进行真正写入。

而 `disk_manager` 真正写入时就会检测这个 `page_id` 是不是超过当前磁盘分配的 page 了。如果是，它来负责在真正在磁盘上新建一个 page，毕竟它是 DISK MANAGER。

## 其他操作

其他的就很简单了，只要细心一点就行。由于先不考虑并发，所以很简单。比如 `CheckedWritedPage`：

1. 检查 page_id 是否在 Buffer Pool 中，如果在那么找对应的 frame 然后构建 WritePageGuard，否则就继续
2. 由于不在 Buffer Pool 中，需要从磁盘加载，并且放在 Buffer Pool 中
3. 首先在 BufferPool 中找一个空闲 Frame，如果有 free_frame，直接拿出来；否则就进行替换，踢出去的 frame 要写回硬盘。要细心，PageTable 等都要更新
4. 从磁盘中的 Page 加载到对应的 Frame 中
5. 有一些数据结构有更新，更新 LRUKReplacer 和 FrameHeader 和 PageTable
6. 构造 WritePageGuard 并且返回

总之就是细心一点，包括 `FrameHeader` 中的 `is_dirty` 等等都要记得更新。有一点：如果是 WritePageGuard，只要执行 `GetDataMut()`，都要把 `dirty` 置为 True，无论最终有没有改变。