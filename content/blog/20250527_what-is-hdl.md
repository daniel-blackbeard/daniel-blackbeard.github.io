---
author: "Daniel Blackbeard"
date: '2025-05-27T16:43:31+02:00'
title: 'What Is HDL? The explanation you did not requested'
cover:
  image: "images/hdl.png"
  alt: "Picture of some HDL code"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
tags: 
- hdl
- log
---

## Introduction
If you are a hardware engineer you need no explanation though, this is more for the programming guys who attempt to make things with an FPGA in the belief that is the next step of Arduino (I do think that though, so no judgment made here)

As someone who wrote a lot of code in my young age (still loving fiddling with Linux kernel), I'm lucky I got the purpose of HDL.

Given the amount of times I have seen the question raised on stackoverflow or reddit on the struggle of programmers that approach the field of digital design, this is not a wasted question. So what is even HDL?

## What is HDL?
Basically it stands for _Hardware Definition Language_. Still a language but not for programming on a Turin machine. 

So, for the software programming team, but those who do real programming (hence know at least Lisp, preferable Haskell), this is kind of functional language. It is actually a netlist language, where you describe the connections of digital circuitry, input to the block (HDL module) and the outputs that will go somewhere. And this pretty much resembles the behavior of functions (more than zero inputs, at least one output). So, if you know your functionals languages, you will fare very well with HDL. Else I pretty much recommend Haskell as your next step in programming muscles.

In this blog I will mainly write in SystemVerilog, a superset of Verilog, which is one of the most popular hardware definition languages. The other one that is out there is VHDL. As usual, they both are capable of doing pretty much what's needed for having the job done, it's a matter of preference (or what you are forced to use in your workplace).

## What is this `wire` and `reg`
In logic design you have ones and zeros. Sometimes some floating piece of metal that is not connected to anything of something which you can't tell what's the current state when you plug a supply (remember, we are still talking about circuits, forget definite initial states).

A `wire` is an element that alike a real wire, can carry information from point A to point B. That's all. In HDL, you need to define wires so that you can differentiate by labels the connections from one module to another. An example in here:

``` systemverilog
wire bus_in1, bus_in2;
wire bus_out;

assign bus_out = and(bus_in1, bus_in2);
```

Let me reiterate once again: we are dealing with logic, which is akin to functions (some inputs, at least one output). With wires, we defined where this `and()` element (the circuit for the logic AND operation) is connected. Perhaps also which whom.

{{< figure src="/images/and_gate.png" caption="They are just labels you use to identify connections" alt="AND gate" align="center">}}

Instead, the `reg` is just like a wire that can store its state, like a memory element. Here comes the confusion oftentimes, as a wire is not a complicated circuit, just a piece of conductor. Instead, whatever gets synthesized by a reg ends having some transistors. Hence, a reg will synthesize either a latch (a memory element that's dependent on the level) or a flip-flop (a memory element that's dependent on the transition of a control signal, almost always a clock). 

If you don't know what latches or flipflops are, here is a brief explanation:
> A latch is a memory element that sets the output equal to the input when actively driven. When no driving is given to the latch (no electrons in the input, or floating input) it preserves the last state. A flip-flop will propagate the state of the input to the output only when a transition either from zero to one (raising or positive edge) or from one to zero (falling or negative edge) happens.

As a general rule, in digital design you avoid latches as are very technology dependent on their implementation (they can be very power hungry sometimes). From now on let's set the rule that we will try to synthesize only flip-flops.

## They said HDL has intrinsic parallelism
HLD are as much parallel as two logic gates working at the same time. You describe the logical circuits and nothing stops it from making two operations at the very same time (disregarding propagation delay in logic gates)

## So can I do a `for` loop?
*Hardware Definition Language!*

Try to make a for loop out of AND/OR/NOT gates on a breadboard. Sure you can, but there are better ways to make a loop that save components (fewer components, less area, less power wasted, polar bears not floating around Sicily).

{{< figure src="/images/poor_beardy.png" caption="Yup, when you shitpost on reddit global warming send beards to Sicily" alt="Polar bear in Sicily" align="center">}}

However, for all the effects, HDL is capable of parsing a for loop, as it will infer the attempt of the engineer and generate the equivalent circuit that reproduces that _behavior_. You can pretty much describe the behavior of your circuit in HDL, no warranties that the synthesis will be the best one for a certain algorithm. 

Again, team Haskell wins here against team Java (lol). Recursion basically is a way of making pipelines in digital design.

## So, am I ready to write HDL after reading this?
Let's be honest: you were always ready to make digital circuits, just that your mindset was obfuscated by the prejudice that HDL are programming languages. Now that you know they aren't, and you know that they basically describe the connections and/or behavior of logical circuits, the doors are open for you.