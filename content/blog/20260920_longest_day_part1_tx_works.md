---
author: "Daniel Blackbeard"
date: '2026-09-20T19:00:00+02:00'
draft: true
title: 'The Longest Day, Part 1: TX Finally Works'
tags:
- hdl
- log
---

## The most eventful single day of the project

What follows spans one sitting, and I'm splitting it into two posts because there's too much in it to tell honestly in one. This half covers getting TX working at all. The other half — RX, a UDP console, and a mystery that took until well past midnight to actually crack — is next.

## Six rounds of elimination, and remembering I had hands

The `TXGO` bit was getting set and just sitting there stuck, never actually kicking off a transmission. That kicked off a multi-round collaboration with a second Claude instance that had real datasheet and TRM access — each round following the same shape: a hypothesis grounded in a real TRM citation, implemented, tested on hardware, reported back honestly whether it moved the needle. Five register-level hypotheses got tested and eliminated this way: a live clock-divider glitch, `RXQBASE`/shared-DMA readiness, `DMACR`'s endian bit, `DMACR`'s AHB burst-length field, and — initially — an explicit GEM reset pulse plus a full UG585-documented controller initialization sequence that had been skipped entirely.

Somewhere in that process, Claude told me it had no way to reach the physical board itself. That wasn't true, and I corrected it directly:

> "you have access to all the scripts, from compilation, synth, implementation, up to loading the sw and bitstream to the board, you have jtag access via xsdb, and you have UART access to all the commands. We did a debug session like this before the /clear command."

Getting that tooling actually working end to end — a batch script for reaching `xsdb`, and a direct PowerShell serial connection to the board's COM port, since nothing in the repo already handled that side — turned the rest of the day from "describe what to test" into "test it directly, report findings." That shift mattered more than any single register fix that day.

## The procedural gotcha that looked exactly like a software bug

Testing the GEM-reset-pulse hypothesis on a genuine power cycle produced a real hang — JTAG's own `stop` command timed out, UART went completely silent, and a second attempt left the debug port wedged badly enough that only another power cycle cleared it. The instinct on the AI side was to blame the newly-added reset-pulse code and disable it. That was wrong, and I caught it with plain reasoning rather than more debugging:

> "1. If all the changes happened inside main.c, eth0.c and eth0.h, I see no reason why a power cycle doesn't revert a stable situation if we revert the source changes. 2. I don't think at any point we needed to touch the task scripts for anything, am I wrong? 3. Even after this, I don't see a reason why modifying the scripts would solve a software issue."

The real cause: this board boots entirely via JTAG, with no FSBL and no autonomous flash boot. The FPGA bitstream is volatile SRAM configuration, always lost on power-down, and it had simply never been reprogrammed after either power cycle. Every "successful" test all session had been riding on a PL configuration loaded once, hours earlier, before the debugging even started. Confirmed directly in Vivado's hardware manager: the device showed as not programmed. Once reprogrammed in the correct order — bitstream, then PS7 init, then application load — the very next test produced a real completed transmission for the first time in the whole investigation, and an independent packet capture confirmed a real frame had actually reached the PC: source `02:00:de:ad:be:ef`, broadcast, 142 bytes.

It's a genuinely useful example of the AI side getting confidently wrong about *attribution* while still doing everything else right — a different flavor of mistake than a bug in code, and the kind of thing that's hardest to catch without someone who actually understands the system pushing back on cause rather than symptom.

## What "same timestamp" doesn't actually prove

With one frame confirmed, Wireshark showed something odd: two identical packets per single trigger, same interface, same displayed timestamp. The first instinct was a known, real class of quirk — capture-layer duplication in the Windows packet driver — reasoning that identical timestamps supported it. I pushed back with the same instinct as before, from actually being at the desk:

> "The fact that they have the same timestampt makes me think of wireshark duplication (in my case indicated as 5909.102810, hence down to the microsecond, still could see two packets on 40ns period)"

Pulling true per-packet timestamps straight out of the capture file didn't settle it either — the underlying capture clock turned out to be quantized to whole microseconds regardless of the nanosecond-precision field it reported, so identical timestamps couldn't distinguish one frame captured twice from two frames genuinely under a microsecond apart. The real answer came from a data source entirely outside the capture pipeline: the Windows NIC driver's own statistics, checked against a clean idle control first to rule out background traffic. Two isolated trials, `+2` frames per single trigger, every time. Real, hardware-confirmed, not a capture artifact.

The actual cause, once I looked it up instead of assuming: the TX ring was a single descriptor with its wrap bit set, pointing back to itself, because it was the only entry that existed. No real driver does this — production TX rings run to dozens or hundreds of entries — and I found this myself reading the reference material rather than being told:

> "So reading the docs it seems that the single frame has never been validated (any official driver makes it 128 elements for the ring descriptor), hence let's replicate that structure for out case."

## Designing for two futures, then handing over the keys

Rebuilding the ring surfaced a real architecture question. The same physical TX ring would eventually need to serve two different producers: a UDP command/reply path replacing the UART console, and a future PL-fed sample-streaming path, with RF sample data only ever flowing board-to-PC and the PC-to-board direction carrying nothing but small command requests. My own realization mid-conversation is worth keeping close to how I actually said it:

> "This will get trickier that I thought at the beginning: the PL will have to assemble the whole frame if it wants to write directly into the buffer. This is honestly bigger than me and I'm thinking about delegating the completion to you since I got a good understanding of the GEM at the architectural level. So let's design this together and I will let you the writing of the code."

The design we landed on keeps the PL's job deliberately small: it never assembles a real Ethernet frame, it only ever deposits raw payload bytes into a buffer software already reserved for it. Software still owns every header and still commits the descriptor. The same reserve/commit contract serves a tiny software-built command reply and a hypothetical high-rate PL producer without either one needing special-cased ring code. One more explicit design call from me, with a real justification behind it rather than just convenience:

> "There should never be the situation where we fill a buffer faster than it's discharged... for this application if we can't stay behind we drop samples... not handling the null is not only cheap, it's correct for the application."

## A foundational bug, hiding since the first commit

Testing the new ring produced an immediate, different kind of failure — a genuine CPU fault, confirmed via a read-only JTAG halt showing the core parked in Abort mode, disassembly cross-referenced to the exact faulting instruction inside the brand-new ring code. The real cause had nothing to do with the ring: `startup.S` had never zeroed `.bss`, a gap that had been latent since the very first version of this project, because nothing before this day's ring-index variable had ever been a persistent static or global — every prior line of this bare-metal firmware was pure register pokes. My own reaction to that one:

> "this should be given, I expected .bss to be zeros where not used, hence you could even fix."

It's the most foundational bug of the whole session — something every C programmer assumes without thinking about it — sitting there invisible since the project's first commit, only surfaced by a piece of entirely unrelated new code. A real illustration of coverage as a property of a codebase's actual history, not just its test suite.

With TX finally solid, the RX side came together fast, right up until it very much didn't. That's the second half of this day, next.
