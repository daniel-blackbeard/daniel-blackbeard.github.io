---
date: '2026-04-07T09:39:21+02:00'
draft: true
title: 'Analog preamplificator for the XADC'
author: "Daniel Blackbeard"
cover:
  image: "images/preamp_pcb.jpg"
  alt: "Picture of a PCB work for a preamplificator"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
tags: 
- analog
- log
math: true
---

## Introduction

Let me take a detour from the HDL and digital things and go back for a while into analog domain. For this whole system I'm building to be any useful, the signal from the guitar has to have enough amplitude to:
* Dominate the noise of the processing chain
* Use as much as possible the XADC dynamic range.

In any case, I need something simple while also thinking I wouldn't be able to respect myself if I do it in a circuit mounted into a breadboard. So, I took also the chance to learn the skill of PCB production in a long afternoon.

But let me go in order of things

## Preamp design
Again, something simple is good enough for this application, I don't need accurate control of non-linearities and although power consumption is not a concern, I want to keep it low. Usual guitar pick-ups tend to have an output amplitude onto a capacitive load of around 10-50mV, so 20dB of gain are enough to avoid clipping and to make this stage to dominate the [noise figure](https://en.wikipedia.org/wiki/Noise_figure) of the chain. Volume control will be done in the instrument (guitar in this case) and on digital.

Also, I will need a pass-band that is adequate for audio, so a lower cutoff frequency bellow 20Hz and a higher cutoff frequency around 3KHz. Why not 20KHz? The guitar on E7 (last fret on the E string) produces ~2640Hz, on audible frequencies it will have another 7 harmonics which I will just decide to remove completely. Also, letting more bandwidth that is not used on the average application means that the noise that exists on those unused frequencies will pass too (thermal noise has a white spectrum). 

Additionally, the XADC required a DC common mode voltage at the input of 500mV. This means that the amplifier has to have a bias point at the output of 500mV, behave differentially and have a decent rejection of the supply noise.

So, let me summarize the specifications for this analog block:

|Spec               |Value  |Unit  |
|:-----------------:|:-----:|:----:|
|Gain               |20     |dB    |
|Low cut frequency  |<20    |Hz    |
|High cut frequency |3K     |Hz    |
|V out (DC)         |500    |mV    |

So, let me tackle the bits of design to come out with a complete proposal.

### Circuit Topology
We will need differential signal at the input of the XADC, but most guitars provide a single ended signal. So we will need to use a topology that allows for single ended to differential. 

This is a classical circuit on analog front-ends when it comes to optical communications:

{{< figure src="/images/preamp_schematic.png" caption="Schematic of a front end capable of single ended to differential conversion" alt="Schematic of a front end capable of single ended to differential conversion" align="center">}}

If you're familiar with analog circuits, this will look like a differential pair with a grounded input. It is. But to recognize better the single ended to differential conversion, it's more useful to see at this circuit like a regular common emitter inverter that it's also an emitter follower, which is driving a common base amplifier. This way, when you see the slight mismatch between single ended gains you will not be that amazed.

### Component values choice
In choosing the component values we have one important criterion: the output voltage has to be around 500mV. So whatever resistor we choose will also determine the current on the emitter of this amplifier and vice versa. Let's do some quick and dirty calculations and throw a few formulas that will drive the design. First, the emitter current will be approximately twice the collector current, hence 

$$ I_E = 2\frac{V_{cm}}{R_C} $$

At the same time, the single ended gain will be determined by a dirty ratio between the resistor of the collector and the small signal emitter impedance:

$$ r_e = \frac{V_T}{I_E} $$

where 

$$ V_T = \frac{K_B T}{q} $$

and is around 25mV at room temperature. Plug everything together and you get the small signal differential gain:

$$ A = \frac{R_C}{r_e} = \frac{V_{cm}}{V_T} = \frac{500mV}{25mV} = 20 $$

