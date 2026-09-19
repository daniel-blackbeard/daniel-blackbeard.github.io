---
author: "Daniel Blackbeard"
date: '2026-09-27T19:00:00+02:00'
draft: true
title: 'The Longest Day, Part 2: RX, a UDP Console, and a Mystery at Midnight'
tags:
- hdl
- log
---

## Picking up where TX left off

The first half of this day — [covered last week]({{< ref "blog/20260920_longest_day_part1_tx_works.md" >}}) — ended with TX finally solid: a rebuilt, properly-sized ring, a real architecture for sharing it between a command console and future sample streaming, and a decade-old `.bss` zero-init bug finally caught. The second half is where RX came together cleanly, a full UDP console got built and delegated end to end, and the project hit its first genuinely open-ended mystery — one that didn't stay open by the time the sun came up.

## RX works, and nobody had to ping anything

RX ring init went through several rounds of review on my own code: a wrap bit in the wrong position, a buffer-stride rounding bug, and an operator-precedence bug inside that same rounding formula — `+` binds tighter than `<<` in C, so an expression I'd written to compute a padded stride wasn't parsing the way I'd assumed — before settling on a clean, fixed 256-byte stride. Final review caught two real bugs before hardware ever saw the code: `RXEN` never set at all in an earlier draft, meaning the ring could be perfectly initialized and nothing would ever land in it; and, after I added `RXEN` myself in the right place, a missing memory barrier between finishing the ring/queue-base writes and enabling `RXEN` — the same DDR write-ordering hazard the TX ring had already taught this project once, just recurring at setup time instead of per-commit, since RX has no software-side kick to hang a barrier off. That one I explicitly delegated:

> "The dsb is not something I ever handled, on that I will trust you."

The milestone landed cleanly on the very first real test: ambient LAN broadcast traffic showed up in the ring within seconds of boot, with no PC-side trigger needed at all. The status register showed a frame received, the first descriptor's ownership bit flipped exactly as the bit layout predicted, and a second frame landed in the second descriptor — proving the ring genuinely walks sequentially rather than being a one-descriptor fluke. Reading the buffer bytes by hand and decoding a real subnet-broadcast UDP packet — destination MAC, source MAC, EtherType, a genuine IPv4/UDP header — closed the milestone out exactly as planned.

The exact same "JTAG init fails" symptom from earlier in the week came back here too, and turned out once again to be the bitstream never having been reprogrammed after a power cycle. The difference this time: I was asked to physically check the board before anyone guessed at a cause, and my own diagnosis was refreshingly blunt:

> "Nothing unusual, I just power cycled. This is purely sw issue"

The first time this gotcha showed up it read as a one-off war story. The second time made it obvious it's a standing property of this board's JTAG-only boot flow, not a fluke — worth calling out explicitly as the kind of thing that isn't a bug in any one session, it's a fact about the rig.

## Full delegation, and a working UDP console

With RX proven, I handed off the rest of the networking work outright:

> "At this point I will just delegate to you. Now I have a decent understanding of GEM and the RTL operation, so the rest is just mechanically adding functionality we kind of already have (uart)."

Before any code got written, four genuinely open design questions got settled up front: the board's static IP — `192.168.3.50`, picked to match the subnet already seen in real captured traffic — the UDP command port (`5555`), whether to build a real ARP responder rather than lean on a static PC-side ARP entry (I chose to build the real thing), and how replies to a multi-command request should be framed — one batched UDP reply mirroring the request's own array shape, rather than one frame per command.

The implementation reused every piece of the TX/RX groundwork instead of inventing new mechanism: RX poll/release functions mirror TX reserve/commit exactly; the UART command dispatch table got extracted into a transport-agnostic function so the same AXI/SPI/CDC/GEM device switch now serves both UART and UDP without duplication; the main loop moved from blocking on UART to polling both UART and Ethernet every pass, since swapping UART for UDP only works if the loop actually reaches the network side promptly. A command-array protocol with a stop sentinel got bounded by the *smaller* of the sentinel and the frame's own received length, specifically so a truncated or malformed payload can't walk the parser off the end of a buffer.

## The bug that ate the rest of the night

Then the console just didn't work. Ping and ARP against the new board IP did nothing, and the obvious question came first:

> "I tried ping and arp, none of those is working. Is the sw loaded already?"

It wasn't — the board was still running the pre-UDP build. Reloading surfaced something much worse than a missing feature: the UART console, which had survived every prior stretch of this project untouched, went completely silent within seconds of a real ARP request arriving. What followed was the longest single debugging arc of the whole project up to that point, and — unusually for this project's history — it ended, for the moment, without a resolved root cause.

