---
layout: post
title:  "QEMU event model"
date:   2021-11-15 20:20:16 +0800
categories: virtualization 
---

# QEMU event model
## event loop
QEMU使用了glib的事件循环分发处理机制来处理异步事件，简单来说就是使用一个线程poll不同的event：
- 文件描述符
- Event notifier
- Timer
- Bottom-half

在这个机制中主要有三个概念：
1. **GMainContext**- 代表主事件循环的上下文，与GMainLooper绑定，GMainContext和GMLooper为一对一的关系，同样GMainLooper和线程也是一对一的关系　
2. **GMainLooper**- 代表一个事件循环，和线程绑定
3. **GSource**- 事件源，多个事件源可以绑定到同一个循环上下文中， 这些事件源包含Idle, Timeout 和 I/O，同时也支持创建自定义的事件源。

处理流程：
1. g_main_context_prepare 会遍历所有注册到GMainContext上的事件源GSource, 找到最近要到期的定时任务（将来这个超时时间会作为poll/select/epoll的超时时间，时间到了可以派发定时任务）
2. g_main_context_query　函数用于收集事件源GSource关心的文件描述符
3. g_main_context_poll　则调用poll/select/epoll观察文件描述符事件，注意超时事件为prepare阶段计算到的timeout时间
4. g_main_context_check() 注意这里已经从poll/select/epoll函数返回了，无非就是两种情况，一种为有观察的文件描述符事件到来，另一种为超时，也可能两种情况都有触发。（其实还有一种情况是注册了新的GSource或者GSource 增加了新的文件描述符或新的超时函数）总之poll/select/epoll返回了，g_main_context_check调用每个事件源来的check方法来确认是否真的是关心的事件到达，这里把就绪的事件源收集起来
5. g_main_context_dispatch() 派发事件，也就是把４中收集到的事件源进行事件派发。
  
## QEMU 线程
QEMU在启动后，会有如下不同的线程：
- vCPU线程，用来执行虚拟机代码
- 主循环线程，用来轮询不同的event loop
- IO线程,用来处理设备的模拟逻辑

### 主循环线程
主线程总共有３个event loop,　一个是glib GMainContext和另外两个AioContext，QEMU主线程使用qemu_poll_ns来轮询处理这三个loop,两个AioContext中，第一个为iohander_ctx，用来实现qemu_set_fd_handler, 主要用于net,audio,vfio等；另外一个使用包含新特性的qemu_aio_ctx，这个context使用aio_poll来轮询，主要用于block I/O, ioeventfd，timer和bottom halves。
1. 主循环的初始化
```
int qemu_init_main_loop(Error **errp)
{
    ...
    qemu_aio_context = aio_context_new(&local_error);//创建qemu_aio_context
    if (!qemu_aio_context) {
        error_propagate(errp, local_error);
        return -EMFILE;
    }
    qemu_notify_bh = qemu_bh_new(notify_event_cb, NULL);
    gpollfds = g_array_new(FALSE, FALSE, sizeof(GPollFD));
    src = aio_get_g_source(qemu_aio_context);
    g_source_set_name(src, "aio-context");
    g_source_attach(src, NULL);
    g_source_unref(src);
    src = iohandler_get_g_source();//创建iohandler_ctx
    g_source_set_name(src, "io-handler");
    g_source_attach(src, NULL);
    g_source_unref(src);
    return 0;
}
```

2. 主循环loop的设置
qemu的主循环main_loop会一直调用main_loop_wait函数来监听所有的异步事件，main_loop_wait函数会遍历定时器链表，找出最近的超时时间，然后调用os_host_main_loop_wait
```
static int os_host_main_loop_wait(int64_t timeout)
{
    GMainContext *context = g_main_context_default();
    int ret;

    g_main_context_acquire(context);

    glib_pollfds_fill(&timeout); //prepare阶段，设定需要监听的fds

    qemu_mutex_unlock_iothread();//释放iothread锁
    replay_mutex_unlock();

    //调用ppoll来监听所关心的文件描述符，并设定超时时间
    ret = qemu_poll_ns((GPollFD *)gpollfds->data, gpollfds->len, timeout);

    replay_mutex_lock();
    qemu_mutex_lock_iothread();

    //用来分发处理poll中结果
    glib_pollfds_poll();

    g_main_context_release(context);

    return ret;
}
```
### IO线程
IO线程中有２个event loop，一个是AioContext，使用aio_poll来轮询处理，一个glib GMainContext，需要使用g_main_loop_run来轮询处理,其伪代码如下：
```
iothread_run
    ->while(iothread->running)
        ->aio_poll(iothread->ctx)
        ->g_main_loop_run(iothread->main_loop)
```
## AioContext新特性
1. 牺牲一点cpu损耗，换更低的延迟
2. 在遍历需要监听的文件描述符时有O(1)的时间复杂度
3. 纳秒级的timer