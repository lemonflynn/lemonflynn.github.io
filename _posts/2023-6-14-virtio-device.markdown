---
layout: post
title:  "virtio device"
date:   2023-6-14 8:00:16 +0800
categories: virtualization 
---

# virtio device

**作者: flynn**

---

# 1. 简介

virtio 一种半虚拟化技术，一般由 virtio 设备和 virtio 驱动构成

![1](/assets/qemu/qemu_virtio_01.png)

# 2. 驱动的整体框架

![1](/assets/qemu/qemu_virtio_02.png)

## 驱动的主要数据结构

![1](/assets/qemu/qemu_virtio_03.svg)

我们可以使用 virtio 设备的发现过程来串联整个数据结构

## Virtio 的设备发现过程

1. 内核使用 module_virtio_driver（）宏来注册各种 virtio 设备的驱动，例如virtio_balloon， virtio_gpu， 将这些驱动都挂载在虚拟的 virtio-bus 总线上，静静的等待被调用。

2. 内核会创建一个 virtio_pci_driver, 并注册到 pci 总线上，如此以来，在开机设备枚举的过程中，如果设备 id 符合 virtio_pci_driver 中填写的 id_table，则驱动的 probe 函数会执行。

3. virtio_pci_driver 的 probe 函数主要会创建一个 virtio_pci_device 结构体 vp_dev，并初始化其中和 pci 设备相关的变量，并且设置 vp_dev.vdev 中的 vendor id 和 device id,  并调用register_virtio_device(&vp_dev->vdev)，将 vp_dev->vdev 注册到虚拟的 virtio-bus 上。

4. 上一步中 register_virtio_device 注册过程，会触发 virtio-bus 的 rescan 操作，将 virtio 设备和正确的驱动绑定，接着会触发 virtio-bus 的 probe 函数，该函数会进行一些 feature 的设置，最终调用 virtio 设备驱动程序的 probe 函数，例如 virtballoon_probe， 来进行和设备相关的初始化。

## Virtio 共享内存的创建过程

1. 驱动首先向偏移为 VIRTIO_PCI_QUEUE_SEL 的 config space 写入需要创建的 QUEUE 的索引

2. 驱动读取 VIRTIO_PCI_QUEUE_NUM 寄存器，得到 QUEUE 的长度信息

3. 驱动使用 dma_alloc_coherent 函数分配一块内存，驱动使用这块内存的虚拟地址来创建 available ring、descriptor table、used ring，并将该内存对应的 dma 地址写入寄存器 VIRTIO_PCI_QUEUE_PFN

4. 虚拟设备在处理寄存器 VIRTIO_PCI_QUEUE_PFN 的 iowrite时，会根据 dma 地址来初始化对应的三个 ring

# 3. QEMU 如何实现 virtio 设备

## 虚拟总线

QEMU 首先会创建一个 VIRTIO BUS 类型的虚拟总线，然后由 VIRTIO_PCI_BUS 或 VIRTIO_MMIO_BUS 来实现这个虚拟总线，这也意味着，对于虚拟机操作系统来说，他会看到一个 pci 或 mmio 设备，并由对应的驱动来 probe.

![1](/assets/qemu/qemu_virtio_04.svg)

## 虚拟设备

QEMU 中大部分 virito 设备都是基于 pci 总线实现的，例如 TYPE_VIRTIO_BALLOON_PCI 的父类是 TYPE_VIRTIO_PCI， 后者的父类是 TYPE_PCI_DEVICE.

以 TYPE_VIRTIO_BALLOON_PCI 设备为例，QEMU 会为创建一个 VirtIOBalloonPCI 结构体来管理该设备，其中包含了一个 VirtIOPCIProxy 类型的变量，该变量对应其父类 TYPE_VIRTIO_PCI， 主要用来进行和 pci 总线相关的操作，例如中断，bar 地址空间的映射管理，配置空间的读写等.

![1](/assets/qemu/qemu_virtio_05.svg)

每个 virtio 设备都至少有一个 virtqueue 用来和 guest 传递数据

![1](/assets/qemu/qemu_virtio_06.svg)

# 4. virtqueue 的操作流程

virtqueue 主要由以下三个部件组成
- Available ring: 驱动程序写，虚拟设备读的区域
- Used ring: 虚拟设备写，驱动读的区域
- Descriptor table: 设备和驱动都可以访问的共享内存

我们来看一个例子：

![1](/assets/qemu/qemu_virtio_07.png)

