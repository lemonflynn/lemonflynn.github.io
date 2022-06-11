---
layout: post
title:  "自问自答系列"
date:   2021-11-19 20:20:16 +0800
categories: Tech 
---

# vfio memory 访问的 fast path & slow path是什么？

vfio fast path是指虚拟机可以直接访问物理PCI设备的设备内存，而不需要QEMU的参与，这是因为在QEMU、VFIO和KVM的帮助下（QEMU不知道HPA，但是QEMU知道mmap后的HVA），建立了GPA和HPA之间的映射关系，这个映射关系被保存在mmu stage-2转换的页表中。
```
    QEMU:     HVA       set_map_of <HVA,GPA>
               ^               ^
               |               |
    --------(mmap)-------(kvm vm ioctl)---------
               |               |
               v               v
    VFIO:    -----     KVM: -------------
             |HPA| <--------|mmu stage-2|---> GPA
             -----          -------------            
               | 
    ---(pci_resource_start)------------
               |
               v
               ---------------
    device:    |memory region|
               ---------------
```
上图中HVA和HPA之间的mmap操作是由VFIO模块完成的，KVM只关注HPA和GPA之间的映射关系。
在有的情况下，我们需要做一些设备模拟的逻辑（比如INTX中断的注入），这时就需要disable fast path，具体步骤是，将PCI设备的各个bar对应的memory region设置为disable，调用flat_view_reset，重新遍历address space中的所有memory region创建新的flat view，当然，被disable的memory region将被排除在外，接着，调用update_address_space，对比新的和之前的address space，发现有被删除的memory region，调用各个memory region listener的region_del回调函数，kvm注册回调为kvm_region_del，这个函数做的事情就是将该memory region的大小改为0，然后发送ioctl（KVM_SET_USER_MEMORY_REGION）来让KVM解除对应的映射关系，KVM最终会调用kvm_arch_flush_shadow_kvmslot来解除GPA和HPA之间的映射关系，这样当VM直接访问GPA时就会触发VMEXIT和KVM EXIT，最终在QEMU中注册的memory region的read/write的回调函数就会被调用。

# linux 的地址空间

首先，对于32位的linux来说，内核和用户层共享了虚拟地址空间，其中用户可以使用的虚拟地址为0-3G，内核虚拟地址为3G-4G，同时每个进程的内核部分是共享一致的，换一种说法，就是linux将内核虚拟地址空间映射到每一个进程中的相同位置，如下图
```
 usr process     usr process      kernel process
       0              1                0
    _______         _______           _______
4G |       |    4G |       |      4G |       |
   |kernel |  ==   |kernel |   ==    |kernel |
3G |_______|    3G |_______|      3G |_______|
   |       |       |       |
   | user  |  !=   | user  |
   |       |       |       |
0G |_______|    0G |_______|

```
这样做的好处是，在进程调度的时候，内核的虚拟地址映射是不变的，这样可以减少TLB flush。那问题来了，每个进程都有自己的一套页表，进程切换时，需要更新CR3的值（对于x86来说），如何做到让每个进程的页表都能包含相同的3G-4G的映射关系呢？这个问题换个问法也就是

## kernel是什么时候创建这样的虚拟地址空间的？
答案是在fork时，整个0-4G的映射关系被复制给子进程，当调用exec时，子进程的0-3G的映射关系将被清除，同时建立新的映射关系（将代码段和数据段映射到程序的entry point address）。这样一来，子进程和父进程就有了如上图usr process0和usr process1所示的虚拟地址空间。虚拟地址和物理地址之间的映射经过多级页表来管理，虽然用户进程的页表中包含了内核虚拟地址空间，但是内核通过设置NX来防止用户程序访问内核虚拟地址空间。对于ARM来说，分别使用两套页表来区分内核虚拟地址空间和用户地址空间，分别是对应于TTBR0_EL0和TTBR1_EL1寄存器。

## current->mm 和current->active_mm是什么关系？
内核可以通过current->mm来判断current进程是否是用户进程，对于用户进程来说current->mm用来管理虚拟地空间，内核进程的current->mm为null，当然内核中也有一部分虚拟内存是需要经过页表转换的，所以内核进程也需要设置页表基地址，进行内核虚拟地址的转换，内核使用current->active_mm来表示真正使用的虚拟地址空间，在进程调度时，这个active_mm是借用自previous进程，因为内核进程不需要访问用户空间，且每个进程的内核部分映射都是一样的，所以内核完全可以使用上一个进程设置好的虚拟地址空间，这样就避免了重新设置CR3寄存器，减少TLB flush，同时，对于用户进程来说，mm和active_mm是一致的。

[Active MM](https://docs.kernel.org/vm/active_mm.html)

## 什么是high memory？
那由于内核要管理所有的物理地址，且它只有1G的虚拟地址空间，它并不能将所有的物理地址进行一个简单的线性映射（虚拟地址和物理地址之间的转换只需要进行简单的加减运算），所以对于大于1G的内存（实际是896M），内核无法进行直接的映射，就要建立页表映射关系，通过MMU来完成转换，对于大于896M的内存，就称之为high memory。

![1](/assets/linux/address_space_0.png)

高端内存有如下几种映射：
- 多个页面的永久映射：VMAP区域是由vmalloc或ioremap分配出来，其虚拟地址连续，但物理地址不一定连续，ioremap的内存关闭了page caching，每个vmap area间隔4K，用来捕获内存访问溢出
- 单页临时映射：atomic_kmap，速度快，因为使用了预留了虚拟地址，所以不需要查找可用的vm area，map和unmap之间时间很短，例如用于中断上下文。
- 单页永久映射：固定的单页线性映射，例如APIC中的0xFFFF_F000，或kmap
![1](/assets/linux/address_space_1.png)