---
layout: post
title:  "TTM Overview"
date:   2023-6-28 7:00:16 +0800
categories: kernel 
---
**作者: flynn**

## 1. TTM 出现的背景

早期的显卡使用 AGP (accelerate graphic port) 接口，这是一个基于 PCI 改造而成的接口，如下图：

<img src="/assets/linux/ttm_2.png" style="zoom:30%;" />

系统将 AGP 的 vram 映射到系统 ram 之上的地址空间，这样 CPU 就可以直接访问 AGP vram，大致流程如下：

1. CPU 发起一个访存（位于 AGP 显卡）请求
2. 北桥芯片对地址进行解析，发现内存属于显卡，则通过 AGP 接口去访问显卡
3. 内存内容经过 AGP 接口，北桥芯片后返回给 CPU

<img src="/assets/linux/ttm_1.png" alt="ttm_1" style="zoom:30%;" />

早期显卡的显存都是很小的，所以当出现显存不够用时，显卡也可以访问系统内存，那就必须要通过 AGP aperture 来访问，如下图，一个古早的 bios 配置界面：

<img src="/assets/linux/ttm_3.png" alt="ttm_3" style="zoom:30%;" />

为了让 GPU 能访问系统内存，这个内存也不能直接划分给显卡专属使用，否则会造成资源浪费，所以需要一个映射表，可以将 GPU 地址映射到不同的系统内存，简单理解，就是让 GPU 以为这段内存是连续的，其实真实的系统物理内存可能是不连续的，这个映射表就是 GART (graphic address remaping table)，早期这个 GART 映射单元位于北桥中，后来逐渐被放到了显卡上，现在的显卡会有自己的 MMU 来管理 GPU 地址，这个当然也就包含了现在 GART 的功能。

<img src="/assets/linux/ttm_4.png" alt="ttm_4" style="zoom:30%;" />

## 2. TTM 显存管理

如此一来，内存就被分成了三个区域：

1. 系统内存
2. GPU 可以访问的系统内存
3. 显卡自身的显存

对于 CPU 来说，我们有伙伴系统和 slab 机制来管理系统物理内存的分配和释放等，但是对于 GPU 来说，在TTM出现以前，没有一个比较好的内存子系统，TTM 的出现就是为了弥补这一缺口。

TTM 对内存域有如下定义:

|    内存域     |   位置   |   访问    |
| :-----------: | :------: | :-------: |
| TTM_PL_SYSTEM | 系统内存 |    CPU    |
|   TTM_PL_TT   | 系统内存 | CPU & GPU |
|  TTM_PL_VRAM  | GPU 显存 | CPU & GPU |

GPU 一般使用 BO (buffer oject) 来管理显存资源，所以对显卡上的显存（以及系统内存）资源， TTM 抽象出一个 **ttm_bo_device** 设备来管理。以下的分析是基于 linux 2.6.32，以及 radeon 显卡驱动。

<img src="/assets/linux/ttm_5.png" alt="ttm_5" style="zoom:40%;" />

**ttm_bo_driver**: 是由具体的显卡驱动实现，主要是负责 bo 的移动 (**这是什么意思，请看下文**) 和 fence  (**本文暂不涉及**) 的实现。

**addr_space_mm**: 用来管理设备的地址空间

**man**: 用来管理不同类型内存域的地址空间，一般用来管理 VRAM 和 GTT



假设 vram 大小为 2G，GTT 大小为 512M，那现在就有了如下三个地址空间：

<img src="/assets/linux/ttm_6.png" alt="ttm_6" style="zoom:30%;" />

1. 设备的地址空间：起始地址从 0x1_0000_0000 开始
2. GTT 地址空间：起始地址从 0 开始，共 512M，gpu_offset 表示对应的 GPU 起始地址，io_offset 表示对应的 io 起始地址（CPU 使用该地址），这里为 0
3. VRAM 地址空间：起始地址从 0 开始，共 2G，这里 io_offset 是对 GPU 上的显存 ioremap 后得到的地址

上面三个地址空间都会由各自的 drm_mm 结构来管理，现在来看一下 mc.gtt_location 和 mc.vram_location 的关系，以下是站在 **GPU** 的视角所看到的内存分布：

~~~c
0x9fffffff |---------------|
           |               |
           |     GTT       |
           |---------------|
           |indirect buffer|  
           |---------------|  
           |  ring buffer  |
0x7fffffff |---------------|<--------rdev->mc.gtt_location
           |     bios      |
           |---------------|
           |  gtt table    |
           |---------------|
           |               |
           |     vram      |
           |               |
           |---------------|
           |  framebuffer  |
   0x0     |---------------|<--------rdev->mc.vram_location

~~~

