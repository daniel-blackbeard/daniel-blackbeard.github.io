---
author: "Daniel Blackbeard"
date: '2026-10-11T19:00:00+02:00'
draft: true
title: 'The DDR Bug That Turned Out to Live in a USB Cable'
tags:
- hdl
- log
---

## A second multi-day mystery, structurally familiar

Right on the heels of the jitter arc, a second hardware mystery opened up — structurally similar in shape, a bug chased for days through firmware and configuration that turned out to live entirely outside the code, but this time the instrument that finally cracked it wasn't a statistical model. It was a human ear.

## Day five, and a fix already decided on

I picked this up mid-investigation, on what I'd already been counting as the fifth day of an intermittent DDR read-corruption bug — random kernel oopses, panics, and hangs, running with ECC disabled so nothing self-corrects along the way and every corruption stays visible. I opened with a specific experiment already decided on, not an open question:

> "Ok, this is the 5th day since the bug (although most of today I was away) I got feed of this stuff so let's apply the fix I was thinking since the beginning: Change PCW_UIPARAM_DDR_FREQ_MHZ from 533 → 400 and soak with our usual protocol for one hour."

The frequency change, plus restoring a separately-derived board-measured DQS delay correction, got applied live through Vivado's own IP customization API rather than by hand-editing the underlying project JSON — the safer path once I'd confirmed this particular property wouldn't hit a range-validator bug that had forced hand-edits on an earlier revert attempt. A clean hour-long soak came back with zero hangs. Then I asked for something bigger overnight:

> "After this one, prepare a 8 hours soak. When I awake I will check the results. Good night."

The real picture only showed up at that scale: 12 hangs across roughly nine hours, no clean periodicity, gaps as short as two minutes and others approaching fifty. The frequency change had clearly moved the failure rate. It hadn't fixed anything.

## An eye scan that came back clean everywhere

Mid-soak, I proposed a direct hardware measurement to settle whether the delay correction had actually pushed the DDR PHY's calibration into an extreme, over-corrected state: reading the PHY's own per-lane delay-line registers directly over JTAG, cross-checked against a maximum tap value supplied by a second Claude instance with direct datasheet access.

> "The idea is that in any case we got delay wrong and we should trust the vendor. After this soak we will calibrate this shit ourselves"

The readback invalidated the premise before any calibration project could even start: every lane's tap value sat within single digits of zero, nowhere near the register's ceiling. The fix's effect on the delay lines was real, but tiny — not the wild over-correction the hypothesis needed. That motivated the cleanest instrument of the investigation: a from-scratch eye-scan tool, stepping every possible tap value, coarse then fine, on both the read-delay and write-delay registers for the two affected byte lanes, verifying a small memory region at each point. Every single tap across the entire range passed. The delay-tap value was conclusively not the failure boundary — a real, well-instrumented negative result, not a shrug. The tool itself needed one round of debugging first, worth a small aside: a Tcl gotcha where `expr {0x$var & $mask}` fails because the expression compiler tries to lex the bare `0x` as a numeric literal before the variable ever gets substituted in — fixed by switching to `scan` instead. Even the diagnostic instrument needed its own bring-up before it could be trusted.

## The moment stock firmware crashed too

The pivotal reframing came from booting the board's genuine vendor reference firmware — a real ADI PlutoSDR Linux image, off an SD card, logged into over serial with the well-known default credentials — a firmware image containing precisely none of this project's own PS7 configuration changes. I confirmed through the kernel's own live clock tree that this stock image runs DDR at the standard 533.33MHz, not the 400MHz under test.

It crashed anyway. Different oopses on different boots, always in the flavor of corrupted CPU or kernel state rather than a reproducible code defect — a fault landing in the exception vector's own entry code, a null pointer inside the scheduler's load balancer, a bad pointer walked during ordinary TCP receive. One crash sequence even revealed the stock image runs a 10-second hardware watchdog, silently self-healing via automatic reboot — meaning this class of failure may have always been a known, tolerated characteristic of this platform, quietly papered over in normal use.

That result reframed the whole investigation: whatever this was, it wasn't specific to anything this project's own firmware had ever touched.

## The click

With both software and configuration cleared, I reported something that had never shown up in any log file:

> "I think I found the issue, but it's an ugly one: while connecting the jtag cable there is an audible electrical 'click'."

The controlled test that followed was about as clean as this kind of bug ever gets: stock firmware, JTAG cable fully disconnected — only UART and Ethernet live — soaked overnight through a serial-console poller. Roughly thirteen-plus hours, one non-fatal oops, one reboot, against every single JTAG-connected run of the entire investigation failing within twenty to fifty minutes, sometimes within minutes of the cable going in. Reconnecting the original cable reproduced a fast crash again almost immediately. A schematic check ruled out any electrical path between the board's other USB port and JTAG, and confirmed JTAG and UART share a single connector — explaining why the serial console had stayed alive throughout every JTAG-only disconnection test.

Then a scrappy, decisive experiment, in my own words:

> "Since I'm poor and dirty but have plenty of USB cables around, I changed the one I'm using for JTAG, is not data isolated, but it might actually block it."

A swapped cable — not even properly isolated, just a different one — JTAG reconnected, running the project's own bare-metal soak protocol for a full five hours: zero hangs. Extended by a harmless supervisor quirk to roughly five and a half hours: still zero. The best result of the entire multi-day investigation, on the one variable nobody had originally suspected, because it isn't code, isn't configuration, and doesn't show up in any log. A bad cable.

## An unfinished ending, on purpose

Investigating how to load firmware without JTAG — since the whole existing dev flow depends on it — turned up a genuine shortcut: the board's QSPI flash already holds a real, bootgen-produced boot image, confirmed via the literal Zynq boot-ROM sync word and Xilinx ID magic bytes at the expected offsets, plus a partition decoded out of bootgen's byte-swapped name-field convention that turned out to hold a working FSBL for this exact hardware — extractable, rather than needing to be generated from scratch.

The session closed on genuine cautious optimism rather than a declared victory:

> "Later on if this works we will have to revert a lot of things. Hopefully this is the solution. Wish us luck and see you later from remote."

I'm resisting the temptation to write this as a clean "and then it was fixed" ending, because that's not how it actually closed — my own framing at the time was explicitly conditional, and that's more representative of real hardware debugging than a tidy bow. It's also worth being upfront that solving this meant walking back several days of DDR configuration changes — frequency, delay correction — that addressed a problem which, in the end, wasn't really there. A real, slightly deflating, entirely typical cost of chasing a bug down the wrong layer for a while before finding the right one.

Two mysteries, two different senses doing the actual diagnosing — a jitter-budget calculation from one, an audible click from the other. What I didn't know yet is that they weren't actually two separate bugs at all. That's next.
