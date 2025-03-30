---
layout: post
title:  "透过一个 bug 窥探 virtio packed-queue 机制"
date:   2025-3-30 9:00:16 +0800
categories: kernel 
---
**作者: flynn**

## 问题描述

在使用 virtio-gpu 设备时，虚拟机的画面会无缘无故卡死，通过查看 xorg 的栈，发现 virtio-gpu 驱动一直在 wait_event，如下图：

![img_v3_02k9_dcea0cfd-1c44-4aee-aafd-ac96c82e7e6g](/assets/linux/packed_vring_12.png)

## packed-queue 如何传递数据

packed-queue 的 descriptor table 成员如下所示：

```c
struct virtq_desc {
 le64 addr;
 le32 len;
 le16 id;
 le16 flags;
}
//FLAG 定义如下：
#define VRING_DESC_F_NEXT   		1
#define VRING_DESC_F_WRITE  		2
#define VRING_DESC_F_INDIRECT   	4
#define VRING_PACKED_DESC_F_AVAIL   7
#define VRING_PACKED_DESC_F_USED    15
```

没有额外的 available queue 和 used queue。

有了确定好的 descriptor table，再加上精细的控制，virtio_packed_queue 就可以工作起来了。

先来看个简单的例子， packed-queue 初始状态如下，除了 desc table 之外，还有一个 desc state 数据数组，用来实现对 free ID 的管理，其管理方式有点类似双向链表。画面中除了灰色框中的变量位于设备端，由设备使用，其余的都位于驱动端，由驱动使用。

### 1 初始状态

假设 packed queue 只有 8 个 entry

![image-20250326162317858](/assets/linux/packed_vring_0.png)

### 2 驱动提交命令

此时，如果驱动想提交一个 virtio-gpu 的命令，则会分配一个 virtio_gpu_vbuffer 结构体，假设该结构体虚拟地址为 **0x7000_0000**，

![image-20250326154939586](/assets/linux/packed_vring_1.png)

标号 2，3，4 对应的内存都需要生成 sg_table，并将其 dma_addr 放在 desc_table 中，以供虚拟设备使用，

![image-20250326155227340](/assets/linux/packed_vring_2.png)

这里为简单起见，假设 sgs[1] 中只有一项，则需要填入 desc_table 中的 dma_addr 分别为 0x8000_0000， 0x8500_0000，0x8800_0000，之后，notify 设备，在此之后，假设设备一次会把所有标记为 avail 的数据一次取出使用，则 desc_table 中的内容如下：

![image-20250326155601002](/assets/linux/packed_vring_3.png)

驱动做了如下事情：

- 通过 num_free 判断是否有足够的空间存放 sgs

- 在 next_avail_idx 指向的 desc table entry 开始依次填入 sgs table 对应的  dma_addr、len、id、flags， ID 由 free_head 确定，并更新 next_avail_idx 和 num_free 变量
- 在  free_head 指向 desc state table entry 中填入 **virtio_gpu_vbuffer** 对应的虚拟地址、占用的 entry 数、还有 last 信息， last 表示该 entry 尾部对应的 ID，用来在回收 ID 时，正确的处理这个 free list
- 当 avail_wrap_cnt 为 true 时，每个 entry 中的 flag A 表示此 entry 有效
- 将 queue idx 写入 VIRTIO_PCI_QUEUE_NOTIFY 寄存器，通知设备去使用这些 buffer

设备做了如下事情：

- 通过判断 last_avail_idx 指向的 desc table entry 中的 flag 判断该 entry 是否有效，判断标准是 A 和 U 标志位没有被同时设置，且 A 标志位的值等于 last_avail_idx_wrap_cnt，无效的话，表示 desc_table 为空，不能取 buffer
- 通过 in_use 来判断是否可以取出 buffer，如果 in_use >= queue_num，则表示 desc_table 中的 buffer 已经被全部取出来了，不需要再取了
- 取出所有 available 的 buffer，更新 last_avail_idx 指向下一次要取数据的 entry，在这里是 3，同时更新 in_use = 3
- 使用 buffer

