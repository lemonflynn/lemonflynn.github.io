---
layout: post
title:  "由一个小小错误导致FPGA上各种奇怪问题"
date:   2021-10-18 08:20:16 +0800
categories: FPGA 
---

# 问题
经过iverilog验证过设计，生成bitstream下载到FPGA开发板上后出现了各种各样奇怪的问题：

1. assign 的值并不完全相等。
![reg0](/assets/fpga/problem01.png)

2. 存入dmem的值，在取出来的时候就不相等了。
将0x4000e9c（函数返回地址）存入地址0x3f3c，在几个时钟周期后取出时，则变成了0x4000e90，这错误的返回地址就导致了程序会跑飞。
![reg0](/assets/fpga/problem02.png)

# 原因
最初以为这些问题是由于设计上的缺陷，或者由于使用了错误的变量类型和verilog语法，经过排查问题依旧。

最后无意发现了如下这条warning信息：
![reg0](/assets/fpga/problem03.png)
![reg0](/assets/fpga/problem04.png)

```create_clock -name sysclk-period 20 [get_ports clk]```

此时真相大白，原来是因为在定义clock时sysclk和-period 20中间少了一个空格...，修改后则一切表现正常。

```create_clock -name sysclk -period 20 [get_ports clk]```