GPU 在访问内存时，首先判断该内存是否属于 GTT 内存，如果不是，那么就是 VRAM，可以直接通过 GPU 上的内存控制器访问，如果是 GTT 内存，则减去 GTT 基地址后，通过 GART 查到系统内存的地址，之后经过 pcie 总线去访问。

<img src="/assets/linux/ttm_7.png" alt="ttm_7" style="zoom:30%;" />

假设 GPU 采用的页表大小为 4k，一级页表，则管理 512M 的系统内存共需要 128k 个页目录项，每项 4 个字节，则页目录表共 512k，假设 GPU 要访问从 0x8010_1000 地址开始，大小为 16k 的内存，那由于这个地址属于 GTT 内存，所以，首先减去一个 GTT 基地址（0x8000_0000），得到 0x10_1000 ，根据这个地址去查找 gtt table 中的第 0x101 项（地址为 0x404），则该页表项中的地址就是设备可以访问系统内存的 io 地址，该地址是经过 **pci_map_page** 后得到的，后面会有更详细的解释。



<img src="/assets/linux/ttm_8.png" alt="ttm_8" style="zoom:30%;" />

既然 GPU 是以 BO 来管理显存资源，那我们来看看如何表示一个 **buffer object**。

<img src="/assets/linux/ttm_9.png" alt="ttm_9" style="zoom:50%;" />

```
ttm_buffer_object: 表示一个 bo
ttm_tt: 如果这个 bo 位于系统内存域或者 GTT 内存域，则需要为这个 bo 分配系统内存和映射，这个结构体就是用来管理系统内存的分配和映射的
ttm_mem_reg: 表示这个 bo 的大小和位置，以及所属内存域
drm_mm_node: 表示这个 bo 在 drm_mm 管理的地址空间中的节点，其中位于 ttm_buffer_object 结构中的 vm_node 表示这个 bo 在 GPU 设备地址空间中的位置，位于 ttm_mem_reg 中的 mm_node，表示 bo 在 GTT 或 VRAM 地址空间中的位置节点
ttm_backend_func: 是由驱动提供的回调函数，主要用来更新 gtt table，由于各家 GPU 对 gtt table 的定义会有差异，所以必须由驱动来提供
```

由于一个 bo 可能会位于不同的内存域，且可以动态的移动，比如可以从 **系统内存域** 移动到 **GTT 内存域** ，或者移动到 **VRAM 内存域**，为了方便管理，每个 bo 都有一个状态机来管理处于不同内存域的 bo ，有如下三个状态：

- **unpopulate**:  bo 创建的初始状态，还没有分配对应的内存页
- **ubound**: 已经为 bo 分配了位于系统内存的内存页，但是映射关系还没建立
- **bound**: 映射关系已建立，GPU 可以通过 gart 访问 bo 的内存页

状态之间的转换如下：

```c
                           ttm_tt_create
                                |
                                v
                          --------------
                          | unpopulate |
                          --------------
                           ^         |
  ttm_free_allocated_pages |         |ttm_tt_populate
                           |         |
                           |         v
                          --------------                   --------------
                          |            | ---ttm_tt_bind--> |            |
                          |  unbound   |                   |   bound    |
                          |            | <--ttm_tt_unbind--|            |
                          --------------                   --------------     
```

这里还得再提一个概念，ttm 对 bo 定义了如下三种类型：

1. ttm_bo_type_device  : 位于 GPU 设备地址空间，可以被映射到用户空间
2. ttm_bo_type_kernel  : 和 1 的差别是，仅限于内核自己使用
3. ttm_bo_type_user     : 属于用户分配的内存，经过 GART 映射后，GPU 通过 aperture 访问

我们来看一下，如何创建一个 bo，我们对位于**系统内存域**、**GTT 内存域** 和 **VRAM 内存域**的 bo 分别分析。

### 位于系统内存

在这里我们创建一个**类型**为 ttm_bo_type_device，其大体流程如下：

1. 在 GPU 设备的地址空间 (addr_space_mm) 查找一个可用的地址区间
2. 分配 ttm_tt 结构体，并初始化 pages 页表指针数组
3. 此时 bo 的状态为 unpopulate

其填充的结构如下图：

<img src="/assets/linux/ttm_10.png" alt="ttm_10" style="zoom:33%;" />

由于 bo 位于系统内存，GPU 不需要也不能访问该 bo，所以 mem.mm_node 为空，表明该 bo 没有 gtt 或者 vram 地址。

### 位于 GTT 内存域

创建位于 GTT 内存域的操作就会复杂很多，直接看图：

<img src="/assets/linux/ttm_11.png" alt="ttm_11" style="zoom:33%;" />

这里主要的操作是在 ttm_bo_validate 函数中完成，该函数会判断 bo 是否处于最终的内存域，如果不是，就要将 bo 移动到指定的内存域，在这个例子中，就是需要将 bo 从初识的 system ram 域移动到 gtt 内存域，创建结束后，地址空间的使用情况如下：