### 3 驱动再次提交命令

此时，如果驱动再放入一个命令，id 为 3，占一个 desct entry，则更新如下：

![image-20250326161002178](/assets/linux/packed_vring_4.png)

此时分两种情况讨论

### 3.1 设备使用完 ID 为 3 的 buffer

![image-20250326180652324](/assets/linux/packed_vring_5.png)

设备做的事情：

- 设备使用完 buffer 后，将使用完的 buffer ID，填入 used_Idx 指向的 desc table entry，并设置 flags 为 `A|U`，并更新 `used_idx += 1`，同时也更新 `in_use -= 1`
- 发起一个中断，通知驱动去处理

驱动做的事情：

- 判断 last_used_idx 所指的 desc table entry 的 flags 是否标记已经被使用，在此情况是 `A|U`
- 根据 entry 中的 ID 对应到 desc state table 中，得到 virtio_gpu_vbuffer 的地址为 0x7100_0000，num 为 1，并更新 `last_used_idx += 1`，`num_free += 1`， free_head 为 3

### 3.2 设备使用完 ID 为 0 的 buffer

![image-20250326180823713](/assets/linux/packed_vring_6.png)

设备做的事情：

- 设备使用完 buffer 后，将使用完的 buffer ID，填入 used_Idx 指向的 desc table entry，并设置 flags 为 `A|U`，并更新 `used_idx += 3`，同时也更新 `in_use -= 3`
- 发起一个中断，通知驱动去处理

驱动做的事情：

- 判断 last_used_idx 所指的 desc table entry 的 flags 是否标记已经被使用，在此情况是 `A|U`
- 根据 entry 中的 ID 对应到 desc state table 中，得到 virtio_gpu_vbuffer 的地址为 0x7000_0000，num 为 3，并更新 `last_used_idx += 3`，`num_free += 3`， free_head 为 0

### 4 在 3.2 的情况下，驱动再次提交命令，ID 为 0，占 5 个 desc entry，则更新如下：

![image-20250326181135536](/assets/linux/packed_vring_7.png)

驱动做的工作和 2 中描述的基本一致，除了当 next_avail_idx = 8 时，需要进行回卷 next_avail_idx 为 0，设置 avail_wrap_counter = 0，并且设置 avail_used_flag 为 `!A|U` 表示该 entry 有效。

你可能会问，一会用 `!A|U` 表示该 entry 有效，一会又用 `A|!U` 表示有效，不会出现混乱吗？

其实并不会，因为对于驱动来说 next_avail_idx 和 last_used_idx 都有自己的 wrap_counter，用来标记对应 idx 是否出现了回卷；同样的，设备的 last_avail_idx 和 used_idx 也有各自的 wrap_counter。

驱动在想把某个 entry 设置为 avail 时，当  avail_wrap_counter 为 1 时，设置 `A | !U`, 否者设置 `!A | U`

驱动要判断某个 entry 是否为 used 时，只需要判断 `avail == used && used == used_wrap_counter`

设备要把某个 entry 设置为 used 时，当 used_idx_wrap_counter 为 1 时，设置 `A | U`, 否者设置 `!A | !U`

设备要把某个 entry 设置为 avail 时，只需要判断 `(avail != used) && (avail == last_avail_wrap_counter)`

这有点像在轮流写一页纸的正反两面，正面用 `A | !U ` 标记数据有效，`A | U` 表示数据已经被使用完毕，而反面用 `!A | U` 标记数据有效，`!A | !U` 表示数据已经被使用完毕，对于驱动或者设备来说，他们是清楚自己正在读写的是正面还是反面，于是就可以用正确的标记来处理数据。

### 5 驱动再次提交命令，ID 为 6，占 2 个 desct entry

![image-20250326181305774](/assets/linux/packed_vring_8.png)

