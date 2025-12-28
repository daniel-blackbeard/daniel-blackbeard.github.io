---
date: '2025-12-27T17:00:43+02:00'
title: 'Arty A7 Guitar Effect Chain - Introduction'
author: "Daniel Blackbeard"
cover:
  image: "images/effect_chain.jpg"
  alt: "Picture of a guitar effect chain"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
tags: 
- hdl
- xadc
- analog
- log
math: true
---

## Introduction

It took me a while to come with worthy content, but here I am. I have been working on getting my FPGA board to sample analog audio with the internal ADC for a project proposed by a colleague as a way to learn the intricacies of digital design. Let me tell you straight: [documentation about the ADC of this Artix 7](https://docs.amd.com/r/en-US/ug480_7Series_XADC) device does a great job at being confusing. I'm not ashamed to say that this is the part that took me the longest to figure, also considering the paranoia about damaging the components with non-compliant voltages.

So, before starting with anything related the actual implementation of this effect/filter chain in an FPGA, let's agree on some architecture. Consider my limited experience on this kind of projects, many this I found they can be done better. But for starters this will work.

## Effect Chain Architecture

Let's set some constraints to the design of this architecture<cite>[^1]</cite>:
1. I want 48KHz sample rate for the audio, for using my already designed [I2C interface](../../blog/20250608_digital_audio_with_i2s/)
2. I want maximum sample rate possible from the XADC (from documentation, 12bits 1MSPS)
3. I want to avoid clock cross-domains, hence the same clock will have to drive the ADC and the I2C interface

The third constraint is tricky, because the XADC takes 26 clock cycles<cite>[^2]</cite> to sample the input, so if I wanted 1MSPS, I would need 26MHz clock. However, I also need 48KHz clock/strobe for the I2S and the audio processing chain in general. Considering the PLL in the Artix 7 device can't go bellow 10MHz, I will have to strobe the audio chain out of the same clock as the ADC while also making them synchronous (an integer number of samples from the XADC processed per 48KHz period). For this reason, I will choose the frequency of the PLL as 60MHz, set the divider on the XADC by 3 (hence it will have an internal clock of 20MHz) and set the divider of the audio chain by 26*48 = 1248. This will make the audio chain to have a sampling rate of 48076.92Hz. Close enough, none will notice.

Still, there is the discrepancy between the sample rate of the XADC (about 770KSPS) and the 48KSPS we want for audio processing. But that comes easy and with benefits: let's just add an accumulator by 16. So basically, we accumulate 16 samples of the XADC with two benefits:
1. It reduces the noise of the analog front-end and XADC by applying a low pass filtering
2. It increases the resolution of the data from 12bits (XADC samples are 12bit) to 16 which I like more.

So, we could put a block diagram for the system as follows:

{{< figure src="/images/audio_chain_arch.png" caption="Block diagram of the effect chain architecture" alt="Audio effect chain block diagram" align="center">}}

In this blog I will just briefly outline the XADC configuration, in case you find this blog while trying to sample with it as I did, and the accumulator code. In the next articles I will be covering the next pieces:
1. Analog front-end design
2. Finite Impulse Response (FIR) filter digital design
3. Control unit for a pseudo register map implementation and UART communication
4. Routing system for the effect chain
5. Brief discussion on the effects that will be created

At the moment of writing, I have everything but the last two points, hopefully by the day I start writing it I have completed the project.

## Quick XADC snippet

As commented before, documentation is fairly complicated to navigate, and it took me a while to figure this out, so I'm just giving you a recipe to start with the XADC for your own projects. I have put the most important comments on the code to set it up properly. For the internals, use the wizard for single channel, continuous sampling from the analog pins as set from the design constraints file. 

Set other settings as you like as they aren't important at all. As I don't need the extra complication of an AXI bus, I will stick with DRP

```systemverilog
xadc_wiz_0 XADC(
          .daddr_in(8'h03),
          .dclk_in(ck_main),          // 60MHz clock
          .reset_in(0),               // set a proper reset signal
          .busy_out(),
          .den_in(xadc_enable),       // wire to enable XADC signal
          .channel_out(),
          .do_out(xadc_out),
          .drdy_out(sampling_done),
          .eoc_out(xadc_enable),      // wire to enable XADC, for continuous sampling
          .eos_out(),
          .ot_out(),
          .vccaux_alarm_out(),
          .vccint_alarm_out(),
          .user_temp_alarm_out(),
          .alarm_out(), 
          .vp_in(vp_in),             // Analog inputs on the FPGA
          .vn_in(vn_in));
```

## Accumulator implementation

This is nothing fancy: we use the main clock and a carry adder for accumulating samples until it counts 16 steps, then providing the result to the output and a signal for `sample_ready` We can do this really quickly without timing issues in the available sampling time of the XADC (1.3us) so simple strategy is better. The XADC even provides a signal for `sampling_done` we can use to not waste transitions on this little block. 

Here is the snippet:

```systemverilog
module accumulator_16(
  input clk,
  input wire [15:0] in_data,
  input wire data_ready,
  output reg [15:0] out_data,
  output reg done
  );
  
  reg data_ready_edge      = 1'b0   ;
  reg data_ready_old       = 1'b0   ;
  reg [19:0] accum         = 20'b0  ;
  wire [19:0] in_data_temp          ;
  reg [3:0] counter        = 4'b0   ;
  
  always @(posedge clk) begin
    data_ready_old <= data_ready;
    data_ready_edge <= data_ready && ~data_ready_old;
    
    if(data_ready_edge) begin
      accum <= accum + in_data_temp;
      counter <= counter + 4'b0001;
      if(counter == 4'b0000) begin
        out_data <= accum[19:4];   // this is like average, divide by 16
        accum <= in_data_temp;
      end
    end
  end
  
  assign done = counter == 4'b0001;
  assign in_data_temp = in_data[15] ? {4'b1111, in_data} : {4'b0000, in_data};
endmodule
```

Now, if you're paying attention, you will see the input of the accumulator is 16bits, not 12bits. Well, it just happens that the XADC will provide 16bits, where the 4LSBs are declared for "averaging". Which is what we are doing right here. Still, I will be keeping the 16bits output. For my purposes this will give me enough resolution for all the computing I have to do on the guitar audio.

## Conclussion
We are providing here the beginning of a hobby project I'm dreaming since childhood: my own guitar effects. We set the strategy and the next steps to make this become reality.

Stay tuned


[^1]: As the good master Aloys said in Fux's book: the mastery of the rules is the way to artistic freedom, it's within the constraints that real creativity is born.
[^2]: I would infer it's a SAR ADC, so 12x2 clocks for comparator/feedback, plus 2 to propagate the sample to output. But this is conjecture from my side