---
author: "Daniel Blackbeard"
date: '2026-12-06T19:00:00+02:00'
draft: true
title: 'Stereo and RDS, a Ghost Stall, and a Bug Hiding in One Missing Letter'
tags:
- hdl
- log
---

## The biggest arc yet

Two days and one night, spanning the stereo/RDS demodulator and a shared-CORDIC, time-multiplexed three-oscillator Costas loop for pilot-tone tracking. It went from nothing to a genuinely locking, audible stereo receiver. The road there had several strong, distinct beats.

## Reverse King Midas

After a diagnostic-build cycle left the AD9361 detuned following a reprogram, I found myself joking to no one in particular that I'd become my own reverse King Midas — every time I touched the board, the signal got worse instead of better. No packets incoming, no audio, 0.00MB/s on the meter. It's a small, honest beat about the real cost of iterating on actual hardware, and the pivot that followed it — stop guessing, verify in simulation instead of burning another build cycle — is the natural turning point for the rest of the day.

## The ghost that never got caught

The demodulator's output-valid signal stalled dead after exactly four pulses on one boot, then ran perfectly for over 500,000 pulses on the very next boot, with zero RTL changes in between. I ran three separate, careful simulations to chase it: the Costas loop alone with a real pilot tone, the loop alone with pure noise, and the entire real chain — decimator, FIR, CORDIC, the new decimator stage, and the loop together — at realistic ADC timing. All three ran clean. It was never reproduced again and never root-caused. The best explanation I have is a recovered-clock lock-acquisition race that only shows up under some specific, unlucky boot-time timing. I moved on. Not every bug gets solved, and that's a real, valid outcome to write down honestly rather than dress up as something it isn't.

## Told my own math was wrong, and it was

Mid-investigation, I talked myself partway into an explanation for why the loop wasn't correcting, built on an even/odd trig-function argument — the idea that the mixer's own product-of-sines structure might be phase-insensitive right at lock. I stopped myself before finishing that thought: I wasn't about to start justifying how a product of sines magically becomes a cosine. The real logic is simpler than that and doesn't need trig identities at all — either that operating point is stable, in which case there's no drift, just a wrong final phase, or it's unstable, in which case the loop moves away from it on its own. There's no third option where a confused derivation matters. Confident-sounding explanations can still be wrong even when you're the one who came up with them, and catching that before it sends you down a dead end is its own skill, distinct from the quieter self-caught errors in earlier entries.

## A verifier with its own bug, caught before it was trusted

To settle whether the loop's math actually held or whether this was a dynamics problem, I hand-verified RTL trace values against an independent Python re-implementation of the same difference equations. The first version of that Python model had a classic blocking-versus-non-blocking translation bug in it — using a register's just-updated value instead of its value from before the clock edge, something real hardware never does. I caught it because the mismatch it produced made no sense on its own terms, fixed the model, and only then trusted what it told me. A callback to the opera day's "who verifies the verifier" question, with the answer this time being: not until you've also verified the reference model itself.

## The actual bug: one missing letter

`32'd128` instead of `32'sd128`, sitting in a loop-filter rounding term. SystemVerilog's rule that any unsigned operand makes a whole expression unsigned meant the filter's arithmetic silently went haywire every time its running difference went negative — which is roughly half of all real operation — producing huge spurious corrections instead of small ones. I confirmed it by hand-deriving the exact corrupted value from first principles and matching it bit-for-bit against the logged RTL trace. This turned out to be the real cause of a loop that had looked broken in every conceivable way across the whole investigation — drifting, "not really closed," unresponsive to a deliberate sign flip — and it fits neatly into this project's recurring signedness-gotcha theme.

## Tuning what you can't calculate

With the bug fixed, the loop's remaining underdamped ringing had no clean analytical fix. The real phase-detector and oscillator gain constants aren't known precisely enough to compute proportional and integral gains from formulas with any real confidence. So I parametrized the loop and swept four gain combinations in one simulation run, sharing a single reference stimulus, and compared them side by side on one plot. I couldn't choose accurate values for those constants from first principles — sometimes the right move is to stop deriving and start measuring.

## An ending that isn't quite the ending

The loop genuinely locks on real hardware — 19.000kHz, 22.3dB of purity, holding solidly. But the very next observation opened the next thread immediately: high-energy content from the stereo subcarrier's own sideband was leaking straight into the RDS channel, through a filter too weak to reject it. This project doesn't really have single triumphant endings; it has a chain of them, each one revealing the next real problem underneath.

The human side of a long hardware campaign is all through this one — the real cost of each build cycle, a ghost bug honestly left unsolved, catching my own reasoning going somewhere it shouldn't — with a technical payoff that, in the end, fits in one character: `s`.
