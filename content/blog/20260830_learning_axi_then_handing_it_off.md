---
author: "Daniel Blackbeard"
date: '2026-08-30T19:00:00+02:00'
draft: true
title: 'Learning AXI, Then Handing It Off'
tags:
- hdl
- log
---

## A rule I set for myself, and later lifted

I want to break from the technical thread for one post, because something happened on this project that's worth writing about on its own: how the working relationship between me and my AI collaborator actually evolves over the course of a real project, not just in theory.

Early on, I made a deliberate rule: `axi_if.sv`, the module handling the low-level AXI protocol handshaking, was off-limits to Claude entirely. I wanted to write and verify it myself, with zero help, because I already knew from the CPU project that the only way I actually learn a protocol is by getting my hands dirty implementing it, mistakes included. That rule held for weeks.

Then, once I'd actually done that — once I understood what was inside `axi_if.sv` because I'd built it — I lifted it, and said so directly:

> "This is a learning project: I already learned how axi works. Hence now, since I know what's inside of it, I let you do the heavy work of re-organizing the already existing code. So all restrictions are lifted, but in doubt ask."

That's a distinction I think is easy to miss if you only think about AI collaboration in binary terms — "do it yourself" versus "let the AI do it." The actual rule I was applying was about *sequencing*: learn it first, delegate the reorganization second. Once I'd done the learning, refusing help on the mechanical reorganization work wasn't protecting anything anymore, it was just slower.

## What came out of lifting the restriction

The refactor that followed ended up more unified than I'd originally planned. Instead of a separate address-range interconnect routing requests to each peripheral, a single shared `axi_if` instance sits directly on `M_AXI_GP0`, fanning `wen`/`ren`/`addr`/data out identically to every peripheral in parallel. Each peripheral self-selects by comparing an address field against its own `PERIPH_ID`. A peripheral that isn't the target simply says "not me" combinationally — which means an unclaimed address gets an immediate DECERR response instead of hanging forever waiting for a `done` signal that will never come. No interconnect, no address-range table, anywhere in the design.

I wouldn't have arrived at that architecture on day one. It only fell out once I'd actually internalized how the individual pieces worked and could evaluate whether the reorganization Claude proposed was actually simpler, not just different.

## The module I wasn't ready to hand over — until I was

The same session, in the opposite direction, something else happened worth mentioning. `axi_spi.sv` — the SPI master module — had started out under the exact same rule as `axi_if.sv`: I'd write the whole thing myself, no hints. Sometime before this, on a day I kept flagging as a low-focus one, I broke that rule and asked Claude to finish it for me. My own words at the time capture it better than I could reconstruct now: "no focus today," "I'm kind of happy that at least I got a sizeable part of the module kept." Fatigue had quietly changed my own priority from *learn by writing it* to *just get this unblocked*, and I made that trade-off consciously rather than pretending otherwise.

That module got reclaimed here, described in my own notes as "my chance for redemption" — a deliberate decision to go back and finish something myself that I'd handed off on a worse day.

## Taking my revenge

A little over a week later, I actually did it. I finished `axi_spi.sv` myself — written at my workplace during a break, then pasted into the project over email, which mangled the formatting and dropped an `endmodule` along the way. I asked Claude to fix the formatting only, logic completely untouched, and to flag rather than fix anything it noticed while it was in there. It flagged two real issues without touching them: `spi_start` never getting reset, and a hazard where holding `wen` high could cause a re-latch mid-transfer.

Running my own corrected testbench against the module found something neither of us had caught by inspection: `w_done`/`r_done` were being deasserted one clock cycle late. That meant `axi_if` sampled the stale prior value and closed out transactions a full operation early, while the real SPI transfer was still finishing in the background — the kind of bug that's genuinely hard to spot by reading the code, and comes right out once a testbench actually exercises back-to-back transactions instead of one operation in isolation. I had it diagnosed in detail, and I left it for myself to fix rather than asking for the fix outright.

Session ended with a line I'm keeping exactly as I wrote it:

> "I took my revenge on the SPI module I wasn't able to do last time ;)"

## Why this is worth writing down

Not every day is a good day to be doing the hard, focused version of learning something. I think that's fine, as long as you're honest with yourself about when you're making that trade — and willing to go back for the redemption round once you're not running on fumes anymore. The technical result of this stretch — a genuinely cleaner AXI fan-out architecture than the interconnect I started with, plus a one-cycle-late deassertion bug caught by a testbench that actually stresses back-to-back transactions — came directly out of taking that sequencing seriously: learn it, then delegate it, then go back for the part you shortcut, once you're actually able to.

With the RTL side in a good place, the next big day was the one I'd been quietly blocked on since the BBPLL post: getting the RX local oscillator synthesizer to actually lock. It turned into the longest single day of the whole project so far.
