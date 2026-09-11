---
author: "Daniel Blackbeard"
date: '2026-09-10T19:00:00+02:00'
draft: false
title: 'Pipelined RISCV32I: Closing Timing to 100MHz'
cover:
  image: "images/riscv_arty.jpg"
  alt: "Picture of some HDL code"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
tags:
- hdl
- arch
- log
---

## Closing a loop

The [single-cycle RISCV32I post]({{< ref "blog/20260724_single_cycle_riscv_datapath.md" >}}) ended with a promise: "this single-cycle design became the baseline for the pipelined RISCV32I CPU I'm currently building... that comparison is next up." That comparison took longer than expected to actually write up — not because the work stalled, but because the pipeline got finished, timing got closed, and I moved on before circling back to put it into words. This is that write-up, reconstructed from the actual RTL and the real Vivado timing reports rather than from memory, because at this point the project genuinely isn't fresh anymore. I'd rather tell it accurately from the evidence than paraphrase what I think I remember.

The short version: five stages, full forwarding, a load-use hazard detector, and a hard self-imposed constraint — 100MHz on an Arty A7-100T, against the board's own onboard clock, no PLL involved. Getting there took a real detour through a wrong fix, a genuine root-cause diagnosis that turned out to be bigger than RTL cleanup could solve, and a handful of deliberate architectural and encoding changes to close the gap — some of which quietly simplified other parts of the design as a side effect, which wasn't the point but was a nice thing to notice afterward.

## Why 100MHz, specifically

I set that number myself, not because the design forced it — the target had to be reached in some way, and 100MHz matching the board's own oscillator was the obvious, unambiguous line to hold rather than a number I'd drift away from once it got inconvenient. Once it's a real constraint instead of an aspiration, it does something useful: it turns "is this fast enough" from a vague feeling into a yes/no line in a timing report.

## Finding the real critical path

A code review pass over the pipeline RTL turned up the usual assortment of real, worth-fixing issues before timing analysis even started: the ALU's `flags` output left completely unconnected at the top level (legal SystemVerilog, but a linter would flag it immediately), a reset-coverage gap where the data memory's block-RAM array had no `$readmemh`/clear on `~rstb`, and — recurring from the single-cycle project's own hazard logic — the load-use detector missing an `x0` qualification:

```systemverilog
//assign stall = (curr_inst_type == L_TYPE) & ((next_rs1 == curr_rd) | (next_rs2 == curr_rd));
assign stall = (curr_inst_type == L_TYPE) & (curr_rd != 5'b0) & ((next_rs1 == curr_rd) | (next_rs2 == curr_rd));
```

Without that `curr_rd != 5'b0` term, `lw x0, 0(x6)` followed by anything reading `x0` — which is extremely common, since `x0` doubles as a literal-zero source in branches — would trigger a completely unnecessary stall bubble every single time, treating an architecturally-void write as a real hazard.

None of that was the actual timing problem, though. Tracing the worst path in the post-synthesis report by hand found the real one: forwarding-mux select logic feeding into the ALU, whose result fed a branch decision, whose result fed the `pc_next` mux — all landing, in the same clock cycle, on the **synchronous reset pin** of `reg_id_ex`'s roughly 45 flip-flops. Vivado had correctly recognized "clear this register to a bubble value when flush is asserted" as a synchronous-reset pattern and wired the `flush` net directly to every one of those SRST pins. One logical signal now had to physically reach 45-plus flip-flops scattered across the module — a fanout of over 70 on the final net — and that routing, not the logic depth, was eating 83% of the delay.

## A bug inside the fix

The first real attempt at shortening that path was structural: instead of re-computing a 5-way `reg_src` mux twice, once for MEM-stage forwarding and again for the final register write, collapse it into one canonical result field computed once, right where the ALU result is first produced, and carry just that one value forward. Good idea in principle. The first implementation of it introduced a real bug:

```systemverilog
always_comb begin
  case(mem_wb_ctrl.reg_src)        // WRONG — this belongs to the MEM-stage instruction
    REGSRC_ALU    : ex_result = alu_res;         // these are all EX-stage (current-cycle) values
    REGSRC_PC_4   : ex_result = ex_pc_plus_4;
    REGSRC_PC_IMM : ex_result = pc_plus_imm;
    REGSRC_IMM    : ex_result = ex_imm;
    default       : ex_result = alu_res;
  endcase
end
```

The selector was reading `mem_wb_ctrl.reg_src` — the control field belonging to whatever instruction currently sits in MEM — to choose between values that all belong to the instruction currently in EX. Whenever the EX and MEM instructions didn't happen to share the same `reg_src` encoding, the mux would silently pick the wrong candidate, breaking register writeback and both forwarding paths for most instruction types. Caught before trusting any timing number that came out of that build — which mattered, because a timing report generated against broken logic proves nothing about the design you actually want to ship.

