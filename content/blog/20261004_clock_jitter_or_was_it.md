---
author: "Daniel Blackbeard"
date: '2026-10-04T19:00:00+02:00'
draft: true
title: 'Clock Jitter, or Was It: Three Days Chasing a Ghost in the Ethernet Stack'
tags:
- hdl
- log
---

## Not actually over

The RX ring-exhaustion bug from the midnight mystery [in the last two posts]({{< ref "blog/20260927_longest_day_part2_rx_and_midnight_mystery.md" >}}) was real, and the fix was real: drain every ready frame per pass instead of one, and explicitly acknowledge the hardware's "buffer not available" condition. On top of that, my own code review turned up three more independently well-motivated hardening fixes: a UDP command preamble/magic-number guard, since any stray packet matching the destination IP and port alone could previously have its payload blindly executed as register commands; SOF/EOF-based fragment rejection in the RX poll path; and a real 1536-byte Ethernet-frame RX buffer stride, making multi-descriptor RX structurally impossible. All three shipped.

The crash did not go away.

## A day of disproving good theories

What followed was methodical, and notable mainly for how many reasonable theories died against direct hardware evidence rather than argument. "JTAG-reload state accumulation, needs a real power cycle" — directly disproved: a genuine cold power cycle plus a completely fresh reload, cable disconnected, still crashed within about five minutes, identical signature. An earlier 40-plus-minute clean run got reframed, once that landed, as just an outlier rather than evidence of anything — and that turned out to be the correct read, since the eventual explanation is a rare, memoryless-ish event where wide run-to-run variance is exactly what you'd expect. Two more source-level hypotheses, both raised with real citations, both evaluated and set aside: an unbounded MMIO address in the command dispatcher (real, independently worth fixing, but the crash reproduces with zero commands ever sent, so it can't be the mechanism), and cacheable, unmaintained DMA buffers (ruled out directly, since the D-cache was confirmed disabled the whole time — the specific "dirty line never reaches DRAM" mechanism simply can't occur if there's no D-cache active to hold a dirty line in the first place).

The day ended on an explicit, unresolved note: "not our bug. I'm out of ideas for now, I will have to sleep over this." A full day of good theories dying one after another, with nothing to show for it by evening, is a real and underrepresented shape of debugging, and I'd rather show that honestly than skip straight to the eventual answer.

## Bisection: removing things until nothing is left to remove

The next stretch is the most methodologically distinctive debugging in the project's history so far: instead of hunting for what's wrong, systematically prove what *isn't* required, one controlled change at a time, each verified on real hardware before moving to the next. In order, all confirmed structurally dead and removed without changing the crash: a call-tracing macro, the internal fragment-discard path inside RX poll, the descriptor ownership check itself, replaced with a hardcoded value, and finally the entire loop body down to a bare early return. A parallel synthetic rewrite — generic functions named purely by call-stack depth, at my own request, specifically to strip away any doubt that something about the real Ethernet driver's code or address layout mattered — reproduced the identical fault at a completely different address in the binary, settling that question directly.

Two statistical detours ran alongside the code bisection, and both caught real methodology mistakes before they became false conclusions. First, a same-configuration control comparison initially looked like it flipped which fault signature appeared, until a proper five-versus-five repeated trial showed both groups landing on the same signature ten times out of ten — correcting a conclusion drawn from a sample size of one each way. Second, once an overnight 43-run batch came back with a sharp, unexpected reversal — roughly 81% clean survival to a 5-minute cap, against a daytime stretch that had crashed in nearly every one of about 14 consecutive attempts — a proper censoring-aware maximum-likelihood fit, treating every clean run's full exposure as real information instead of discarding it, put the true mean time-to-failure at roughly 24 minutes, converging within 1% of a completely independent estimation method run minutes earlier. What actually stopped a 5-minute soak window from being mistaken for "fixed" was that statistics work, not intuition:

> "sure you're a machine, but aren't you happy? we found the root cause"

## Every crash lands inside the stack, and only with a real function call

