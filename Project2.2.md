# 并发的再实现

刚在上一篇说这次比想象中顺利多了，晚上通过本地测试后，提交到 gradescope，打算第二天早上看看 QPS 到多少。结果第二天过来一看，LeaderBoard 的测试居然有了 SegmentFault，于是本地多跑了几次，发现确实有这个问题。当时打算偷懒，多次提交，总有一次是全部通过的吧，果然提交多次之后终于有过的了，结果看了一眼 QPS，傻眼了：

![image](./images/Project2_02.png)

WriteQPS 只有几十，这这这... 如果几百我都能忍，睁一只眼闭一只眼也就过来了，几十真是没眼看了。于是就开了艰难的解决之路，一共经历两天:
1. 周二工作事情比较多，白天基本没考虑，晚上下班后简单做了会没效果，也没啥好的思路，很迷茫；
2. 周三正好事情很少，摸鱼做了会，期间一度感觉绝望，觉得能做的都做了，代码完美无缺，根本想不到好的解决方法；
3. 直到晚上十一点半左右，通过层层分析，加上突然的意识，做了一次细微的调整，才终于完成。

本文不是侧重分享知识的笔记，更多的是复述我的艰难解决之路。如果你也遇到了 WriteQPS 离谱的低，直接看最后即可。

## 解决单线程上的错误

在解决并发的问题前，先解决一些单线程上忽略的错误点。

### 解决错误一（易错点）：判断是否为上下限需要注意叶子/内部的类型要正确

这个意思可以看下面的代码，我一开始为了代码简洁，在循环中这样用：
```cpp
  // 非空，那么开始查找，直到最底层
  auto now_page_id = root_page_id;
  while (true) {
    auto now_btree_page_guard = bpm_->WritePage(now_page_id);
    auto now_btree_page = GetInternalPage(now_btree_page_guard);

    if (now_btree_page->IsUpperBound()) {
      ctx.write_set_.clear();
      ctx.header_page_.reset();
    }
    ctx.write_set_.push_back(std::move(now_btree_page_guard));

    if (now_btree_page->IsLeafPage()) {
        break;
    }

    // ...
  }
```
其实这里就有问题，如果 `now_btree_page` 是叶子节点的话，我们执行 `IsUpperBound` 的时候是当成 `InternalPage` 的... 所以就错了，应该先判断是否是叶子节点，然后再去判断上下限：

```cpp
  // 非空，那么开始查找，直到最底层
  auto now_page_id = root_page_id;
  while (true) {
    auto now_btree_page_guard = bpm_->WritePage(now_page_id);
    auto now_btree_page = GetInternalPage(now_btree_page_guard);

    bool is_upperbound = now_btree_page->IsUpperBound();
    bool is_leaf = now_btree_page->IsLeafPage();
    if (is_leaf) {
      auto now_btree_page_leaf = GetLeafPage(now_btree_page_guard);
      is_upperbound = now_btree_page_leaf->IsUpperBound();
    }
    if (!(is_upperbound)) {
      ctx.write_set_.clear();
      ctx.header_page_.reset();
    }
    if (is_leaf) {
        break;
    }
    // ...
  }
```

### 解决错误二（个人问题）：ReadPageGuard Drop 错误

这个错误是个人自己的错误，属于是 Project 1 里的一个笔误，Debug 了大概三个多小时，期间一度绝望，感觉没有任何问题。当时是想试试利用 Header_page 读取完 root_page_id 之后，立马释放掉，然后看看会不会好一点。结果死锁了...

一开始是用脑子想，进行了很多种打印尝试，结果都没问题。很绝望，然后才用 gdb，现在想想，**这种多线程死锁应该先用 gdb 看看各个线程卡在哪儿的**。用 gdb 看，发现他们都是卡在了请求 RootPage 的读锁那里，但是按理来说没问题呀，我的 WritePage 没有任何毛病，每次析构函数都会把当前的写锁释放掉的。