此时，达到一种极端情况，表明 ring 已经满了，驱动无法再提交更多的命令，而设备也无法再 pop 更多的数据了。

此时，细心的你会发现，在上述的例子中，desc state table 中有很多 entry 是空的，而且标称有 8 个 entry 的 desc table 实际上可能发送的命令远远少于 8 个，这无疑造成了很多资源浪费，其实当一条 virtio 命令占用的 dest table entry 大于 1 个时，会使用 indirect desc，此时每个 entry 指向一个二级 desc table，而这个二级 table 则表示一个 virtio 命令，desc state table 中也不会有这么多空洞了， 具体实现，可以直接参考驱动 ***virtqueue_add_indirect_packed*** 函数。

## packed-queue 如何通知彼此

VIRTIO_RING_F_EVENT_IDX 这个 feature 可以让设备和驱动更精细的控制 notify（驱动通知设备） 和 interrupt（设备通知驱动），这需要借助一个结构体，vring_packed_desc_event

```c
struct vring_packed_desc_event{
    /* Descriptor Ring Change Event Offset/Wrap Counter. */
    __le16 off_wrap;
    /* Descriptor Ring Change Event Flags. */
    __le16 flags;
}
```

off_wrap 的最高位表示 wrap_bit，其余 0-15 bits 表示 descriptor table 中的 offset。

flags 有三个值，ENABLE，DISABLE，RING_EVENT_FLAGS_DESC，分别对应使能、禁用和 event 中断，这三者是互斥的。



驱动中 **vq->packed.vring.driver** 和 **vq->packed.vring.device** 分别对应两个 **vring_packed_desc_event** 结构体，前者被 driver 使用，用来控制 device 要不要产生中断；后者被 device 使用，用来控制驱动要不要发 notification，这两个结构体对应的 dma_addr 会在 set_vq 时通过写 PCI 的 **VIRTIO_MMIO_QUEUE_AVAIL** 和 **VIRTIO_MMIO_QUEUE_USED** 寄存器更新到设备端，建立共享内存。

### 简单的例子

我们来看个简单的例子，假如初始状态如下图：

![image-20250327172247061](/assets/linux/packed_vring_9.png)

此时驱动会设置

```c
vq->packed.vring.driver->off_wrap = last_used_idx | used_wrap_counter << 15;
// 此时是 0x8000
```

表示当设备把第 0 个 entry 标记为 used 时，就可以触发中断了。

设备这边的逻辑如下：

```c
// 读取驱动写的 vring_packed_desc_event 信息
vring_packed_event_read(vdev, &caches->avail, &e);
//获取上次的 signal idx
old = vq->signalled_used;
// 更新此次的 signal idx
new = vq->signalled_used = vq->used_idx;
// 判断
vring_packed_need_event(vq, vq->used_wrap_counter, e.off_wrap, new, old);
```

```c
static bool vring_packed_need_event(VirtQueue *vq, bool wrap,
                                    uint16_t off_wrap, uint16_t new,
                                    uint16_t old)
{   // off 是 int 类型，有可能是负数
    int off = off_wrap & ~(1 << 15);

    if (wrap != off_wrap >> 15)
        off -= vq->vring.num;
    return vring_need_event(off, new, old);
}

// 此处 event_idx 竟然又是 uint16_t 类型
static inline int vring_need_event(uint16_t event_idx, uint16_t new_idx, uint16_t old)
{
    // 为什么要这样写，new_idx 看似可以去掉，公式化简为 ‘old < event_idx + 1’ 行不行？
    // 在比较之前为什么要转换成 uint16_t 类型？
    return (uint16_t)(new_idx - event_idx - 1) < (uint16_t)(new_idx - old);
}
```

我觉得这段代码写的不友好，也许其中蕴藏着什么 dark magic 吧。

我们可以代入假定值验证一下其逻辑，假设设备的 vq->signalled_used 为 0，vq->used_idx 为 1，则

