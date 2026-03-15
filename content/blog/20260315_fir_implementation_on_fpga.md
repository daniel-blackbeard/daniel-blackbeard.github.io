---
date: '2026-03-15T11:12:58+01:00'
draft: false
title: 'Arty A7 Guitar Effect Chain - FIR Implementation'
author: "Daniel Blackbeard"
cover:
  image: "images/fir_resources_count.jpg"
  alt: "Picture of a verilog snippet for a FIR"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
tags: 
- hdl
- log
math: true
---

## Introduction

On this little project of making a guitar chain effect, the equalizer/filter will be a central component. The guitar has a particular spectrum and not all the 20KHz humans can hear are useful to be processed. In filtering out the unnecessary frequencies we get some benefits:
* All the spectral content is useful information, increasing the amount of signal we can process
* We are cutting noise, which usually is very wide spectrum, oftentimes modeled as uniformly distributed

All of this translated into better signal-to-noise ratio (SNR)

Although that's not the only benefit we get from a filter, as we can also use the filter to shape the spectrum of the sound, boosting or attenuating certain bands as we wanted. For this implementation I choose a finite impulse response (FIR) filter. This is quite easy to implement in an FPGA in principle, since it's just a buffer of samples in some kind of FIFO, a memory to store the correcting taps, some multiplicators and a final summation of all the calculated values. It would look something like this: 

{{< figure src="/images/fir_example.png" caption="Example of FIR implementation with 4 taps" alt="Example of FIR implementation with 4 taps" align="center">}}

In the picture there is a 4 tap FIR filter, it will need 4 coefficients previously calculated and stored in some memory, some shift registers and a final sum. As you can see, quite simple circuit that abides to simple mathematics. With its simplicity comes a great flexibility on how to implement, and this is what we will be discussing here.

## Design Tradeoff

On first approximation, the granularity of the frequency response, that is, how good this filter is at manipulating narrow bands, depends on the number of taps. However, in my board I have DSP cells that implement multiplication, not so many mind you, in my case 240. This means that from 48KHz, if I used blindly the DSP cells to implement the filters I would be able to manipulate bands of 200Hz, which is not bad if it weren't because I'm left without hardware for anything else. So, let's make some considerations here:
* The Arty operates on the megahertz, I need to process data at 48KHz
* I can afford some audible delay, on the order of 10ms at most

Hence, what I will do is that I will be using only 8 DSP cells, that will run through the FIFO and the tap memory, and multiply and accumulate (a feature of Artix DSP cells) the results. This has the additional advantage of reducing the complexity of the final adder in case of big tap numbers. This operation will happen at 60MHz, hence if we wanted 256 taps, with 8 DSP cells we have to run through 32 data samples, taking something around 2 microseconds, we have a budget of 20 microseconds from a sampling rate of 48KHz, hence we can be good up to some hundred of FIR coefficients. This technique is called `time-multiplexion`.

## FIR Design

As commented before, we need a way to store many samples for the calculation, so a basic shift register is implemented here

```systemverilog
  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      for (int i = 0; i < NUM_TAPS; i++)
        shift_reg[i] <= '0;
    end else if (sample_edge) begin
      for (int i = 0; i < NUM_TAPS-1; i++)
        shift_reg[i] <= shift_reg[i+1];
      shift_reg[NUM_TAPS-1] <= data_in;
    end
  end
```

Then, a Finite State Machine (FSM) will be used to control the flow of the computation. Every time we get a new sample from the previous steps (at 48KHz) we start the machine to calculate the FIR value. When it's done calculating it will be set on IDLE and await for the next sample. Fairly simple.

In doing those operations, we defined the `PARALLELISM`, which is the number of DSP cells we want to use for the FIR implementation and `NUM_TAPS` which is the number of FIR coefficients. The FSM will run until `NUM_MAC_STEPS` calculations has been performed.

```systemverilog
  localparam int NUM_MAC_STEPS = NUM_TAPS / PARALLELISM;

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
      state <= IDLE;
    else
      state <= next_state;
  end

  always_comb begin
    next_state = state;
    unique case (state)
      IDLE: if (sample_edge)                   next_state = MAC;
      MAC : if (mac_step == NUM_MAC_STEPS-1)   next_state = DONE;
      DONE:                                    next_state = IDLE;
    endcase
  end

  assign fir_idle  = (state == IDLE);
  assign fir_state = {6'b0, state}; // simple encoding for debug
  
    always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      tap_idx  <= '0;
      mac_step <= '0;
    end else begin
      unique case (state)
        IDLE: if (sample_edge) begin
          tap_idx  <= '0;
          mac_step <= '0;
        end

        MAC: begin
          tap_idx  <= tap_idx + PARALLELISM;
          mac_step <= mac_step + 1;
        end

        DONE: begin
          tap_idx  <= tap_idx;   // hold
          mac_step <= mac_step;  // hold
        end
      endcase
    end
  end
```

