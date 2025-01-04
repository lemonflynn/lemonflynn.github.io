---
layout: post
title:  "Ara co-processor 分析(2) MASKU and VLSU"
date:   2025-1-1 9:00:16 +0800
categories: kernel 
---
**作者: flynn**

# VLSU

VLSU 包含三个模块，AddGen、Load Unit、Store Unit，其核心功能分别如下

1. **ADDGen**

- VA->PA 地址转换
- 根据 load-store 的 index 或 stride 类型，生成 AXI 请求，控制 AXI 总线的地址通道，并控制 load unit 或 store unit

2. **Load Unit**

- 控制 AXI 总线的 R 通道读取数据
- 把读到的数据写回 VRF
- 处理 exception

3. **Store Unit**

- 从 lane 中读取要写到内存的数据
- 控制 AXI 总线的 W 通道写数据，以及 B 通道
- 处理 exception 

其中 ADDGen 是核心模块，同时只能执行一条 load 或 store 指令，其内部逻辑由两个状态机控制：

FSM1 主要处理来自 main_sequencer 的 request，以及控制 FSM2：

<img src="/assets/fpga/ara14.png" alt="image-20250103082644542" style="zoom:40%;" />

FSM2：

<img src="/assets/fpga/ara15.png" alt="image-20250103083940532" style="zoom:30%;" />

最后来一个 overview 吧

![image-20250103085252310](/assets/fpga/ara16.png)

# MASKU

mask unit 是一个较为复杂的模块，因为它要实现的功能比较多，且需要同时处理来自多个 lane 的数据，并且要访问的数据还需按照下图的规则进行重新排列：

![image-20250103075322623](/assets/fpga/ara13.png)

指令集定义的指令中，需要 mask unit 参与的指令可以分为如下五类：

1. **vector integer compare instruction** 

   *VMSEQ, VMSNE, VMSLTU, VMSLT, VMSLEU, VMSLE, VMSGTU, VMSGT*，由 ALU 计算第一阶段的结果，再由 MASKU 进行最终的计算

2. **vector float point compare instruction**

   *VMFEQ, VMFLE, VMFLT, VMFNE, VMFGT, VMFGE*，由 FPU 计算第一阶段的结果，再由 MASKU 进行最终的计算

3. **vector mask instruction**

   - *VMANDNOT, VMAND, VMOR, VMXOR, VMORNOT, VMNAND, VMNOR, VMXNOR*，由 ALU 计算第一阶段的结果，再由 MASKU 进行最终的计算
   - *VMSBF, VMSOF, VMSIF, VIOTA, VID, VCPOP, VFIRST*，结果完全由 mask unit 进行计算

4. **Vector Integer Add-with-Carry/Subtract-with-Borrow instruction**

   需要由 V0 (mask register) 提供进位信息，供 Lane 中的 ALU/FPU 使用

5. **vector mask control**

   由  V0 (mask register)  进行 vector 指令的 **mask** 控制，由 MASKU 提供给 Lane 中的 ALU/FPU，设置对应 result_queue 的 **BE**（Byte Enable） 位

前两类指令的处理较为复杂，其逻辑如下图所示：

![image-20250101120143889](/assets/fpga/ara12.png)

每个 lane 中的 ALU 或者 FPU 对相应的元素进行比较计算，每次处理来自两个 memory bank 的 64 bit 的数据，假设处理一个 64 bit 需要一个时钟周期，那么处理完整的 vector register 需要 8 个时钟周期，在生成最终的结果后，需要将结果写回 vector register。

第三类指令的处理比较简单，因为其源数据就是以 mask register layout 进行布局的，不需要进行特殊转换，可以直接进行逻辑（OR, AND, ...）运算。

第四，五类指令也是按批次处理，每次同时处理所有 lane 中的某个 bank 的 mask 信息，由 mask_pnt 进行 offset 管理。