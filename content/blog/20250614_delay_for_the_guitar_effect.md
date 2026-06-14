---
date: '2026-06-14T19:40:36+02:00'
draft: false
title: 'Delay Effect Implementation'
author: "Daniel Blackbeard"
cover:
  image: "images/arty_with_afe.jpg"
  alt: "Picture of the analog front end"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
tags: 
- hdl
- ddr
- log
math: true
---

## Introduction
While keeping sight of my target when doing this project, there were some components on the Arty that I don't really know how to use. One of this was the DRAM mounted on the board. Hence, I wanted to implement the most obvious effect that needs some kind of memory of previous samples.

Basically the delay will take at some point of the chain the samples at a rate of 48KHz, store them in a circular buffer in RAM and set up a memory pointer for reading at some distance of the writing pointer in such a way that the reading is yielding back audio from the past. The maths for this are quite simple in the end, with a sampling rate of 48KHz every memory address of distance between read and write adds ~20.8 microseconds of delay. So let's say I want a delay of up to ~2 seconds we will need 192000 bytes of memory. It's honestly not even that much when the mounted memory is 1GByte.

So, on the specification side, I need a circular buffer that can store the 16bits samples at 48KBaud and read back with a variable distance. I will treat this module as a FIFO for all purposes from now on.

## Talking with the memory controller
Setting up the controller from Vivado was honestly a painful experience. I had to figure many things on the way, so let me put here a recipe that will be useful for the next time I have to implement a memory controller.

### DDR Configuration
Almost all of the configuration can be intuitive, but Vivado might not recognize/remember you set up a Digilent board. The pin association I had to figure it out from the schematic and place it by hand. If for some reason I had to reconfigure again those settings were lost, so I spent a lot of time dealing with inserting many rows of pins on the configuration form. So, here it is the configuration of the Arty A7:

```systemverilog
NET   "ddr3_addr[0]"                           LOC = "R2"    |    ;
NET   "ddr3_addr[10]"                          LOC = "R6"    |    ;
NET   "ddr3_addr[11]"                          LOC = "U6"    |    ;
NET   "ddr3_addr[12]"                          LOC = "T6"    |    ;
NET   "ddr3_addr[13]"                          LOC = "T8"    |    ;
NET   "ddr3_addr[1]"                           LOC = "M6"    |    ;
NET   "ddr3_addr[2]"                           LOC = "N4"    |    ;
NET   "ddr3_addr[3]"                           LOC = "T1"    |    ;
NET   "ddr3_addr[4]"                           LOC = "N6"    |    ;
NET   "ddr3_addr[5]"                           LOC = "R7"    |    ;
NET   "ddr3_addr[6]"                           LOC = "V6"    |    ;
NET   "ddr3_addr[7]"                           LOC = "U7"    |    ;
NET   "ddr3_addr[8]"                           LOC = "R8"    |    ;
NET   "ddr3_addr[9]"                           LOC = "V7"    |    ;
NET   "ddr3_ba[0]"                             LOC = "R1"    |    ;
NET   "ddr3_ba[1]"                             LOC = "P4"    |    ;
NET   "ddr3_ba[2]"                             LOC = "P2"    |    ;
NET   "ddr3_cas_n"                             LOC = "M4"    |    ;
NET   "ddr3_ck_n[0]"                           LOC = "V9"    |    ;
NET   "ddr3_ck_p[0]"                           LOC = "U9"    |    ;
NET   "ddr3_cke[0]"                            LOC = "N5"    |    ;
NET   "ddr3_cs_n[0]"                           LOC = "U8"    |    ;
NET   "ddr3_dm[0]"                             LOC = "L1"    |    ;
NET   "ddr3_dm[1]"                             LOC = "U1"    |    ;
NET   "ddr3_dq[0]"                             LOC = "K5"    |    ;
NET   "ddr3_dq[10]"                            LOC = "U4"    |    ;
NET   "ddr3_dq[11]"                            LOC = "V5"    |    ;
NET   "ddr3_dq[12]"                            LOC = "V1"    |    ;
NET   "ddr3_dq[13]"                            LOC = "T3"    |    ;
NET   "ddr3_dq[14]"                            LOC = "U3"    |    ;
NET   "ddr3_dq[15]"                            LOC = "R3"    |    ;
NET   "ddr3_dq[1]"                             LOC = "L3"    |    ;
NET   "ddr3_dq[2]"                             LOC = "K3"    |    ;
NET   "ddr3_dq[3]"                             LOC = "L6"    |    ;
NET   "ddr3_dq[4]"                             LOC = "M3"    |    ;
NET   "ddr3_dq[5]"                             LOC = "M1"    |    ;
NET   "ddr3_dq[6]"                             LOC = "L4"    |    ;
NET   "ddr3_dq[7]"                             LOC = "M2"    |    ;
NET   "ddr3_dq[8]"                             LOC = "V4"    |    ;
NET   "ddr3_dq[9]"                             LOC = "T5"    |    ;
NET   "ddr3_dqs_n[0]"                          LOC = "N1"    |    ;
NET   "ddr3_dqs_n[1]"                          LOC = "V2"    |    ;
NET   "ddr3_dqs_p[0]"                          LOC = "N2"    |    ;
NET   "ddr3_dqs_p[1]"                          LOC = "U2"    |    ;
NET   "ddr3_odt[0]"                            LOC = "R5"    |    ;
NET   "ddr3_ras_n"                             LOC = "P3"    |    ;
NET   "ddr3_reset_n"                           LOC = "K6"    |    ;
NET   "ddr3_we_n"                              LOC = "P5"    |    ;
```

