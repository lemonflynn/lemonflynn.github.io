# non-iothread
早期的qemu并不支持多线程，它只有一个线程，在这个线程中完成vcpu、设备模拟和事件处理等逻辑；当该线程正在执行vcpu虚拟机的代码时，如果此时虚拟机发生了一个异常或该线程收到了一个信号，则cpu从执行虚拟机的代码切换到qemu代码，然后根据select的返回结果去处理对应的文件描述符，等完成设备的模拟逻辑后，就接着执行虚拟机的代码。
# iothread
non-iothread的模型并不能利用多核处理器的性能，假如传入-smp 2，即要虚拟出2个vcpu，此时qemu也只有一个线程，在该线程中轮流交替的选择一个vcpu来运行，其也只是用到了一个物理cpu。

新的线程模型是为每一个vcpu分配一个线程，外加一个main loop线程（用来监听文件描述符、eventfd、定时器和中断下半部），可能还会有设备的worker thread（用来卸载main loop的负载，例如vnc）。因为QEMU的代码不是thread-safe的，也就意味着QEMU的代码不能同时被多个线程运行，所以需要一个全局mutex来保证。当vcpu从guest code中退出到qemu中时，需要运行`qemu_mutex_lock_iothread`来获取这个全局锁，之后才能执行QEMU的代码；在进入guest code前使用`qemu_mutex_unlock_iothread`来释放该锁。

# 引用
[QEMU Internals: Overall architecture and threading model](http://blog.vmsplice.net/2011/03/qemu-internals-overall-architecture-and.html)

[QEMU Internals: Event loops](http://blog.vmsplice.net/2020/08/qemu-internals-event-loops.html)