Worth saying plainly: the `ex_result` idea itself never made it into the final design. It got abandoned once the real fix — described below — closed timing on its own. The final `reg_ex_mem` still carries `alu_res`, `pc_plus_4`, `pc_plus_imm`, and `imm` as four separate fields, exactly like the original structure. Not every good-sounding optimization survives contact with the actual bottleneck.

## The diagnosis: not really an RTL problem

With the selector bug fixed and the flush fanout narrowed down toward just the bits that actually needed gating (`reg_w_en`, `mem_w_en`, the load-hazard-relevant type info, and `pc_src`), the numbers barely moved: **WNS -5.427ns post-route**, same source, same "lands on a bubble-insertion reset pin" pattern, just migrating to a different bit of the same flush-gated set. Roughly half the clock period, averaged across 364 to 366 failing endpoints.

That magnitude is the actual tell. Even a perfectly narrow, minimal-fanout flush signal still has to pay for a real 32-bit ALU operation plus forwarding-select logic plus the OR gate merging it into `flush`, before it can reach *any* register — realistically 8 to 10 logic levels at minimum, which is already several nanoseconds on an Artix-7 once routing is accounted for. Local RTL cleanup was never going to fully close a half-period gap. The actual options were a deeper pipeline — resolving branches a full stage later, trading a bigger misprediction penalty for a shorter same-cycle path — or accepting a slower clock.

100MHz wasn't the part I was willing to move. Among the valid ways to get there — a longer pipeline being the more general one — resolving branches one stage later was the most immediate, so that's the one I actually built.

## Moving resolution to MEM

Branch and jump resolution moved from EX to MEM. `pc` and `flush` now key off the *registered* MEM-stage signals instead of the raw EX-stage combinational output:

```systemverilog
always_comb begin
    case(mem_if_ctrl.pc_src)
        PCSRC_PC_4   :  pc = pc_plus_4;
        PCSRC_PC_IMM :  pc = mem_pc_plus_imm;                        // JAL
        PCSRC_ALU    :  pc = {mem_alu_res[31:1], 1'b0};               // JALR
        PCSRC_BRANCH :  pc = mem_alu_branch ? mem_pc_plus_imm : pc_plus_4;
        default      :  pc = pc_plus_4;
    endcase
end

assign flush = (mem_alu_branch | mem_inst_type == JAL_TYPE | mem_inst_type == JALR_TYPE);
```

That one extra pipeline register between "the ALU has decided" and "the PC has to act on it" is what actually bought back the slack — the forward-mux-to-ALU-to-branch chain no longer has to land on a wide-fanout reset net in the same cycle it's computed. The real cost is a fatter misprediction penalty: a taken branch or JAL/JALR now flushes three stages — `reg_if_id`, `reg_id_ex`, and `reg_ex_mem` — instead of two. I'm naming that plainly rather than glossing over it: it's a genuine, deliberate trade, correctness and Fmax bought at the price of more bubbles on every taken branch. For this project, that's the right trade. Nothing here is latency-sensitive in a way that makes a bigger branch penalty actually expensive, and 100MHz was the constraint I'd already decided not to negotiate on.

## A side effect: the stall signal stopped needing to travel

Something I only noticed going back through the diff after the fact: moving branch resolution to MEM changed how the PC itself gets computed, and that change quietly made the load-use stall a lot cheaper to plumb everywhere else in the design.

In the earlier pipeline, `pc` was a real register holding the current fetch address, explicitly held during a stall: `pc <= stall ? pc : pc_next`. `instruction_memory` needed its own `stall` port to hold its output steady. `reg_if_id` needed a `stall` input to hold `o_pc` steady. Three separate modules, three separate places a stall had to be threaded through correctly.

In the current design, `pc` is purely combinational, computed off the already-registered MEM-stage decision — and the one place stall actually matters is a single line upstream of everything else:

```systemverilog
assign pc_plus_4 = stall ? pc_next : pc_next + 4;
```

When stalled, the combinational `pc` output simply holds at its current value instead of advancing. Because the address handed to the instruction ROM never changes during a stall, the ROM keeps re-reading the exact same word — freezing the fetched instruction without the ROM needing to know a stall is happening at all:

```systemverilog
// instruction_memory.sv — no stall port anymore
always_ff @(posedge clk) begin
    inst <= rom[addr[INST_MEM_ADDR_SIZE-1:2]];
end
```

`reg_if_id` shrank the same way — no `stall` port, no `flush` port, just a plain pass-through:

```systemverilog
module reg_if_id(
    input  logic                          clk,
    input  logic                          rstb,
    input  logic                   [31:0] i_pc,
    input  logic                   [31:0] i_inst,
    output logic                   [31:0] o_pc,
    output logic                   [31:0] o_inst
);

always_ff @(posedge clk) begin
    o_pc <= i_pc;
end

assign o_inst = i_inst;

endmodule
```

That's not a coincidence, and it's not something I set out to do — it fell out of fixing the actual bottleneck. Freezing the PC at the source meant every downstream consumer of "what instruction is currently being fetched" saw a naturally stable value during a stall, without needing to be told to hold. Two modules stopped needing a control input entirely, not because I went looking to simplify them, but because the thing they were protecting against stopped being able to happen.

## The register file went back to being boring, on purpose

The earlier pipeline had made the register file's read ports synchronous — matching how block RAM naturally behaves — which required a genuinely clever but fragile alignment: the register file's own output flop supplied the one cycle of delay that `rs1`/`rs2` needed to line up with everything else crossing the ID/EX boundary, so `reg_id_ex` bypassed its own register for just those two fields and passed them through combinationally instead. It worked, but it only worked because two separate modules' timing happened to line up exactly right, which is a subtle thing to have to keep true every time either module changes.

Once the real bottleneck turned out to be the branch-resolution chain instead, that cleverness stopped earning its keep. The final register file went back to a plain combinational read — the same shape the single-cycle project always used:

```systemverilog
logic [31:0] regs[0:31];

always_ff @(posedge clk) begin
    if(~rstb) begin
        for(int i=0; i<32; i++) begin
            regs[i] <= '0;
        end
    end
    else if(wr_en) begin
        regs[addr_rd] <= (addr_rd == 5'b0) ? '0 : wr_reg;
    end
end

assign rs1 = regs[addr_rs1];
assign rs2 = regs[addr_rs2];
```

And `reg_id_ex` picked up the one honest register that used to be split across two modules:

```systemverilog
o_rs1 <= i_rs1;
o_rs2 <= i_rs2;
```

A 32-entry register file is small enough that a plain async read was never going to be the thing standing between this design and 100MHz. I'd rather have the conventional, easy-to-reason-about version once I know that for certain, instead of carrying a clever structure that was solving a problem that had already moved somewhere else.

## Decoupling the branch decision from the adder

One more change worth showing on its own, because it attacks the same critical path from a different angle: the ALU's comparison flags stopped being derived from the arithmetic result.

Previously, a branch instruction ran through the ALU configured for `SUB`, and the branch condition was read off the subtraction's own zero flag and sign bit — meaning "is this branch taken" had to wait for a full 32-bit subtractor to finish, even though all a comparison actually needs is a compare, not a subtraction. The final ALU computes the flags directly off the raw operands instead, independent of whichever operation `op_sel` actually selects that cycle:

```systemverilog
// Flags — all computed directly from op1/op2, independent of op_sel/res
assign flags[3] = (op1 == op2);                                // zero / equal
assign flags[2] = ($signed(op1) < $signed(op2));                // slt
assign flags[1] = (op1 < op2);                                  // sltu
assign flags[0] = (op1[31] == op2[31]) && (res[31] != op1[31]); // overflow (arith-only, not on branch path)
```

A comparator is a shorter combinational structure than a subtractor followed by a zero-detect, and — more importantly here — it no longer shares logic with whatever the ALU's actual arithmetic result is doing that cycle. The branch decision comes straight off the two operands, in parallel with (rather than downstream of) the adder. On a path that was already the worst offender in the design, shortening one more link in that specific chain is exactly the kind of thing worth doing even after the bigger architectural fix has already closed timing — belt and suspenders on the one path that's already caused the most trouble.

## Every control signal became one-hot

The last change, and the most visible one if you diff the two packages side by side: every enumerated control type in `riscv_pkg.sv` moved from a compact binary encoding to a one-hot encoding.

```systemverilog
// riscv_pipeline_base
typedef enum logic [3:0] {
  ADD  = 4'b0000,
  SUB  = 4'b0001,
  AND  = 4'b0010,
  OR   = 4'b0011,
  XOR  = 4'b0100,
  SLL  = 4'b0101,
  SRL  = 4'b0110,
  SRA  = 4'b0111,
  SLT  = 4'b1000,
  SLTU = 4'b1001
} alu_op_t;

// riscv_pipeline_improved
typedef enum logic [9:0] {
  ADD  = 10'b00_0000_0001,
  SUB  = 10'b00_0000_0010,
  AND  = 10'b00_0000_0100,
  OR   = 10'b00_0000_1000,
  XOR  = 10'b00_0001_0000,
  SLL  = 10'b00_0010_0000,
  SRL  = 10'b00_0100_0000,
  SRA  = 10'b00_1000_0000,
  SLT  = 10'b01_0000_0000,
  SLTU = 10'b10_0000_0000
} alu_op_t;
```

