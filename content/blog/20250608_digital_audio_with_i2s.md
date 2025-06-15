---
author: "Daniel Blackbeard"
date: '2025-06-08T14:12:26+02:00'
title: 'Digital audio with I2S'
cover:
  image: "images/i2s.jpg"
  alt: "Picture of some HDL code"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
tags: 
- hdl
- i2c
- log
---

## Introduction
As you perhaps know, I'm working on this blog also as a way to log my learning journey. So, to support the lack of interesting peripherals on the Arty A7, I bought a set of modules for Arduino. Most of them quite uninteresting, but I got this I2S DAC module from [Adafruit](https://www.adafruit.com/product/3678) that implements the UDA1334A from NXP ([datasheet](https://www.nxp.com/docs/en/data-sheet/UDA1334ATS.pdf)).

This device interest me the most among the ones I got from the assortment, because since I was a kid I wanted to make my own guitar effects (in fact, I believe many electronic engineers are born this way or similar), so the end goal is quite ambitious as you can see. But let's start from something easy.

## I2S protocol
I2S (not to be confused with I2C, it's a different thing) is a serial protocol for transmitting audio that needs three wires:
1. Left and Right selection, called Word Select (WS)
2. Data stream
3. A clock for sampling the data (BCK)

Those three elements have simple rules:
* The data stream has to be at least 8bit (24bit is best) of __signed data__
* The MSB is transmitted first
* The WS makes its transition during the falling edge of the BCK
* After the WS transition, the LSB is processed to give time to the receiver to elaborate the stream before switching audio side

So, basically the stream will look something like this:
{{< figure src="/images/i2s_waveform.png" caption="WS comes before the data stream ends" alt="I2S waveform" align="center">}}

With this said, let's take a look at a simple implementation in systemVerilog for this circuit.

## Implementation
From a module perspective, I want to make it very easy with few requirements for the modules that will use this one. Hence, we will need a clock, the input data to be sent, the tree outputs to the I2S receiver and a `invert` flag in case I wanted to exchange the left and right audio channels. The user of this module will be in charge of sending the data at the right moment with the wide timing margin that the WS offers.

I will send 24bit data at 48KHz because it's easy. This way, I can start from 60MHz clock I will generate by using a PLL in the Artix 7 FPGA and use a strobe to generate all the other signals. A counter from zero to the division ratio will act as state for the system. 

The divisor has to be such that takes 60MHz to 48KHz audio as suggested before, but the streamed data is serial, so the BCK will have a clock frequency of `24bit x 2(LR channel) x 48KHz = 2.304MHz`. Now, it just happens that is hard to make perfect ratios for the many clocks I plan to use, so I will settle with a divide ratio of 26, accepting a non-audible sampling rate at the UDA1334A of 0.16% higher frequency. Not a bad deal.

So, all together, this will end like this:

```systemverilog
always @(posedge clk) begin
  if(strobe==6'b011001) begin
    temp     <= (counter == 5'b1) ? data_in : {temp[22:0], 1'b0};
    strobe   <= 6'b0;
    counter  <= (counter == 5'b10111) ? 5'b0 : counter + 5'b1;
    dx_or_sx <= (counter == 5'b0) ? ~dx_or_sx : dx_or_sx;
  end
  else begin
    strobe   <= strobe + 6'b1;
  end
end

assign ws = invert ? ~dx_or_sx : dx_or_sx;
assign i2c_out = temp[23];
assign bck = strobe > 6'b001100 ? 1'b1 : 1'b0;
```

Quite easy as promised.

So, when the strobe is `6'b011001`, the strobe is reset and the state is increased by 1. If the state is zero it switches the audio channel, then when the state is 1 loads the next piece of data. The BCK is just a cheap 1bit alternating counter, as making this with PLL is not possible.

In simulation, we can obtain this:
{{< figure src="/images/i2s_simulation.png" caption="A stream of 24'hE38E38 is sent here" alt="I2S simulation" align="center">}}

## Conclusion
This piece of verilog, together with a I2S capable device, will allow you to send pieces of audio waveform out to the external world. This is the first step for making a small synthesizer (on the next post) and in the future a complete digital audio processor.

If you want to get the sources of these blocks, as I write this blog I will place them at [my github repo](https://github.com/daniel-blackbeard/verilog_modules)
