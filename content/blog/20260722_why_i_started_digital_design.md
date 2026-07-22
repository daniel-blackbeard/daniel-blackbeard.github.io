---
date: '2026-07-22T04:53:37+02:00'
draft: false
title: 'Why I Started Learning Digital Design (After 15 Years of Waiting)'
cover:
  image: "images/riscv-pipeline-paper.jpg"
  alt: "Picture of some HDL code"
  relative: false
tags: 
- log
---

## Why I Started Learning Digital Design

I was ten years old when I first touched a computer. I don't remember much of that first encounter, except one thing: I became obsessed with the idea of the Arithmetic and Logic Unit.

My teacher mentioned it once, briefly, the way you mention something to a room full of ten-year-olds — no details, just the name. But in my head, the ALU became this almost mythical machine. Something that could take any calculation, any problem, and just... solve it. I didn't know how. I just knew I wanted to.

That question followed me. At 14, I started learning electronics because of it. At 16, I learned to program, hoping that would finally explain how a CPU actually did what it does. It didn't, not really. I became a physicist instead — drawn to how the universe organizes itself, from particles to galaxies. Later I became an analog IC designer, chasing a different but related fascination: wireless communication, the invisible signals that connect everything.

But the ALU was still there, in the back of my mind. The dream never left. I just didn't have the means to chase it — no hardware, no board, nothing to actually put my hands on. And I've always been a hands-on learner. Theory alone was never going to satisfy me on this one.

It took fifteen years to get my first FPGA board.

When I finally opened up the ALU, it was almost anticlimactic

Here's the thing nobody tells you: the ALU doesn't do everything. It does almost nothing, on its own. A handful of simple operations — add, shift, compare, a bit of logic. That's it. Anything that looks like real computation is actually a sequential algorithm, built by combining those few simple operations again and again, in the right order, at the right time.

Simple? Absolutely. Elegant? Without a doubt. But it deflated the picture I'd been carrying since I was ten. It's a bit like being catfished on Tinder — the profile promised something bigger than what showed up.

And yet, that deflation turned into something better than the myth: real understanding. Because it turns out the beauty was never in the ALU doing something magical. It was in how something so simple, repeated and sequenced correctly, becomes powerful enough to run everything we build on top of it.

## Why this, and why now

If you know how I work as an analog designer, you know I've never been comfortable staying inside my own lane. I do my best work when I understand the language of the teams around me — layout, firmware, system-level, digital. I think of myself as a kind of diplomat between disciplines. So the shift toward learning digital design wasn't really a shift at all. It was the natural next step of a habit I already had: understanding things one building block at a time, then seeing how they assemble into something larger. That's the same instinct that made me a physicist in the first place.

So I bought an FPGA board, and I'm now building a pipelined RISC-V CPU from scratch — one stage, one hazard, one bug at a time. I'll be documenting the whole thing here as I go.

## The point of all this

I'll admit this is a fairly self-absorbed story. But if there's one thing I want someone reading this to take away, it's this: keep chasing. It's never too late to keep discovering. The universe is vast and, when you look closely, wonderful — nature spent billions of years organizing atoms into life, and now we get to spend our short time here organizing that same universe further, building things nature alone never would have.

I waited fifteen years to build my first CPU. I'm glad I never stopped wanting to.

Follow along as I document the build — from single-cycle to pipelined, hazards, forwarding, and everything else I'm learning along the way.