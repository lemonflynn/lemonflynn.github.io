---
layout: post
title:  "slab 内存管理器"
date:   2023-6-26 7:00:16 +0800
categories: kernel 
---
**作者: flynn**

slab 是一个对象缓存管理器，他可以将相同类型的数据缓存起来，当要分配一个该类型的对象（用对象更好理解）时，直接从 slab 中分配，而不是直接调用 kmalloc， 同样的，释放该对象时，也是将该对象返回给 slab。slab 适用于经常分配并释放的对象。

可以使用 cat /proc/slabinfo 查看当前系统中 slab 的相关信息。

```
# name     <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
...
shmem_inode_cache   7344   7920    720   22    4 : tunables    0    0    0 : slabdata    360    360      0
kernfs_node_cache 156903 158610    136   30    1 : tunables    0    0    0 : slabdata   5287   5287      0
mnt_cache           3054   3175    320   25    2 : tunables    0    0    0 : slabdata    127    127      0
filp               44287  50912    256   16    1 : tunables    0    0    0 : slabdata   3182   3182      0
inode_cache        33329  35126    608   26    4 : tunables    0    0    0 : slabdata   1351   1351      0
dentry            375643 470232    192   21    1 : tunables    0    0    0 : slabdata  22392  22392      0
names_cache           80     80   4096    8    8 : tunables    0    0    0 : slabdata     10     10      0
iint_cache             0      0    120   34    1 : tunables    0    0    0 : slabdata      0      0      0
lsm_file_cache     23377  27030     24  170    1 : tunables    0    0    0 : slabdata    159    159      0
buffer_head       1076593 1156311    104   39    1 : tunables    0    0    0 : slabdata  29649  29649      0
uts_namespace        198    198    440   18    2 : tunables    0    0    0 : slabdata     11     11      0
nsproxy             1022   1022     56   73    1 : tunables    0    0    0 : slabdata     14     14      0
vm_area_struct    143193 164749    208   19    1 : tunables    0    0    0 : slabdata   8671   8671      0
mm_struct           5640   5640   1088   30    8 : tunables    0    0    0 : slabdata    188    188      0
files_cache         4577   4577    704   23    4 : tunables    0    0    0 : slabdata    199    199      0
..

kmalloc-96         11569  12432     96   42    1 : tunables    0    0    0 : slabdata    296    296      0
kmalloc-64         73845  77504     64   64    1 : tunables    0    0    0 : slabdata   1211   1211      0
kmalloc-32        132704 136448     32  128    1 : tunables    0    0    0 : slabdata   1066   1066      0
kmalloc-16         32512  32512     16  256    1 : tunables    0    0    0 : slabdata    127    127      0
kmalloc-8          22016  22016      8  512    1 : tunables    0    0    0 : slabdata     43     43      0
kmem_cache_node     4224   4224     64   64    1 : tunables    0    0    0 : slabdata     66     66      0
kmem_cache          3366   3438    448   18    2 : tunables    0    0    0 : slabdata    191    191      0
```

可以看到系统中有很多不同类型的缓存。

# 原理分析

slab 内存管理器的整体结构如下所示：

<img src="/assets/linux/slab_1.png" width="40%">

每个缓存(cache)只负责一种对象类型(例如 mm_struct)，每个缓存中的 slab 数目各不相同，所有的缓存被保存在一个双链表里。一个缓存用 struct kmem_cache 来表示。

<img src="/assets/linux/slab_2.png" width="60%">

```
struct kmem_cache {
​    struct array_cache *array[NR_CPUS];
​    unsigned int batchcount;
​    unsigned int limit;

​    unsigned int buffer_size;
​    unsigned int num;

​    unsigned int gfporder;
​    size_t colour;
​    unsiged int colour_off;
​    struct kmem_cache *slabp_cache;
​    ...
​    void(*ctor)(struct kmem_cache *, void*);
​    const char *name;
​    struct list_head next;

​    struct kmem_list3 *nodelists[MAX_NUMNODES];
}
```

- 其中 array 是一个指针数组（长度等于 CPU 个数），代表每个 CPU 的 array_cache
- batchcount 指定在 array_cache 为空的情况下，从缓存的 slab 中获取对象的数目
- limit 指定 array_cache 最大可以保存的多少对象
- buffer_size 表示对象的大小
- num 表示每个 slab 中对象的数量
- gfporder 表示每个 slab 中的页数的幂
- colour 和 colour_off 是和着色相关的

对于只有一个内存节点的系统 nodelists 只有一项，kmem_list3 结构中主要是 3 个 slab 列表(free, partial, full)

array_cache 的结构如下

```
struct array_cache {
​    unsigned int avail;
​    unsigned int limit;
​    unsigned int batchcount;
​    unsigned int touched;
​    spinlock_t lock;
​    void *entry[]
}
```