但 gdb 显示就是卡在那儿，我无可奈何了。又是冥思苦想了一个多钟头，多次测试 WritePage 的各种情况，打印了层出不穷的语句。但就是没有问题，已经打算直接摆了。后来去吃晚饭前，还是不死心。结果... 发现居然是 ReadPage 那儿有问题... ReadPage 的锁释放有问题，也就是别的线程 GetValue 结束之后还占着锁...

就是这一个小小的笔误，让我费心好久。

```cpp
    // ???? 我服了，这里本来放在 if 里面，调试了半天
    read_lock_.unlock();
    if (frame_->pin_count_ == 0) {
      replacer_->SetEvictable(frame_->frame_id_, true);
    }
```

## WriteQPS 过低的解决之路

### 1. 提前释放 HeaderPage
最先想到的就是一个改进就是提前释放 Header Page，也就是如果发现 Root Page 不会变，那么 `ctx.header_page` 也 `reset` 掉就行。这个确实是应该做的，但最后发现没有太大提升。

### 2. 改成乐观锁
然后就考虑是不是大家都实现了乐观锁？于是吃完晚饭后就去把乐观锁实现了。乐观锁的实现真是巨简单了，直接一路用读锁，读到叶子节点后，上写锁。如果发现叶子是直接普通的插入删除，那么就直接相应操作就行了。我感觉应该会有很大帮助，结果发现还是维持在几十的 QPS...

### 3. 只看 insert 的情况
遵循最小化 Debug 的原则，尽可能让场景简单化，于是我就将 `btree-bench.cpp` 中的文件修改为只插入，看看他统计的 writeQPS 怎么样，结果果然只有几十。好，那么我们现在就只来关注 insert 。

### 4. 统计乐观锁占比
于是我就想到，是不是乐观锁占比其实很低，不然为什么 QPS 还是只有几十？所以，我增加一些用来计数的成员变量，每次 insert 的时候进行相关的计数。结果发现，乐观锁居然占比有 95% 往上！所以乐观锁是有必要的。

### 5. 统计各个代码块时间
既然乐观锁占了 95% 以上，那为什么 QPS 还是只有几十？所以我想到应该统计各个块的时间，比如我把 Insert 分为：乐观锁完成时间、获取 Header 的时间、加锁加到叶子节点时间、叶子节点插入时间、上溯时间...

问了一下 AI 如何统计时间，给了很好的代码段：
```cpp
auto ClockNs() -> uint64_t {
  auto now = std::chrono::steady_clock::now();
  return std::chrono::duration_cast<std::chrono::nanoseconds>(now.time_since_epoch()).count();
}
```

结果发现：乐观锁的时间很快，是有效果的。瓶颈是获取 HeaderPage 的时间，平均下来用了接近 `3*10^7` ns！而其他最长的，也只是 `10^5` 这个级别，乐观锁的时间甚至只有 `10^4` 级别。

### 6. 思考请求 HeaderPage 为什么这么慢
所以一切都指向了 HeaderPage 的请求问题。我一开始以为是上面第一小节中，提前释放 HeaderPage 的语法错了，但查阅资料后，确实 `ctx.header_page.reset()` 是释放了锁的。又想了半天，想不出来，只能得出结论：或许 HeaderPage 就该这么慢，毕竟它是树的开始，所有的操作都要经过它。

### 7. 比较乐观锁和不用乐观锁的各个代码块时间
但是转念一想，不对啊，为什么我用了乐观锁，而且占比很高、时间很短，为什么 QPS 还是几十？？所以就又统计了代码完全不用乐观锁的情况，发现请求 HeaderPage 的时间骤减，但依然是瓶颈，依然比其他代码片段的时间要高不少。

但是这就奇怪了，为什么不用乐观锁了，HeaderPage 的请求时间会减短？这个当时没怎么思考到原因，回过头来看，这个问题一定程度上能指引我们朝着正确的方向走。

### 8. 减小请求 HeaderPage 的时间