Here we generate two variables: `tap_idx` which is basically the address of the coefficient in memory, and `mac_step` which will keep track of the parallel operation to conclude the FSM `MAC` state. With `tap_idx` we can select the taps to be used on the current multiplication step from the memory `taps_flat`. For the DSP cells, we will also have to extend the data to 18bit signed, we do this per a sample basis to avoid extra storage of extended data.

 ```systemverilog
  generate
    for (genvar p = 0; p < PARALLELISM; p++) begin : SEL
      // Samples from shift register
      assign sample_s[p] = shift_reg[tap_idx + p];
      assign tap_s[p] = taps_flat[((NUM_TAPS-1) - (tap_idx + p))*TAP_WIDTH +: TAP_WIDTH];

      // Sign extension for DSP-friendly 18-bit operands
      assign sample_ext[p] = {{2{sample_s[p][DATA_WIDTH-1]}}, sample_s[p]};
      assign tap_ext[p]    = {{2{tap_s[p][TAP_WIDTH-1]}},    tap_s[p]};
    end
  endgenerate
 ```

 And then the last step, where the magic happens. On a new sample, the accumulator and partial sum gets reset. Then on `MAC` state the multiplications happen. Here, it's important to communicate Vivado that we will be using DSP cells for multiplication with `(* use_dsp = "yes" *)`, else it will try to use LUTs to synthesize them, which is hardware expensive and fairly inefficient.

 Also, for my specific use case, I unrolled the sum depending on how many DSP cells I want to use. I will mainly stick with 8, but one never knows.

 ```systemverilog
   always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      accum       <= '0;
      done        <= 1'b0;
      data_out    <= '0;
      partial_sum <= '0;
    end else begin
      done <= 1'b0;

      unique case (state)

        IDLE: if (sample_edge) begin
          accum       <= '0;
          partial_sum <= '0;
        end

        MAC: begin
          // Parallel DSP multiplies
          for (int p = 0; p < PARALLELISM; p++) begin
            (* use_dsp = "yes" *)
            mult[p] <= sample_ext[p] * tap_ext[p];
          end

          // Fully unrolled adder tree, truncating from 36->24 bits as [31:8]
          if (PARALLELISM == 1) begin
            partial_sum <= mult[0][31:8];

          end else if (PARALLELISM == 2) begin
            partial_sum <= mult[0][31:8] +
                           mult[1][31:8];

          end else if (PARALLELISM == 4) begin
            logic signed [ACC_WIDTH-1:0] s0 = mult[0][31:8] + mult[1][31:8];
            logic signed [ACC_WIDTH-1:0] s1 = mult[2][31:8] + mult[3][31:8];
            partial_sum <= s0 + s1;

          end else if (PARALLELISM == 8) begin
            logic signed [ACC_WIDTH-1:0] s0 = mult[0][31:8] + mult[1][31:8];
            logic signed [ACC_WIDTH-1:0] s1 = mult[2][31:8] + mult[3][31:8];
            logic signed [ACC_WIDTH-1:0] s2 = mult[4][31:8] + mult[5][31:8];
            logic signed [ACC_WIDTH-1:0] s3 = mult[6][31:8] + mult[7][31:8];
            logic signed [ACC_WIDTH-1:0] t0 = s0 + s1;
            logic signed [ACC_WIDTH-1:0] t1 = s2 + s3;
            partial_sum <= t0 + t1;
          end

          accum <= accum + partial_sum;
        end

        DONE: begin
          data_out <= accum[ACC_WIDTH-2 -: DATA_WIDTH];
          done     <= 1'b1;
        end

      endcase
    end
  end
 ```

 At every `MAC_STEP` we accumulate the `partial_sum` and in state `DONE` we propagate the result to the output to avoid glitches, also informing the next cell that the sample is ready.

 ## Conclussion

 Here we presented a digital implementation of a filter, where we trade off area/power consumption with delay. Since we can afford some delay, this tradeoff has overall benefits on the implementation without any extra complexity on the users of this block. For my use I will stick to 256/512 taps and 8 DSP units in parallel, still giving me the feel of real time, no matter how hard I try to hear any kind of delay. Next article I will be delving a bit on the analog part of this project, as I need a front-end for the ADC.

 Stay tuned