Cross-referencing the recurring garbage return-address values against the linker's own stack symbols found something concrete underneath the noise: every single captured fault, across dozens of otherwise-unrelated configurations, put the corrupted program counter inside the live stack region — never in code, never in scratch memory, always the stack. That reframed the whole hunt: not "which line of code is wrong," but "something is corrupting a saved value sitting on the stack between when it's written and when it's trusted again."

The cleanest single experiment of the whole arc tested that directly: two versions of the same repeating loop, same iteration count, same hardware state — one using a real function call, push/pop-based return included, the other doing the identical work inlined with a plain backward branch and no call at all. The no-call version ran clean for a full 60 minutes, three times running. Restoring nothing but the function call reproduced the crash within 5 minutes, three times running, no exceptions. The branches were never the issue. Only a real call and return, saving something to the stack and later trusting it, was.

## A self-checking return address inverts the theory

The natural next question — does the corruption happen when the return address gets written, or does it change while it's sitting there — led to genuinely novel instrumentation: a hand-written assembly routine that checks its own saved return address twice, once right after pushing it and once right before using it to return, trapping into a small controlled handler with full diagnostics instead of ever blindly jumping on a bad value. That's meaningfully different from every prior fault capture on this project, which could only ever show the aftermath of a wild jump; this one catches the corruption non-destructively, before anything acts on it.

It caught something within five minutes, and inverted the working theory on the spot. The value sitting on the stack was completely correct. What came back wrong was the comparison constant itself, loaded from a literal pool baked into the program's compiled code — supposedly static, read-only, unchanging memory, not the stack at all. Relocating the entire test's stack to a completely untouched region of DRAM reproduced the identical pattern: correct stack value, corrupted code-region read — plus a bonus finding in the same capture, where the trap handler's own diagnostic-address constant, an entirely separate literal pool read in a different function, came back corrupted too. Two independent bad reads, two different functions, two different memory regions, one halt.

I raised two real objections rather than accepting the result outright:

> "The only thing really bugging me however is: 1. we did a ram stress test: we never caught this 2. I tried the firmware with this board... the operation was stable."

Both got taken seriously. The RAM stress test's clean result is consistent with the day's own findings — it's the same raw read/write access pattern that an earlier all-branches control had already shown doesn't trigger this at all. The "stable" reference firmware result is harder to fully close: either it never produces this project's specific calling pattern at the volume needed, or a short bring-up check simply isn't a long enough window against a fault this rare — this project's own data already showed the identical vulnerable code surviving a clean hour by chance. Left open, not resolved.

## The diagnosis: clock jitter

The final turn came from my own day job — PLL design — reframing three days of software findings as a straightforward electrical calculation. Modeling the fault as a Gaussian timing-jitter excursion past a fixed setup/hold margin, and solving for the jitter magnitude implied by the measured roughly 1.3-billion-cycle mean time to failure at the CPU's actual clock period, gives roughly 125 picoseconds of RMS jitter — verified independently against the data to within 1 picosecond. For comparison:

> "in my job we do PLLs with 150fs of jitter at 30GHz, so this is still big in comparison with the 6GHz PLL in the zynq7"

Six orders of magnitude worse than professional practice, even before adjusting for the frequency difference. The same model predicted that underclocking the CPU from 666MHz to 500MHz should improve the failure rate by roughly a million times, and it checked out precisely against an independent calculation. It also retroactively explained nearly everything the prior three days had found empirically: why the failure statistics looked exponential (a fixed per-clock-edge probability is exactly a memoryless process), why only this one loop ever exposed it (nothing else in the program repeats enough), why the call/return pattern specifically mattered (a tight store-then-read of the same address is a real timing pressure point), and why the corruption landed equally in code memory as in the stack (a bad clock edge doesn't care which transaction it hits).

Three days of rigorous, textbook software debugging — bisection, controlled A/B trials, statistical modeling, a genuinely novel self-checking diagnostic — converging on a conclusion that no amount of software could ever fix, only work around. I want to flag something up front, though, since I already know how this story keeps going: this diagnosis turned out later to be real math, precisely fit, to the wrong physical mechanism. I'll get to why in a couple of posts. For now, it's worth telling this exactly as it landed at the time, elimination by elimination, because the eventual correction only means something once you've seen how convincingly the wrong answer fit first.
