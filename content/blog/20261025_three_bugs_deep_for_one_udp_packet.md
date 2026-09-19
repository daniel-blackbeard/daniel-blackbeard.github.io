---
author: "Daniel Blackbeard"
date: '2026-10-25T19:00:00+02:00'
draft: true
title: 'Three Bugs Deep to Send One UDP Packet'
tags:
- hdl
- log
---

## A testbench that passed, and hardware that said nothing

With the DDR/D-cache saga finally closed, I moved on to `axi_dsp` — the PL write-master module responsible for streaming decimated I/Q samples from the AD9361 straight into DDR. Its testbench passed clean: 3160 checks, zero failures. On actual hardware it produced exactly zero real notifications. Not flaky — zero, permanently, no matter how long the board ran.

It took three real, independent bugs stacked on top of each other before a single sample-stream packet ever reached a PC, and each one was only visible once the previous one was out of the way.

## Bug one: a flip-flop with two drivers

`axi_dsp.sv` had a reset-branch assignment setting `pl_wen` to zero, with no other assignment to it anywhere in that same clocked block — and, separately, a combinational assignment to the same signal at module scope. That's legal-looking and actually illegal: two drivers on the same net. Vivado's synthesizer resolved the conflict by keeping the flip-flop, which, driven only by that one reset-time assignment, synthesizes to a permanent constant zero, and silently discarded the real logic. The warning was sitting in the synthesis log the whole time:

```
CRITICAL WARNING: [Synth 8-6859] multi-driven net on pin Q with 1st driver
pin 'u_axi_dsp/pl_wen_reg/Q'
CRITICAL WARNING: [Synth 8-6858] multi-driven net Q is connected to at
least one constant driver which has been preserved, other driver is
ignored
```

Simulation resolved the same conflict the other way, which is exactly why 597 passing testbench runs never caught it — a genuine simulation-versus-synthesis mismatch from one illegal construct that both tools happily accepted without an error, just opposite silent resolutions.

Getting to that finding took two earlier, more exotic hypotheses first: a missing clock-domain reset synchronizer, then a same-cycle set/clear race in the handshake logic. Both were real, defensible things worth fixing on their own merits. Neither was the actual bug. What actually made it obvious was building a small internal-state debug tap, mirroring the module's FSM state out into spare status registers — since the PL write-master's own address counter turned out to be advancing beautifully the entire time, the notification logic itself was what was dead on arrival.

## Bug two: fixing a real problem broke a different rule

Giving sample-stream packets their own dedicated range of GEM TX descriptors, separate from the existing single-buffer console/ARP/reply ring, correctly avoided a genuine hazard: the hardware only ever reports completion on a multi-descriptor frame's *first* descriptor, so mixing 1- and 2-descriptor allocations on one shared producer index can't reliably tell whether a "second descriptor" slot is actually free.

But GEM walks its descriptor list strictly sequentially with zero ability to skip a dormant region, and since nothing was exercising the original ring anymore, its queue pointer got permanently stuck the first time it reached that region. I'd called this one before it was even confirmed:

> "I suspect that would be an issue eventually. Try your solution, let's see."

The fix: unify both traffic types onto one ring, where every frame — rare console replies and continuous sample-stream packets alike — always consumes exactly two descriptors. Uniform occupancy solves the first-descriptor-only-completion problem; one shared, constantly-rotating ring, kept warm by the frequent sample-stream traffic, solves the dormant-region problem. Both fixes were needed together. Either alone was insufficient.

## Bug three: the CPU and the network chip reading two different memories

Even with the ring unified, GEM kept halting on descriptors that looked perfectly armed every time the CPU read them back. They were correctly armed — in the CPU's data cache. GEM reads DDR directly, bypassing the cache entirely, and the region backing GEM's descriptors and frame buffers was still marked cacheable. A descriptor the CPU had "written" could sit dirty in cache indefinitely while GEM kept reading stale content straight out of physical memory — structurally the same class of bug as the sample-streaming buffer's own cache-coherency fix earlier the same day, just not yet applied to this particular region. The original single-descriptor ring likely got away with it purely by luck: slow, on-demand console traffic left enough idle time for the cache to flush naturally, while continuous high-rate sample-stream traffic removed that luck entirely.

This is also where I hit the edge of my own domain knowledge and asked for it plainly:

> "This is not my real[m], so explain to me like if I was stupid what exactly the issue is."

The explanation that followed led directly to my own, independently-reasoned proposed fix:

> "let's try some stupid given your explanation to stupid me: first flag as ready the descriptors associated to the data from dsp, then the descriptor for the udp header. That was it will not attempt the packet until every single component is ready. Might this work?"

It turned out the code already did exactly that ordering, which is what prompted a closer look at what "ordering" even meant here, and surfaced the real, different-in-kind cache-visibility bug underneath. Even mid-fix, I pushed back on a specific, hard-won piece of this project's own history before letting the change happen:

> "apply it, but I think we used d-cache to solve mysterious hangs of the board, let's see what happens."

The two fixes don't actually conflict — the earlier one, [covered a couple of posts back]({{< ref "blog/20261018_the_bit_nobody_had_flipped.md" >}}), was about an uncached stack causing crashes; this one is about a cached DMA region going stale — but it was worth making that connection explicit before letting it happen, given how much of this project's history that exact subsystem has already cost.

With all three fixed: over 100,000 real UDP packets landed on the receiving PC in a 15-second capture window, correctly addressed, at sustained high volume. The digital chain from AD9361 sample to a packet on my desk actually works, end to end, for the first time.

## The real shape of the day

The narrative spine here isn't any one of the three bugs, it's the shape of the investigation: three bugs, each one completely invisible until the one in front of it was cleared, two of them only reachable by first landing on a plausible-but-wrong hypothesis and ruling it out with real hardware evidence instead of reasoning alone. One more thing worth a callout: a recurring, separate red herring ran through this entire session, unrelated to any of the three real bugs — reprogramming a new bitstream without power-cycling the board reliably wedged the JTAG debug port, costing several power-cycle round-trips before I explicitly called a stop to chasing it: "let's go back to the main issue, forget about looking for frequent power cycling reason." Knowing which mystery is worth solving right now, and which one is a problem for later, turned out to be its own skill on this project — right up there with knowing how to actually find the bug once you've decided to chase it.

That's where the digest I've been working from ends for now — this project moves fast enough that there's already more material piling up behind it. I'll keep posting as the next stretch of bring-up happens.