array_cache 的设计是为了更好的利用 CPU 缓存，每个 array_cache 对应一个 cpu (可以理解为 per-CPU 的一个指针)，其中 entry 数组保存了专用于这个 cpu 的对象，该 cpu 释放了一个对象时，先将这个对象保存在对应 cpu 的 array_cache 中，这样下次分配对象就可以直接从这里拿，并且拿到的对象很有可能还位于 cpu 的 cache line 中，这样就大大增加对象的访问速度。当 entry 为空，想再分配对象时，就需要去 slab 中空闲的对象中拿 batchcount 个对象来填充 entry。

```
cache_alloc
​    -> ac = cachep->array[smp_processor_id()]
​    -> if(ac->avail)
​            objp = ac->entry[--ac->avail]
​       else
​            objp = cache_alloc_refill(cachep, flags)
cache_free
​    -> ac = cachep->array[smp_processor_id()]
​    -> if(ac->avail < ac->limit)
​            ac->entry[ac->avail++] = objp
​        else
​            cache_flusharray(cachep, ac)
​            ac->entry[ac->avail++] = objp
```

于是对象的分配有如下三个层级，第 1 层和第 2 层都会管理一定数量的对象，当这两层中可用的对象不足时，就会从下一层拿，如果对象太多了，就会还给下一层，同样的，当系统物理内存不足时，buddy system 也会从 slab 层回收可用的内存(free slab)。

```
1    per-CPU array cache
​            ^
​            |
​            v
2   free or partial free slab
​            ^
​            |
​            v
3       buddy system
```

我们现在来看看 slab 的结构

<img src="/assets/linux/slab_3.png" width="20%">

slab 头部管理数据结构如下：

```
struct slab {
​    struct list_head list;
​    unsigned long colouroff;    //着色偏移
​    void *s_mem;                //首对象地址
​    unsigned int inuse;         //被使用的对象，不能超过 slab 中最大可管理的对象数
​    kmem_bufctl_t free;
​    unsigned short nodeid;
};
```

每个 slab 有 slab head，后面跟着一个对象管理数组，剩余的内存则按对象的大小（对齐后）平均的划分，一般还会有剩余的内存，这部分内存用来做着色处理。着色处理的好处是位于不同 slab 中的对象可以同时加载到 CPU 的 L1 cache，如果不做着色处理，由于 slab 都是位于 4k 边界处，所以不同 slab 中的对象很有可能会映射到同一个 cache line 上，从而造成 cache 冲突。着色处理如下图所示 （需要更详细的说明？）

<img src="/assets/linux/slab_4.png" width="60%">

slab 对空闲对象的管理设计的比较巧妙

<img src="/assets/linux/slab_5.png" width="60%">

图中，灰色的代表已经被使用的对象，free 代表了下一个可使用对象的索引

slab 不仅可以作为对象的缓存，也可以作为 kmalloc 和 kfree 的前端。在内核初始化过程中，会让 slab 创建不同大小(32、64、128 ...)的 kmem_cache(包括通用内存和 DMA 内存)，如此一来，当内核使用 kmalloc 时，首先去查找合适大小（向上对齐）的 kmem_cache，然后从该 kmem_cache 中分配内存。

```
struct cache_sizes {
​    size_t          cs_size;
​    struct kmem_cache   *cs_cachep;
​    #ifdef CONFIG_ZONE_DMA
​    struct kmem_cache   *cs_dmacachep;
    #endif
};
```

<img src="/assets/linux/slab_6.png" width="60%">

这里再放一张 kmem_cache_alloc 的流程图，方便贯穿理解 slab 整个分配器

<img src="/assets/linux/slab_7.png" width="60%">

# 如何使用

slab 缓存的使用非常简单

```
1. 创建 kmem_cache
// ctor 为构造函数，为每个对象执行该构造函数
struct kmem_cache * kmem_cache_create(const char *name, size_t size, size_t align,
​          unsigned long flags, void (*ctor)(void *))

2. 分配对象
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)

3. 释放对象
void kmem_cache_free(struct kmem_cache *cachep, void *objp)

4. 销毁 kmem_cache
void kmem_cache_destroy(struct kmem_cache *s)
```

我们来看一下示例

```
// 创建一个名为 pgd_cache 的 kmem_cache
pgd_cache = kmem_cache_create("pgd_cache", PGD_SIZE, PGD_SIZE,
​                          SLAB_PANIC, NULL);

// 分配一个 pgd 对象
pgd = kmem_cache_alloc(pgd_cache, PGALLOC_GFP)

// 释放 pgd 对象
kmem_cache_free(pgd_cache, pgd)

// 销毁 pgd_cache
kmem_cache_destroy(pgd_cache）
```