Other settings are kind of enforced accordingly to what you want to do. I didn't wanted AXI so I disabled that interface. Another tricky part was that the exact component of my board was not the one set by default. I set MT41K128M16XX-15E for my memory and data width of 16, with 4 banks and normal ordering. 

While I don't need crazy speed I will stick with correctness when possible, hence I set the impedance controls to RZQ/6 and using a memory address mapping as BANK-ROW-COLUMN.

In my application I have two clocks: system clock at 100MHz and DSP clock at 60MHz. I use the system clock for the controller, so I have to set in the configuration as "No Buffer" with reset active low and disabling the XADC because I want exclusive control from the analog front end.

Other settings aren't complicated to understand, so I'm skipping. After this you get a memory controller with native interface that's not too hard to use.

### Communication protocol
Here things got a bit messy on my side. I did a controller integrated itself with the delay function, instead of implementing the controller and having a delay module using it. Hence, what comes next is not necessarily standard. 

The memory interface generator (MIG from now on) exposes few controls:

```systemverilog
  // MIG interface
  output logic [27:0]  app_addr,
  output logic [2:0]   app_cmd,
  output logic         app_en,
  input  logic         app_rdy,
  
  output logic [127:0] app_wdf_data,
  output logic         app_wdf_wren,
  input  logic         app_wdf_rdy,
  
  input  logic [127:0] app_rd_data,
  input  logic         app_rd_data_valid,
  input  logic         app_rd_data_end,
  input  logic         calib_done,
  output logic         status
```
the rules are simple and the best way to make the control is with an FSM

```systemverilog
case (ram_state)
    3'b000 : ram_state <= host_wr_ff2                      ? 3'b001 : 3'b000;          // IDLE
    3'b001 : ram_state <= (app_wdf_rdy & app_rdy)          ? 3'b010 : 3'b001;          // WRITE
    3'b010 : ram_state <=                                    3'b011;                   // COMMIT 
    3'b011 : ram_state <=                                    3'b100;                   // FINISH WRITE
    3'b100 : ram_state <=                                    3'b101;                   // SKIP
    3'b101 : ram_state <= (app_rdy & app_en)               ? 3'b110 : 3'b101;          // READ PREPARE
    3'b110 : ram_state <= app_rd_data_valid                ? 3'b111 : ram_state;       // DEASSERT AND READ   
    3'b111 : ram_state <= host_wr_ff2                      ? 3'b111 : 3'b000;          // WAIT ACK
  endcase
```

This FSM starts the moment there is the request from the host side to write a sample by setting `host_wr_ff2`, then waiting for the memory controller to be ready and available for writing. For writing into the memory, there is the limit of being able to write in packets of 128 bits, so I have to pack 8 of my 16 bits samples before executing a writing operation. For this, you have to set a memory address (that will have to move in steps of 16 because a byte is 8 bits), the content to be written and when the memory controller notifies that it's ready for receiving commands, set `app_wdf_wren` and `app_en`. Something like this:

```systemverilog
  if(ram_state == 3'b001) begin
    ram_wr_flag  <= '0;
    app_cmd      <= '0;
    app_addr     <= ram_addr;
    app_wdf_data <= {fifo_wr_mem[7],fifo_wr_mem[6],fifo_wr_mem[5],fifo_wr_mem[4],fifo_wr_mem[3],fifo_wr_mem[2],fifo_wr_mem[1],fifo_wr_mem[0]};
    if (app_wdf_rdy & app_rdy) begin
      app_wdf_wren <= '1;
      app_en       <= '1;
    end else begin
      app_wdf_wren <= '0;
      app_en       <= '0;
    end
  end
```

A consequence of the steps of 8 bytes in writing is that the granularity of the delay is in that many samples. Before writing/reading I have to collect those 8 samples. I could just insert a single sample every 8 memory address, but I consider the granularity is good enough for the application, sitting at ~166us.

For reading is similar, but `app_cmd` changes in this case:

```systemverilog
  if(ram_state == 3'b101) begin
    ram_wr_flag  <= '0;
    app_en       <= '1;         // set the command and wait until app_rdy is set
    app_cmd      <= 3'b001;
    app_addr     <= ram_addr_delay;
  end
```

the MIG will process the request and will notify when data is ready to be fetched.

```systemverilog
  if(ram_state == 3'b110) begin
    ram_wr_flag  <= '0;
    app_en       <= '0;
    if(app_rd_data_valid) begin
      ram_addr <= (ram_addr == 28'b1111_1111_1111_0000) ? '0 : ram_addr + 28'h10;
      fifo_rd_mem[0] <= app_rd_data[15:0];
      fifo_rd_mem[1] <= app_rd_data[31:16];
      fifo_rd_mem[2] <= app_rd_data[47:32];
      fifo_rd_mem[3] <= app_rd_data[63:48];
      fifo_rd_mem[4] <= app_rd_data[79:64];
      fifo_rd_mem[5] <= app_rd_data[95:80];
      fifo_rd_mem[6] <= app_rd_data[111:96];
      fifo_rd_mem[7] <= app_rd_data[127:112];
    end
  end
```

This is not particularly complicated to be honest, worth avoiding the AXI interface for this simple application.

### Cross Domain Clocks
As an analog designer, the different clocks from controller and host was kind of an obvious problem to be solved: at some points one clock will sample on the flip-flops when the data is not ready, resulting in what's defined as [metastability](https://en.wikipedia.org/wiki/Metastability_(electronics)). 

The trick to solve the problem? just put another flip-flop. Basically a metastable latch will take relatively some clocks to resolve the metastability (depending on the architecture) so the additional flip-flop will let the previous one time to resolve the metastability (which is an undefined state, not necessarily mid-range between zero or one). This is almost a stochastic process and usually one FF is enough to avoid audible glitches, but if that was not the case, a third one will reduce the probability of a propagating bad state to extremely low values. I discovered this is something digital designers have to give a lot of thought. For me, it was expected.

The synchronizer has to exist on the own clock domain receiving from the cross domain, so for example in the host, the piece of module will look like this:

```systemverilog
always @(posedge clk_host) begin
  ram_wr_ff1 <= ram_wr_flag;
  ram_wr_ff2 <= ram_wr_ff1;
  // wr_en edge detection
  wr_en_last <= wr_en;
  if(wr_en & ~wr_en_last) begin
    fifo_wr_mem[fifo_ptr]  <= wr_data;
    rd_data                <= fifo_rd_mem[fifo_ptr];
    sample_ready           <= wr_en;
    fifo_ptr               <= fifo_ptr + 3'b001;
  end
  else if(~wr_en & wr_en_last) begin
    // Fire the FSM only right after the host FIFO is done sampling
    // Consider the sampling happens once every ~1250 clock cycles
    // (i.e. 60MHz clock to sample 48KHz audio)
    host_wr_flag <= (fifo_ptr == '0);
  end
  else begin
    host_wr_flag <= ram_wr_ff2 ? '0 : host_wr_flag;
    sample_ready <= '0;
  end
end
```