1. 填写 descriptor table 的第 0 和第 1 项，这两项是连接在一起的，其中的 address 是虚拟机物理地址

![1](/assets/qemu/qemu_virtio_08.png)

2. 此时available ring 中的 index 为 0， 驱动程序把刚刚填入 descriptor table 的索引 0 填入 avaible ring[index]，并递增 index 并通过写 pci 设备的特定内存区域来通知设备
3. 当虚拟设备收到驱动的通知后，一般会使用 virtqueue_pop 来从 descriptor table 中获取 buffer 信息，设备端会使用一个 last_avail_idx 来记录这次需要从 descriptor table 的何处来取数据，如果 last_avail_idx 和 available ring 中的 index 一致，则说明没有新的数据，否则就从 descriptor table 的特定位置取 iov 信息

![1](/assets/qemu/qemu_virtio_09.png)

4. 同样的，当虚拟设备使用完 buffer 后，会更新 used ring， 并触发 pci 中断来通知驱动程序
5. 驱动程序会使用 last_used _idx 来记录需要从 descriptor table 的何处来取数据，用法和 available ring 类似

# 5. 虚拟设备如何使用 virtqueue

virtio 设备在从 ring 中取数据时，会先将元数据保存在 VirtQueueElement 结构体中

```
typedef struct VirtQueueElement
{
    unsigned int index;
    unsigned int len;
    unsigned int ndescs;
    unsigned int out_num;
    unsigned int in_num;
    hwaddr *in_addr;
    hwaddr *out_addr;
    struct iovec *in_sg;
    struct iovec *out_sg;
} VirtQueueElement;
```

如果数据比较简单，则按照如下方式使用:

1. elem = virtqueue_pop(vq, sizeof(VirtQueueElement))
2. s = iov_to_buf(elem->out_sg, elem->out_num, 0, &cursor_info, sizeof(cursor_info))
3. virtqueue_push(vq, elem, 0)
4. virtio_notify
5. free(elem)

第一步是从 virtqueue 中取出所有可用的 sg buffer ( 包括 in 和 out ) 的地址和长度信息，分配的内存大小由第二个参数指定；第二步是将 iov 中的数据拷贝到一个临时变量中使用；第三步则调用 virtqueue_push 来更新 virtqueue 内部的相关变量，表示数据已经被使用

假如数据相对比较复杂，例如是需要处理 virtiogpu cmd 命令，则可以定义一个和设备命令相关的结构体，然后使用宏 ```VIRTIO_GPU_FILL_CMD``` 来从 iov 中提取数据

```
struct virtio_gpu_ctrl_command {
    VirtQueueElement elem;
    VirtQueue *vq;
    struct virtio_gpu_ctrl_hdr cmd_hdr;
    uint32_t error;
    bool finished;
    QTAILQ_ENTRY(virtio_gpu_ctrl_command) next;
};
```

命令处理如下：

1. cmd = virtqueue_pop(vq, sizeof(struct virtio_gpu_ctrl_command))
2. VIRTIO_GPU_FILL_CMD(cmd->cmd_hdr)
3. 根据 cmd_hdr.type 命令类型来调用不同的解析函数，例如 virgl_cmd_context_create
4. 不同的命令来提取对应的数据，例如 VIRTIO_GPU_FILL_CMD(cc)， 是从 iov 中拷贝数据到类型为 ```struct virtio_gpu_ctx_create``` 的变量 cc 中
5. 调用 ```virgl_renderer_context_create(cc.hdr.ctx_id, cc.nlen, cc.debug_name);```
6. virtqueue_push
7. virtio_notify
8. g_free(cmd)

* 第一步 virtqueue_pop 中有个神奇的操作，注意第二个参数的值明显和 VirtQueueElement 结构体的大小不等，那 VirtQueueElement 中 *in_addr, *out_addr, *in_sg, *out_sg 指针应该指向哪一块内存区呢？ 答案是在 virtio_gpu_ctrl_command 后面，如下图所示：

![1](/assets/qemu/qemu_virtio_10.png)

所以 virtqueue_pop 分配的内存是大于第二个参数指定的，多余分配的内存用来存放 in_addr, out_addr, in_sg, out_sg 数组

# 6. 参考引用

1. [Virtio: An I/O virtualization framework for Linux](https://developer.ibm.com/articles/l-virtio/)
2. [virtqueues-and-virtio-ring-how-data-travels](https://www.redhat.com/en/blog/virtqueues-and-virtio-ring-how-data-travels)