vring_need_event 要计算的就是

```c
(1 - 0 - 1) < (1-0)
       |
       V
     0 < 1
// 返回 true，表示需要产生中断
```

### 复杂的例子

假设驱动的 last_used_idx 为 7，且 used_wrap_counter 为 1，则

```c
vq->packed.vring.driver->off_wrap = last_used_idx | used_wrap_counter << 15;
// 此时是 0x8007
```

假设设备的 vq->signalled_used 为 7，vq->used_idx 为 0，used_wrap_counter = 0，则

```c
vring_need_event(-1, 0, 7)
-> (uint16_t)(0 - (-1) - 1) < (uint16_t)(0 - 7)
                     |
                     V
                   0 < 0xFFF9
// 返回 true，表示需要产生中断
```

是不是很神奇？竟然可以 work！

我们看个真实的例子，这个 packed ring 有 16 个 entry，

驱动设置 vq->packed.vring.driver->off_wrap 为 0x2800f

![image-20250327175626764](/assets/linux/packed_vring_10.png)

在设备将最后一个 entry 标记为 used 之后的内存布局，

![image-20250327175830512](/assets/linux/packed_vring_11.png)

## 回到最初的问题

通过下图可以看出，驱动设置的 vq->packed.vring.driver->off_wrap 为 0x20008

![img_v3_02kg_64d33852-6ec7-461d-8ce5-ce5bce9ac1cg](/assets/linux/packed_vring_13.png)

通过下图可以看出，desc table 其实已经全部被标记为 used 了，按理说驱动应该可以继续下发命令。

![8BOHcE0ykr](/assets/linux/packed_vring_14.png)

为什么会产生这种情况呢？这是因为设备端有如下的代码：

```c
    for (;;) {
        elem = virtqueue_pop(vq, sizeof(VirtQueueElement));
        s = iov_to_buf(elem->out_sg, elem->out_num, 0,
                       &cursor_info, sizeof(cursor_info));
        update_cursor(g, &cursor_info);
        virtqueue_push(vq, elem, 0);
        // 出于某些原因，schedule a qemu bh to execute virtio_notify
    }
```

这段代码在运行时会出现一种极端情况，就是设备在标记所有 desc table 都为 used 之后，才去调用 virtio_notify。我们可以推测在 for 循环执行之前，设备端的状态如下：

```c
0  A                                          A|U
1  A                                          A|U
2  A                                          A|U
3  A                                          A|U
4  A                                          A|U
5  A                                          A|U
6  A                                          A|U
7  A                                          A|U
8 !A|U                          ->           !A|!U
9 !A|U                                       !A|!U
a !A|U                                       !A|!U
b !A|U                                       !A|!U
c !A|U                                       !A|!U
d !A|U                                       !A|!U
e !A|U                                       !A|!U
f !A|U                                       !A|!U
      
used_idx = 8                               used_idx = 8
signal_used = 8                            signal_used = 8
used_wrap_count = false                    used_wrap_count = true
```

根据这种情形，我们再来分析一下 **vring_packed_need_event** 的逻辑，

```c
vring_need_event(-8, 8, 8)
-> (uint16_t)(8 - (-8) - 1) < (uint16_t)(8 - 8)
                     |
                     V
                  15 < 0
// 返回 false，表示不产生中断 ！
```

可以看出，此时 **vring_need_event** 就会计算出错误的值，从而漏掉了一次中断，导致驱动一直在死等待

那么，知道了为何会卡死，解决起来就很容易了，直接在 **virtqueue_push** 之后，立马调用 **virtio_notify** 就可以了。

## 参考引用

1. [packed-virtqueue-how-reduce-overhead-virtio](https://www.redhat.com/en/blog/packed-virtqueue-how-reduce-overhead-virtio)
2. [virtio-spec v1.2](https://docs.oasis-open.org/virtio/virtio/v1.2/csd01/virtio-v1.2-csd01.html)