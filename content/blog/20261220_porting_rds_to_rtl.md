---
author: "Daniel Blackbeard"
date: '2026-12-20T19:00:00+02:00'
draft: true
title: 'Porting RDS to RTL, One Red Row at a Time'
tags:
- hdl
- log
---

## Starting the homework

The plan from the previous entry finally started: the Python tools had already decoded a real Italian station, and the job now was to rebuild the whole chain in RTL by hand, stage by stage, using the Python decoder as a known-good oracle. Nine stages in total: a CIC decimator, a low-pass biquad, a symbol-recovery oscillator, biphase soft/hard-bit detection, differential decode, the block's syndrome check, offset-word matching, block synchronization and lock, and finally the actual program-identification, program-service, and RadioText extraction. By the end of the day, the first seven were green.

## A testbench that doubles as a to-do list

The interesting decision was inverting the usual order. Instead of writing a testbench per module after the module exists, I wrote one golden testbench first, with a companion Python generator that builds a synthetic 240kHz RDS stream and a golden vector for every single stage. Then I added output ports to the RTL — width guesses and all — and let the compiler and one summary table tell me what was actually missing: one row per stage, failing until the stage existed, passing once the numbers matched.

Stage zero, the CIC decimator that already existed from the earlier entry, passed bit-exact on the very first run — 43,700 out of 43,700 samples — which also quietly validated the Python model of that same CIC against the real RTL for free. The wrong port widths I'd guessed at up front — 16-bit placeholders standing in for what should have been a 1-bit valid flag, a 32-bit phase word, and a 512-bit text buffer — compiled with nothing worse than warnings. I only found them by actually reading the compiler's log, not because the build refused to proceed.

Before trusting the generator itself, I checked it against independent references: its block encoder against the decoder's own syndrome function, its demodulated bits against the transmitted bits it started from, its final decoded output against the earlier Python decoder, and its calibrated symbol phase against an analytic estimate derived from the filter chain's own group delay — about 0.28 samples apart, close enough to trust. The oracle needed its own verification before I was willing to rely on it for nine stages of hardware.

## The same mistake, twice, weeks apart

The first real bug of the day showed up in the golden generator itself: its block encoder failed its own cross-check against the decoder because it flushed ten extra zero bits through a direct-form LFSR — one that already multiplies the message by x^10 internally, so the extra flush double-counted that factor. I'd written that generator days earlier and hadn't caught it until it collided with the decoder side.

Hours later, the RTL syndrome stage failed for the exact mirror-image reason: I'd XORed the incoming bit into the LFSR's feedback path — the form that computes the message times x^10, modulo the generator polynomial, which is the *encoder's* structure — instead of injecting it at bit zero, which is what the receiver's syndrome computation actually needs. Same underlying misunderstanding about which LFSR form does what, made independently, in two different pieces of code, written on two different days. That's proof this particular point about LFSR structure is genuinely easy to get backwards, not a one-off slip. There was a second, separate flaw riding alongside it: a running LFSR never forgets old bits, so it isn't a sliding-window syndrome on its own — the outgoing bit needs explicit cancellation with the constant `x^26 mod g`, which works out to `0xEE`.

Before changing anything, I reproduced the failing values myself with a plain running LFSR in a quick Python script — confirming the problem was structural, not a timing artifact — and my own first attempt at a windowed recurrence to fix it *also* failed to match the golden vectors. Only once I found a version that matched bit-for-bit across the whole stream did I actually touch the RTL. A third, smaller bug rode along with the rest: the feedback bit itself had been registered, delaying it by one symbol longer than it should have been.

Looking back at the whole detour afterward: it wasn't actually that complicated, I'd just gotten confused by the wording of what each LFSR form computes — and it's a good enough reminder that GF(2) polynomials and LFSRs are something worth studying properly at some point, rather than re-deriving under pressure every time they come up.

## A tolerance stage that hit its own floor

The low-pass biquad — a Butterworth filter, 2.5kHz cutoff at 48kHz — was specified against a ±4 LSB tolerance relative to a floating-point reference. Trying realistic fixed-point versions in Python before locking in the RTL spec turned up something non-obvious: with the rounded 16-bit output fed straight back into the recursion, the error sat at exactly 4 LSB *even with infinite coefficient precision*, because the filter's poles amplify feedback rounding error by roughly 11.6x. Truncating instead of rounding made it worse, at 10 LSB. The actual fix was four guard bits carried in the feedback state, which brought the error down to 1 LSB. Quantization noise in the feedback path, not coefficient width, was the real limiting factor.

