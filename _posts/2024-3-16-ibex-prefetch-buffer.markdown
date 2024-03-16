---
layout: post
title:  "ibex prefetch buffer 分析"
date:   2024-3-16 7:00:16 +0800
categories: FPGA 
---
**作者: flynn**

ibex 是一个开源的 risc-v 处理器，其中 prefetch buffer 是位于 Instruction fetch stage 的一个模块，用来预取指令，其实现方式有一些比较难懂但有意思的地方，这篇文章就用来分析一下其实现细节。

先来看一段[官方的解释](https://ibex-core.readthedocs.io/en/latest/03_reference/instruction_fetch.html):

**Instructions are fetched into a prefetch buffer (rtl/ibex_prefetch_buffer.sv) for optimal performance and timing closure reasons. This buffer simply fetches instructions linearly until it is full. The instructions themselves are stored along with the Program Counter (PC) they came from in the fetch FIFO (rtl/ibex_fetch_fifo.sv). The fetch FIFO has a feedthrough path so when empty a new instruction entering the FIFO is immediately made available on the FIFO output. A localparam DEPTH gives a configurable depth which is set to 3 by default.**

总结如下：

1. prefetch buffer 会按指令顺序取指令，直到 buffer （默认深度为 3）满了
2. 指令会和指令地址一起存储在 FIFO 中
3. 当 FIFO 为空的时候，从 instruction memory/cache 中取得的指令会 bypass fifo，立即送到解码模块

<img src="/assets/fpga/ibx_1.png" alt="ibx_1" style="zoom:30%;" />

首先，先看一下模块定义：

```verilog
module ibex_prefetch_buffer #(
  parameter bit ResetAll        = 1'b0
) (
  input  logic        clk_i,
  input  logic        rst_ni,

  // 来自下级流水线的 request，一般都有效，除非遇到 WFI 等指令
  input  logic        req_i,	

  input  logic        branch_i,	// 正在处理的指令为 branch，需要清空 fifo
  input  logic [31:0] addr_i, // 取指令的地址

  // goes to ID stage
  input  logic        ready_i,
  output logic        valid_o,
  output logic [31:0] rdata_o,
  output logic [31:0] addr_o,
  output logic        err_o,
  output logic        err_plus2_o,

  // goes to instruction memory / instruction cache
  output logic        instr_req_o,
  input  logic        instr_gnt_i,
  output logic [31:0] instr_addr_o,                                                                                                                                                 
  input  logic [31:0] instr_rdata_i,
  input  logic        instr_err_i,
  input  logic        instr_rvalid_i,

  // Prefetch Buffer Status
  output logic        busy_o
);
```



这里 handshake 的接口和 AXI-4 的类似，不做过多解释。这里的代码比较难理解的是**如何实现取指令，直到 FIFO 满**。为方便分析，暂不考虑分支代码的处理，且没有 instruction cache。

ibex 中定义了下面三个寄存器：

```verilog
//                            next state               shift         current state
logic [NUM_REQS-1:0] rdata_outstanding_n, rdata_outstanding_s, rdata_outstanding_q
```

ibex 并未对_n, _s, _q 后缀进行解释说明，这里按我的理解，分别对应 next state, shift , current state，NUM_REQS 为 FIFO 深度，默认为 3。接下来看下面这段代码：

```verilog
  ///////////////////////////////
  // Request outstanding queue //
  ///////////////////////////////
  for (genvar i = 0; i < NUM_REQS; i++) begin : g_outstanding_reqs
    // Request 0 (always the oldest outstanding request)
    if (i == 0) begin : g_req0
      // A request becomes outstanding once granted, and is cleared once the rvalid is received.
      // Outstanding requests shift down the queue towards entry 0.
      assign rdata_outstanding_n[i] = (valid_req & instr_gnt_i) |
                                      rdata_outstanding_q[i];
    end else begin : g_reqtop
    // Entries > 0 consider the FIFO fill state to calculate their next state (by checking
    // whether the previous entry is valid)
      assign rdata_outstanding_n[i] = (valid_req & instr_gnt_i &
                                       rdata_outstanding_q[i-1]) |
                                      rdata_outstanding_q[i];
    end
  end

  // Shift the entries down on each instr_rvalid_i
  assign rdata_outstanding_s = instr_rvalid_i ? {1'b0,rdata_outstanding_n[NUM_REQS-1:1]} :
                                                rdata_outstanding_n;
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      rdata_outstanding_q  <= 'b0;
    end else begin
      rdata_outstanding_q  <= rdata_outstanding_s;
    end
  end
```

这段稍显复杂的代码是用来干嘛的呢？

其实这里是用移位寄存器实现了一个状态机，其工作流程如下：

1. 最初 rdata_outstanding_q 为 3'b000，表示当前有 三个 entry，且 entry 都为空。

2. 对于 entry 0 来说，会设置 

   `assign rdata_outstanding_n[0] = (valid_req & instr_gnt_i) | rdata_outstanding_q[0];`

如果 **valid_req & instr_gnt_i** 有效，则表示 data_memory 模块已经接受到了请求，设置 entry 0 的下一个状态为 1，表示 entry 0 已经被占用，如果 **valid_req & instr_gnt_i** 无效，则 entry 0 的下一个状态和当前状态一       致

3. 对于剩下的 entry 来说，设置

   ```verilog
   assign rdata_outstanding_n[i] = (valid_req & instr_gnt_i & rdata_outstanding_q[i-1]) |
                                     rdata_outstanding_q[i];
   ```

如果 **valid_req & instr_gnt_i** 有效，且前一个 entry 被占用时，才会设置当前 entry 为 1，如果前一个 entry 为空，则当前 entry 保持不变。如果 **valid_req & instr_gnt_i** 无效，则 entry 的下一个状态和当前状态一致

4. 当 instr_rvalid_i 有效时，即从 instruction memory 取得了一个有效指令后，将 rdata_outstanding_n 右移一位，高位清零。

下图也许能更形象的理解这个过程：

<img src="/assets/fpga/ibx_2.png" alt="ibx_2" style="zoom:40%;" />

这幅图左边表示 prefetch buffer 依次向 instruction memory 发出请求，直到最多可以允许的 3 个 outstanding 请求都已经发出后，但都没有接受到有效数据的状态变化。右边表示依次接收了指令数据后的状态变化。当然，这个图比较简单，一般不会是 instruction memory 接收了三次请求后，才返回数据，这只是一个极端的情况。

接下来分析，**instr_req_o** 信号的处理机制：

```verilog
// Make a new request any time there is space in the FIFO, and space in the request queue
assign valid_new_req = req_i & (fifo_ready | branch_i) & ~rdata_outstanding_q[NUM_REQS-1];
assign valid_req = valid_req_q | valid_new_req;

// Hold the request stable for requests that didn't get granted
assign valid_req_d = valid_req & ~instr_gnt_i;
  
always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      valid_req_q          <= 1'b0;
    end else begin
      valid_req_q          <= valid_req_d;
    end
end    

assign instr_req_o  = valid_req;
```

这一段也比较难理解，不过要是画成如下的图，就能更好的理解这段代码了：

<img src="/assets/fpga/ibx_3.png" alt="ibx_3" style="zoom:30%;" />

至此，prefetch buffer 的主要逻辑已经分析结束，还剩如下细节没有分析

- FIFO 入队和出队的处理
- FIFO bypass 的实现
- FIFO 中需要存储指令地址如何自动生成
- 分支指令的处理

这些细节可以通过阅读代码理解，比较简单。