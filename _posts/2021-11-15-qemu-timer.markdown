---
layout: post
title:  "QEMU Timer"
date:   2021-11-15 20:20:16 +0800
categories: virtualization 
---

# Timer implementation in QEMU

## Clock type
QEMU中定义了如下四种时钟：
1. QEMU_CLOCK_REALTIME: 真实时钟，即使vcpu暂停运行，时钟继续更新
2. QEMU_CLOCK_VIRTUAL: 虚拟时钟，如果vm暂停了，该时钟也不更新
3. QEMU_CLOCK_HOST: host时钟，也就是墙上时钟
4. QEMU_CLOCK_VIRTUAL_RT: 根据instruction counter来更新时钟，如果使用了硬件加速，则等同于虚拟时钟

## 相关数据结构
![1](/assets/qemu/qemu_timer.png)

如上图所示，QEMU定义了一个包含４类QEMUClock的qemu_clocks数组，同时QEMU会创建类型为QEMUTimeListGroup的全局变量main_loop_tlg, 这个变量用来管理使用*timer_new_ms*注册的不同时钟类型的定时器，同时针对每一个的AioContext,其中也有一个QEMUTimeListGroup类型的成员变量tlg，用来管理使用*aio_timer_new*注册的定时器。新建的定时器用QEMUTimer代表，根据其截止时间排序后插入指定时钟类型的QEMUTimelist中的active_timer链表中。

当某一类型的lock发生状态变化时（使能或失能，溢出等）， QEMU会调用QEMUClock挂在链表timerlist中的所有QEMUTimeListGroup的notify_cb函数。

## 更新机制
对于使用*timer_new_ms*注册的定时器，QEMU在main_loop_wait中调用qemu_clock_run_all_timers来处理定时器。 对于使用*aio_timer_new*注册的定时器，则在aio_ctx_dispatch阶段使用timerlistgroup_run_timers来处理。处理步骤也很简单，根据当前时间判断，依次调用定时器链表中所有满足截止时间的定时器的callback函数。

## 备注
![2](/assets/qemu/qemu_timer2.png)

所以在QEMU中注册的所有定时器中，假如有两个定时器的截止时间是一样的，他们也不是同时执行，而是有先后顺序的。所以定时器中不能做耗时长的操作，否则会影响其他定时器的准时执行。