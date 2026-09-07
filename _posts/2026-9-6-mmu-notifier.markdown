---
layout: post
title:  "MMU notifier 详解"
date:   2026-9-5 6:00:16 +0800
categories: GPU
---

## 什么是 MMU notifier？

当 KVM 运行虚拟机或者 GPU 运行计算任务时，它们有自己的页表（称为 **SPTE，Shadow PTE / 次级页表**）。这些 SPTE 必须和 Linux 主内存的 **PTE（主页表）** 保持一致。 如果 Linux mm 子系统决定要把某个内存页换出（Swap）或者释放，mm 子系统必须通知 KVM/GPU 把他们的 SPTE 也删掉。`invalidate_range_start()` 和 `end()` 就是这个通知的“开始”和“结束”标志。这个通知机制就是 MMU notifier。

但是内核中 mmu notifier 的实现还挺复杂的，因为其要解决的问题本身就挺复杂，而且为了实现并发处理页表更新操作，最大化的优化性能，还涉及内存同步、lock、RCU、异步等待、还使用了一些隐晦的机制( [seqlock 机制](https://docs.kernel.org/locking/seqlock.html) )来实现特定功能，所以代码整体理解起来有一定的难度，不过也能学习一些硬核的设计思想。

Ps. 到了最后文章都写好了才找到对 mmu notifier 大改的 patch，哎 ... 不过能看出来，这个 patch 确实很复杂

https://lists.freedesktop.org/archives/nouveau/2019-November/035069.html

## 数据结构

![image-20260630143504632](/assets/gpu/image-20260630143504632.png)

**`mmu_notifier`** 是为**全虚拟化/硬件全镜像**量身定制的。结构体简单，全空间覆盖，适合不需要频繁变动、掌控全局的组件（KVM、SMMU），有以下特点：

- 这些组件需要保持和 mm 子系统完全一样的映射
- 低频全局挂载

**`mmu_interval_notifier`** 是为高性能异构计算（GPU、HMM、RDMA）量身定制的。它引入区间红黑树和奇偶状态机，把各驱动原本需要重复实现的“冲突重试”与“区间检索”收归内核，从而能够支撑 GPU 动辄成千上万个动态内存对象的超高并发监控需求，有以下特点

- GPU 不关心全空间，只关心自己申请的 Memory Buffer
- 高频、动态的生命周期，与虚拟机不同，GPU 的应用会频繁地创建 Buffer、销毁 Buffer。这就意味着驱动需要**成百上千次地注册和注销**监控。 如果这些频繁的操作都去直接修改全局的 `mm_struct` 的全局链表，多核并发下的锁竞争会把整个核心 mm 子系统卡死。

为了解决 GPU 和 RDMA 驱动开发者在驱动层各写各的、效率低下的乱象，2019 年 Jason Gunthorpe 在内核中引入了`mmu_interval_notifier`。它的本质是**把区间查找过滤的逻辑，从“驱动层”上移到了“主内核 mm 层”统一管理**。

## 使用流程

```c
// 注册 mmu notifier 的只有 kvm 和 iommu 
// 没有 GPU 吗？GPU 使用 interval notifier
mmu_notifier_register(&kvm->mmu_notifier, current->mm)
  
static const struct mmu_notifier_ops kvm_mmu_notifier_ops = {
    .invalidate_range   = kvm_mmu_notifier_invalidate_range,
    .invalidate_range_start = kvm_mmu_notifier_invalidate_range_start,
    .invalidate_range_end   = kvm_mmu_notifier_invalidate_range_end,
    .clear_flush_young  = kvm_mmu_notifier_clear_flush_young,
    .clear_young        = kvm_mmu_notifier_clear_young,
    .test_young     = kvm_mmu_notifier_test_young,
    .change_pte     = kvm_mmu_notifier_change_pte,
    .release        = kvm_mmu_notifier_release,
};

// 注册一个 mmu_interval_notifier
amdgpu_mn_register(bo, addr) //addr is cpu va
-> mmu_interval_notifier_insert(&bo->notifier, current->mm, addr, size, &amdgpu_mn_gfx_ops)
   -> subscriptions = smp_load_acquire(&mm->notifier_subscriptions)
   -> if(!subscriptions || !subscriptions->has_itree)
      -> mmu_notifier_register(NULL, mm)
         -> alloc and init an mmu_notifier_subscriptions
         -> set subscriptions->has_itree = true
   -> subscriptions = mm->notifier_subscriptions
   -> __mmu_interval_notifier_insert(&bo->notifier, mm, subscriptions, start, leng, ops)
      -> init interval_sub->interval_tree.start
      -> init interval_sub->interval_tree.last = start + len -1
      -> some fine control logic
      -> interval_tree_insert(&interval_sub->interval_tree, &subscriptions->itree)
```

## 工作机制 - collision-retry

主要工作逻辑就在这个注释里，这里的 write-side 指 mm-core，read-side 指的是 kvm 或 GPU driver

```c
/* This is a collision-retry read-side/write-side 'lock', a lot like a
 * seqcount, however this allows multiple write-sides to hold it at
 * once. Conceptually the write side is protecting the values of the PTEs in
 * this mm, such that PTES cannot be read into SPTEs (shadow PTEs) while any
 * writer exists.
 *
 * Note that the core mm creates nested invalidate_range_start()/end() regions
 * within the same thread, and runs invalidate_range_start()/end() in parallel
 * on multiple CPUs. This is designed to not reduce concurrency or block
 * progress on the mm side.
 */
 
// 1. core mm 可能会嵌套式调用 invalidate_range_start/end
// 2. 同一个进程的 mm 可能会在不同 cpu 上同时执行
   
// 假设你有一个多线程的程序，运行在一个 4 核 CPU 上，多个线程共享同一个进程的内存空间（即同一个 mm_struct）。
// CPU 0： 线程 A 正在调用 madvise(DONTNEED) 主动释放 A 内存段，这会触发 invalidate_range_start()。
// CPU 1： 线程 B 恰好因为内存不足（Page Fault），触发了内存回收（Reclaim），
// 内核决定要换出（Swap out）该进程的 B 内存段，这也会触发 invalidate_range_start()。
// 如果这里使用传统的互斥锁（Mutex），那么当 CPU 0 在清理 A 内存时，CPU 1 就必须死等（Block），
// 直到 CPU 0 搞完。在大型服务器（比如几百个 CPU 核心）上，这种死等会带来巨大的性能灾难，
// 因此，mmu_notifier 采用了无锁计数器的方式：
// 无论是哪个 CPU 先触发了 start()，它只需要把一个全局的 notifier_count 计数器 ｜=1 
// 变为奇数（表示当前有多个写者正在修改页表）。
// 写者（mm 侧）绝不阻塞： CPU 0 和 CPU 1 都可以各自开开心心地去修改自己的页表
// （这里不对？如果使用了 split_page_table_lock，则可以同时修改同一个 mm 内不同 va 对应的 pte），
// 不需要互相等待。可以参考 
/Documentation/mm/process_addrs.rst
/Documentation/mm/mmu_notifier.rst

// 压力给到读者（KVM/GPU 侧）： 此时，如果 KVM 尝试去读取 PTE 并建立 SPTE，
// 它发现 notifier_count > 0（有写者在搞事情），KVM 就会知道：
// “哦，现在有别的 CPU 正在拆房呢，我这时候建地基肯定会出问题。” 
// 于是 KVM 放弃本次操作，等一会儿再来重试（Collision-Retry）。
 
 /*
 * As a secondary function, holding the full write side also serves to prevent
 * writers for the itree, this is an optimization to avoid extra locking
 * during invalidate_range_start/end notifiers.
 *
 * The write side has two states, fully excluded:
 *  - mm->active_invalidate_ranges != 0      
 *  - subscriptions->invalidate_seq & 1 == True (odd)
 *  - some range on the mm_struct is being invalidated
 *  - the itree is not allowed to change     
 *
 * And partially excluded:
 *  - mm->active_invalidate_ranges != 0      
 *  - subscriptions->invalidate_seq & 1 == False (even)
 *  - some range on the mm_struct is being invalidated
 *  - the itree is allowed to change
 * Operations on notifier_subscriptions->invalidate_seq (under spinlock):
 *    seq |= 1  # Begin writing
 *    seq++     # Release the writing state  
 *    seq & 1   # True if a writer exists    
 *
 * The later state avoids some expensive work on inv_end in the common case of
 * no mmu_interval_notifier monitoring the VA.
 */
```

这里使用 seq 序列号的奇偶表示读写状态， seq 为奇数（odd） 代表此时写者正在修改页表中，偶数（even）表示写者已经修改完了。

上面的注释提到，write-side 有 ‘**fully excluded**’ 和 ‘**partially excluded**’ 两种状态，设计后面这个状态主要是为了优化 common case 里的 expensive work，具体就是指操作红黑树(应该是 interval tree)，这个状态的进入也比较隐晦，因为这两个状态并不是将一个长时间的操作拆成了两个状态，以便 partial exclude 状态在一定程度的并行运行。也就是说，在这个 invalidate 过程中，并不是有一段时间是 **fully excluded** ，剩下的是 **partially excluded**， 这里先留个悬念。。。

我们来看 invalidate_range_start 的流程

```c
__mmu_notifier_invalidate_range_start
-> if (subscriptions->has_itree)
   -> mn_itree_invalidate(subscriptions, range)
-> if (!hlist_empty(&subscriptions->list))
   -> mn_hlist_invalidate_range_start(subscriptions, range)

mn_itree_invalidate
    for (interval_sub =
             mn_itree_inv_start_range(subscriptions, range, &cur_seq);
         interval_sub;
         interval_sub = mn_itree_inv_next(interval_sub, range)) {
        ret = interval_sub->ops->invalidate(interval_sub, range,
                            cur_seq);
        ...
    }

mn_itree_inv_start_range(subscriptions, range, &seq)
-> subscriptions->active_invalidate_ranges++
-> node = interval_tree_iter_first(&subscriptions->itree, range->start, ..
// 这里很关键，只有当 node 存在时，才会将 seq 置为奇数，从而进入 fully excluded 状态
// 否则就是 partial excluded，表明当前要 invalidate 的 range ‘无 GPU 关心‘
-> if (node) subscriptions->invalidate_seq |= 1
-> *seq = subscriptions->invalidate_seq
```

现在可以来看 mn_itree_inv_end 做了什么

```c
static void mn_itree_inv_end(struct mmu_notifier_subscriptions *subscriptions)                               {
    spin_lock(&subscriptions->lock);
    if (--subscriptions->active_invalidate_ranges ||
        !mn_itree_is_invalidating(subscriptions)) { // seq 为奇数表示正在 invalidting， 有可能是偶数，表明正在进行
        spin_unlock(&subscriptions->lock);          // partial exclude 的 invalidating，也就不用考虑后续操作了
        return;
    }

    /* Make invalidate_seq even */
    subscriptions->invalidate_seq++;
    ...
//    处理 deferred_list，操作红黑树，这就是所谓的 common case 里的 expensive work
```

spin_lock 后面的 if 判断需要进行下面三种情况的判断：

1. 如果 active_invalidate_ranges > 0, 则还有未完成的 invalidate，此时还不能修改 itree，要提前退出

2. 如果 active_invalidate_ranges == 0，且 seq 为偶数，表示现在是 partial excluded 状态，无人修改 itree 和 deferred list，可以提前退出，跳过 耗时 的 itree 相关操作

3. 如果 active_invalidate_ranges == 0，且 seq 为奇数，表示现在是 fully excluded 状态，且是最后一个调用 invalidate_end 的操作，此时需要标记 invalidate_seq 为偶数，且要更新 itree

我们在前面提到过如何注册 mmu_interval_notifier，

```c
   -> __mmu_interval_notifier_insert(&bo->notifier, mm, subscriptions, start, leng, ops)
      -> init interval_sub->interval_tree.start
      -> init interval_sub->interval_tree.last = start + len -1
      -> some fine control logic
      -> interval_tree_insert(&interval_sub->interval_tree, &subscriptions->itree)
```

其中有一个  fine control logic

```c
    spin_lock(&subscriptions->lock);
    if (subscriptions->active_invalidate_ranges) {
        if (mn_itree_is_invalidating(subscriptions))
            // 此时为 fully exclude 状态，不允许修改 itree，
            // 先将 interval 插入 deferred_list
            hlist_add_head(&interval_sub->deferred_item,
                       &subscriptions->deferred_list);
        else {
            // 此时为 partial exclude 状态，可以直接更新 itree，
            // 但是也要设置 invalidate_seq 为奇数，
            // 切换到 fully exclude 状态
            subscriptions->invalidate_seq |= 1;
            interval_tree_insert(&interval_sub->interval_tree,
                         &subscriptions->itree);
        }
        interval_sub->invalidate_seq = subscriptions->invalidate_seq;
    } else {
        // 此时没有在 invalidate，可以直接更新 itree，
        // 但是为什么设置 invalidate_seq = subs->invalidate_seq - 1 ?
        // 在设计中，interval_sub->invalidate_seq 只能是奇数
        // invalidate_seq 如果和 subs->invalidate_seq 相等，则表明该 interval 正在被 invalidate，
        // 显然当前并没有在 invalidate，且 invalidate_seq 是个递增的数字，
        // 所以设置为 subs->invalidate_seq-1 表示这个 interval 没有在被 invalidate，
        // 那为啥不直接设置成 1？可能是担心刚好和 subs->invalidate_seq 相等，
        // 导致误判 ？
        interval_sub->invalidate_seq =
            subscriptions->invalidate_seq - 1;
        interval_tree_insert(&interval_sub->interval_tree,
                     &subscriptions->itree);
    }
    spin_unlock(&subscriptions->lock);
```

此时再看 mn_itree_inv_end 中的 expensive work

```c
    /*
     * The inv_end incorporates a deferred mechanism like rtnl_unlock().
     * Adds and removes are queued until the final inv_end happens then
     * they are progressed. This arrangement for tree updates is used to                                                               
     * avoid using a blocking lock during invalidate_range_start.
     */
    hlist_for_each_entry_safe(interval_sub, next,
                  &subscriptions->deferred_list,
                  deferred_item) {
        // RB_EMPTY_NODE 表示这个 interval_sub 不在红黑树中，所以需要插入到 itree 中
        if (RB_EMPTY_NODE(&interval_sub->interval_tree.rb))
            interval_tree_insert(&interval_sub->interval_tree,
                         &subscriptions->itree);
        else
        // 如果 interval_sub 已经在红黑树中了，表示这个节点是需要被删除的，具体可以
        // 参考 mmu_interval_notifier_remove 函数
            interval_tree_remove(&interval_sub->interval_tree,
                         &subscriptions->itree);
        hlist_del(&interval_sub->deferred_item);
    }
    spin_unlock(&subscriptions->lock);
```

**mmu_interval_read_begin** 是 read 操作的入口函数，Begin a read side critical section against a VA range，比如说 GPU 想去更新一段 VA 的页表地址到 GPU 中，在 GPU 驱动中，每个 bo 都对应一个 mmu_interval_notifier，则更新 bo 的映射前需要调用 mmu_interval_read_begin 获取当前 interval_notifier 的 seq，此函数中还会判断如果 seq 和 subs->seq 相等，则表明正在进行 invalidating 操作，此时显然不是跟新 GPU 页表的正确时机，会 ***挂起*** 当前进程，当 inv_end 结束后，再 wakeup 此进程。

```c
    spin_lock(&subscriptions->lock);
    /* Pairs with the WRITE_ONCE in mmu_interval_set_seq() */
    seq = READ_ONCE(interval_sub->invalidate_seq);
    is_invalidating = seq == subscriptions->invalidate_seq;
    spin_unlock(&subscriptions->lock);
    /*
     * interval_sub->invalidate_seq must always be set to an odd value via
     * mmu_interval_set_seq() using the provided cur_seq from
     * mn_itree_inv_start_range(). This ensures that if seq does wrap we
     * will always clear the below sleep in some reasonable time as
     * subscriptions->invalidate_seq is even in the idle state.
     */
    // 这个注释也很难理解，他这里提到的 wrap 应该是这样一种情况：
    // 假设区间 A 的 seq(即 interval seq)一直保持比如说 3， 但是全局 seq(即 subs->seq) 
    // 一直从 3 一路增加，在此期间从来没有动过区间 A 的 seq，那么如果全局 seq 发生了 wrap，
    // 从 0xfffffffff 跳到了 3，那么此时会错误的认为区间 A 在 invalidating，
    // 此时就会错误的 wait，不过这也不会无限制的等待下去，因为当变为 idle state 时，
    // 全局 seq 最终会变为偶数，且会 wakeup all。
    // 如果 区间 seq 可以为偶数呢？这个考虑起来比较费脑，
    // 先忽略吧。。。反正设计上不允许区间 seq 为偶数。
		if (is_invalidating)
        wait_event(subscriptions->wq,
               READ_ONCE(subscriptions->invalidate_seq) != seq);

> > I think this comment should be with the struct mmu_range_notifier
> > definition and you should just point to it from here as the same
> > comment would be useful down below.
> 
> I had it here because it is critical to understanding the wait_event
> and why it doesn't just block indefinitely, but yes this property
> comes up below too which refers back here.
> 
> Fundamentally this wait event is why this approach to keep an odd
> value in the mrn is used.
```

假如运气比较好，此时 mm core 没有在修改 pte，该函数会返回此 seq，之后 GPU 驱动读取 VA range 对应的 PA，待到后面要开始更新  GPU 页表时，需要通过 mmu_interval_read_retry 判断，seq 是不是还和 subscriptions->invalidate_seq 相等，如果不等，说明 mm core 又修改了 pte，此时需要放弃修改 GPU 页表，进行 retry 。

mmu_interval_read_begin 中有更详细的注释说明，区分了 collision-retry 机制中，正在 invalidating 和 已经被 invalidate 两种情况，这里就不重复解释了。

那么，会不会 itree 中的各个 mmu_interval_notifier node 的 seq 不一致？

完全有可能，比如下面的例子

```c
                  base     size
interval node A, 0x10000 + 0x100, seq 9
interval node B, 0x20000 + 0x100, seq 11
interval node C, 0x80000 + 0x100, seq 13

// 而 subs->seq 为 15，这表明，
// 在第 9  次时只修改了覆盖 node A 地址范围的页表
// 在第 11 次时只修改了覆盖 node B 地址范围的页表
// 在第 13 次时只修改了覆盖 node C 地址范围的页表
// 此时正在进行第 15 次修改，而这次的修改，并没有覆盖 node A、B、C
```

现在的机制能很好的处理上面这种情况。这应该也算一种优化。

/Documentation/mm/mmu_notifier.rst

```c
.. _mmu_notifier:

When do you need to notify inside page table lock ?
===================================================

When clearing a pte/pmd we are given a choice to notify the event through
(notify version of \*_clear_flush call mmu_notifier_invalidate_range) under
the page table lock. But that notification is not necessary in all cases.

For secondary TLB (non CPU TLB) like IOMMU TLB or device TLB (when device use
thing like ATS/PASID to get the IOMMU to walk the CPU page table to access a
process virtual address space). There is only 2 cases when you need to notify
those secondary TLB while holding page table lock when clearing a pte/pmd:

  A) page backing address is free before mmu_notifier_invalidate_range_end()
  B) a page table entry is updated to point to a new page (COW, write fault
     on zero page, __replace_page(), ...)

Case A is obvious you do not want to take the risk for the device to write to
a page that might now be used by some completely different task.

Case B is more subtle. For correctness it requires the following sequence to
happen:

  - take page table lock
  - clear page table entry and notify ([pmd/pte]p_huge_clear_flush_notify())
  - set page table entry to point to new page

```

## 举例说明

amdgpu 注册的 interval_notifier_ops 是

```c
static bool amdgpu_mn_invalidate_gfx(struct mmu_interval_notifier *mni,
                     const struct mmu_notifier_range *range,
                     unsigned long cur_seq)
{
    struct amdgpu_bo *bo = container_of(mni, struct amdgpu_bo, notifier);
    struct amdgpu_device *adev = amdgpu_ttm_adev(bo->tbo.bdev);
    long r;

    if (!mmu_notifier_range_blockable(range))
        return false;

    mutex_lock(&adev->notifier_lock);
    // 这个 cur_seq 是 mn_itree_invalidate 时，设置为 subs->seq
    mmu_interval_set_seq(mni, cur_seq);

    r = dma_resv_wait_timeout(bo->tbo.base.resv, DMA_RESV_USAGE_BOOKKEEP,
                  false, MAX_SCHEDULE_TIMEOUT);
    mutex_unlock(&adev->notifier_lock);
    if (r <= 0)
        DRM_ERROR("(%ld) failed to wait for user bo\n", r);                                                                            
    return true;
}
```

这个 adev->notifier_lock 会在 amdgpu_cs_submit 中使用，用来保护命令提交，在 GPU 命令提交时，不允许修改 bo 的 cpu 映射，如果要修改，请等 GPU 使用完再说。

**mmu_interval_read_begin** 函数中有如下注释：

![image-20260702162355358](/assets/gpu/image-20260702162355358.png)

这个图理解起来太费劲了，过于简化：

1. 我觉得框 1 内的代码应该是 3 处，才能表示 read 时，正在发生 colliding，此时，会挂起 GPU 驱动，直到 seq 变得不一样，也就是 mn_itree_inv_end 执行完之后，interval_sub->seq （奇数，比如 7）本来就和 subs->seq （偶数，比如 8）不一样，如果在 1 处，应该是 before collide，会在最后 read_retry 时，判断是否发生了 collide，在 2 处时，也一样，7 != 9，会判断为 before collidate。**mmu_interval_read_begin 只会在判断 seq 相等时，判定正在 colliding，需要 wait collide 结束**。
2. **user_lock** 的作用是什么？user_lock 作用应该是将驱动的 teardown 和 setup spte 工作串行化起来，他们不能同时进行，但是上图中完全没体现这一点，我猜测，在**左侧的 user_lock/unlock** 中需要 teardown BO 的 GPU page table，amdgpu driver 的操作就是更新 bo 的 mmu_interval_notifier->invalidate_seq 并 dma_resv_wait 其被 GPU 使用完（那何时销毁 pte？bo_unmap，或 close_bo 时）， **右侧的 user_lock/user_unlock** 应该是保护这种情况，“如果没有发生过 collide， GPU 在建立映射期间，免受 mm core 的 invalidate 打扰”。如果不相等，直接报错退出。但是我看实际的代码是，这个 user_lock 应该就是 **adev->notifier_lock**，其会在 amdgpu_cs_submit 中使用

```c
amdgpu_cs_submit
-> drm_sched_job_init
-> drm_sched_job_arm
-> mutex_lock(notifier_lock)
-> foreach userptr bo check if any bo has collided
   if yes, mutex_unlock, return -EAGAIN
-> ...
-> insert job's finish fence into related bo's dma_reservation
-> mutex_unlock(notifier_lock) 
```

## 参考引用

https://lists.freedesktop.org/archives/dri-devel/2019-November/243933.html

[A last-minute MMU notifier change](https://lwn.net/Articles/732952/)

https://docs.kernel.org/locking/seqlock.html

