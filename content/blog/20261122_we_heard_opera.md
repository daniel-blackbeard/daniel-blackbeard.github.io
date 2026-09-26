---
author: "Daniel Blackbeard"
date: '2026-11-22T19:00:00+02:00'
draft: true
title: 'We Heard Opera'
tags:
- hdl
- log
---

## The richest day this project has had

This is a full arc from "the discriminator is broken and nobody knows why" to real, audible Italian opera coming out of my PC speakers, through a receiver that had been essentially non-functional at the start of the very same day.

## The mystery that wasn't

For days the suspect had been the CORDIC discriminator: on real hardware the demodulated phase angle looked frozen, and I'd reproduced the simulation-versus-hardware divergence twice without explaining it. It had already eaten a lot of effort. The resolution came from building an on-chip logic analyzer into the design specifically to catch it in the act — and the capture proved the CORDIC was innocent the entire time. The bug was never where I'd been looking.

Before the payoff, a good beat worth keeping: the very first capture out of that logic analyzer itself *looked* broken — values jumping around in a pattern that made no sense. It turned out to be a bit-reversal bug in my own probe-connection ordering in the diagnostic script, not the hardware. The instrument lied first.

## One symptom, three unrelated root causes

"The audio sounds quiet" unraveled into three genuinely independent bugs across the FIR filter. A pipeline-drain corruption bug had been invisible to every earlier test, because every earlier test used a constant input — it only showed up once I wrote the first dynamic-input testbench. Separately, there was a plain 16-bit output overflow. And the real root cause of the whole gain mystery: a coefficient generator that had auto-scaled its taps to a non-power-of-two, which no integer RTL shift can ever exactly undo. I'd silently mis-attributed that one as "the filter's designed gain" for days.

That last bug is a good example of resisting the urge to stop at the first plausible explanation. I fixed it at the source — an explicit power-of-two quantization scale — and verified it with a new gold standard: a Python golden-vector generator running the RTL's exact bit-accurate algorithm against the real synthesized coefficient file. Six hundred out of six hundred samples, bit-exact, not "within tolerance."

## The bug that almost slipped through

A Python script had written its output to the project root instead of the source directory, so a "verified, bit-exact" test had quietly been checking the wrong, unfixed file the whole time. I only caught it because I stopped and asked myself a sharper question instead of accepting the reported result at face value: wait — so DC gain isn't actually 1? That question is what surfaced the stale file. A verification claim needs re-checking, not just re-asserting, no matter how clean the summary line looks.

## Can a reference model catch its own bug?

An honest dead end, worth naming rather than glossing over: the golden-vector test proves the RTL implements a *described* algorithm, not that the algorithm itself is right, since I'd written both the RTL and the reference model myself. Agreement between them proves consistency, not correctness. What actually gave me real independence was a frequency-domain check running through a completely different code path, plus an earlier, separately-verified hand trace of the RTL's tap ordering. A concrete answer to a question worth asking of any self-checking test: who verifies the verifier?

## The reprogramming trap that kept getting worse

The JTAG debug-port wedge started as "just retry after a power cycle," escalated over the day to "can now kill the DSP clock itself, not just the debug port," and ended with a deliberate, documented attempt to clear it in software that flatly did not work — confirmed by actually trying it, not assumed. A sharp-edged Zynq JTAG quirk nobody warns you about.

## Working around a constraint instead of fighting it

Rather than risk another wedge-prone RTL reload just to get mono audio demodulated, I decided to do it in Python on the PC side instead, reusing the sample stream already flowing over UDP. The PC console got an opt-in pipeline — off by default, since I didn't want it running unless I asked for it — that low-passes and decimates the demodulated signal down to 48kHz and plays it through the sound card. No new hardware risk, and audio working in under an hour. Knowing when to stop reaching for the tool I'd been fighting all day. Along the way, a new decimator stage narrowed the FFT window enough that the 19kHz FM pilot tone became visible for the first time — the seed of the stereo work that came next.

## The ending

Real, audible opera, from a local Italian broadcaster, through a receiver that had a frozen discriminator, a corrupted FIR, and the wrong gain everywhere at the start of that very same day. Everything else in this entry is really just the setup for that one payoff.

The reusable lessons here are all about instruments and claims: the logic analyzer that cleared the actual suspect, the probe script that lied first, the verification that was quietly checking the wrong file, and the reference model whose independence had to be argued for rather than assumed. None of them are one-off — they're the same discipline that's carried this whole project, just compressed into a single, unusually eventful day.
