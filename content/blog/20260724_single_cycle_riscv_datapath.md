---
author: "Daniel Blackbeard"
date: '2026-07-24T10:16:21+02:00'
draft: false
title: 'Single Cycle RISCV32I Datapath'
cover:
  image: "images/single_cycle_fpga.jpg"
  alt: "Picture a scientific paper and an Arty A7"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
tags: 
- hdl
- arch
- log
---

## Introduction
 
Perhaps it's only me, but I believe anyone who is serious at digital design should at least try to implement a simple processor. This is the first of a series aimed to end with a usable processor for my projects, in an attempt to learn some of the concepts of computer architecture, while at the same time honing my hardware design skill. This post is a technical overview of the design choices behind it: where I made reasonable calls, where I got it wrong, and what I learned along the way.
 
## Why single-cycle, why RISC-V
 
Out of every digital machine you could build, a CPU is arguably the one with the most day-to-day impact on people's lives — not necessarily the most complex circuit out there, but the one that's everywhere. I'd read Hennessy & Patterson years earlier, but only recently got my hands on hardware to actually build one. Rather than follow someone else's RTL, I extracted the logic directly from the RISC-V unprivileged ISA specification, trying to guess at the right microarchitectural blocks as I went. Decoding the spec into hardware — not writing the Verilog itself — turned out to be the real challenge. So single cycle is the most spontaneous piece of hardware that can test my understanding and the correctness of my implementation. As to why RISCV, simply because it's open and simple, the 32bit integer specification is no more than 20, quite detailed, pages you can find at the [unprivileged specification](https://docs.riscv.org/reference/isa/_attachments/riscv-unprivileged.pdf)

The project: SystemVerilog, targeting my Arty A7 board. I haven't run it on real hardware yet — everything so far has been verified in simulation and disregarding for now about timing optimizations.
 
## Verification approach
 
For the smaller building blocks (register file, ALU, memory), I wrote basic self-checking testbenches. For the datapath as a whole, I built one big self-checking testbench that ran actual programs — compiled with a real RISC-V toolchain, not hand-written instruction sequences — and checked that memory ended up in the state the program expected. Working from a real compiled binary instead of hand-picked instructions caught a class of bugs that manually-crafted test vectors would have missed entirely.
 
```systemverilog
task automatic check(string name, logic [31:0] actual, logic [31:0] expected);
    checks++;
    if (actual !== expected) begin
        errors++;
        $display("FAIL [%0d] %-22s got=0x%08h expected=0x%08h", checks, name, actual, expected);
    end else begin
        $display("PASS [%0d] %-22s = 0x%08h", checks, name, actual);
    end
endtask

check("x1  (10)",           u_riscv32_dut.u_reg.regs[1],  32'd10);
check("x2  (20)",           u_riscv32_dut.u_reg.regs[2],  32'd20);
check("x3  (add)",          u_riscv32_dut.u_reg.regs[3],  32'd30);
check("x6  (load)",         u_riscv32_dut.u_reg.regs[6],  32'd50);
check("x8  (load)",         u_riscv32_dut.u_reg.regs[8],  32'd51);
check("x0  (write-blocked)",u_riscv32_dut.u_reg.regs[0],  32'd0);
check("x10 (x0 read-back)", u_riscv32_dut.u_reg.regs[10], 32'd0);
check("x11 (post-beq)",     u_riscv32_dut.u_reg.regs[11], 32'd111);
check("x12",                u_riscv32_dut.u_reg.regs[12], 32'd5);
check("x13 (post-bne)",     u_riscv32_dut.u_reg.regs[13], 32'd222);
check("x14 (-3)",           u_riscv32_dut.u_reg.regs[14], 32'hFFFFFFFD);
check("x15 (post-blt)",     u_riscv32_dut.u_reg.regs[15], 32'd333);
check("x16 (post-bge)",     u_riscv32_dut.u_reg.regs[16], 32'd444);
check("x17 (post-bltu)",    u_riscv32_dut.u_reg.regs[17], 32'd555);
check("x19 (post-jal)",     u_riscv32_dut.u_reg.regs[19], 32'd666);
check("x20 (beq skip)",     u_riscv32_dut.u_reg.regs[20], 32'd0);
check("x21 (bne skip)",     u_riscv32_dut.u_reg.regs[21], 32'd0);
check("x22 (blt skip)",     u_riscv32_dut.u_reg.regs[22], 32'd0);
check("x23 (bge skip)",     u_riscv32_dut.u_reg.regs[23], 32'd0);
check("x24 (sltu)",         u_riscv32_dut.u_reg.regs[24], 32'd1);
check("x25 (bltu skip)",    u_riscv32_dut.u_reg.regs[25], 32'd0);
check("x26 (jal skip)",     u_riscv32_dut.u_reg.regs[26], 32'd0);
check("x30 (jalr skip)",    u_riscv32_dut.u_reg.regs[30], 32'd0);

check("x31 == x27 (jalr &~1)", u_riscv32_dut.u_reg.regs[31], u_riscv32_dut.u_reg.regs[27]);

check("mem[0] (sw x5)", u_riscv32_dut.u_data_mem.ram[0], 32'd50);
check("mem[4] (sw x7)", u_riscv32_dut.u_data_mem.ram[1], 32'd51);
```