So `ram_wr_flag` is getting generated on the MIG clock domain, but it passes through two registers `ram_wr_ff1` and `ram_wr_ff2` sampled on the host domain before being used (`ram_wr_ff2` being the useful bit, never use `ram_wr_ff1`). Same for the MIG domain, reasoning on the contrary.

## Implementation at the top
The MIG will launch a calibration step at power up, so on the top module, we have to hold the controller in reset until the calibration is done. Usually there is a flag to be read, but in my case I just wait for some time

```systemverilog
// -------------------------------------------------------------------------
// Reset - active HIGH for MIG, held until PLL locked + 200us
// -------------------------------------------------------------------------
logic [15:0] rst_cnt  = '0;
logic        mig_rst;                       // active HIGH
localparam   HOLD_CYCLES = 16'd12000;       // 200us @ 60MHz

always_ff @(posedge ck_main) begin
    if (!pll_locked)
        rst_cnt <= '0;
    else if (rst_cnt < HOLD_CYCLES)
        rst_cnt <= rst_cnt + 1'b1;
end

assign mig_rst = ~(rst_cnt < HOLD_CYCLES);  // HIGH while counting, LOW when done
```

with the actual instantiation of the module being like: 

```systemverilog
// -------------------------------------------------------------------------
// MIG instantiation
// -------------------------------------------------------------------------

// Temperature: tie to a safe mid-range constant until XADC temp
// readback is wired properly (25C aprox 12'h2C0 in Xilinx format)
localparam logic [11:0] TEMP_CONST = 12'h2C0;

mig_7series_0 u_mig (
    // DDR3 physical
    .ddr3_dq            (ddr3_dq),
    .ddr3_dqs_n         (ddr3_dqs_n),
    .ddr3_dqs_p         (ddr3_dqs_p),
    .ddr3_addr          (ddr3_addr),
    .ddr3_ba            (ddr3_ba),
    .ddr3_ras_n         (ddr3_ras_n),
    .ddr3_cas_n         (ddr3_cas_n),
    .ddr3_we_n          (ddr3_we_n),
    .ddr3_reset_n       (ddr3_reset_n),
    .ddr3_ck_p          (ddr3_ck_p),
    .ddr3_ck_n          (ddr3_ck_n),
    .ddr3_cke           (ddr3_cke),
    .ddr3_cs_n          (ddr3_cs_n),
    .ddr3_dm            (ddr3_dm),
    .ddr3_odt           (ddr3_odt),

    // Clocks and reset
    .sys_clk_i          (ck_100),
    .clk_ref_i          (ck_200),
    .sys_rst            (mig_rst),
    .ui_clk             (ui_clk),
    .ui_clk_sync_rst    (ui_rst),
    .init_calib_complete(calib_done),

    // User interface - placeholder, delay FSM goes here
    .app_addr           (app_addr),
    .app_cmd            (app_cmd),
    .app_en             (app_en),
    .app_rdy            (app_rdy),
    .app_wdf_data       (app_wdf_data),
    .app_wdf_wren       (app_wdf_wren),
    .app_wdf_end        (app_wdf_wren),   // always last word for BL8
    .app_wdf_mask       (16'h0000),        // write all bytes
    .app_wdf_rdy        (app_wdf_rdy),
    .app_rd_data        (app_rd_data),
    .app_rd_data_valid  (app_rd_data_valid),
    .app_rd_data_end    (app_rd_data_end),

    // Maintenance - tied off
    .app_sr_req         (1'b0),
    .app_ref_req        (1'b0),
    .app_zq_req         (1'b0),
    .app_sr_active      (),
    .app_ref_ack        (),
    .app_zq_ack         (),

    // Temperature
    .device_temp_i      (TEMP_CONST),
    .device_temp        ()
);
```
remember I wanted the XADC module exclusive for the front end, hence the temperature is just hard-coded into the module and we disregard the effect on the calibration for this application.

## Conclussion

This is almost a recipe for a MIG implementation, where the key lessons are
* Next time, separate the controller from the user. In the way I did, I can't really use the memory for something else
* The CDC was an expected behavior, with an intuitive solution that happened to be also the standard solution
* As the project increases in complexity, I will be needing some way for testing (DFT). I will be writing about my solution on this in the next blog