所以就想方法减小这个瓶颈时间呗，于是尝试了如下方法：

1. 一开始想的，HeaderPage 全部用读锁，直到最后发现 RootPage 要改了，即 HeaderPage 需要修改时，把读锁变成写锁。但查阅资料发现，这是标准的锁升级问题，这有风险，即读锁释放再去加写锁，这中间有风险。

2. 于是想着，是不是用条件变量？我之所以这样想，是因为觉得可能是请求锁导致频繁等待，频繁空转了。然后查阅资料，问了 AI，发现其实 C++ 里面的锁并不是空转的，是会主动睡眠的，所以使用条件变量应该改变不大。

3. 感觉没别的招了，于是就想效仿乐观锁的方法。我先用读锁，然后呢，直到我发现 RootPage 要改了，我再从头开始用写锁，一路加到底。这个差点就打算做了，但后来觉得自己越走越远了，别人不可能这么来实现的，肯定是自己哪里还有疏漏。

4. 每个操作都要请求 HeaderPage，只不过有的是读锁有的是写锁。读的 QPS 很高，那是不是可以改变锁的策略，优先安排写锁的程序？和上一条一样，我感觉复杂了，根本原因肯定不在这。

### 9. 持有写锁的时间

继续思考原因。是不是因为每次 Insert 请求到写锁之后，后面花费的时间过长了？也就是说，虽然第一条中我发现 RootPage 如果不变会提前释放 HeaderPage，但是这个占比很低？于是依旧是增加一些统计变量，结果显示 Insert 中一直持有 Header 的占比只有 10%，显然不是这个原因。

那干脆，我统计一下持有写锁的时间，所以我在 `WritePageGuard` 增加了一个统计时间的变量，并且析构函数中会打印它的时间，并且由于 HeaderPage 是 0，所以我们可以加个判断，这样打印条目很少。
```cpp
  auto now = std::chrono::steady_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(now - acquire_time_);
  // Page0(HeaderPage) 才打印
  if (page_id_ == 0 && duration.count() > 10) {
    std::cout << "WritePageGuard Destructor, duration = " << duration.count() << "ms" << std::endl;
  }
```
结果显示，持有写锁的时间也不长，所以真的不知道怎么解决了。


### 灵光一闪的解决

已经是晚上十一点半了，就要睡觉了，就要放弃了。突然，脑子里蹦出一个想法：承接上面的 8.4，请求 HeaderPage 的写锁很长，因为读锁很多？那我试一下把 GetValue 的线程都禁掉看看？果然提高了一些。再看 GetValue 的代码，终于逮到你了：读锁没有及时的释放！！

```cpp
auto BPLUSTREE_TYPE::GetValue(const KeyType &key, std::vector<ValueType> *result) -> bool {
  auto header_page_guard = bpm_->ReadPage(header_page_id_);
  auto root_page = header_page_guard.As<BPlusTreeHeaderPage>();
  auto root_page_id = root_page->root_page_id_;
  // !!! 这里需要及时释放
  header_page_guard.Drop();

  // 开始查找，先无脑查找，直到最底层的叶子节点
  // ...

  // 叶子节点搜索
  // ...
}
```

因为 `header_page_guard` 是一开始就申请的，所以只有不主动 Drop，那么函数体内就一直会存活！！！所以每次读锁占用时间特别长！同理，乐观插入、乐观删除等都需要将 HeaderPage 读锁主动释放掉。

最后一测试，果然.... 瞬间提升到了一万多... （其实还有一个并发问题，截图是解决其之后的结果，该问题我觉得挺有意思，留在第三篇讲）。虽然排名还是很低，但总体思路基本是对的，剩下的就是花时间做一些优化了，我个人感觉就觉得没必要了。

![image](./images/Project2_03.png)

本来是打算当天晚上看一部电影的，但解决这个的快乐堪比看十部、看百部电影啊。完成之后的那一瞬间的轻松和欢愉，或许就是人类活着的意义啊... 