---
author: "Daniel Blackbeard"
date: '2026-11-15T19:00:00+02:00'
draft: true
title: 'Mission Mode, Finally, and a Reset Pulse That Lived One Layer Up the Stack'
tags:
- hdl
- log
---

## Finishing what the night left mid-sentence

The day started by finishing what the previous session had left hanging: the mixer's output ports were undriven, the FIR needed a real I/Q-sharing redesign, and none of it had ever run on real hardware. It ended with a working radio — real antenna signal, visible on a live FFT, the gain slider audibly doing something to it. In between sat one of the longer single-day debugging arcs this project has produced, and the satisfying part is that almost none of the eventual fixes were where the symptoms first pointed.

## Proving the mixer with a trick instead of a testbench

Before any of the hardware chasing started, there was a nice small moment: how do you *prove* a freshly-wired oscillator/mixer chain is actually rotating phase correctly on real silicon, with no spectrum analyzer and no known off-air signal to reference against? The AD9361's built-in test tone turned out to be exactly the right tool for the job — it parks at a precisely known digital frequency, derived straight from the chip's own tone-frequency configuration. I set the NCO's phase step to the same magnitude in the opposite direction: if the mixer is doing real math, the tone should land exactly on DC. It did. No new test infrastructure needed, just noticing that two numbers I already understood should cancel, and watching them actually cancel on the bench.

## The rate collapses, and every obvious suspect is wrong

Then the real chase started. I added RX gain control — a genuinely necessary bring-up step nobody had done yet, since the AD9361 ships with no built-in gain table at all. Right after flashing it, the sample-stream rate, which had been a rock-solid ~9.6MB/s minutes earlier, fell to ~191 packets per second. A 46x drop. Toggling mission mode did nothing. UART writes looked like they weren't landing.

My first instinct, and the wrong turn, was to suspect the new gain-control code directly — maybe a missing settle pulse the reference driver documents, needed after forcing the chip into RX while in manual gain mode. I added it. No change. So I bisected for real: compiled the entire new block out with `#if 0`, rebuilt, reflashed, remeasured. The rate stayed at exactly 191 packets per second. The new code wasn't the cause — which meant my next suspicion, a flaky protocol bug in the console client, also wasn't the whole story either, even though a separate investigation around the same time *had* just found a genuinely real bug in that area: a fresh serial connection could pick up two unsolicited boot-time debug words as part of its first command reply, permanently desyncing every reply after that. Real bug. Just not this one.

## The fix was never in the code that changed

The actual cause lived one layer up the stack from anything a firmware diff could show. I'd done several live ELF-only JTAG reloads in a row that session — fast and convenient, since only C code was changing and a full bitstream reprogram wasn't needed. Except `RESETB`, the AD9361's physical reset pin, is driven by a plain FPGA-fabric register that a JTAG ELF reload never touches. Firmware only ever releases that pin; it never asserts it low first. Across several reloads, the AD9361 had been quietly running on whatever degraded analog and PLL state an earlier bad boot had left behind — and no amount of *correct* SPI reconfiguration on top of that state could fix it, because the chip had never actually gotten a hardware reset in between.

A full bitstream reprogram, which does reset that pin, plus a physical power cycle — needed anyway, since live reprogramming had wedged the PS7 debug port for the third time this project, a now-thoroughly-documented hazard — brought the rate back. Not just back: to 9.6MB/s, *better* than a figure from earlier in the same session I'd already accepted as "the real rate." That number, too, turned out to have been measured on chip state degraded by the same mechanism. The lesson worth keeping: a tool that reloads fast is not the same tool as one that resets everything, and conflating the two cost a genuinely long stretch of the day chasing firmware bugs that were never there.

## The tone test's biggest tell was what it didn't show

With the rate fixed, real antenna signal was still weak — while the built-in test tone, moved around freely by the NCO, looked perfectly healthy. That contrast was the actual diagnostic: the test tone is injected digitally, downstream of the entire analog RF front end. A real-signal-only weakness meant the bug had to be specifically in the analog path, not the digital chain the tone had already proven correct. The register controlling which physical RX pins are actually electrically active had never been written by this firmware, at all, in the project's entire history. One SPI write later — the standard balanced default used on essentially every AD9361 reference board — real signal appeared, the gain slider started visibly doing something, and the FFT showed real, filtered content for the first time.

The whole day is a case study in symptom locality lying to you. A rate collapse that started right after a firmware change looked like a firmware bug and wasn't. A tone that worked while real signal didn't looked like it ruled out the analog front end, and instead pointed straight at it once I actually reasoned through the data flow — the tone bypasses the front end, real signal doesn't — instead of assuming. Twice in one day, the fix lived exactly one abstraction layer away from where the evidence seemed to point, which is a much better story than "I found a bug and fixed it," because it's about the reasoning move, not the bug itself. The session closed with the rate dropping again, unexplained — a reminder that "fixed" in embedded RF work is often provisional until it's been stable for a while, not a single clean green checkmark.
