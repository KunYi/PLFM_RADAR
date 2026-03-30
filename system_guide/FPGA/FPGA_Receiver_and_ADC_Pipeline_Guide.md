# FPGA Receiver and ADC Pipeline Guide

## Overview

The receiver pipeline is the first stage that turns raw analog radar information into digital signals that the FPGA can trust.
This chapter explains how the design captures high-speed ADC samples, moves them into stable clock domains, and converts them into lower-rate complex baseband data for later radar processing.

The key question behind this chapter is:

- how does the design move from a fragile high-speed electrical interface to stable digital samples that later algorithms can use confidently?

## At a glance

- Main capture modules: `ad9484_interface_400m.v`, `adc_clk_mmcm.v`, `ddc_400m.v`
- Main focus: ADC capture, fast-domain cleanup, and conversion into baseband I/Q
- Best companion page: `FPGA_Signal_Processing_Guide.md`
- Common source of confusion: capture-domain or CDC mistakes here can look like downstream DSP problems

## Main stages

This part of the design can be read as three connected jobs:

- capture ADC bits reliably at high speed
- move those bits into a usable FPGA clock domain
- convert the IF signal into lower-rate baseband I/Q samples

## ADC capture and LVDS interface

The radar front end uses an ADC that sends data over LVDS pairs.
`ad9484_interface_400m.v` performs this interface.

The design uses a source-synchronous capture strategy:

- the ADC sends data together with a data clock
- the FPGA samples using that same clock relationship rather than treating the samples as ordinary system-clock data

To make that work, the RTL uses:

- `IBUFDS` to receive differential signals
- `IDDR` to capture on both clock edges
- `BUFIO` to feed the I/O capture path
- `BUFG` for the buffered fabric clock

This combination is what allows the 400 MSPS stream to be captured reliably.

## Clock cleanup and reset handling

The 400 MHz domain is cleaned and managed carefully because high-speed capture is sensitive to timing quality.
`adc_clk_mmcm.v` is used to clean the clock, and reset release is synchronized to the fast domain.

The reset strategy follows a common FPGA pattern:

- asynchronous assertion for immediate reset
- synchronous de-assertion for clean recovery

That approach reduces the risk of metastability when the fast domain comes out of reset.

## Reconstructing the 400 MSPS stream

After DDR capture, the design alternates between rising-edge and falling-edge samples to reconstruct the full 400 MSPS stream.
This is an important conceptual step:

- the LVDS interface first captures edge-separated data
- the logic then rebuilds it into an ordered sample stream

If this stage is wrong, every later processing stage will inherit the error.

## Digital down-conversion

Once the ADC stream is stable, `ddc_400m.v` converts the IF signal into 100 MHz complex baseband outputs.
This is the stage that changes the data from "high-rate sampled IF" into "lower-rate I/Q samples suitable for radar DSP".

The DDC path includes:

- sign handling for ADC samples
- NCO-based mixing
- phase dithering to reduce spurs
- low-pass filtering and decimation
- diagnostics for saturation, overflow, and sample counting

This is one of the most instructive places in the design because it connects digital communication concepts directly to radar processing needs.

## Why the 400 MHz to 100 MHz step matters

The ADC side operates at a rate that is natural for the front end.
The later radar processing stages operate at a lower rate that is more practical for fixed-point arithmetic, buffering, and FPGA timing closure.

The DDC is the bridge between those two worlds:

- keep enough information to preserve the radar return
- reduce the rate enough that later blocks can process it efficiently

That is why the DDC is not a convenience block.
It is the point where the design trades raw bandwidth for manageable computation.

## Common problems this stage is solving

- sampling on the wrong edge or with the wrong clock relationship
- moving fast I/O data into fabric logic unsafely
- carrying excessive sample-rate burden into later DSP blocks
- creating hidden sign, overflow, or saturation errors that only appear downstream

Many later "algorithm bugs" actually begin here.

## Practical concepts

### Source-synchronous design

This stage is a concrete example of source-synchronous capture:

- the transmitter of the data also provides the timing reference
- the receiver must honor that timing relationship

That idea appears often in high-speed converter interfaces.

### Clock domain crossing is everywhere

The receiver pipeline interacts with several domains:

- 400 MHz ADC capture domain
- 100 MHz processing domain
- 120 MHz DAC/control-related interactions indirectly through the wider system

Understanding why each domain exists is essential for interpreting the RTL correctly.

### Fixed-point robustness matters early

The receiver path is also where sign extension, scaling, overflow, and saturation start to matter.
If fixed-point handling is wrong here, later matched filtering and Doppler stages will produce believable-looking but incorrect outputs.

## Known caveats

- This chapter describes the implemented structure, not measured analog performance such as ENOB, clock-jitter impact, or calibration quality.
- The DDC flow is clear at the architectural level, but a future revision could still explain more of the tuning constants and gain-scaling assumptions.

## Future questions to explore

- Should more calibration or alignment logic live closer to the ADC interface?
- How would this design change for a wider ADC, higher sample rate, or JESD-style interface?
- What observability hooks would most help when captured waveforms look wrong?
- Which parts of the current pipeline are chosen mainly for clarity, and which are chosen mainly for timing closure?

## Future improvement ideas

- Move more DDC filtering into dedicated FPGA IP if lower latency or higher efficiency becomes important.
- Add auto-calibration for NCO phase and gain balance.
- Improve diagnostics to correlate events across clock domains.
- Add support for alternate ADC sample depths or data formats.