What I could confirm, all from real hardware: the crash was a genuine CPU exception, not a soft lockup, and it was deterministic — the same fault address and the same link register recurred across multiple independent fresh reloads, hours apart. That address traced to the MDIO command word inside the PHY's boot-time "restore default page" write — except that function only ever runs once, at boot, and a targeted breakpoint test proved conclusively it was never being re-entered. A hardware watchpoint confirmed something really did dereference that exact address, even though the watchpoint's own reported program counter at the stop point was nonsense — a real, documented limitation of this debug hardware for this fault class, not a dead end so much as a reminder that the tool itself has limits worth knowing about. The clearest lead: an instruction breakpoint caught execution *inside* `.bss` itself — the tiny region holding this project's only three static globals — in clean supervisor mode with a completely sane stack pointer. The CPU was genuinely executing whatever bit pattern happened to be sitting in those globals as if it were machine code.

Every function in the ARP reply path got reviewed instruction by instruction against its own disassembly and matched exactly — no out-of-bounds write, no stack overflow, double-checked against worst-case stack usage figures, no bad pointer arithmetic anywhere. What made this arc different from every other bug in the project's history to date: bisection results were inconsistent between otherwise-identical runs. One attempt showed the ARP reply completing cleanly; the very next attempt crashed before reaching that same checkpoint. Every earlier mystery in this project had a single, fixed, reproducible mechanism once found. This one's symptom was rock-solid reproducible; its exact path to that symptom wasn't — which pointed toward a real timing or concurrency interaction between GEM's autonomous DMA and the CPU, rather than a fixed logic error sitting on one line.

## What got built instead of a fix

Rather than a fix, what came out of this stretch is genuine infrastructure I still use: the exception handlers in `startup.S` no longer just spin silently. They save the original fault-time registers to a fixed, JTAG-readable scratch address and self-report the fault type, status, and address over UART, written in hand-rolled assembly that deliberately touches zero stack so it stays trustworthy even when the crash itself corrupted the stack pointer. A test-only function synthesizes a valid ARP request directly into the RX ring on command, turning "wait an unknown number of seconds for real traffic and hope to catch it" into "send one UART byte, get an answer immediately." Together these took what had been an hours-long, luck-dependent JTAG chase down to a repeatable few seconds per attempt.

The day closed, twice, with the same honest question asked from opposite directions — first handing the investigation over outright, then pulling back to check what other tools were even available:

> "I will still let this debug session up to you, but that being said, what other options do we have other than jtag?"

## Handed off for the night, and the mystery actually breaks

Past midnight, I handed it off for real:

> "I didn't saw the notification, but this is what I wanted to see. I'm not familiar with debugging race conditions, but now we know there is one that happens 'right after a branch to a call'. Here I will trust you, it's late hence my next reply will be when I awake, you keep processing this last evidence and apply better techniques."

What followed was the payoff to the whole day. A "trace every function call" instrumentation pass had already produced the most important clue purely as a side effect: tracing slowed the loop down enough that the crash stopped reproducing for a clean three-minute window — real evidence of timing sensitivity, not a fixed bad line. But UART tracing turned out to be exactly the wrong instrument for it — each traced call could block tens of microseconds waiting for transmit space, adding enough latency to mask the very problem it was built to catch. The fix was to redesign the trace mechanism entirely: instead of streaming bytes live, each call writes one byte into a plain DDR ring buffer, a few non-blocking memory accesses instead of a blocking UART send, and the self-reporting fault handler got extended to dump that entire buffer automatically as part of its one-shot post-crash report. The overhead that mattered moved from "every loop iteration, forever" to "once, after the fact" — the same principle as the fault handler itself, one level up.

With that running on real hardware, the actual root cause surfaced within one capture. The true pre-crash registers held a completely valid frame pointer and a completely valid descriptor base address, ruling out corrupted data outright — while the GEM hardware's own status register showed the "buffer not available" flag set, with all eight RX descriptors simultaneously marked owned by software. The RX ring had run completely out of buffers. A direct follow-up made the mechanism unambiguous: a fresh board, deliberately driven into that same fully-exhausted state by a burst of forced ARP broadcasts, sat there for over a hundred seconds — every descriptor full — running perfectly healthy the entire time, no crash at all. Exhaustion alone wasn't the bug. It was a confirmed precondition for a rarer event riding on top of it, almost certainly a bus-arbitration hazard from the hardware's own handling of an incoming frame with nowhere to put it, colliding with whatever the CPU's instruction fetch happened to be doing at that exact moment — which is exactly why the fault type kept changing between occurrences: different collision timing corrupts a different bit pattern each time.

Root cause, in one sentence: the RX service loop only ever drained one frame per pass, real traffic can arrive faster than that for long enough to exhaust an 8-entry ring, and GEM's behavior once genuinely starved has a real chance of corrupting whatever the CPU is doing at that instant. Not a mystery defect needing a workaround — an ordinary, fixable gap in drain rate, plus a hardware condition the software had never once acknowledged or cleared. I left the actual fix unwritten on purpose: real driver logic, the same kind I'd written every other piece of this Ethernet stack myself, waiting for me to wake up to a fully solved mystery and a clear next step instead of an open one.

It's a genuinely rare thing to be able to show honestly: going to sleep mid-investigation, explicitly on trust, and waking up to a real answer instead of either silence or a fabricated one. It's also, as the next couple of posts will show, not quite the end of this particular story — the ring fix held, but the underlying crash signature would come back twice more before its actual cause was found.
