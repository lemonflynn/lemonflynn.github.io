---
layout: post
title:  "CVA6 流水线分析"
date:   2024-8-17 9:00:16 +0800
categories: kernel 
---
**作者: flynn**

## 前言

cva6 是 64 位 risc-v 处理器的一个实现，采用 system verilog 语言，具备 6 级流水线，分支预测，指令单发射，TLB，MMU，Cache 和乱序执行等功能模块，其乱序执行核心采用 scoreboard 机制，整个流水线可以简单划分为前端和后端，前端负责取指令、PC 值更新和分支预测处理，它和后端通过 instruction queue 来传递指令，后端主要负责指令解码、指令发射、指令执行、指令提交和中断的处理，[流水线全貌](https://docs.openhwgroup.org/projects/cva6-user-manual/03_cva6_design/intro.html)如下图所示：![ariane_overview.drawio](/assets/fpga/ariane_overview.drawio.png)

在阅读流水线后端代码时，有一些比较难懂的地方，在此做个记录，也顺便梳理一下整个流水线后端的工作流程。个人认为，在后端的设计中，精髓在 issue 和 LSU模块(包括 MMU 的实现)，官方文档对 [LSU 模块](https://docs.openhwgroup.org/projects/cva6-user-manual/03_cva6_design/ex_stage.html#load-store-unit-lsu)的介绍已经很详细了，这里就不过多赘述，本文将主要围绕 issue 阶段来展开。

## 指令发射 issue

在 instruction decode 阶段，会根据从 I$ 中取得的指令，构造一个 scoreboard entry，通过以下接口传递到 issue 阶段：

```verilog
// Handshake's data between decode and issue - ISSUE
output ariane_pkg::scoreboard_entry_t issue_entry_o,
// Instruction value - ISSUE
output logic [31:0] orig_instr_o,
// Handshake's valid between decode and issue - ISSUE
output logic issue_entry_valid_o,
// Report if instruction is a control flow instruction - ISSUE
output logic is_ctrl_flow_o,
// Handshake's acknowlege between decode and issue - ISSUE
input logic issue_instr_ack_i,
```

其中，scoreboard_entry_t 包含了一条指令送入执行阶段的所有信息，定义如下：

![image-20240810111838400](/assets/fpga/cva6_1.png)

```verilog
typedef enum logic [3:0] { 
  NONE,       // 0 
  LOAD,       // 1 
  STORE,      // 2 
  ALU,        // 3 
  CTRL_FLOW,  // 4 
  MULT,       // 5 
  CSR,        // 6 
  FPU,        // 7 
  FPU_VEC,    // 8 
  CVXIF,      // 9 
  ACCEL       // 10
} fu_t;

typedef struct packed { 
  riscv::xlen_t cause;  // cause of exception
  riscv::xlen_t tval;  // additional information of causing exception (e.g.: instruction causing it),
  // address of LD/ST fault
  logic valid;
} exception_t;

typedef struct packed { 
  cf_t                    cf;               // type of control flow prediction
  logic [riscv::VLEN-1:0] predict_address;  // target address at which to jump, or not
} branchpredict_sbe_t;

typedef enum logic [2:0] { 
  NoCF,    // No control flow prediction
  Branch,  // Branch
  Jump,    // Jump to address from immediate
  JumpR,   // Jump to address from registers
  Return   // Return Address Prediction
} cf_t;
```

在图中，虽然画了一个 issue buffer，其实 buffer 深度为 1。

指令发射阶段的工作，主要由 **issue_read_operand** 和 **scoreboard** 两个模块完成。scoreboard 是核心组件，其功能框图如下：

![image-20240813082057574](/assets/fpga/cva6_2.png)

scoreboard 中有一个 issue queue 来管理要发射的指令，这个 queue 就是 scoreboard，其结构如下：

```verilog
// this is the FIFO struct of the issue queue
typedef struct packed {
    logic issued;  // this bit indicates whether we issued this instruction
    logic is_rd_fpr_flag;  // redundant meta info, added for speed
    scoreboard_entry_t sbe;  // this is the score board entry we will send to ex
} sb_mem_t;

sb_mem_t [NR_SB_ENTRIES-1:0] mem_q, mem_n;
```

<img src="/assets/fpga/cva6_3.png" alt="cva6_3" style="zoom:30%;" />

**scoreboard** 模块会从 issue buffer 中取得指令，它的逻辑是判断 scoreboard 是否还有空余的 entry，如果有，则将 issue buffer 中的 sbe 存储在 scoreboard 中 issue_poiner 对应的 entry 中，并更新 issue_pointer_q 为 sbe 的 transaction ID，接着将 sbe 传递给 **issue_read_operand** 模块。

#### Data forward

**issue_read_operand** 模块从 sbe 中解析出 rs1/2/3 地址，从寄存器文件中读取对应的内容，在此之前，会将 rs1/2/3 再送回 scoreboard 模块，目的是判断是否有正在执行的指令将要更新这几个寄存器，这种指令可以分为两种情况，第一种是已经执行完但是还没有提交的指令，第二种是在这个指令周期内会更新这几个寄存器的指令，其处理逻辑是：

根据 issue_read_operand 模块传入的 rs1, rs2, rs3, 以及 write_back port 和 issue fifo 中的数据构建 rs_fwd_req 和 rs_data。

```verilog
logic [NR_SB_ENTRIES+NrWbPorts-1:0] rs1_fwd_req, rs2_fwd_req, rs3_fwd_req;
logic [NR_SB_ENTRIES+NrWbPorts-1:0][XLEN-1:0] rs_data
```

对于 rs1_fwd_req, rs2_fwd_req, rs3_fwd_req，构建过程分为两部分

a. 遍历 write_back ports，根据每个 write back 信号的 transaction ID 得到对应 sbe，判断 sbe.rd 是否等于 rs，以及 write_back 信号是否有效，write_back 没有异常产生，要更新的寄存器是否类型一致，如果满足条件，设置对应 req 为 1，以及设置对应的 `rs_data = wbdata_i`

b. 遍历所有 sbe entry，判断  `(mem_q[k].sbe.rd == rs1_i) & mem_q[k].issued & mem_q[k].sbe.valid` 且要更新的寄存器是否类型一致，如果满足条件，设置 req1_fwd_req[k] = 1。

将 rs1_fwd_req 和 rs_data 送入 rr_arb_tree，由这个仲裁器判从 rs1_fwd_req 中的众多 req 中，选出一个 req，以及对应的数据，在 rr_arb_tree 选择的逻辑中，a 的优先级比 b 高。

**这里有个疑问**，按照 CVA6 的设计，为了解决 WAW(write after write)冲突，发射一条指令之前，都会检查该指令的 rd 寄存器是否要被某个功能单元 clobber，也就是说，是否有正在执行的指令也要更新相同的 rd 寄存器，如果是的话，就不会允许发射这条指令。所以上述 a 和 b 所要处理的情况，并不会同时发生，也就是说 rs_fwd_req 中最多只有一位为 1，应该不需要 rr_arb_tree 来进行仲裁选择，不知为什么要引入这个模块，即便如此，也不需要让 a 的优先级比 b 高，这是为何呢？

#### Clobber process

与此同时，scoreboard 模块还需要维护一个 rd_clobber_gpr 结构体，定义如下：

```verilog
fu_t [2**REG_ADDR_SIZE-1:0] rd_clobber_gpr_o
```

用来表示寄存器中的值是否是最新的，例如，如果 rd_clobber_gpr_o[2] = none, 则表示没有正在执行的指令要更新 X2 寄存器，如果  rd_clobber_gpr_o[2] = ADD，则表示，某个正在执行的 ADD 指令将要更新 X2 寄存器。这个信号会被 issue_read_operand 模块使用，用来判断是否接受 forward 数据。

我们来分析一下 rd_clobber_gpr_o 的构建逻辑，首先看看所需要的数据结构：

```verilog
logic [2**REG_ADDR_SIZE-1:0][NR_SB_ENTRIES:0] gpr_clobber_vld;
logic [2**REG_ADDR_SIZE-1:0][NR_SB_ENTRIES:0] fpr_clobber_vld;
fu_t [NR_SB_ENTRIES:0]                             clobber_fu;
```

注意这里第二维度用的是 NR_SB_ENTRIES 而不是 NR_SB_ENTRIES - 1，因此 NR_SB_ENTRIES 不对应任何 sbe，将会被用作默认值。假设 NR_SB_ENTRIES 为 8。

<img src="/assets/fpga/cva6_4.png" alt="cva6_4" style="zoom:33%;" />

clobber process 核心要做的事情是判断各个寄存器的值是否要被哪个功能单元（但是 issue_read_operand 模块只是用到了是否被功能单元更新这个信息，并没有对不同功能单元进行区分）更新，进而在 issue 阶段需要根据 issue 的不同指令做 forward 或 stall 处理，其步骤如下：

1. 依次设置 gpr_clobber 结构中每个寄存器(x0-x31)对应 scoreboard 顶部 entry 的数组项(sb8) 1
2. 设置 clobber_fu 结构中，标记对应 scoreboard 顶部 entry 的数组项为 none，和步骤 1 的作用是设置默认值
3. 遍历所有 scoreboard entry，在上述两个结构体中依次标记每个已经 issued 的 scoreboad entry 要更新的目的寄存器和所使用的 fu

4. ~~因为多个 scoreboard entry 可能会对同一个寄存器进行写回操作，所以依次对每个寄存器进行 round robin arbitration~~，对每个寄存器 k 都进行 rr_arb_tree 操作，其作用是从 sb0-sb7 和 sb8 之间选择一个 sbi，设置 rd_clobber_gpr_o[k] = clobber_fu[sbi]，其中 sb8 的优先级最低，这样一来，如果 sb0-sb7 都是 0 ，则 sb8 被选中，得到的 fu 为 none，表示没有正在执行的指令要更新寄存器 k

<img src="/assets/fpga/cva6_5.png" alt="cva6_5" style="zoom:33%;" />

#### Read operand

这个结构体信号会在 issue_read_operand 模块中使用，用来判断某个寄存器是否要被 function-unit 操作，如果是的话，就要进行 forward （如果 forward 信号是 valid）或 stall 操作等待执行结果，否则就直接从寄存器文件(由 issue_read_operand 模块定义)中读取值。

issue_read_operand 模块在判断指令不需要 stall 且指令运行所需的功能单元可用的情况下，会继续判断，该指令的 rd 是否和正在执行的某个指令的 rd 相冲突，如果不冲突则设置 issue_ack_o = 1，表示发射这条指令，否则设置 issue_ack_o =0，该信号会被 scoreboard 模块使用，`decoded_instr_ack_o  = issue_ack_i & ~issue_full;` 用来 ack 来自 issue buffer 的握手信号。

#### Write back

指令在被 function unit 执行完之后，其结果会被更新到 scoreboard 中，也就是 mem_n。scoreboard 模块根据 write_back_port 中的 trans_id 找到对应的 sbe，如果该 sbe 已经被 issue，则设置 sbe.valid 和 sbe.result， 如果有异常，则设置 sbe.ex 以及 sbe.ex.cause。此时此刻，该指令已经执行完毕，等待被 commit，也就是还没有更新 architecture state，一切都还可以回退。

## 指令提交/退休 commit

scoreboard 和 commit 模块交互的逻辑也比较简单，在 scoreboard.sv 中定义了这样一个二维数组

```verilog
logic [NrCommitPorts-1:0][TRANS_ID_BITS-1:0] commit_pointer_n, commit_pointer_q;
```

commit_pointer_n[0] 表示第一个 commit port 要 commit 的指令对应的 trans_id，下一个指令周期要 commit 的指令位于 scoreboard 中的第 trans_id + 1，依次类推。

```verilog
assign commit_instr_o[i] = mem_q[commit_pointer_q[i]].sbe
```

其中 i 根据 commit port 的数量，依次递增，在收到来自 commit 模块的 commit_ack_i[i] 后

```verilog
if (commit_ack_i[i]) begin
	mem_n[commit_pointer_q[i]].issued    = 1'b0;
	mem_n[commit_pointer_q[i]].sbe.valid = 1'b0;
end
```

那么，commit 模块会在何时 commit_ack 呢？
这里仅对要 commit 的第一个指令进行示例，当要同时 commit 多条指令时，指令之间需要做额外的判断

```verilog
if inst valid & !exception & !halt
case(inst)
store:
	-> if lsu_ready, commit_ack = 1, else 0
fpu:
	-> commit_ack = 1
csr:
	-> if csr_excep not valid, commit_ack = 1, else 0
vma or fence_i or fence:
flush_dcache_i & cachetype == wb & inst is not store:
	-> commit_ack = no_st_pending_i
amo:
	-> commit_ack = amo_resp_i.ack
```

除此之外，为了实现精确异常模型，所有来自流水线内部的 exception 和外部的 interrupt 也会在 commit 阶段被处理。
