# Project1.6 BufferPoolManager 记录

上一篇说，我想要优化每一页进行加锁，这样各页之间读写不会受到影响。但经过分析，发现各种加解锁顺序下，都会死锁，感觉没有解决方案了。但没想到最后居然完成了，有点出乎意料，就是一瞬间的想法。


## 同时上锁

既然一个一个锁，不行，那就同时上锁？而 C++ 恰好有这样的属性，这个回答是 AI 帮我的，后来提交通过，但是效率甚至比单线程还要低。

```cpp
// 每次开始需要请求这个 Page 的锁
auto nowpage_latch = GetPageLatch(page_id);
std::lock(*bpm_latch_, *nowpage_latch);

// 把锁托管给 unique_lock（不再重复 lock）
std::unique_lock<std::mutex> bpm_lock(*bpm_latch_, std::adopt_lock);
std::unique_lock<std::mutex> page_lock(*nowpage_latch, std::adopt_lock);

// ...

// 之后可以单独 unlokc
bpm_lock.unlock()
```

## 舍弃完全并行

于是换种思路，那既然想要完全并行不可以，我舍弃一部分。如果我不加 Page1 锁（Page1 是最后进来的），而只对 Page2 加锁（Page2 是最后出去的），是不是可以？

但其实还是回到上一篇文章一开始讨论的：Page1 要加锁，相当于告诉其他线程，这个 Page 是我的，谁也不许动，否则：

1. 我们 Write(page2) 的时候现在是没有 bpm 锁的，不然改成一页一页的锁也没有意义了
2. 也就是此时 Write，我们只有 Page2 的锁，这个线程目前在慢慢写，岁月静好
3. 外面已经满天飞了，因为只要处理其他 page，都不受到影响，想想看是不是很可怕
4. 就比如跑了好多个线程，现在已经有 Free Frame 了，这时 OtherThread 申请了 Page1，放到某个 Frame 中
5. 我们的线程终于跑完 `Write(page2)`，现在再 `Read(page1)`，Oops，一个 Page 映射有两个 Frame 了

## 空转

还想到用空转，或者用条件变量。但后来想想，感觉越想越复杂了，这样做可能负优化了。

## 最终解决

最后还是解决了，突然想到的解决方法。回归到上篇文章的某次分析：

```
BPM.lock() Page1.lock()
Page2.lock() BPM.unlock()
Page2.unlock() BPM.lock()
Page1.unlock() BPM.unlock()
```

1. ThisThread 执行完第二组，即释放 BPM 锁后，开始进行 I/O，他现在有 Page1 和 Page2 两个锁
2. OtherThread 此时申请 Page1，在第一组时，会拿到 BPM 锁，但是拿不到 Page1 锁，等待
3. ThisThread 的 I/O 结束，即到了第三组，想要 BPM 锁，结果被 OtherThread 阻塞，而 OtherThread 等待 Page1 锁，死锁

关键点就是第二步，我们想要的锁被别的线程拿走了。那怎么办？可以让 OtherThread 放弃+睡眠，似乎可以。但其实有个更好的操作：如果线程有意识，这个时候最好怎么做？OtherThread 知道这个 Page1 已经被 MyThread 锁定了，并且结束后会把内容更新到 Frame 中。所以 OtherThread 等 MyThread 执行完，然后立马就和对应 Frame 打交道。

怎么想到的，其实就是第二步中，如果 OtherThread 拿不到 Page1 能放弃就好了。有的处理可能是回退，但回退显然不好；所以就想着如果释放掉 BPM 锁，如何保证正确性，于是就逐渐知道怎么做了。

上面有几点注意点，下面进行说明：

### 第一点：MyThread 提前更新某些结构

OtherThread 知道 MyThread 结束之后会把 Page 放在对应 Frame 中；这个如何知道？
 
MyThread 来做，即 MyThread 在进行实际磁盘操作前，就把 `page_table_` 和 `frames_` 等变量修改了。此时 MyThread 即使休眠了，OtherThread 也会通过 `page_table_` 等知道，哦，这个 Page 和这个 Frame 对应上。

总之，MyThread 的原则就是，我休眠了，但别发生幺蛾子。MyThread 也要确保这个 Frame 不会被换出去，需要执行 `SetEvictable(false)`。

### 第二点：OtherThread 找到对应后释放锁

但是死锁问题还是没有解决。OtherThread 知道 Page 和 Frame 的对应了，但它还有 BPM 锁啊，最后还是系统会停住。

