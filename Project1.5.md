# Project1.5 BufferPoolManager 记录

这个做完之后，就想着进行优化了。优化的方向很明确，每一页都有一个锁，这样各个页之间的读写互相不受影响。

这篇文章我加了很多的折叠模块，非常推荐先自己去想想问题，感觉蛮有意思的。

## 每一页加锁的准备过程

之前的代码如下，可以看到，如果要替换的话，需要经历 `WriteFrameToDisk` 和 `ReadFrameFromDisk` 这两个巨慢的 IO 操作（这两个也是 bpm 的函数），而在这期间，一直保持着 bpm 的锁，那肯定效率很低。

```cpp
CheckedWritePageGuard (...) {
// ...

// 不在 Buffer Pool 中，需要从磁盘加载，并且放在 Buffer Pool 中
// 换出去的 frame，需要写入到硬盘
if (!WriteFrameToDisk(frame_id, frames_[frame_id]->GetPageId())) {
    return std::nullopt;
}
// 换出去后修改 PageTable 和 LRU_K ...

// 从磁盘中加载 page_id 的 page 到 frame_id 中
if (!ReadFrameFromDisk(frame_id, page_id)) {
return std::nullopt;
}

// ...
}
```

所以自然地，想到每一页都加一个锁，数据结构如下：

```cpp
// 维护一个 Page 的锁，现在还没有动态管理，一旦初始化后了即使 Page 没了也不销毁
std::unordered_map<page_id_t, std::shared_ptr<std::mutex>> page_latch_;
std::mutex page_latch_mutex_;
```

**为了便于称呼，我们把最终进来的叫 Page1，踢出去的叫 Page2**

## Page1 和 Page2 的加锁顺序
现在有一个问题：Page1 和 Page2 哪个先加锁？很好的问题，请先思考。

<details>
<summary> 解答 </summary>

答案是 Page1 要先加锁。是想如果 Page2 加锁，然后进行 `WriteFrameToDisk(page2)`：

1. 请明确我们 Write 的时候现在是没有 bpm 锁的，不然改成一页一页的锁也没有意义了
2. 也就是此时 Write，我们只有 Page2 的锁，这个线程目前在慢慢写，岁月静好
3. 外面已经满天飞了，因为只要处理其他 page，都不受到影响，想想看是不是很可怕
4. 就比如跑了好多个线程，现在已经有 Free Frame 了，这时 OtherThread 申请了 Page1，放到某个 Frame 中
5. 我们的线程终于跑完 `Write(page2)`，现在再 `Read(page1)`，Oops，一个 Page 映射有两个 Frame 了

上面这个是无法通过对 Frame 加锁来规避的。所以，我们必须要先 Page1 加锁，告诉其他线程：这个 Page1 是我的，谁也不许动。

</details>

## 整体构建及系统性分析

那么我们可以写出这样的架构来：

```cpp
CheckedWritePage(...) {
    // 1. bpm.lock() && page1.lock()
    // finding.. find page2

    // 2. page2.lock() && bpm.unlock()
    // WriteToDisk(page2)

    // 3. bpm.lock() && page2.unlock()
    // ReadFromDisk(page1)

    // 4. page1.unlock() && bpm.lock()
    // WritePageGuard(page1)
}
```

### 确认第二组

一共四组，每次有两种情况，那就是十六种情况？太多了！我们可以看到第二组的顺序一定是固定的，一定是 `Page2.lock(); BPM.unlock()`，否则：

<details>
<summary> 死锁情况 </summary>

1. ThisThread 执行到第 2 组前，无论第 1 组顺序如何，他肯定有了 BPM 和 Page1
2. ThisThread 释放 BPM 锁，还没上 Page2 锁
3. OtherThread 申请 Page2，顺利执行，放在别的 Frame 中，然后读取/写入磁盘
4. ThisThread 回来，重新上 Page2 锁，写回，数据彻底乱了
</details>

### 忽略第四组

现在就还有八种情况。我们就依次开始，先忽略第四组，我们来看看第一组和第三组四种情况有什么问题？

#### 选择一
```
BPM.lock() Page1.lock()
Page2.lock() BPM.unlock()
BPM.lock() Page2.unlock()
Page1.unlock() BPM.unlock()
```

<details>
<summary> 死锁？ </summary>

1. ThisThread 执行到第二组，即释放 BPM 锁后，开始进行 I/O，他现在有 Page1 和 Page2 两个锁
2. OtherThread 此时申请 Page1/Page2，会拿到 BPM 锁，但是拿不到 Page1/Page2 锁，等待
3. ThisThread 的 I/O 结束，即到了第三组，想要 BPM 锁，结果被 OtherThread 阻塞，而 OtherThread 等待 Page 锁，死锁

</details>

#### 选择二
```
BPM.lock() Page1.lock()
Page2.lock() BPM.unlock()
Page2.unlock() BPM.lock()
Page1.unlock() BPM.unlock()
```
<details>
<summary> 死锁？ </summary>

1. ThisThread 执行完第二组，即释放 BPM 锁后，开始进行 I/O，他现在有 Page1 和 Page2 两个锁
2. OtherThread 此时申请 Page1，在第一组时，会拿到 BPM 锁，但是拿不到 Page1 锁，等待
3. ThisThread 的 I/O 结束，即到了第三组，想要 BPM 锁，结果被 OtherThread 阻塞，而 OtherThread 等待 Page1 锁，死锁

</details>

#### 选择三
```
Page1.lock() BPM.lock()
Page2.lock() BPM.unlock()
Page2.unlock() BPM.lock()
Page1.unlock() BPM.unlock()
```
<details>
<summary> 死锁？ </summary>

1. ThisThread 执行到第二组之前，即想要 Page2 前，其他线程开始执行，此时有 BPM 和 Page1 两个锁
2. OtherThread 想要 Page2, 在第一组中拿到了 Page2 锁，但是拿不到 BPM 锁，等待
3. ThisThread 开始执行第二组，结果拿不到 Page2 锁，但是有 BPM 锁，等待，死锁

</details>

#### 选择四
```
Page1.lock() BPM.lock()
Page2.lock() BPM.unlock()
BPM.lock() Page2.unlock()
Page1.unlock() BPM.unlock()
```
<details>
<summary> 死锁? </summary>

1. ThisThread 执行到第二组之前，即想要 Page2 前，其他线程开始执行，此时有 BPM 和 Page1 两个锁
2. OtherThread 想要 Page2, 在第一组中拿到了 Page2 锁，但是拿不到 BPM 锁，等待
3. ThisThread 开始执行第二组，结果拿不到 Page2 锁，但是有 BPM 锁，等待，死锁
</details>


#### 结语

完蛋！全部都死锁。不给活路，于是在这里卡了很久。而且这个不好复现，必须魔改 `bpm-bench`，然后打印很多东西才能分析原因。关于打印，可以同样设置一个用于打印的全局锁，这样打印出来才能看，不然乱序根本没法看的。

但是没有死心，下一篇文章继续探讨解决之路。