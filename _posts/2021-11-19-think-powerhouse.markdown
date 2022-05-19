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