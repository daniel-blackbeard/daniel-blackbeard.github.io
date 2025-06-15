---
date: '2025-06-15T15:00:43+02:00'
title: 'Sine Synthesis with FPGA'
author: "Daniel Blackbeard"
cover:
  image: "images/sine-scope.jpg"
  alt: "Picture of a sine waveform"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
tags: 
- hdl
- i2c
- log
math: true
---

## Introduction
Last time we did together a circuit using verilog to communicate to a I2S device. But we never delved into how to make a waveform for it. 

Now, don't get me wrong: if you did the blinky led example while starting your first projects, this can be easily achievable for you:
1. A square pattern is just zeros and ones
2. A saw pattern is a counter from zero to max
3. A triangle wave is a counter that goes back after hitting max or min values
4. A sine wave is just some [CORDIC](https://en.wikipedia.org/wiki/CORDIC)...

Now, we will not be using CORDIC and will not be doing simple counters. I want to show you a couple of tricks to generate a clean sine waveform that you can later compose to create more fancy sounds.

## Single LUT<cite>[^1]</cite>

Instead of CORDIC, we will encode the values of the sine function within a lookup table (LUT) and then use the values and transform them as needed. This will allow us to create a shape that resembles the trigonometric function, just without frequency content. Instead, for the frequency we will use a counter and call it _phase accumulator_ because it sounds better. This counter will keep track of the phase per time step (clock or strobe), hence the frequency will be directly a ratio between the phase advancement and this time step.

### Sine ROM
Let's take a look at the ROM:
```systemverilog
data[0]  = 8'b00000000;
data[1]  = 8'b00000110;
data[2]  = 8'b00001100;
data[3]  = 8'b00010010;
data[4]  = 8'b00011001;
data[5]  = 8'b00011111;
data[6]  = 8'b00100101;
data[7]  = 8'b00101011;
data[8]  = 8'b00110001;
data[9]  = 8'b00111000;
data[10] = 8'b00111110;
data[11] = 8'b01000100;
data[12] = 8'b01001010;
data[13] = 8'b01010000;
data[14] = 8'b01010110;
data[15] = 8'b01011100;
data[16] = 8'b01100001;
data[17] = 8'b01100111;
data[18] = 8'b01101101;
data[19] = 8'b01110011;
data[20] = 8'b01111000;
data[21] = 8'b01111110;
data[22] = 8'b10000011;
data[23] = 8'b10001000;
data[24] = 8'b10001110;
data[25] = 8'b10010011;
data[26] = 8'b10011000;
data[27] = 8'b10011101;
data[28] = 8'b10100010;
data[29] = 8'b10100111;
data[30] = 8'b10101011;
data[31] = 8'b10110000;
data[32] = 8'b10110101;
data[33] = 8'b10111001;
data[34] = 8'b10111101;
data[35] = 8'b11000001;
data[36] = 8'b11000101;
data[37] = 8'b11001001;
data[38] = 8'b11001101;
data[39] = 8'b11010001;
data[40] = 8'b11010100;
data[41] = 8'b11011000;
data[42] = 8'b11011011;
data[43] = 8'b11011110;
data[44] = 8'b11100001;
data[45] = 8'b11100100;
data[46] = 8'b11100111;
data[47] = 8'b11101010;
data[48] = 8'b11101100;
data[49] = 8'b11101110;
data[50] = 8'b11110001;
data[51] = 8'b11110011;
data[52] = 8'b11110100;
data[53] = 8'b11110110;
data[54] = 8'b11111000;
data[55] = 8'b11111001;
data[56] = 8'b11111011;
data[57] = 8'b11111100;
data[58] = 8'b11111101;
data[59] = 8'b11111110;
data[60] = 8'b11111110;
data[61] = 8'b11111111;
data[62] = 8'b11111111;
data[63] = 8'b11111111;
```

In here, I encoded the values of the sine function from zero to pi/2. This is because to save memory and resource, we would rather leverage on the sine symmetries and make a little calculation depending on the phase.

The resolution of this function is 8bit and there are 64 values. This means that using the symmetries, we add a sign, making a 9bit value for a total of 256 samples per period. If we use as sampling rate the same we used in the former blog about I2S, so 48KHz, that means that a full period will be of 48KHz/256 or 187.5Hz. This basically defines the frequency resolution, and as such, I don't like this number. So I will boost artificially the number of samples in the LUT by using linear interpolation. This interpolation at the same time will boost by additional bits the resolution of the signal with some penalty on the Signal to Noise plus Distortion Ratio ([SNDR](https://en.wikipedia.org/wiki/SNDR)) with respect to use a bigger and more accurate LUT, but with huge savings in power, area and complexity.

###Synthesizer
We use this sine ROM with a circuit that fetch the values by encoding the phase into a memory address and transforms it into a complete waveform. So let's put some framework to begin with:

```systemverilog
// It produces samples at ~48KHz, that given the constraints of i2s_tx module
// means we can use a frequency of 60MHz and a divisor of 1248
// the frequency error will be around 0.16%
parameter DIV = 12'b10011011111;

reg  [11:0] strobe;
reg  [17:0] phase_accum;      // each bit increases freq by 0.732Hz, we will use sine symmetries
wire  [7:0] samp;
reg   [8:0] data, data_last;  // this is the actual waveform, on this we will make interpolation
reg   [5:0] addr;

reg [23:0] dtemp, dtemp_interp;

initial begin
  addr = 6'b0;
  strobe = 12'b0;
  phase_accum = 18'b0;
  data = 9'b0;
  data_last = 9'b0;
  d_out = 24'b0;
end
```

We have as inputs a clock, that for compatibility with the I2S circuit, will be 60MHz, we will need a volume control, the phase step per time step (proxy of the frequency) and put out the current value of the numerical oscillator.

Let's see how to make this quarter of wave into a complete wave:

```systemverilog
always @(phase_accum, samp) begin
  case(phase_accum[17:16])
    2'b00: addr <= phase_accum[15:10];
    2'b01: addr <= 6'b111111  - phase_accum[15:10];
    2'b10: addr <= phase_accum[15:10];
    2'b11: addr <= 6'b111111  - phase_accum[15:10];
  endcase
  
  case(phase_accum[17:16])
    2'b00: data <= {1'b0,  samp};
    2'b01: data <= {1'b0,  samp};
    2'b10: data <= {1'b1, ~samp};
    2'b11: data <= {1'b1, ~samp};
  endcase
end
```

Not sure if intuitive enough, but basically two things happen in here:
1. On even quadrants the address to the room goes to the other side to have continuity after pi/2 and 3pi/2.
2. On third and fourth quadrant, the sine changes sign, hence we add this sign and transform into 2-complement signed value

Then, we update via synchronous logic the strobe (to convert from 60MHz to 48KHz), the phase accumulator and keep track of the last value of the ROM. Consider that we will move only forward in phase here.

```systemverilog
always @(posedge clk) begin
  if(strobe==DIV) begin
    strobe <= 12'b0;
    phase_accum <= phase_accum + f_mult;
    data_last <= ((phase_accum>>10) != (phase_accum + f_mult)>>10) ? data : data_last;
  end
  else begin
    strobe <= strobe + 12'b1;
  end

  // This will attempt to infer DSP slices
  // Linear interpolation here
  dtemp <= {{15{data[8]}}, data} - {{15{data_last[8]}}, data_last};
  dtemp_interp <= dtemp*phase_accum[9:0] + ({{15{data_last[8]}}, data_last}<<10);
  d_out <= dtemp_interp * vol;
end
```

You can see, `data_last` changes only when the data from the calculation made on the ROM values changes itself. This value will be used for linear interpolation.
### Linear interpolation
Let's put it blunt in here because either you know what it is, or you don't: between two points on the sine table, we will draw a line and divide it by a power of two number of points. This is not necessarily a good approximation, but the quantization error made is way smaller with respect to not doing anything.

{{< figure src="/images/linear_interpolation.png" caption="The white dots are the values in the LUT, the curve is the theoretical values of the function in continuous time, the black dots are inferred from a line between the white dots." alt="Linear interpolation example" align="center">}}

The values at the middle points between two values of the LUT follow a simple linear relationship. Basically divide the $\Delta Y$ by the $\Delta \phi = X_{n+1} - X_{n}$ to obtain the slope, then multiply by the current slice in the phase and add the last value:

$$ Y_n = \frac{Y_{n+1} - Y_{n}}{X_{n+1} - X_{n}}m $$

here, $m$ is the current slice between 2 points in the LUT. We will choose a power of 2 for the maximum value of $m$ for simplicity, but it can be any value. The trick here, is that the maximum value of $m$ is equivalent to the number of slices between $X_{n+1} - X_{n}$, and if this value is a power of 2, then the division becomes a left shift. Actually, we don't even have to do this shift, as the data is 9bit (8 + 1 of sign) and we extended by additional 15bit up to 24bit. Hence, we can just make a rest and call it a day. Then, what we do instead is leverage on the 18bit phase accumulator and make the next partition:
1. bit `[17:16]` tracks the sine quadrant
2. bit `[15:10]` tracks the address in the sine ROM
3. bit `[9:0]` are used for interpolation

So we can assume 18bit data with huge chunks of quantization, shift back 10bit, subtract, then multiply by the interpolation partition that tracks the current slice between to ROM points. The data is already shifted so few less circuits for us.

With 10 bits of interpolation, we are converting the 8bit ROM data into 18bit, add one for the sign, and then we can multiply this value for the volume setting, getting a total of 24bit data to the I2S circuit. 

Neat, isn't?

One more notes about the interpolation: by using the multiplication with this recipe, we are leveraging on the DSP slices in the Artix 7 FPGA, there are something like 120 or so of those slices, so they are kind of precious and is better to save them for critical calculations. In this example we just used two of those for a synthesizer.

## Conclusion
This is just a toy example of how to make a LUT based synthesizer. More improvements can be made and perhaps I will. You can send this 24bit data and chain it with the I2C circuit we did last time, put headphones and listen the result.

As usual, if you want to get the sources of these blocks, as I write this blog I will place them at [my github repo](https://github.com/daniel-blackbeard/verilog_modules)

[^1]: Disclaimer: don't abbreviate the word "Single" please.