<img src="/assets/linux/ttm_12.png" alt="ttm_12" style="zoom:33%;" />

假设这个 bo 在设备地址空间中的起始地址是 0x1_0001_0000，在 gtt 内存域的起始地址是 0x105_0000，则 bo 中对应的变量值如下：

```c
bo.mem.mm_node->start = 0x105_0000
//offset = bo.mem.mm_node->start + bdev->man[GTT].manager.gpu_offset
bo.offset = 0x8105_0000
bo.vm_node->start = 0x1_0001_0000
```



<img src="/assets/linux/ttm_13.png" alt="ttm_13" style="zoom:33%;" />

### 位于 VRAM 内存域

创建位于 vram 内存域的 bo 最为复杂，因为首先创建的 bo 是位于 system ram 的，需要先将 bo 从 system ram 移动到 gtt 内存域，再从 gtt 内存域移动到 vram 内存域，这里，你可能会问，为什么不能直接把 bo 从 system ram 移动到 vram 呢？我猜测这是因为如果是让 CPU 把 bo 拷贝到 vram，则会占用 CPU 的时间，所以一般会使用 GPU 中的 DMA 来完成数据的搬运，但是 GPU 又不能直接访问 system ram，所以就需要把 bo 先拷贝到 GTT 内存域中，这样 GPU 就可以用 DMA 引擎将数据拷贝到 VRAM 中。

<img src="/assets/linux/ttm_14.png" alt="ttm_14" style="zoom:33%;" />

同时，也可以注意到，在将 bo 移动到 vram 内存域后，bo 所管理的 system ram 就会被释放，ttm_tt 结构体也就没有存在的理由了，```bo->ttm``` 被设置为 null。

### 应用程序访问 bo

在创建了 bo 之后，用户态驱动或者程序就需要去访问这个 bo 资源，我们来看一个例子：

<img src="/assets/linux/ttm_15.png" alt="ttm_15" style="zoom:40%;" />

这是用户态驱动程序 mesa 中 winsys 中的一个函数，其中绿色的 fd 是 rendernode 对应的 fd，linux 图形子系统会为每个 bo 分配一个 handler，首先通过调用 DRM_RADEON_GEM_MMAP 后，就可以得到这个 bo 的设备地址，也就是 args.addr_ptr ，值就是 0x1_0001_0000，再经过 mmap 后得到一个用户虚拟地址 0x6000_0000，现在，用户态程序就可以通过 0x6000_0000 这个地址来读写 bo 了。

那 mmap 到底做了哪些工作？请看图：

<img src="/assets/linux/ttm_16.png" alt="ttm_16" style="zoom:40%;" />

Linux 内存子系统在用户虚拟地址空间分配一个可用的 vma，然后调用 radeon_mmap，细心的朋友可能已经发现，GPU 设备地址空间是从 4G 开始的，所以对于 offset 低于 4G 的地址，直接调用 drm 框架的通用 mmap 函数 （drm_mmap），如果 offset 落在了 GPU 设备地址空间，根据 offset 找到对应的 bo，将 bo 保存在 vma->vm_private_data 中，最后更新 vma->vm_ops， 其中最主要的就是 fault 函数，ttm_bo_vm_fault 函数完成了主要的映射建立关系，这里就不再过多解释了：

<img src="/assets/linux/ttm_17.png" alt="ttm_17" style="zoom:40%;" />

下面以 bo 位于 GTT 内存域为例来说明，建立好映射关系后，GPU 就可以使用 0x8105_0000 这个地址来访问系统内存，而 CPU 使用 0x6000_0000 这个地址来访问同一个内存；同理，当 bo 位于 vram 域时，也就不难理解了。

<img src="/assets/linux/ttm_18.png" alt="ttm_18" style="zoom:25%;" />

TTM 框架中实现了以下几个重要的函数，来让 GPU 驱动使用：

```c
ttm_bo_kmap: 映射 bo，得到一个内核虚拟地址
ttm_bo_pin: 将 bo 固定到某个内存域，并得到 GPU 地址
ttm_bo_reserve: 相当于一个锁，防止 CPU 多线程同时访问 bo
```

下图是DRM 中 TTM 和 radeon 驱动程序中各组件的关系。

