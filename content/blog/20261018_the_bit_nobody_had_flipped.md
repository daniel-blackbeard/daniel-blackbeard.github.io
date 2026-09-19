---
author: "Daniel Blackbeard"
date: '2026-10-18T19:00:00+02:00'
draft: true
title: 'The Bit Nobody Had Ever Flipped'
tags:
- hdl
- log
---

## The crash comes back a third time

With the DDR/cable mystery closed and the UDP console resumed, a pasted external code review — sixteen findings, ranked — kicked off what looked like routine hardening: a missing memory barrier in the RX release path, missing frame-length guards, missing alignment checks on the console's register-write path. All three got implemented, verified compiling, and soak-tested in turn. None of them changed anything. The exact same crash, same address, same registers, kept reproducing on the same 60-to-120-second timescale regardless of which finding was applied.

## A false all-clear, caught by insisting on a second data point

The recurring signature — program counter frozen inside the fault handler's own UART-dump busy-loop, link register landing on the aligned base address of the program's stack region — coincided suspiciously with something every single test shared: a tight JTAG polling loop running alongside it. A single 5-minute run with zero JTAG contact came back clean, which read, briefly, like confirmation that repeated debug halting — not the firmware — had been corrupting state the whole investigation. That conclusion didn't survive a second sample: a longer JTAG-free run, checked through a passive UART listener instead of any halt at all, crashed at 4.5 minutes — faster than the "clean" run that had seemed to prove the opposite. The retraction was immediate and unhedged: "I need to retract my earlier conclusion... That earlier clean result was just a lucky sample, not a fix — I overclaimed on one data point."

It's the exact same trap as the "40-plus-minute clean run" that got reframed as an outlier during the jitter arc, recurring a session later — except this time it got caught in one exchange instead of costing a full day. Worth noticing on its own: this project actually learned the lesson, rather than just having it happen once.

## "My bisection always finds answers faster"

Handed a wrong theory and a stalled investigation, I pulled things back to method rather than more hypotheses:

> "listen, we have been here before, if I let you assemble a prejudice we start wasting days chasing theories. My bisection always finds answers faster. So let's start removing functionality."

What followed was the same "remove things until nothing is left to remove" discipline as before, run faster, on a different bug: strip the UDP command dispatch out of the RX path — still crashed — strip ARP too, down to a bare poll/release drain touching no frame content at all — still crashed, faster — bump the RX ring from 8 to 512 entries on the theory that wrap frequency mattered — no change, ruled out — strip the RX poll function itself to an unconditional null return, never touching a descriptor — still crashed, in 15 seconds — disable receive entirely, so the hardware never receives a single frame — still crashed, in 40 seconds — empty the main loop down to a bare infinite loop with zero function calls after boot. That one came back clean for a full 10 minutes, the first clean result of the whole arc, isolating the cause to the act of repeatedly *calling functions*, independent of Ethernet, the AD9361, or anything else in the application.

Reading the crash dump correctly took one more piece of understanding: the fault handler's own UART-dump loop reuses several registers for its own bookkeeping, so most of what JTAG had been showing all along was the handler's own scratch state, not the real fault. The handler does save the true original registers to a fixed DDR scratch address before touching anything, though, and reading that back showed the live fault was happening inside the RX release function — a leaf function that never writes to the link register anywhere in its own body — at the exact moment it returned, with the link register corrupted to the aligned base address of the program's own stack region. A hand-derived assembly test program from the earlier DDR arc, a tight call/return loop, but deliberately using on-chip memory for its scratch stack rather than the normal DDR-resident one, ran clean for over 35 combined minutes across two runs, on the exact same call/return pattern. The one structural difference between "always crashes" and "never crashes" was which memory the repeated call/return activity actually touched.

## D-cache, and a trap hiding inside the fix itself

The stack-versus-on-chip-memory distinction pointed at one specific, checkable difference: `startup.S` had enabled the instruction cache since the very first hang investigation, back near the start of this project, but had never enabled the data cache, project-wide, since the first line of assembly this project ever wrote. Every stack push and pop on the DDR-resident stack had always been an uncached, full-latency round trip to external memory. Enabling the data cache on the fastest bisected reproducer — a build crashing 100% of the time within 15 to 40 seconds — ran clean for 30 minutes straight. A striking result, and I treated it with exactly the suspicion the JTAG-polling false alarm a few sections earlier had earned:

> "yeah, but before that, remove D-cache again and soak for 30min total, restoring the test if a crash is detected. Count the number of crashes to understand if this is a coincidence or statistical evidence."

That comparison immediately surfaced a second, unrelated methodology bug: a JTAG warm reset doesn't reliably restore ARM coprocessor state to its true power-on default, so simply reverting the source and reloading wasn't actually guaranteeing the data cache was off. It produced two different new crash signatures across successive attempts, neither matching the original bug, before this was understood. The real fix wasn't a workaround, it was a correctness bug in the test rig itself: `startup.S` needed to explicitly clear the cache-enable bit before setting anything else, rather than assuming a known starting state — the same "don't trust inherited hardware state" lesson the cable arc had already taught about JTAG resets a week earlier, now recurring for a coprocessor register instead of a whole board.

Once the bit was forced deterministic in both directions, the comparison was completely clean: D-cache off crashed on every single boot, byte-identical registers, within one to two seconds, 679 times in a row with zero exceptions. D-cache on ran the same firmware, the same reload path, the same real network traffic, for 30 straight minutes without a single failure. Restoring full application functionality on top of the fix — real RX polling, the ARP responder, the UDP command console, the original 8-entry ring — and soaking that for another clean 30 minutes closed the arc out on the actual target configuration, not a stripped-down stub.

> "I flies over my head why d cache helps, but I will accept the fact."

## The reveal

I settled the two investigations side by side myself:

> "Jitter was a false flag: it was a statistical explanation (among many one could come up) that explained the random phenomenon of the cable creating issues on DDR."

This unifies what had looked like two separate multi-day mysteries into one timeline. The real, underlying vulnerability — the instruction cache enabled but the data cache never touched, at any point in this project's history, leaving every stack push and pop on the hot call/return path exposed to a raw, full-latency DDR round trip — was present for both arcs identically. What made the jitter investigation so much worse, and so much more mysterious, was the bad JTAG cable actively injecting electrical noise into those exact DDR transactions at the same time, independently confirmed by the direct cable swap and the 5.5-hour clean soak. A jitter model can describe random, memoryless-looking corruption just as well as a bad cable can produce it — the statistics fit beautifully to within a picosecond precisely because the model was curve-fitting the right *shape* of randomness to the wrong physical cause. Once the cable got replaced, the D-cache gap didn't go anywhere. It just dropped to a much lower baseline failure rate, which is exactly what resurfaced as this session's "new" bug, days later.

I don't want to hedge on the actual lesson here: a jitter-shaped fit to bad data is still just a fit, and "the numbers work out precisely" is not the same claim as "the numbers work out for the reason I think." That's a better, tighter ending than either investigation's own closing line promised on its own — and it's the one I'm choosing to end this stretch of the project's story on.
