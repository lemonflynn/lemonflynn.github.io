---
layout: post
title:  "linux workqueue implementation"
date:   2021-08-01 08:25:16 +0800
categories: kernel 
---


linux中的workqueue可以用来执行延迟执行一些操作，这些操作可以不是原子操作，一般可以用来实现中断的下半部。我们来分析一下它的实现机制。

首先，使用如下函数创建workqueue:

```c
#define create_workqueue(name) __create_workqueue((name), 0, 0, 0)
#define create_singlethread_workqueue(name) __create_workqueue((name), 1, 0, 0)
```
 create_workqueue会在每个cpu上创建一个worker thread，create_singlethread_workqueue只会在cpu0上创建一个worker thread。其数据结构如下所示：
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/a82a47041fe348189052f0ac8dcf3a00.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzc4MDI2MA==,size_16,color_FFFFFF,t_70)
kernel使用list来管理所有的workqueue，list head为workqueues，每个workqueue_struct中包含了多个（和cpu核数相同）cpu_workqueue_struct，内核为每个cpu_workqueue创建了一个worker_thread，其中使用worklist来管理插入该queue中的work，以及一个wait_queue_head用来唤醒worker_thread的执行， 在worklist为空的时候，worker_thread会使用prepare_to_wait是自身进入sleep状态，并将自身加入到等待队列more_work中。

使用如下接口来定义work：
```c
静态定义
DECLARE_WORK(name, void (*function)(void *))
动态定义
INIT_WORK(name, void (*function)(void *)
```
有了workqueue和work后，就可以使用

```c
int queue_work(struct workqueue_struct *wq, struct work_struct *work)
```
该函数将work插入到wq中对应cpu（调用该函数的cpu）的cpu_workqueue中的worklist，并使用wake_up(more_work)来唤醒workerthread去处理worklist。

内核还准备了一个全局的workqueue，这样就只需要调用`schedule_work(struct work_struct *work)`就可以将work放入keventfd_wq中。