Same treatment landed on `pc_src_t`, `reg_w_src_t`, `mem_d_size_t`, `mem_d_sign_t`, `alu_comp_t`, and the forwarding-select type `reg_fw_t`. Ten bits of register state to encode ten ALU operations is objectively wasteful by the usual metric — the resource report shows 1458 flip-flops used against a device with over 126,000 available, so the extra bits cost nothing that mattered here. What one-hot buys back is decode simplicity: every `case` arm downstream of one of these signals collapses from "match an N-bit binary pattern" into "check whether one specific bit is set," which is a narrower, shallower piece of logic than a real binary comparator, repeated at every single consumer of that signal throughout the datapath. On a design where the critical path was already about logic depth stacking up in front of a wide-fanout net, trading flip-flop bits — which this device has in huge surplus — for shallower decode logic at every consumer is close to a free lunch.

## Where it landed

```
    WNS(ns)      TNS(ns)  TNS Failing Endpoints  TNS Total Endpoints
      0.485        0.000                      0                 2931

All user specified timing constraints are met.
```

Post-synthesis comes in a little tighter at +0.427ns, same story. Resource usage on the XC7A100T: 1403 LUTs (2.21%) and 1458 flip-flops (1.15%) — plenty of headroom left on the board for whatever comes next. And the regression suite — the same stress test that exercises EX/MEM and MEM/WB forwarding, both directions of the load-use hazard, every branch condition, JAL, and the JALR `&~1` edge case that a naive target-address mask gets wrong — still passes clean, all 31 checks, on the exact design that closed timing:

```
PASS [4]       x4  (EX/MEM fwd) = 0x0000003c
PASS [5]       x5  (MEM/WB fwd) = 0x00000032
PASS [7]     x7  (load-use rs1) = 0x00000033
PASS [9]     x9  (load-use rs2) = 0x0000003d
PASS [28]  x31 == x27 (jalr &~1) = 0x0000009c
=== ALL 31 CHECKS PASSED ===
```

Later drafts of the stress test even carry hand-annotated program-counter comments next to the tricky instructions — `# PC=0x34`, `# PC=0x70` — the kind of detail that only shows up when you're cross-checking expected behavior against a real disassembly rather than trusting the assembler's output blind.

## One thing I noticed and want to flag rather than paper over

Going through the final data memory for this write-up, I found something I'm genuinely not certain is intentional. The earlier pipeline's data memory split storage into four separate byte-lane block RAMs with real per-lane write masking and a downstream sign/zero-extension mux, correctly supporting `LB`/`LH`/`LBU`/`LHU`/`SB`/`SH` alongside full-word access. The final data memory is a single 32-bit-wide block RAM with no width or sign-extension handling left in the module at all:

```systemverilog
module data_memory(
    input  logic                          clk,
    input  logic                          w_en,
    input  logic [DATA_MEM_ADDR_SIZE-1:0] addr,
    input  logic                   [31:0] data_in,
    output logic                   [31:0] data_out
);
// RISCV addressing is per byte, however this module will return an entire word of 4bytes
// Processing of this word happens outside this block
```

— except nothing downstream in the current `riscv_datapath.sv` actually does that processing anymore; `data_out` feeds straight into the writeback mux unchanged. The regression suite never catches this because it only ever exercises `sw`/`lw`. I don't know, from the evidence alone, whether this was a deliberate scope-narrowing — "prove word-aligned access closes timing first, byte/half support can come back later" — or a gap that fell out of some other refactor and just never got noticed because nothing exercises it. I'd rather flag it honestly than guess at a tidy explanation, and it's worth deciding before this is the version anyone builds on: either restore the byte-lane handling, or note explicitly that this iteration is word-access-only by design.

## Where this leaves the CPU thread

I've said before that the ALU was a genuine childhood fixation — the whole reason I got into any of this in the first place. Building an actual CPU by hand, single-cycle through pipelined, closing real timing on real hardware, is that thread actually closed out rather than left as a someday-project. I don't have a specific next step queued up for this line of work, and I'm fine leaving it there for now. If I need real CPU horsepower for something serious later, I'll reach for a consolidated, production-grade core or a proper SoC rather than extending this one further — this project did what it was for. The interest might come back around someday. For now, it's done, and it's nice to be able to say that plainly.
