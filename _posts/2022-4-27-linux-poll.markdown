---
layout: post
title:  "poll implementation"
date:   2022-4-27 20:20:16 +0800
categories: kernel
---

在用户层调用poll后，内核使用do_sys_poll来处理需要poll的fds，主要过程如下：
1. 首先会将需要监听的fd从用户层拷贝到内核
2. 然后创建一个poll_wqueues结构，初始化polling_task为current，用来被唤醒时传入try_to_wake_up函数
3. 设置poll_table中的poll_queue_proc默认为__poll_wait
4. 轮询所有fd，调用file->f_op->poll(), 设备的驱动程序一般会在注册的poll中调用poll_wait(filp, &dev's_wait_queue_head_t, poll_table)，实际会调用__poll_wait, poll_table_entry主要包含wait_queue_t wait和wait_queue_head_t wait_address，同时poll_wqueues结构中的poll_table_entry分为两类，一类是静态分配的，大小固定，另一类是按页动态分配，所以该函数首先为需要监听的fd分配一个poll_table_entry，接着初始化entry中的wait的wait_queue_func_t为pollwake，wait_address由设备驱动程序实现的poll传入，最后将这个wait_queue插入到对应设备驱动的wait_queue_head中
```add_wait_queue(wait_address, &entry->wait)```
5. 调用poll_schedule_timeout让当前进程sleep，从而block住用户程序
6. 当驱动程序准备好数据后，会调用wake_up_xxx函数，来唤醒挂载在驱动程序wait_queue_head上的所有进程
7. 清除挂载在设备wait_queue_head上的用户进程

```
void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
{
    if (p && wait_address)    
        p->qproc(filp, wait_address, p);
}

//example of eventfd
static unsigned int eventfd_poll(struct file *file, poll_table *wait)
{
    struct eventfd_ctx *ctx = file->private_data;
    unsigned int events = 0;
    unsigned long flags;

    //ctx->wqh is the wait address
    poll_wait(file, &ctx->wqh, wait);

    spin_lock_irqsave(&ctx->wqh.lock, flags);
    if (ctx->count > 0)
        events |= POLLIN;
    if (ctx->count == ULLONG_MAX)
        events |= POLLERR;
    if (ULLONG_MAX - 1 > ctx->count)
        events |= POLLOUT;
    spin_unlock_irqrestore(&ctx->wqh.lock, flags);

    return events;
}

//this function would wake up all the task on the wait_queue_head
__u64 eventfd_signal(struct eventfd_ctx *ctx, __u64 n)
{
    unsigned long flags;

    spin_lock_irqsave(&ctx->wqh.lock, flags);
    if (ULLONG_MAX - ctx->count < n)
        n = ULLONG_MAX - ctx->count;
    ctx->count += n;
    if (waitqueue_active(&ctx->wqh))
        wake_up_locked_poll(&ctx->wqh, POLLIN);
    spin_unlock_irqrestore(&ctx->wqh.lock, flags);

    return n;
}
```

![1](/assets/qemu/poll1.png)

![1](/assets/qemu/poll2.png)

![1](/assets/qemu/poll3.png)

当设备准备好数据后，调用wakeup来唤醒wait_queue_head_t中所有的wait_queue，进而pollwake会被调用，该函数会重新将poll_wqeueues中的polling_task（也就是调用了poll的应用程序）进程重新放入调度器的run_queue中。