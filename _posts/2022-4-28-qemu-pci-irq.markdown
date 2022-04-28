---
layout: post
title:  "QEMU如何处理pci设备的中断"
date:   2022-4-28 18:00:16 +0800
categories: virtualization 
---

## 8259芯片处理一次中断的过程如下：

1. 当一个或多个中断引脚被拉高时，IRR对应的位置1.
2. 如果此中断没有被屏蔽，8259向CPU发送INT信号。
3. 当CPU接收到中断信号后，通过INTA引脚发送低电平脉冲信号通知8259芯片。
4. 当8259芯片第一次收到来自CPU的INTA信号后，将最高优先级中断对应的位在ISR中置1，同时清零位于IRR中对应的位。
5. 这时，CPU会第二次发送INTA信号给8259芯片，8259芯片收到信号后，将第四步计算得到的中断号通过数据总线发送给CPU。
6. 如果8259芯片工作在AEOI模式，则ISR中对应的位自动清零。否则，该位一直保持位1，直到CPU处理完中断服务程序后，通过EOI命令来清零该位。

## 中断路由

![1](/assets/qemu/irq1.png)
kvm使用kvm_irq_routing_table来管理路由信息，其中map[x]，表示gsi为x的irq以何种方式传递到cpu，有可能是
- slave pic->master-pic->cpu
- master-pic->cpu
- ioapic->cpu
- msi->cpu

## vvgpu 设备assert一个irq的过程

![1](/assets/qemu/irq2.png)
![1](/assets/qemu/irq3.png)
![1](/assets/qemu/irq4.png)

在vcpu进入guest之前，kvm会检查是否有pending的事件需要处理，如果有irq需要注入，则调用inject_pending_event,该函数会找到优先级最高的irq，然后设置vmcs的VM_ENTRY_INTR_INFO_FIELD，这样当cpu切换到guest时，会自动产生一个对应的irq.