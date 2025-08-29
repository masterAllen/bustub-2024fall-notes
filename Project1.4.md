# Project1.4 BufferPoolManager 记录

开始考虑并发，做完了 P1-P3，我感觉 P1 花费的时间是最最长的，而 70% 的时间都是花在对并发处理进行的 DEBUG，太痛苦了，很多只能在网页上测评。不过可以在本地开 Leaderboard Task，那个可以检测很多问题，等确认可以再去提交到网上，不然每次都等结果，太慢了。

## 并发处理一：Frame 前先释放 BPM 锁

最符合人类思维的申请 Frame 方式，肯定是我先有 bpm 锁，然后申请完 Frame 释放，即：

```cpp
auto BufferPoolManager::CheckedWritePage(page_id_t page_id, AccessType access_type) -> std::optional<WritePageGuard> {
    std::scoped_lock lock(*bpm_latch_);

    // ...
  
    // WritePageGuard 中会对 Frame 上锁
    return WritePageGuard(page_id, frames_[frame_id], replacer_, bpm_latch_);
}
```

这个感觉很合逻辑。但是这会同时拥有两把锁，这是比较冒险的事情。这也是很经典的死锁问题，其实本地测试中的 `DeadlockTest` 已经给了我们具体的样例了，具体是：

1. MyThread 申请 Page0：申请 bpm 锁，申请 Page0 锁，释放 bpm 锁，完成度 100% 的一个操作
2. OtherThread 申请 Page0：申请 bpm 锁，再申请 Page0 锁，卡住
3. MyThread 申请 Page1：申请 bpm 锁，结果锁再 OtherThread 上，卡住
4. 死锁

很经典的问题，解决方式就是尽量不同时有两把锁：每次在进入 PageGuard 前释放掉 bpm 锁，所以我们要主动释放锁，此时就不能用 `scoped_lock`，因为它不能主动释放，而是改用 `unique_lock`。

```cpp
auto BufferPoolManager::CheckedWritePage(page_id_t page_id, AccessType access_type) -> std::optional<WritePageGuard> {
    std::unique_lock lock(*bpm_latch_);

    // 主动释放掉 bpm 锁
    lock.unlock();
    // WritePageGuard 中会对 Frame 上锁
    return WritePageGuard(page_id, frames_[frame_id], replacer_, bpm_latch_, disk_scheduler_);
}
```

## 并发处理二：`SetEvictable` 位置

一开始，我是在 PageGuard 中进行 `SetEvictable`，即如果一个 Page 被占用了，那它肯定不能被替换出去。但测试之后，有并发问题：

```cpp
auto BufferPoolManager::CheckedWritePage(page_id_t page_id, AccessType access_type) -> std::optional<WritePageGuard> {
    std::unique_lock lock(*bpm_latch_);

    // ...

    // 这里一开始在 PageGuard 的构造函数做的，但会有并发问题：
    // 1. This Page 之前被读取过并且 Drop 掉，所以 pin_count 为 0，可替换，但还在 PageTable 中
    // 2. 本线程读取 This Page，由于还在 PageTable 中，所以进入到这个条件分支
    // 3. 在执行 ThisPageGuard 的构造函数前，此时这个 Frame 还是可替换的，切换到其他线程
    // 4. 另一个线程想要写 OtherPage，但是满了，需要替换，恰好选中了 This Page 对应的 Frame
    // 5. 另一个线程把 This Frame 替换成了 OtherPage，而 ThisPageGuard 开始构造，这样就读取错了错误的 Page
    replacer_->SetEvictable(frame_id, false);

    // 返回 WritePageGuard
    lock.unlock();
    return WritePageGuard(page_id, frames_[frame_id], replacer_, bpm_latch_, disk_scheduler_);
}
```

一个很不容易想到的问题，得测试才发现。


## 并发处理三：`pin_count` 位置

同理，原来是在 PageGuard 中进行 `pin_count_++`，那么会有什么问题？请自己思考。

<details>

<summary> 解答 </summary>

1. 在本线程执行之前，有一个线程 A 访问到了 ThisPage
2. 本线程执行，进入到这里的条件分支，在执行 ThisPageGuard 构造函数前，切换到线程 A
3. 线程 A 进行 Drop，此时 ThisPage 的 pin_count 为 0，Drop 中设置为可替换，但还在 PageTable 中，切换到另一个线程 B
4. 线程 B 想要写 OtherPage，但满了，需要替换，恰好选中了 This Page 对应的 Frame
5. 线程 B 把 This Frame 替换成了 OtherPage，而 ThisPageGuard 开始构造，这样就读取错了错误的 Page

</details>


## 结语

上面的操作弄完之后，就可以通过测试了。后续是进行优化。