Interestingly, this topology and the request of a fixed common mode voltage imposes on a first order a gain of 20dB for each side, hence ~26dB differential regardless of any choice of component of current (that doesn't break my quick and dirty formulas). Good enough, I will keep it.

However, the choice of current will impact the input impedance of the stage, and this input impedance together with the decoupling capacitor at the input will determine the lower cutoff frequency. Again, let's do a quick and dirty calculation of the values required to abide the spec.

$$ Z_{in} = R_1 | R_6 | Z_{b} $$

where 

$$ Z_{b} = \beta r_e = \beta \frac{V_T}{I_E} $$

with $\beta$ around 100 (typical values for small signal transistors) and let's say around 15uA current to not starve too much the transistors, $r_e$ end being in the range of 1.5KOhm, so $Z_{b}$ being around 150KOhm or more. Assume overall 100KOhm $Z_{in}$. So, with a capacitor of around 1uF we get a cutoff frequency of 

$$ f_{low} = \frac{1}{2\pi Z_{in} C} \approx 1.6Hz $$

All of this is working given the availability of the components in my desk, hence input cap will be 1uF, and I will stick with collector currents of around 15uA. So, if this is the collector current, that means the $R_C$ has to be around 33KOhm. For simplicity, I will stick 33KOhm on the emitter, that will have a drop twice as much. This means that with a supply of 3.3V, the $V_{CE}$ is 1.8V. This is plenty to keep the transistor in active region when the swing on the collector goes down. The emitter choice of resistor however, binds the choice of biasing resistors (their ratio mainly), as I will need the emitter to be at ~2.3V, this means the base has to be at ~1.6V. Interestingly enough, this will work with what I have, as I can make this ratio 1/2 the supply voltage and have 1.65V. Slightly more, which will mean more emitter current, lower input impedance, higher inferior cutoff frequency. Given the values already obtained this might work for me. 

There is a last question to be answered and is the high cutoff frequency. The XADC has an equivalent input circuit like in the figure

{{< figure src="/images/arty_xadc_input_impedance.png" caption="Model of XADC input impedance, from Arty documentation" alt="Model of XADC input impedance" align="center">}}

So, we have those 33KOhm collector resistors being the main contributor to output impedance, with the 1nF capacitor. This leads to a cutoff frequency of 

$$ f_{high} = \frac{1}{2\pi Z_{out} C} \approx 2.4KHz $$

I'm taking this value as well. If I weren't happy about it and wanted to increase it or decrease it, I would change the values of $R_C$ and $R_E$ while preserving their ratio until I get the frequencies where I want them to be. This would also mean to fix again the base biasing voltage. Usually an iterative process, aided with simulator. 

### Simulation results

So, indeed, let's see what the simulation says and how far am I from what I wanted.

First an DC OP simulation and check the voltages at the different nodes: 

{{< figure src="/images/preamp_sim_dcop.png" caption="DC OP simulation of the preamplifier" alt="DC OP simulation of the preamplifier" align="center">}}

We got higher collector current, leading to higher $V_{cm}$. Does this bother me? not too much, I could tweak a bit the emitter resistor but discrete components have some granularity, and it's not really worth over-optimizing. The problem was indeed the ratio of the base biasing resistors, those extra 50mV added some emitter current. Nothing crazy, let's see now the small signal behavior:

{{< figure src="/images/preamp_sim_ac.png" caption="AC simulation of the preamplifier" alt="AC simulation of the preamplifier" align="center">}}

The lower cutoff frequency is ~1Hz, good. The higher cutoff frequency came little than expected. This might be mainly to overestimating the $r_c$ of the transistor and its associated parasitic capacitance. Again, I can raise the bandwidth a bit by tweaking with the resistors, but this will work for my uses. Any needed emphasis can be done in digital if necessary.

I guess now you understand the meaning of "quick and dirty". 

A soft specification is the [PSRR](https://en.wikipedia.org/wiki/Power_supply_rejection_ratio). I will not go crazy optimizing this either as it will depend a lot on the circuit topology, but let's characterize it.

{{< figure src="/images/preamp_sim_psrr_se.png" caption="PSRR simulation of the preamplifier, single ended output" alt="PSRR simulation of the preamplifier" align="center">}}

The single ended PSRR will not be that good, considering the disturbance on the biasing resistors just split it on half to get amplified by the very same stage. However, on differential mode, this disturbance gets completely rejected (aside from mismatch issues on the components) thanks to the differential conversion is not happening on the other side of the amplifiers (the AC grounded base). Just trust me on this, values better than -80dB are easy to obtain and the XADC itself will reject any common mode disturbance anyway.

## PCB design

Let me be honest with you: since I was 12 years old and did this with [iron chloride](https://en.wikipedia.org/wiki/Iron(III)_chloride) and sharpies, I haven't done any PCB until now. All my design experience focuses on [finFET](https://en.wikipedia.org/wiki/Fin_field-effect_transistor). So I wanted to take the challenge and learn how to do this myself, considering I have much experience on analog design I expect this to be feasible, just a fight against the tool. Well, as you have noticed, also my simulations I did them with KiCAD and I have to say I'm happy about this tool also for simulation. In a few hours I went from nothing to a simulated design and on another two hours and the help of AI for forum related questions I got my PCB. For now let me just show you a couple of choices I made, and I promise I will test it completely when I get my board back from factory.

{{< figure src="/images/preamp_pcb.png" caption="My first PCB design on CAD ever" alt="PCB design of a preamp" align="center">}}

First, there is the main part of the amplifier, with all the bunched components. I do this to reduce parasitic routing, but honestly I didn't think much about the parasitic effect at the output (already underestimated). This will lower further my bandwidth, probably to 2KHz, but let's see. I placed also probing places for the different voltages and points for the oscilloscope at the output (extra capacitance). At this point I'm thinking if I'm not happy I can always optimize the resistors choice, but I will not fret too much. None really uses those very high frequencies while playing guitar (or I'm not skilled enough to do it anyway). There are headers that are to be compatible with the Arty A7 headers and two headers to hold the I2S module I used in a former blog article.

{{< figure src="/images/preamp_pcb_3D.png" caption="3D view of the PCB, not too shabby" alt="3D view of the PCB" align="center">}}

## Conclusion

This was a quick and dirty design of a preamp to be used as analog front end for the Arty A7 XADC. Few things to improve, but I guess that's the beauty of analog design. I regard it as one of the last artisanal jobs that managed to modernize to the space era.