而这就是我们出发的目的，OtherThread 要直接释放掉 BPM 锁！意思就是，OtherThread 现在就等 MyThread 执行完了，他一结束，我就处理 Frame 构造 PageGuard。

### 小结

其实我们现在做的就是上面四组中第一组的 `BPM.lock(), Page1.lock()` 这里，希望能 `Page1.lock()` 的时候，可以 `BPM.lock()` 释放掉：
1. 简单的做法就是失败后回退，但显然很不好
2. 我们这里提前把 `page_table` 改掉，这样找到了 Page 和 Frame 的映射，就可以大胆地 `BPM.unlock()` 了。
3. 为什么之前不敢大胆 `BPM.unlock()`？因为怕别人把 Page 和 Frame 的映射改掉了。为什么我们这里可以？我们对这个 Frame 进行了 `SetEvictable(false)`！

### 第三点：OtherThread 如何确保不被第三者干扰

还有个很微妙的地方：在 OtherThread 释放 BPM 锁，加载 Page1 锁的时候，它什么也没锁，此时有什么风险。

MyThread 执行完了，MyThread 也结束各种操作了。第三者又开始对这个 Frame 操作了，如果还是这个 `page_id` 还好说，但如果是别的 Page 映射到这个 Frame 上怎么办？？

这是有可能的，MyThread 的那个线程万一已经对 Page 各种操作完毕了，进行了 `Drop()`，Frame 重新变成可以替换，完蛋。所以 OtherThread 自己要避免这种情况？怎么避免？`pin_count++` 呗！

### 最终框架

综上，最后的框架是这样的：

```cpp
auto BufferPoolManager::CheckedWritePage(page_id_t page_id, AccessType access_type) -> std::optional<WritePageGuard> {
    // 无论如何，都进行 bpm.lock()
    bpm.lock();

    // 检查 page_id 是否在 Buffer Pool 中
    if (page_table_[page_id] is gooooood) {
        // page_id <--> frame_id
        // ...

        // 尤其是 pin_count_，确保自己休眠了，这个映射不会被打断
        replacer_->SetEvictable(frame_id, false);
        frames_[frame_id]->pin_count_++;

        // 做了上面的事情，我可以大胆释放了
        bpm_lock.unlock();

        // 在这期间，我即使睡眠很久很久，也不会被别的线程影响正确性了

        // 等待其他占有 Page 的人释放了
        page_lock.lock();

        // lock 和 unlock 之间不需要做任何事情，只要别人一释放，我就知道，这 Page 我可以处理了

        // 一旦别的 Page lock 了，那说明至少 Page 和 Frame 的内容是对应的
        page_lock.unlock();
        return WritePageGuard(page_id, frames_[frame_id], replacer_, bpm_latch_);
    }

    // 不在 Buffer Pool 中，需要从磁盘加载，并且放在 Buffer Pool 中
    
    // 找到要被替换的 swap_page_id（有可能没有，假如有 Free Frame 的话）
    auto swap_page_id = std::optional<page_id_t>();

    // 同样，要更新 PageTable。这里的意义是让别的线程就进入上面的 if 等待去吧
    replacer_->RecordAccess(frame_id, access_type);
    replacer_->SetEvictable(frame_id, false);
    frames_[frame_id]->SetPageId(page_id);
    frames_[frame_id]->pin_count_++;
    page_table_[page_id] = frame_id;

    // 对这个进来的 Page1 加锁，一定要加锁
    page1.lock();

    // 对这个踢出去的 Page2 加锁
    page2.lock();

    // Bpm 解锁，要开始漫长的写磁盘 IO 操作咯
    // 正因为有 bpm 的锁，所以上面更新 PageTable、加 Page1 和 Page2 锁的顺序可以无所谓
    bpm_lock.unlock();

    // 写入到硬盘
    WriteFrameToDisk(frame_id, swap_page_id.value());

    // 写完后：释放 Page2 锁
    page2_lock.unlock();


    // BPM 继续加锁
    bpm_lock.lock();

    // 处理一些数据结构，比如 is_dirty（当然不处理放在上面更新 PageTable 那里也可以）

    // BPM 解锁，要开始漫长的读磁盘这个 IO 操作咯
    bpm_lock.unlock();

    // 读取磁盘
    ReadFrameFromDisk(frame_id, page_id, bpm_lock);

    // 释放 Page1 锁
    page1_lock.unlock();

    return WritePageGuard(page_id, frames_[frame_id], replacer_, bpm_latch_);
}
```

## 结语

终于完成了，虽然看起来写的很少，但真是花了好多功夫。虽然对我没有用处，不过解决完这种并发问题，感觉真爽。
