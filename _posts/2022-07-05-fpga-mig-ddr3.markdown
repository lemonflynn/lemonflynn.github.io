---
layout: post
title:  "MIG DDR3"
date:   2022-7-5 8:00:16 +0800
categories: FPGA 
---

MIG is memory interface generator, implemented by xilinx, could be used to control SDRAM, like DDR3 RAM.

![1](/assets/fpga/mig_1.png)

| Signal                        | Direction     | Description       |
| ------------------------------| ---------     |    -----          |
| app_addr[ADDR_WIDTH – 1:0]    | input         |This input indicates the address for the current request.|
| app_cmd[2:0]                  | input         |this input selects the command for the current request.|
|app_en                         | Input         |This is the active-High strobe for the app_addr[], app_cmd[2:0], app_sz, and app_hi_pri inputs.|
|app_rdy                        |output         |This output indicates that the UI is ready to accept commands. If the signal is deasserted when app_en is enabled, the current app_cmd and app_addr must be retried until app_rdy is asserted.|
|app_hi_pri                     | Input         |This active-High input elevates the priority of the current request.|
|app_rd_data [APP_DATA_WIDTH – 1:0]|Output      |This provides the output data from read commands.|
|app_rd_data_end                | Output        |This active-High output indicates that the current clock cycle is the last cycle of output data on app_rd_data[]. This is valid only when app_rd_data_valid is active-High.|
|app_rd_data_valid              | Output        |This active-High output indicates that app_rd_data[] is valid.|
|app_sz                         | Input         |reserved and should be tied to 0|
|app_wdf_data [APP_DATA_WIDTH – 1:0]|Input      |This provides the data for write commands.|
|app_wdf_end                    | Input         |This active-High input indicates that the current clock cycle is the last cycle of input data on app_wdf_data[].|
|app_wdf_mask [APP_MASK_WIDTH – 1:0]|Input      |This provides the mask for app_wdf_data[].|
|app_wdf_rdy                    | Output        |This output indicates that the write data FIFO is ready to receive data. Write data is accepted when app_wdf_rdy = 1’b1 and app_wdf_wren = 1’b1.|
|app_wdf_wren                   | Input         |This is the active-High strobe for app_wdf_data[].|
|app_correct_en_i               | Input         |When asserted, this active-High signal corrects single bit data errors. This input is valid only when ECC is enabled in the GUI. In the example design, this signal is always tied to 1.|
|app_sr_req                     | Input         |reserved and should be tied to 0|
|app_sr_active                  | Output        |reserved|
|app_ref_req                    | Input         |This active-High input requests that a refresh command be issued to the DRAM.|
|app_ref_ack                    | Output        |This active-High output indicates that the Memory Controller has sent the requested refresh command to the PHY interface.|
|app_zq_req                     | Input         |This active-High input requests that a ZQ calibration command be issued to the DRAM.|
|app_zq_ack                     | Output        |has sent the requested ZQ calibration command to the PHY interface|
|ui_clk                         | Output        |This UI clock must be a half or quarter of the DRAM clock|
|init_calib_complete            | Output        |PHY asserts init_calib_complete when calibration is finished|
|app_ecc_multiple_err[7:0]      | Output        | ECC specific|
|ui_clk_sync_rst                | Output        |This is the active-High UI reset.|
|app_ecc_single_err[7:0]        | Output        | ECC specific|

# Command Path
When the user logic **app_en** signal is asserted and the **app_rdy** signal is asserted from the UI, a command is accepted and written to the FIFO by the UI. The command is ignored by the UI whenever **app_rdy** is deasserted. The user logic needs to hold **app_en** High along with the valid command and address values until **app_rdy** is asserted as shown in below.

![1](/assets/fpga/mig_2.png)

- 这个图不知道为何和我理解的时序图不一样，按道理不应该是红色修改后的那样吗？

# Write Path

A non back-to-back write command can be issued as shown in below. This figure depicts three scenarios for the app_wdf_data, app_wdf_wren, and app_wdf_end signals, as follows:

1. Write data is presented along with the corresponding write command (second half of BL8).
2. Write data is presented before the corresponding write command.
3. Write data is presented after the corresponding write command, but should not exceed the limitation of two clock cycles.

For write data that is output after the write command has been registered, as shown in Note 3, the maximum delay is two clock cycles.

![1](/assets/fpga/mig_3.png)


# Read Path

![1](/assets/fpga/mig_4.png)

~~读的时候比较简单，但是需要特别注意的是，在发送了读命令后，最好等返回的数据变成valid后，再读取下一个地址（此时app_rdy_o可能还是高电平，意味着可以继续发送命令），否则会出错。~~

![1](/assets/fpga/mig_5.png)

在时刻1处，开始发送读命令，到2时，才收到valid信号，但是此时读取的数据是不对的，正确的数据要到3时才是对的。

~~经过修改后，每次发送读命令后，状态就从READ切换到WAIT_READ，app_en信号也只有在state==READ时才为高电平，在WAIT_READ状态，qpp_en为低电平，直到app_rd_data_valid变为有效时，才从WAIT_READ切换到READ，重新开始读取下一个地址。~~

![1](/assets/fpga/mig_6.png)

这个错误是由于代码编写不当造成的,

```
always@(posedge ui_clk)
begin
    if(state == READ) begin
        r_w_cnt         <= app_rd_data_valid_o ? (r_w_cnt + 12'd1) : r_w_cnt;
        app_addr_reg    <= app_rd_data_valid_o ? (app_addr_reg + 28'd8) : app_addr_reg;
    end
end
```

在read状态，如上的代码中r_w_cnt表示已读取数据的个数，app_addr_reg表示要读取的地址，仅靠这两个变量来驱动读是不读的，因为app_addr_reg不是非要等到app_rd_data_valid_o有效时才增加，用r_w_cnt > TEST_DATA_RANGE来判断是否发送完读的命令也是不对的，应该改为如下：

```
always@(posedge ui_clk)
begin
    if(state == READ) begin
        r_cmd_sent      <= app_rdy_o ? (r_cmd_sent + 12'd1) : r_cmd_sent;
        app_addr_reg    <= app_rdy_o ? (app_addr_reg + 28'd8) : app_addr_reg;
        r_w_cnt         <= app_rd_data_valid_o ? (r_w_cnt + 12'd1) : r_w_cnt;
    end else if(state == WAIT_READ) begin
        r_w_cnt         <= app_rd_data_valid_o ? (r_w_cnt + 12'd1) : r_w_cnt;
    end
end
```

使用r_cmd_sent表示已经发送了多少读命令，当r_cmd_sent > TEST_DATA_RANGE时，state跳转为WAIT_READ，此时就不需要再发送读命令了，只需要等接受完所有的数据即可，用r_w_cnt表示，当r_w_cnt > TEST_DATA_RANGE时，读操作就完成了。

![1](/assets/fpga/mig_7.png)

完成代码，可参考

https://github.com/lemonflynn/fpga_lab/tree/main/mig_test

# reference
1. UG586