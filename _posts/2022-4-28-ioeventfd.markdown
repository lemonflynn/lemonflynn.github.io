---
layout: post
title:  "kvm ioeventfd"
date:   2022-4-28 16:20:16 +0800
categories: kernel
---

ioeventfd的设计初衷是为了减少heavy-weight的vmexit，适用于当vm操作的PIO/MMIO只是当成一个信号用来控制其他的设备模拟逻辑。

```
struct _ioeventfd {
    struct list_head     list;
    u64                  addr;
    int                  length;
    struct eventfd_ctx  *eventfd;
    u64                  datamatch;
    struct kvm_io_device dev;
    bool                 wildcard;
};
```

ioeventfd的实现方式是创建一个kvm_io_device并注册到kvm_io_bus(mmio/pbo bus)上，这个设备包含addr, length, eventfd等，这些是在创建ioeventfd时由qemu传入，当vm由于读写io产生vmexit后，kvm首先会遍历kvm_io_bus上的所有设备，判断如果这个设备可以处理这个io，则由这个kvm_io_device处理，就不需要再退出到qemu了，判断的依据是kvm_io_device->ops，为ioeventfd创建的kvm_io_device的ops->write为
```
ioeventfd_write(struct kvm_io_device *this, gpa_t addr, int len,
        const void *val)
{
    struct _ioeventfd *p = to_ioeventfd(this);

    if (!ioeventfd_in_range(p, addr, len, val))
        return -EOPNOTSUPP;   

    eventfd_signal(p->eventfd, 1); 
    return 0;
}
```
ioeventfd_in_range用来判断产生vmexit的addr是否可以由这个kvm_io_device处理，如果可以，则使用eventfd_signal来通知应用层。

[KVM: add ioeventfd support](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d34e6b175e61821026893ec5298cc8e7558df43a)