A side note that fell out of the same analysis: the fixed-point coefficients happen to fit inside a DSP48 slice's 18-bit multiplier port exactly, with the largest coefficient sitting comfortably under the port's ceiling — so the whole filter can time-share a single multiplier. The later out-of-context synthesis run confirmed exactly that: one DSP slice used, with the CIC decimator's fifteen 28-bit registers and ten 28-bit adders actually dominating the resource count, not the filter.

## "It doesn't fire" — and four bugs standing behind it

My own state machine for the filter never actually pulsed its output valid signal — it jumped from one waiting state straight to the next without ever passing through the one state that was supposed to emit. That was the reported symptom. Digging further, before touching anything, turned up four more bugs sitting behind it: the accumulator's clear was silently overridden by a later unconditional assignment inside the same block, so the clear never actually took effect; the input-history registers shifted at the start of the computation instead of the end; the feedback coefficients had the wrong sign and were declared as 16-bit literals that silently truncated a value that needed more bits; and the emit condition fired one multiply-accumulate step too early.

A few iterations later, the log showed the filter under-responding — reporting -13 where the golden model expected -18 at one particular sample — and reading the recorded waveform state by state showed the history registers holding values one step behind where they needed to be. Non-blocking assignments read the *old* output value, not the one just computed, and I'd built the history update assuming otherwise. The same class of bug that had bitten the Python model of the Costas loop a few entries back, now showing up in real RTL instead of a reference model.

## The unsigned accumulator, one more time

The biphase-detection stage failed in three separate ways at once. It was detecting the rising edge of the phase accumulator's top bit — the middle of a symbol — instead of the actual wrap event, which is the real symbol boundary. It was using the *previous* sample's half of the symbol to decide whether to add or subtract, one step later than it should have. And its accumulator was declared as a plain unsigned register, so every negative filter sample got zero-extended instead of sign-extended, silently adding 65,536 to the result. The golden model expected -9,612 for one particular symbol; the RTL produced 454,486 — almost exactly seven multiples of 65,536 too high. This is the third time this exact project has been bitten by one missing `signed` qualifier, after the Costas loop's rounding term and the raw ADC input declarations weeks earlier. I found the last piece myself by reading the failure log carefully: the output bit needed to be `(soft > 0)`, not the raw sign bit, which reads 1 for negative values — the polarity was simply backwards.

## Tooling had its own bad day

A simulation run that sat for over ten minutes looked like a genuinely heavy testbench. It wasn't — it was the compiler hanging on a reused simulation library, a trap I'd already written down in my own build notes from an earlier session and apparently needed to relearn. My first guess, offered without actually measuring anything, was that a debug elaboration flag was the real slowdown; I had to walk that back once the real number came in at sixteen seconds, waveform dump included — another small instance of the same "verify, don't just assert" lesson, this time aimed squarely at my own unverified guess.

A separate scare followed right behind it: I caught myself almost trusting a waveform whose file timestamp was older than the RTL edit it was supposed to represent. That one ended with a small permanent fix to the simulation script — write a fresh, windowed dump on every single run, and delete the old file first, so a stale waveform can never look current by accident. The smaller tooling pratfalls of the day were their own kind of comedy: the MSYS Python install couldn't open Windows-style `/c/...` paths, the simulator's own compile step doesn't expand shell wildcards, and a batch-file wrapper needs an explicit `call` or the whole script silently stops dead after its first command.

By the end of the day, block synchronization was still sitting as a state machine I was thinking through on paper rather than in code — a clean, deliberate stopping point rather than a stall.

The strongest thread running through today is verifying the verifier, twice over — checking the generator against independent references before trusting it, and checking my own hand-derived fixes numerically before committing them, including one honest dead end where my first windowed-syndrome recurrence simply didn't match. The other good thread is the recurring shape of the bugs themselves: a missing `signed`, a non-blocking assignment read one cycle too early, a structure that's *almost* the right LFSR — small, local, invisible to a plain read-through, and each one immediately visible as one specific wrong number in a golden-vector table. A testbench that scores nine stages in a single pass turned what could have been a week of guessing into "read the row, read the value, fix the line" — probably the most practical lesson worth carrying out of this entry.
