---
layout: post
title: PDM Microphones and Sigma-Delta A/D Conversion
date:  2020-08-29 00:00:00 -1000
categories:
---

* TOC
{:toc}

# Introduction

Over the past months, I've been reading on and off about MEM microphones, 
[pulse density modulation](https://en.wikipedia.org/wiki/Pulse-density_modulation) (PDM), conversion
from PDM to [pulse code modulation](https://en.wikipedia.org/wiki/Pulse-code_modulation) (PCM), 
and various audio processing techniques and interfaces.

In the coming series of blog post, I'll be writing down some of the things that I learned. Rather than focussing on the 
math behind it, I'm looking at this from an intuitive angle: I'll forget the math anyway, and there are plenty of articles 
online where you can look up the details if it's necessary.

*Just to avoid misunderstandings: I didn't know anything about this until a few months ago, so I'm about the opposite of 
an expert on any of this. If you're serious about learning about PDM signal processing in depth, this blog post is not for you.
Along the way, I will include a list of references that may point you in the right direction.*

# PDM MEMS Microphones

[MEMS microphones ](https://en.wikipedia.org/wiki/Microphone#MEMS) can be found in all moderns
mobile phones. They are very small in size, are low cost, have low power consumption, offer decent quality, and
can be mounted to a PCB with a standard surface mounted assembly process.

Some MEMS microphones transmit the audio data in PCM format over an I2S interface, but most use 1-bit pulse density modulation.

On Digikey, the cheapest PDM microphones go for around $1 a piece. Or you can buy [a breakout board on Adafruit](https://www.adafruit.com/product/3492) 
for $5. It has an [ST MP34DT01-M](https://cdn-learn.adafruit.com/assets/assets/000/049/977/original/MP34DT01-M.pdf) microphone
with a 61dB SNR.

![Adafruit PDM MEMS Microphone](/assets/pdm/adafruit-pdm-mems-microphone.jpg)


# Pulse Code Modulation and Quantization Noise

Humans are able to hear audio in a range from 20 to 20000Hz, though the range goes down with age.
Digital signal theory dictates that audio needs to be sampled at a frequency of at least twice the bandwidth 
to accurately reproduce the original signal. That's why most audio is recorded at 44.1 or 48kHz sample rate.

The number of bits at which the audio signal is sampled determines how closely the sampled signal matches
the real signal. The delta between sampled and real signal is the quantization noise. For a certain number
of bits N, the signal to noise ratio (SNR) of a full amplitude sine wave follows the formula: 

SNR = 1.761 + 6.02 * Q db, where Q is the number of quantization bits.

![Quantization Level 8 Noise Level](/assets/pdm/quantization_noise_8.svg)


![Quantization Level 8 Noise Level](/assets/pdm/quantization_noise_8.svg)


It is, however, possible to trade off the number of bits for clock speed. In the case of PDM microphones, 
the signal is usually sampled at 64 times the traditional sample rate, or 48kHz * 64 = 3.072 MHz.

# From Analog Signal to PDM with a Sigma-Delta Convertor

PDM microphones use so-called sigma-delta A/D convertors that continuously toggle between -1 and 1 to approximate
the original signal, while feeding back the cumulative errors between the 1-bit output and the real input value.

The ratio of the number of -1's and 1's, the pulse density, represents the original analog value.

Here's an example of a PCM sine wave that's converted to PDM with a first order, 16x oversampling sigma-delta convertor:

![Sigma-Delta Sine Wave](/assets/pdm/sinewave_to_pdm.svg)

At the peak of the sine wave, the green output consists of primarily ones, while at the trough, the output
is primarily minus ones.

PDM microphone encodes -1 as a logic 0 and 1 as a logic 1.

Jerry Ellsworth has [great video](https://www.youtube.com/watch?v=DTCtx9eNHXE) that shows how to build a 
sigma-delta A/D convertor yourself with just 1 FF and a few capacitors and resistors.

Sigma-delta convertors are complex beasts. The [Wikipedia article](https://en.wikipedia.org/wiki/Delta-sigma_modulation)
goes into a decent amount of intuitive detail without going totally overboard on theory. 

When you reduce the number of output bits of an A/D convertor, increasing the oversampling rate doesn't
automatically reduce quantization noise. In fact, the noise is inevitable! However, if you can
push the frequency component of that noise to a range that's outside of the freqency range of the
original input signal, then it's easy later on to recover the original signal by using a
low pass filter.

It turns out that this is exactly what happens in a sigma-delta convertor! 

We can see this in the power spectral density of the PDM signal above:

![Sigma-Delta Sine Wave PSD](/assets/pdm/sinewave_pdm_psd.svg)

The bandwidth of the signal that we're interested in lays on the left size of the green dotted line. That's
where we see the main spike: this is our sine wave. Everything else is noise.

We can also see that the bandwidth of our signal is 1/16th of the total bandwith. With a perfect
low pass filter, we could remove all the noise to the right of the green line, after which we'd end
up with a SNR of 35dB. 

(Note that the input signal has an amplitude of 0.5. Increasing the amplitude
would increase the SNR of this case a little bit, but soon you run into limitations of sigma-delta
convertors that prevent you from using the full input range.)

# Higher Order Sigma Delta Convertors

A SNR of 35dB is far from stellar (it's terrible), but keep in mind that we're using a very simple 
*first order* sigma-delta convertor here. Remember how I wrote earlier that there is an error feedback
mechanism that keeps track of the difference in error between output and input? Well, it turns out that you 
can have multiple nested error feedback mechanism. If you have 2 feedback loops, you have a *second order*
convertor etc.

Let's see what happens when we use higher order convertors on our sine wave:

![Sigma-Delta Sine Waves with higher order convertors](/assets/pdm/sinewave_to_pdm_different_orders.svg)

Do you notice the difference between the different orders? Neither do I! There's no obvious
way to tell apart the PDM output of a first and a 4th order sigma-delta convertor.

But let's have a look if there were any changes in the frequency domain:

![Sigma-Delta PSD with higher order convertors](/assets/pdm/sinewave_pdm_psd_different_orders.svg)


Nice! Despite the constant oversampling rate of 16, the SNR increased from 35dB to 50.2dB!

Higher order sigma delta convertors are significantly more complex and at some point they become
difficult to keep stable. Most contemporary PDM microphones use a 4th order sigma-delta convertors.

For a given oversampling rate, there's a point where increasing the order of the convertor stops
increasing the SNR. For this example, that point is right around here: the maximum SNR
was 51dB.

# Increasing the Oversampling Rate

50dB is better than 35dB, but it's still far away from the 96dB (and up) that one expects from high-end
audio.

Luckily, we have a second parameter in our pocket to play with: oversampling rate.

Until now, I've used an oversampling rate of 16x because it makes it easier to see the frequency
range of the input signal in the frequency plot. But let's see what happens when we increase
oversampling rate of the 4th order convertor from 16 to 32 to 64:

![Sigma-Delta Sine Waves with higher oversampling rate](/assets/pdm/sinewave_to_pdm_different_osr.svg)

This time, the effect on the PDM is immediately obvious in the time domain.

And here's what happens in the frequency domain:

![Sigma-Delta PSD with higher oversampling rate](/assets/pdm/sinewave_pdm_psd_different_osr.svg)

WOW!!! Now we're talking!

An oversampling rate of 64 and a 4th order sigma-delta convertor seems to be do the trick.

And that's exactly what today's PDM microphones support.

Notice how the green vertical line shifts to the right as the oversampling rate increases.
This is, of course, expected: we are doubling the sampling rate with each step while bandwidth of
our input signal stays the same. 


# SNR Slope Depends on Sigma Delta Order

We've seen how higher order sigma-delta convertors have a much lower noise floor in the frequency
range of the sampled signal, but we overlooked that the noise increases much faster for higher
order convertors once the frequency is to the right of the green dotted line.

Here's a graph that makes this more obvious:

![Noise slope for different orders](/assets/pdm/noise_slope_different_orders.svg)

For clarity, I only show the normalized frequency range up to 0.1 instead of 0.5.

As we saw before, on the left of the green line, the noise SNR is lower for higher order convertors, but
on the right side, the SNR for higher order convertors is soon higher than for the
lower order convertors!

In other words: the slope of the SNR curve is steeper for higher order convertors.

This will be important later when we need to design a low pass filter to remove all higher
frequencies when converting back from PDM to PCM format: the low pass filter needs to be 
steeper for higher order sigma-delta convertors.

# PDM and Sigma-Delta Convertors Summary

The most important things to remember about PDM microphones and their sigma-delta convertors are that: 

* they use oversampling to trade off bits for clock speed
* they push the quantization noise into the frequency range above the bandwidth of the sampled
  signal of interest.
* the rate at which the noise increases with frequency depends on the order of the sigma-delta
  convertor.

# The Road from PDM to PCM

We now know the characteristics of PDM-encoded audio signal. 

What do we need to do to convert it to a traditional PCM code stream of samples?

There are 2 basic components:

* send the PDM signal through a low pass filter 

    This increases the number of bits per sample back to a PCM value. It also
    removes all the higher order noise.

* decimate the samples back to a lower sample rate

# PDM Decimation without Filtering

If, just for fun, we made the worlds dumbest PDM sample rate downconvertor by just throwing out 
1-bit samples, we'd get something like this:

![Decimation without Filtering](/assets/pdm/decimation_without_filtering.svg)

Starting with a beautiful 102 dB, our SNR drops down to 10 dB after removing every other sample, 
and down to 4.6 dB after doing that again. Forget about a 48 kHz sample rate, even after going
down from 3.072 MHz down to 768kHz, our signal has already entry disappeared.

After the first step, the noise that's present in the original frequence range form 0.25 to 0.5 was folded onto
the range from 0 to 0.25, drowning out the original signal.

# Best Case Low Pass Filtering

It should be abundantly clear now that we need that low pass filter before we can decimate to
a lower sample rate.

Let's use the following parameters:

* Original sample rate: 3.072 MHz
* Oversample rate factor: 64
* Desired sample rate: 48 kHz
* Original signal bandwidth: 20 kHz
* Desired pass band: 0 dB
* Desired stop band: 96 dB

I chose 96 dB because that's the theoretical maximum SNR for 16-bit audio. Most PDM microphones only
have an SNR in the low sixties, so this is overly aggressive, but let's just see what we can do.

The transition from pass band to stop band will start at 20 kHz. And if we look at the graph for
a 64x oversampling, 4th order sigma-delta convertor, we see that the noise goes above 96 dB 

Since the noise starts going up immediately above 24 kHz (=48/2), we have 4 kHz to construct a filter
that goes from a pass band to the stop band. 

# References

## Sigma-Delta AD Convertors

* [How delta-sigma ADCs work, Part 1](https://www.ti.com/lit/an/slyt423a/slyt423a.pdf) 
   and [Part 2](https://www.ti.com/lit/an/slyt438/slyt438.pdf)

    Texas Instruments article.  Part 1 is a pretty gentle introduction about the sigma-delta basics.
    Part 2 talks about filtering to go from PDM to PCM, but it's very light on details.

## General DSP

## Filter Design

* [Tom Roelandts - How to Create a Simple Low-Pass Filter](https://tomroelandts.com/articles/how-to-create-a-simple-low-pass-filter)

    Simple explanation, Numpy example code.

## Decimation

* [Optimum FIR Digital Filter Implementations for Decimation, Interpolation, and Narrow-Band Filtering](https://web.ece.ucsb.edu/Faculty/Rabiner/ece259/Reprints/087_optimum%20fir%20digital%20filters.pdf)

    Paper that discusses how to size cascaded filter to optimized for FIR filter complexity.

## Filter Tools

* [FIIIR1](https://fiiir.com)

* [LeventOztruk.com](https://leventozturk.com/engineering/filter_design/)


