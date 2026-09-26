---
author: "Daniel Blackbeard"
date: '2026-12-13T19:00:00+02:00'
draft: true
title: 'RDS, Decoded From Scratch'
tags:
- hdl
- log
---

## The leftover becomes its own arc

Directly continuing from the leaked-sideband thread the stereo work ended on: the RDS channel that had been a noisy afterthought became its own full investigation, ending in a real, decoded station name.

## A CIC decimator, built the hard way

The RDS path's five-stage CIC decimator went through the same pattern as earlier DSP modules in this project: a comb recurrence that was structurally wrong, differencing against its own previous output instead of a properly delayed input; register widths undersized against the theoretical bit-growth bound; a valid pulse timed about one strobe period off; and a missing output descaling stage that caused wraparound "noise" which looked like a real design flaw but wasn't. Each of these got caught by a dedicated cycle-accurate testbench before I moved on to the next one — a good, unglamorous illustration of what CIC bring-up actually looks like in practice, bug by bug.

## Two tools that caught themselves lying

Both of the new Python analysis scripts I wrote for this stage had a real, load-bearing bug that would have produced a confident wrong answer if I hadn't self-tested them first.

The autocorrelation script assumed the symbol-period correlation peak would be positive. Running it against synthetic random Manchester data first — where I already knew the right answer should be close to nothing — showed almost no correlation at that lag at all. The real signature turned out to be a negative dip at *half* the symbol period. It was the detection theory itself that was wrong here, not an implementation bug, and I only caught it because I tested the theory against data I already understood instead of trusting it on sight against a real capture.

## The false "VERDICT: real"

The decode script reported "structured RDS group sync found," with roughly 150 different candidate program-identification codes. The question that actually broke this open was simple: can this be turned into something human-readable, or is it just gibberish? A real station's identification code never changes — so 150 different values in one capture was proof of nothing at all, not confirmation. The root cause was a verdict threshold calibrated against a short synthetic test that never accounted for how many chance hits a real capture's much larger window count would produce. The fix was to compute the actual chance rate for real operating conditions instead of reusing the synthetic test's assumptions.

That one is worth calling out on its own: every earlier self-caught bug in this project got caught by inspection or by a mismatch that didn't add up. This one shipped a confident false positive first, and it took asking the right question about what the output actually meant — not staring harder at the code — to catch it.

## The payoff: real Italian text

Once fixed — requiring a run of three or more consecutive synced blocks instead of an isolated pair, dropping the chance-collision rate from roughly 0.5 down to about 0.004 — the same 30-second capture produced 823 offset-word hits against a chance expectation of about 174, a 74-block consecutive run, a station code repeating 134 times, and a decoded program-service name that rendered as legible Italian.

That last part is the real proof, more than any statistic: it rendered actual Italian, which means the tool wasn't cheating. A confidence interval can tell you something is probably real. Being able to read what the station is actually broadcasting settles the question outright.

## Homework, on purpose

Rather than build the full RTL decode chain in one push, I chose to keep the Python chain as ground truth and wrote up a staged plan for porting the biphase demodulator, block sync, and content extraction into RTL myself, deliberately, stage by stage, as a real learning exercise rather than a race to the finish. Verify with the tool first, then build the real thing by hand — that's been the pattern running through this whole project, and RDS is where it gets applied one more time, on the hardest module yet.
