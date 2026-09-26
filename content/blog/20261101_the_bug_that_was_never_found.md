---
author: "Daniel Blackbeard"
date: '2026-11-01T19:00:00+02:00'
draft: true
title: 'The Bug That Was Never Found, and the Two Hiding Behind It'
tags:
- hdl
- log
---

## When "fully working" stops being true

The UDP command console had tested clean the day before — sixty out of sixty round-trips, no drops. Then I ran the sample stream at genuine full rate for the first time, and the console started dropping out. What followed was the most exhaustive investigation this project has produced so far, and it ends, honestly, without a resolution.

I ruled out DDR bandwidth first — the required load sat under 4% of the HP0 AXI port's peak, nowhere near saturation. I built a TX-ring reply-priority scheme, measured it, and it did nothing. I gave replies their own dedicated TX buffer pool, separate from the sample stream, measured that too, and it also did nothing. I went through the PC's own NIC driver settings one by one, disabling anything that looked like it could be buffering or coalescing packets. A live capture showed replies arriving up to 2.4 seconds late and out of order, rather than simply lost — which ruled out a clean drop somewhere and pointed at something queuing or stalling instead. At one point I caught the PC's own ARP cache sitting in a stuck `Incomplete` state that Windows wasn't even bothering to retry on its own.

None of these threads closed the loop. My own read, which held up under everything I threw at it: this was never a TX problem. TX was nowhere near saturated. If anything was actually starving, it had to be on the RX side. That diagnosis is very likely right, and it's the open lead going into whatever comes next — but as of today, the honest state is that the root cause was never found.

Most bug-hunt write-ups end with the bug. This one doesn't, and I'd rather write about that directly than smooth around it. Every step here was a real measurement, not a guess — several plausible hypotheses individually built, tested, and killed with hardware evidence, and still no closure. The workaround that shipped instead — a live UART/UDP transport toggle in the PC-side console, so register access isn't blocked by whichever mystery this turns out to be — is its own small lesson: sometimes the right move isn't "find the bug," it's "stop being blocked by it."

## A wrong assumption, not a bug hunt

Along the way, something that looked like it belonged in the same investigation turned out to be a completely different kind of problem. A four-way mismatch between the sample stream's theoretical and measured data rate had been flagged and set aside as a puzzle for later. Working it through from the AD9361 datasheet directly settled it: for one clock cycle, the chip sends 6 bits of I on the positive edge and another 6 bits on the negative edge. At 30MHz that's 24 bits arriving at an effective rate of 15MHz. That data path is shared between the two RX channels, so in 2RX mode the rate per channel is actually half of that again.

Checked directly against the LVDS interface's frame-match logic, this was exactly right: the valid-sample pulse, in the dual-RX configuration actually in use, only fires once every four `dsp_clk` cycles — a fact nothing in the RTL had ever made explicit, since the signal existed but nothing downstream actually consulted it. My own decimator had been treating every single `dsp_clk` cycle as if it carried a fresh sample, when only one in four genuinely did.

## The same landmine, twice, right next to each other

Gating the accumulator on the real valid pulse fixed the obvious instance. On review I found a second, subtler one sitting right beside it: the `dec_done`/seed branch, one `if` away from the branch I'd just fixed, was still unconditional. I fixed it and then simplified the resulting three-times-repeated valid-signal guard down into one — deliberately keeping the change small, since I was addressing the symptom in front of me, not rewriting the module.

Even after both fixes, the eye diagram still looked wrong — values clustered tightly around 14,000 to 22,000 instead of anywhere near zero. My first instinct was to suspect the signed/unsigned display toggle I'd just added to the PC console. It wasn't that: two's-complement reinterpretation only changes values above 32,768, and every value I was seeing sat well below that. The toggle *couldn't* have caused this — and that was the actual tell. The real cause was a second occurrence of a bug from several nights earlier: the RX data registers had already been marked `signed`, but the four raw ADC input operands feeding into the addition that produced them had not been. SystemVerilog's rule that a mixed signed/unsigned expression gets evaluated entirely as unsigned if even one operand is unsigned had been quietly defeating my earlier fix on every single addition, with no warning from the toolchain. Adding `signed` to the four ADC input declarations resolved the same values that had been sitting at a nonsensical +17,000-ish offset down to a small, physically sane cluster around -375 — a real bug disappearing, not a display artifact being tuned away.

## Closing an old approximation with the real source

One last thread, tidied up rather than left dangling: a long-standing comment in the repo claimed the RX built-in-self-test tone sits at "~931kHz," derived from `dsp_clk/32`. Once I'd correctly worked out that the sample clock is `dsp_clk/4`, not `dsp_clk` itself — the same distinction the datasheet math above had already forced me to nail down — and cross-checked it directly against the AD9361's own reference driver source, the real expected frequency turned out to be 232.8kHz. As a clean structural consequence of the tone generator's own 32-samples-per-cycle definition against the decimator's fixed 8-sample window, the decimated tone lands at exactly one quarter of the decimated sample rate, independent of the exact clock value. A small thing, but a good reminder to trust the actual vendor source sitting in the repo over an old comment that had quietly inherited someone's earlier mistake.

With the DSP correctness work in a good place, this is where the project turned toward the radio itself rather than the bring-up plumbing underneath it. The eth0 mystery stays open, tracked, and deliberately not blocking anything else.
