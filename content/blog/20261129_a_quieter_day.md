---
author: "Daniel Blackbeard"
date: '2026-11-29T19:00:00+02:00'
draft: true
title: 'A Quieter Day: a Leaky Integrator and a 1000x Arithmetic Error'
tags:
- hdl
- log
---

## After the climax, a smaller beat

A short follow-up to the opera day, mostly spent building a rudimentary automatic frequency control loop — a leaky integrator tracking the slow, DC component of the demodulated signal as a proxy for residual tuning error, feeding that correction back into the oscillator's phase step. Nothing here matches the previous entry's climax, but there are three small, honest details worth keeping.

## The same typo, twice

The previous session it was one mistyped signal name in a data-path signal. This session, independently, it was the same class of typo in a different signal — the accumulator this time, not the descaled sample. A small, very human detail about pattern-matching mistakes when moving fast: it's not a one-off slip, it's a recurring failure mode of typing quickly from memory instead of copy-pasting a name you already know is correct.

## Off by exactly 1000x

Mid-design, I hand-calculated a rough ceiling for the loop's maximum correction — about 0.23 Hz — and moved on without double-checking it. When I actually ran the real numbers through Python later, the true figure was about 229 Hz: three orders of magnitude off. My own mental estimate was wrong by 1000x, and the only reason it got caught at all is that I didn't fully trust a number I'd produced in my head and went back to verify it properly before building anything on top of it. It's the same "verify, don't just assert" discipline that's run through this whole project, aimed at my own quick arithmetic this time instead of a piece of RTL.

## Sizing a gain by a real constraint

Rather than "more gain is better," I deliberately capped the AFC loop's correction range using the FIR channel filter's own passband and stopband — roughly 130 to 272kHz. Past that boundary, the desired station's energy is already filtered out before the discriminator ever sees it, so any extra correction gain literally cannot help; there's nothing left there to correct toward. The nice moment was realizing, as a direct follow-up, that a chosen shift amount translates directly into how many bits of the 32-bit tuning register I actually have to get right by hand — call it a nine-bit tunability tolerance. It ties back to an earlier, exact-to-the-last-bit phase-step calculation I'd done for 99.9MHz: with the AFC loop doing the fine correction now, that precision is no longer load-bearing. A good guess is enough.

The same "verify, don't assert" theme as the day before, just turned on my own back-of-envelope math this time — plus a small, satisfying example of letting a real physical constraint, the filter's actual passband, size a design parameter instead of picking one out of thin air.
