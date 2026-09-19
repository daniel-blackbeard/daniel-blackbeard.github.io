---
author: "Daniel Blackbeard"
date: '2026-09-06T19:00:00+02:00'
draft: true
title: 'The Day the RX Synth Locked'
tags:
- hdl
- log
---

## Setting up the day

Two weeks earlier I'd hit a wall: the AD9361's state machine could reach `ALERT` but nothing I did would push it further, because reaching real `RX` state needed a second PLL — the RX local oscillator synthesizer — that I hadn't touched at all. This post is about the day that wall came down. It ended up being the longest single day of the project so far, and it's the one I'd point to first if someone asked what this project has actually been like to work on.

## A hard push on trust, before touching any RTL

It started as a scope question. RX lock kept dropping, and the obvious FPGA-side fix was `IDELAYE2`/`IDELAYCTRL` sample-delay calibration — tunable delay taps on the LVDS input path. Before committing to that, though, a cheaper hypothesis got proposed: try the AD9361's own coarse, chip-side `REG_RX_CLOCK_DATA_DELAY` register first, testable by hand over SPI without touching any RTL. I set the protocol explicitly:

> "1. Ping the board and verify it's still awake... 2. Send the spi commands by hand... checking that the error count stays stable... 3. You can emulate the found algorithm if you want, but first externally before committing to a sw rewrite. understood?"

And before trusting any result from that channel, I asked the question that actually mattered more than the hypothesis itself:

> "okay for the jtag to spi trick, but do you even know it does work? like, can you read using that spi to confirm you're talking to the device?"

That got answered properly — reading back a register set once at boot and never rewritten since, and separately a genuine hardware status bit (`BBPLL_LOCK`) that software never writes to. Both read back correctly, which ruled out the test methodology itself producing a "write echoes back as read" artifact. It's a small exchange, but it's the whole discipline of this project in one place: verify the instrument before you trust what it tells you.

## The sweep that did nothing, and a signal that turned out to be dead

The actual sweep — 18, then 22 settings across the full `REG_RX_CLOCK_DATA_DELAY` range — produced exactly zero effect. A real, well-verified negative result. That redirected the investigation toward raw signal capture: new debug taps into the LVDS interface reading its pre-decode state directly, ungated by any lock status.

What they showed was worse than misalignment: the frame and data lines weren't toggling-but-offset, which delay taps would have fixed. They were frozen solid, bit-for-bit identical across 20 consecutive samples, while a heartbeat counter in the same clock domain kept incrementing the whole time. The sampling clock was alive. The signal content was dead.

At that point I stopped and asked for research, not action:

> "make a small research, this sounds like a misconfiguration on ad3961 side, take no action, just dig for info, we stop here"

The research turned up something real: `ENABLE_RX_DATA_PORT_FOR_CAL` — the bit my test-mode path had been relying on this whole time to force RX data active — is only ever asserted in ADI's actual reference driver inside a block wrapped in `#if 0`. Dead code. It had never been a working mechanism to begin with; test mode "working" all along had been an illusion.

Then I reversed course and asked for one more experiment before stepping away:

> "wait, let's do the last one: test that theory, I come back in 30min."

A full power cycle and from-scratch reinit reproduced the same brief-burst-then-frozen pattern — but with a *different* frozen value every time: 133, then 27, then 38. That inconsistency was the actual tell. A real, deterministic pattern would lock to the same sequence every reset. Different garbage every time meant the earlier "valid" reads had been transient noise, not real signal — there had never been genuine locked data in the first place, just something that looked like it briefly.

## The real answer, cross-checked against the actual driver

In parallel, hours of independent reading had produced a writeup on the AD9361's ENSM bring-up sequence, sourced from ADI's BIST FAQ, the register map reference, the datasheet, and an EngineerZone forum thread — proposing the real fix was walking the ENSM state machine through SPI (`WAIT → ALERT → RX`, using a `FORCE_RX_ON` bit) instead of relying on the calibration-only bit.

Cross-checked directly against the real reference driver already sitting in the project's docs: `REG_STATE` had read `0x05` (ALERT) in every single capture, all day. It had never once left ALERT. The actual `ad9361_ensm_set_state()` function explained exactly why — reaching RX requires `FORCE_RX_ON`, and `ENABLE_RX_DATA_PORT_FOR_CAL` is never a substitute for it, matching the dead-code finding exactly. Writing `FORCE_RX_ON` directly still didn't move `REG_STATE`, though — a second, deeper finding: the RX synthesizer's own VCO-lock status register read `0x00`. The chip's ENSM hardware was refusing to transition, gated on a PLL that had simply never been programmed. Everything all day — the frozen signal, the dead BIST bit, the failed forced transition — had been the same single root cause, the whole time.

## Deriving the synth, live, from the real driver

What followed was a from-first-principles derivation, register by register, against the actual reference driver functions handling synth calibration and integer/fractional divider math — the same rigor the BBPLL bring-up had used two weeks earlier. One assumption got corrected mid-derivation and flagged rather than quietly patched over: I'd assumed the 40MHz reference got doubled to 80MHz for the synth path, and once the actual written register bits were decoded, that turned out wrong — it's a straight 40MHz passthrough, which changes every downstream divider number.

Final numbers for a 98MHz RX LO: VCO divider 5, integer divider 156, fractional word 6710874 — hand-verified on real hardware, working on the first try. VCO lock asserted immediately, `REG_STATE` moved from `0x05` to `0x08` — genuine RX state, not test mode — and with BIST armed on top, the RX data output changed on every single sample across more than 20 consecutive polls. The digital RX path had never been genuinely alive before that moment, in three weeks of work.

> "We got what we wanted."

## The payoff, twice over

With a real signal finally flowing, I re-ran the exact same coarse delay sweep that had done nothing at the very start of the day. This time it found a sharp, unambiguous transition: below a threshold setting, the frame pattern stayed dead and an error counter free-ran continuously; above it, the frame pattern settled, the error counter went flat, and a valid-sample counter started incrementing fast enough to wrap its own 8-bit range between two 300-millisecond-apart reads. Same experiment, same code, only meaningful once there was something real underneath it to align.

## A correction on where this belongs in the firmware

Planning the permanent port, my first draft put the whole RX synth bring-up inside "mission mode," gated behind a UART command. That was wrong, and I caught it myself:

> "there are common settings that are valid for both mission mode (real radio waves) and test mode (PRBS, tone from digital inside AD9361)... the common function gets always called at startup."

The day's own results proved the point: reaching real RX state is what had unblocked BIST/PRBS data in the first place. It was never mode-specific. The synth bring-up, delay tuning, and state transition all moved into the always-run boot path, and `enter_test_mode()`/`enter_mission_mode()` shrank down to just their genuinely mode-specific single steps.

The day closed with a UART protocol extension — read access to the RX diagnostic counters, previously JTAG-only — and a cleanup pass removing bring-up-era debug scaffolding that had outlived the bugs it was built to chase.

Looking back at the whole day: a wrong theory, a red herring that turned into a real clue, an independent research thread that got fact-checked against source instead of trusted outright, a live "it just worked" moment, and a clean architectural correction at the end that's really a callback to the day's own central insight — state, not mode, was always the real variable. If I could only tell one day from this project in full detail, it would be this one.
