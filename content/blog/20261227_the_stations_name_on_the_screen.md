---
author: "Daniel Blackbeard"
date: '2026-12-27T19:00:00+02:00'
draft: true
title: "The Station's Name on the Screen"
tags:
- hdl
- log
---

## Two days, same station

The RDS port finished today, and I took it straight onto real hardware. It ended with RTL I'd built by hand, stage by stage, reading the same Italian station the Python decoder had read weeks earlier: the same program-identification code, the same program-service name, and a long RadioText string. Two days before this, that text had come out of a script. Today it came out of the FPGA.

## Block sync, and the day the framing changed

The block-synchronization stage was the hard one. My first design was three staggered state machines, each one hunting for a hit, verifying it 26 bits later, and handing the baton to the next machine in line. It failed in ways that were individually easy to explain and collectively confusing: an off-by-one, a counter starting at 1 and comparing at 25 — one bit early — that I'd fixed in the first machine and forgotten to fix in the other two; a "valid" signal that was a level instead of a single-cycle pulse, which made a testbench count nearly 22 million events where only 38 were actually expected; and a deeper structural flaw underneath both of those — after a successful verification, each machine dropped its current chain and picked up whatever hit came next, so "three matches" could quietly mean three links belonging to three completely different alignments, not one coherent lock.

The golden semantics turned out to be far simpler than the machinery I'd built around them. A real hit only ever looks at the position exactly 26 bits earlier than itself, so a chance hit landing between two genuine ones must never be allowed to interfere with the sequence. Reframed as a look-back over the last 52 bits of offset types instead of three separate hunting machines, lock collapsed into one comparison at three fixed positions — 52, 26, and 0 bits ago — plus a single 26-bit block counter once locked. A hundred and fifty-six flip-flops standing in for what had been open-ended counters tracking every possible alignment at once.

It felt almost embarrassingly simple once I saw it that way — coincidence matching across a fixed 3-by-26 window, rather than a fleet of independent state machines chasing events through time. The honest answer for why I didn't see it sooner is that a state machine is the natural way to think about events unfolding in time. The actual win here was re-framing the same problem in space instead.

## The content stage, and a state machine that "never moved"

The stage extracting the program identification, service name, and RadioText had a classic waiting-state bug: each state decided whether to advance or fall back to zero based on a one-cycle pulse, but the next block of data doesn't arrive for another 26 symbols — roughly 22 milliseconds, or something like 650,000 clock cycles later. Every waiting state fell right back to zero on the very next clock, long before the real data it was waiting for ever showed up. The waveform viewer never showed it happening, for the simple reason that a one-cycle blip is invisible once you've zoomed out far enough to see 650,000 cycles at once.

Behind that one sat two more. A value was being captured at the wrong instant — one character was being sampled during a different character's pulse, so it silently held a duplicate instead of the value it should have. And a whole-register overwrite meant each new group of received bits was replacing the *entire* output register instead of just its own 16-bit lane — the corrupted log value decoded exactly to what you'd get from the last group's two letters getting mangled by a bit-shift formula that assumed it owned the whole word. My own earlier design notes on this stage had stressed *where* each character ends up in the final string and had under-stressed the much more important detail that a single group update should only ever touch its own lane. A spec can be completely correct and still leave out the one sentence that actually mattered.

## Sending 592 bits over a 48kHz, 16-bit channel

Getting the decoded result onto the debug bus raised a real design question I sat with for a while: this channel carries one 16-bit sample per clock at 48kHz, and I had 592 bits of decoded text and status to expose through it, continuously, without adding any real complexity.

The answer was to stop thinking of it as a stream at all. Every sample on the channel became one address-and-byte pair, and a free-running scanner sweeps all 75 decoded bytes — the identification code, the service name, the RadioText, and a lock flag — once every 1.56 milliseconds. The receiving side just keeps a table indexed by address. Nothing needs framing, a sync word can never be confused with ordinary text, a dropped sample just means one byte gets refreshed on the next sweep instead of being lost, and a listener can start reading at literally any point in the cycle and still end up with a complete, current picture within one sweep.

I checked the whole scheme before it ever touched hardware — extracting the actual scanner logic out of the mixer module, simulating it against the known-good decoded text, and decoding its output with the real console-side functions, including a run where I deliberately threw away 35 percent of the words to make sure partial loss degraded gracefully instead of corrupting the whole table. On the PC side, the interface simplified at the same time it grew a new feature: the old eye-diagram tab and the RDS spectrum panel came out, since this channel isn't really a signal anymore, the audio channels got labeled for what they actually are, and stereo playback came back — it had been forced down to mono a few entries back, while the Costas loop still wasn't locking reliably.

## On the air

First real contact with a live signal: data flowing, the identification code, service name, and RadioText all correct, signal-to-noise quite poor, and lock dropping repeatedly before recovering on its own. I have a few candidate explanations, none of them investigated yet: the symbol clock runs at a fixed nominal 48,000Hz while the real decimated rate measures closer to 48,000.3Hz, which works out to roughly half a symbol of drift per minute; the block-sync policy drops lock after the very first bad block and needs three consecutive good ones to recover, with no hysteresis and no error correction built in anywhere; or it's simply a genuinely noisy signal doing exactly what a noisy signal does. Any of the three, or some combination, would produce exactly what I'm seeing.

## Closing the project

Later the same day, I settled on a verdict for the lock drops: pure signal-to-noise ratio, varying with the environment, not a design flaw in anything I'd built. On that basis, I'm calling this project officially concluded.

The last thing I did before closing it out was a small quality-of-life change to the PC console, and it feels like the right note to end on. Tuning used to mean writing a raw frequency value directly into a register by hand. It's now six buttons — minus 250, minus 25, minus 10, plus 10, plus 25, plus 250kHz — that convert a frequency step into the oscillator's phase word behind the scenes, with the AD9361's actual center frequency fixed at 98MHz and hidden from the interface entirely. A fitting last beat for a project that started with hand-typed hexadecimal register writes: it ends with a radio you tune by pressing a plus sign.