While not extensive as the real tests, it will stress many tricky instructions like the conditional branches and jal/jalr requirements.

## Block diagram
 
Before getting into individual blocks, here's how they connect in the single-cycle datapath:

{{< figure src="/images/riscv_single_cycle_block_diagram.png" caption="Block diagram of a single cycle RISCV without control unit" alt="Block diagram of a single cycle RISCV without control unitr" align="center">}}

 
At a glance: the PC drives instruction fetch, the control unit decodes the instruction into every downstream select signal, the register file and immediate decoder feed the ALU, and the ALU's result branches out to either data memory (loads/stores) or back into the register write-back mux — all within a single clock cycle, which is exactly what makes this a *single-cycle* design: no pipeline registers, no stalls, just one long combinational path from fetch to write-back on every clock edge.
 
## Design choices
 
### Program Counter
 
The PC is about as simple as sequential logic gets, but it's a good entry point into how control flow decisions ripple through the datapath:
 
```systemverilog
always_ff @(posedge clk or negedge rstb) begin
  if (~rstb) pc <= 15'b0;
  else       pc <= pc_next;
end
 
always_comb begin
  case(pc_src)
    2'b00: pc_next = pc_plus4;                         // normal
    2'b01: pc_next = pc_plus_imm;                      // JAL
    2'b10: pc_next = alu_res & ~1;                     // JALR
    2'b11: pc_next = branch ? pc_plus_imm : pc_plus4;  // conditional branches
  endcase
end
 
assign pc_plus4    = pc + 32'd4;
assign pc_plus_imm = pc + imm_out;
```
 
`pc_src` is a 2-bit control signal that selects between four sources for the next PC value: sequential execution, JAL's PC-relative jump, JALR's register-relative jump (masking bit 0 per spec), and conditional branches gated by the ALU's `branch` output.
 
Note the two dedicated adders, `pc_plus4` and `pc_plus_imm`, sitting right next to the ALU. That wasn't the first version of this design — more on that below.
 
### Register File
 
```systemverilog
logic [31:0] registers[31:0];
 
always_comb begin
  rs1_data_r = registers[addr_rs1];
  rs2_data_r = registers[addr_rs2];
  rd_data_r  = registers[addr_rd];
end
 
always @(posedge clk) begin
  if(~rstb) begin
    for(int i=0; i < 32; i++)
      registers[i] <= 32'b0;
  end
  else if(wen) begin
    registers[addr_rd] <= (addr_rd == 5'b0) ? 32'b0 : rd_data_w;
  end
end
```
 
Two read ports (`rs1`, `rs2`) decoded combinationally straight off the instruction fields, one write port gated by `wen`. The one detail worth calling out for anyone implementing this for the first time: register x0 has to read as zero *and* stay zero on writes — the spec requires it, and it's an easy thing to forget to enforce on the write path specifically, since the read path naturally returns whatever is stored at index 0.
 
### Data Memory
 
```systemverilog
always_comb begin
  case(d_width)
    2'b01   : dout = mem_signext ? {{24{ram[addr][7]}},ram[addr]} : {24'b0,ram[addr]};
    2'b10   : dout = mem_signext ? {{16{ram[addr+1][7]}},ram[addr+1],ram[addr]} : {16'b0,ram[addr+1], ram[addr]};
    2'b11   : dout = {ram[addr+3],ram[addr+2],ram[addr+1],ram[addr]};
    default : ;
  endcase
end
```
 
Memory is modeled as a byte array, with `d_width` selecting byte/halfword/word access and `mem_signext` controlling sign- vs zero-extension on loads — directly mirroring RISC-V's `LB`/`LBU`, `LH`/`LHU`, and `LW` distinctions. Writes follow the same width logic on the synchronous side. This was my first pass at a memory block, so it's more of a working baseline than a polished design — but it got the datapath running correctly against compiled test programs.
 
## The AUIPC lesson: not everything belongs in the ALU
 
The one moment that really stuck with me: I got stuck trying to make AUIPC work *through* the ALU, treating it like just another instruction that needed `op1 + op2` in the same functional unit as every other ALU operation. It didn't click for a while that PC-relative addition doesn't have to go through the ALU at all — it's arguably not even an "ALU operation" conceptually, it's PC arithmetic.
 
The fix ended up being the two dedicated adders you can see in the PC section above: `pc_plus4` for sequential fetch and `pc_plus_imm`, reused for both AUIPC's `PC + imm` and JAL's target calculation. Once I stopped trying to force every addition through one shared unit, the design actually got simpler, not more complex — a dedicated adder is cheap, and it decouples PC-relative arithmetic from whatever the ALU is doing that cycle.
 
It's a small thing, but it's the kind of assumption ("reuse the general-purpose block for everything") that's worth questioning early — especially coming from an analog background where dedicated, purpose-built circuits are often the default instinct anyway.
 
## Conclusion
 
This single-cycle design became the baseline for the pipelined RISCV32I CPU I'm currently building — same ISA, same lessons carried forward, very different set of problems. That comparison is next up.