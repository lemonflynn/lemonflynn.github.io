---
layout: post
title:  "Confusing about wire and reg in Verilog"
date:   2021-08-28 10:20:16 +0800
categories: FPGA 
---

#### 1. declare pc_alu as a register, and use always(*) to assign i_test to pc_alu. Xilinx synthesis tool will treat pc_alu as a wire.
   
```c
module test(
    input i_clk,
    input i_reset,
    input i_test,
    output reg[3:0] o_val
    );

reg pc_alu;

always@(posedge i_clk)
begin
    o_val <= {2'b0, pc_alu, i_reset};
end

always@(*)pc_alu = i_test;
endmodule
```

![reg0](/assets/verilog/reg0.png)

#### 2. Then if use non-block assignment to set pc_alu, pc_alu will be a register, and schematic would be very strange.

```c
reg pc_alu;

always@(*)pc_alu = i_test;

always@(posedge i_clk or negedge i_reset)
begin
    if(!i_reset)
       pc_alu <= 0;
end
```

![reg1](/assets/verilog/reg1.png)

#### 3. Declare pc_alu as a wire, then use Continuous Assignment, the result is what we want, same as 1.
   
```c
wire pc_alu;

always@(posedge i_clk)
begin
    o_val <= {2'b0, pc_alu, i_reset};
end

assign pc_alu = i_test;
```

![wire](/assets/verilog/wire.png)

#### 4. Declare pc_alu as a wire, hten use non-Continuous Assignment, the synthesis would fail.

```c
wire pc_alu;

always@(posedge i_clk)
begin
    o_val <= {2'b0, pc_alu, i_reset};
end

always@(*)pc_alu = i_test;
```