---
author: "Daniel Blackbeard"
date: '2026-08-16T19:00:00+02:00'
draft: true
title: 'The Hang That Took a Week: Chasing a Ghost in the Instruction Cache'
tags:
- hdl
- log
---

## The symptom

With SPI alive and the AXI address map sorted out, the console would run fine for a while and then just stop. Completely unresponsive — sometimes while idle, sometimes under active traffic — and the only fix was a full software reload. No bitstream reload needed, no power cycle. Whatever was going wrong lived in software state, not in the PL.

That's a genuinely unpleasant kind of bug to chase, because "sometimes, after a while, for no obvious trigger" rules out almost every fast debugging technique you'd normally reach for. What follows is the order I ruled things out, because the order matters as much as the answer.

## Ruling things out, one at a time

**UART clock drift** was the first suspect, since a slightly wrong baud generator can produce console corruption that looks exactly like a hang if it happens on the wrong byte. I measured it: about 64ppm of error, several orders of magnitude too small to explain what I was seeing. It also didn't fit the symptom — the hang happened after idle periods too, when nothing had been transmitted to drift against.

**Generic DDR or PS7 corruption** was next. I ran a JTAG-driven soak test that hammered DDR continuously, including through a live hang, for over 300 passes with zero errors. If the memory controller itself were flaky, that test should have caught it.

**ARM Cortex-A9 erratum 794073** looked promising for a while — a real, documented silicon bug in this exact core family. I applied the standard workaround, confirmed it was actually active by reading `SCTLR` back directly over JTAG, and the hang still reproduced. Not that.

**Errata 742230 and 743622**, both from the vendor's own FSBL board support package, were next on the list — except both are explicitly gated to silicon revision r2p2 or earlier. I read `MIDR` directly: `0x413fc090`, which decodes to r3p0. This chip doesn't need either workaround, and applying them would have been chasing a bug this specific silicon revision can't have.

**MMU-disabled as a standalone cause** was the last conventional suspect. I enabled the MMU with a flat, Strongly-Ordered identity map — behaviorally identical to running with the MMU off, but isolating just the `M` bit in `SCTLR` as a variable. Same fault, same instruction, still reproduced.

## The actual signature

At this point I stopped guessing and started reading what the diagnostic halts were actually showing me. Every single capture caught the corruption on the same instruction sequence: a `movw`/`movt`-then-load pair computing the address of a UART peripheral register — which, because it's on the hot path of every single byte the console reads or writes, is by a wide margin the single most frequently re-executed instruction sequence in the entire program.

The clearest capture showed the faulted value differing from the correct one by exactly one bit, sitting in the half-word that a `movt` instruction had just written. That's a specific, testable claim: not a downstream computation going wrong, but the *fetched instruction encoding itself* coming back wrong on that particular DDR read.

## The fix, and why it worked

Once I had that framing, the fix was to enable the instruction cache (`SCTLR.I`). On this core, cacheability is a page-table attribute, which meant enabling I-cache required the MMU to actually be on — so I built a minimal flat table marking DDR as Normal/Cacheable and everything else as Strongly Ordered. The effect: the hot loop gets fetched from DDR exactly once and served out of L1 on every subsequent execution, instead of being re-fetched from DDR on every single pass through the console loop.

That configuration ran clean for over two hours of continuous use — dwarfing every prior configuration's under-ten-minute failure window.

## What I still don't know

I want to be honest about what this fix actually proves, because it's less than it might sound like. It identifies *where* the corruption was getting exposed — a repeated, uncached DDR instruction fetch — not the exact underlying physical defect. This board's Zynq is a reclaimed part, with its package markings and QR code deliberately sanded off. Genuine Xilinx silicon, confirmed via JTAG device ID, but of an unconfirmable and plausibly lower-than-assumed speed grade, running with no heatsink. My best theory is a timing margin issue that only bites on the specific access pattern of a tight, uncached fetch loop — but that's a theory, not a proven root cause, and I'm not going to dress it up as more certain than it is.

What I can say for certain: the fix works, it's held for two-plus hours where nothing else held for ten minutes, and the reasoning behind why it works is sound even if the deepest "why" behind the underlying defect stays open. Sometimes that's where a debugging story actually ends, and I'd rather show that honestly than pretend to a certainty I don't have.

With that resolved, the next stretch of work was actually getting the AD9361's local oscillator chains running — starting with the chip's internal BBPLL, and ending at a wall I didn't expect to hit so early.