```c
   ---------------------------------------------------------
   | DRM                                                   | 
   |                    radeon                             |
   |   ---------        ---------------------------------  |
   |   |       |        |                               |  |
   |   |       |        |   -------------------------   |  |
   |   |       |        |   |  radeon_bo_kmap       |   |  |
   |   |       |        |   |  radeon_bo_pin        |   |  |
   |   |       |<-----------|  radeon_bo_evict_vram |   |  |
   |   |       |        |   |  radeon_ttm_init      |   |  |
   |   |       |        |   |  radeon_ttm_fault     |   |  |
   |   |       |        |   |  radeon_ttm_mmap      |   |  |
   |   |       |        |   -------------------------   |  |
   |   |       |        |                               |  |
   |   |       |        |      ttm_bo_driver            |  |
   |   |       |        |   ----------------------------|  |
   |   |       |        |   | .create_ttm_backend_entry||  |
   |   |  TTM  |        |   | .invalidate_cache        ||  |
   |   |       |----------->| .init_mem_type           ||  |
   |   |       |        |   | .move                    ||  |
   |   |       |        |   | .verify_access           ||  |
   |   |       |        |   | ...                      ||  |
   |   |       |        |   ----------------------------|  |
   |   |       |        |                               |  |
   |   |       |        |   ttm_backend_func            |  |
   |   |       |        |   ------------                |  |
   |   |       |        |   | .populate|                |  |
   |   |       |        |   | .clear   |                |  |
   |   |       |----------->| .bind    |                |  |
   |   |       |        |   | .unbind  |                |  |
   |   |       |        |   | .destroy |                |  |
   |   |       |        |   ------------                |  |
   |   ---------        ---------------------------------  |
   |                                                       |
   --------------------------------------------------------
```



## 3. 与 MESA 的关系

MESA 对渲染管线的资源有如下的分类:

- **PIPE_USAGE_DEFAULT**: Optimized for fast GPU access.

- **PIPE_USAGE_IMMUTABLE**: Optimized for fast GPU access and the resource is not expected to be mapped or changed (even by the GPU) after the first upload.

- **PIPE_USAGE_DYNAMIC**: Expect frequent write-only CPU access. What is uploaded is expected to be used at least several times by the GPU.

- **PIPE_USAGE_STREAM**: Expect frequent write-only CPU access. What is uploaded is expected to be used only once by the GPU.

- **PIPE_USAGE_STAGING**: Optimized for fast CPU access.

Radeon 驱动会把 default 和 immutable 的资源认为是存放在 vram 上，而 stream 和 staging 是放在 gtt 内存域。

在 mesa  winsys 模块中，和 bo 相关的函数在这里初始化：

```
void radeon_drm_bo_init_functions(struct radeon_drm_winsys *ws)
{   
    ws->base.buffer_set_metadata = radeon_bo_set_metadata;
    ws->base.buffer_get_metadata = radeon_bo_get_metadata;
    ws->base.buffer_map = radeon_bo_map;
    ws->base.buffer_unmap = radeon_bo_unmap;
    ws->base.buffer_wait = radeon_bo_wait;
    ws->base.buffer_create = radeon_winsys_bo_create;                                                                                                   
    ws->base.buffer_from_handle = radeon_winsys_bo_from_handle;
    ws->base.buffer_from_ptr = radeon_winsys_bo_from_ptr;
    ws->base.buffer_is_user_ptr = radeon_winsys_bo_is_user_ptr;
    ws->base.buffer_get_handle = radeon_winsys_bo_get_handle;
    ws->base.buffer_get_virtual_address = radeon_winsys_bo_va;
    ws->base.buffer_get_reloc_offset = radeon_winsys_bo_get_reloc_offset;
    ws->base.buffer_get_initial_domain = radeon_bo_get_initial_domain;
}
```

当 state tracker 模块需要分配一个资源时，会一次穿过 driver 和 winsys，最终通过 drmCommand 来发送命令到 linux DRM 子系统：

<img src="/assets/linux/ttm_19.png" alt="ttm_19" style="zoom:33%;" />

熟悉 OpenGL 的朋友，一定对 **glBuffefData** 函数不陌生，这个函数定义如下：

```c
void glBufferData(GLenum target,
 	GLsizeiptr size,
 	const void * data,
 	GLenum usage);
```

这个函数主要的作用是将 data 指针指向的内存拷贝到绑定在特定 target 上的 buffer object 中，整体流程如下：

<img src="/assets/linux/ttm_20.png" alt="ttm_20" style="zoom:33%;" />

这里的 ```ws->buffer_map``` 就是上文分析过的 ```radeon_bo_do_map```。

## 4. 总结

相信看到这里，大家对以下几个概念应该就不会陌生了吧。

```
AGP
AGP aperture
VRAM
GART
TTM
GTT
```

## 5. 参考引用

1. [Accelerated_Graphics_Port](https://en.wikipedia.org/wiki/Accelerated_Graphics_Port)
2. [System address map initialization in x86/x64 architecture part 1: PCI-based systems](https://resources.infosecinstitute.com/topic/system-address-map-initialization-in-x86x64-architecture-part-1-pci-based-systems/)
3. [Memory management for graphic processors](https://lwn.net/Articles/257417/)
4. [DRM memory manager core](https://lwn.net/Articles/257230/)
