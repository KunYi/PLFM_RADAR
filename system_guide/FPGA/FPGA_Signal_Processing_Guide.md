# FPGA Signal Processing Guide

## Overview

This chapter explains the main radar signal-processing algorithms implemented in the FPGA.
It covers pulse compression, range bin reduction, MTI, Doppler processing, and CFAR detection.

The chapter is easiest to read as a sequence of engineering questions:

- how do we improve range resolution without losing chirp energy?
- how do we reduce data volume before later stages become too expensive?
- how do we suppress clutter?
- how do we estimate target velocity?
- how do we detect targets without relying on one fixed threshold?

## At a glance

- Main processing path: matched filtering, range reduction, MTI, Doppler FFT, and CFAR
- Main focus: what each DSP block is solving and what tradeoffs it introduces
- Best companion page: `FPGA_Receiver_and_ADC_Pipeline_Guide.md`
- Common source of confusion: algorithm names sound familiar, but fixed-point assumptions and stage ordering matter just as much as the math

## Processing chain at a glance

The implemented radar flow is:

ADC -> DDC -> pulse compression -> range decimation -> MTI -> Doppler FFT -> CFAR

This chapter focuses on the middle and later parts of that chain: the stages that turn baseband samples into range-Doppler detections.

## Pulse compression

Pulse compression allows a long transmitted chirp to behave like a narrow range response after matched filtering.
That is useful because a long chirp carries energy, while the compressed response still gives useful range resolution.

The FPGA implements pulse compression mainly with:

- `matched_filter_processing_chain.v`
- `matched_filter_multi_segment.v`

The core idea is:

1. collect time-domain samples
2. compute FFTs
3. multiply the received spectrum by the conjugate of the reference spectrum
4. perform the inverse FFT

This is the classic frequency-domain matched-filter approach.

The multi-segment path exists because long chirps do not always fit cleanly into one FFT-sized buffer.
The design therefore uses an overlap-save style segmentation strategy so a long waveform can still be processed with repeated FFT-sized blocks.

## Range bin decimation

After matched filtering, the design may have far more range samples than later stages can process efficiently in real time.
`range_bin_decimator.v` reduces the 1024-point range output to a smaller set of useful bins.

It supports several strategies:

- simple decimation
- peak selection
- averaging

This is not only a computational shortcut.
It is also a statement about where the design chooses to spend resources:

- preserve enough range information to remain useful
- reduce enough data that Doppler and detection stages stay manageable

## MTI cancellation

`mti_canceller.v` implements a 2-pulse canceller:

- current value minus previous value

Conceptually, MTI suppresses stationary returns and preserves changes across chirps.
That makes it especially helpful when strong static clutter would otherwise dominate the later processing.

The tradeoff is equally important:

- suppressing static clutter can also suppress very slow targets

So MTI is a good example of a useful algorithm that still needs interpretation in context.

## Doppler processing

Doppler processing estimates target velocity by examining pulse-to-pulse variation.
In this design, `doppler_processor.v` uses a staggered-PRF structure:

- 32 chirps per frame
- two 16-chirp sub-frames
- separate handling of long-PRI and short-PRI data

This is important because staggered PRF helps reduce velocity ambiguity while keeping the FFT size relatively small.

The design also applies a window before the FFT to control sidelobe behavior.
That is a small implementation detail with a large practical effect: cleaner spectral behavior often matters as much as nominal FFT size.

## CFAR detection

`cfar_ca.v` implements adaptive detection using CA-CFAR and related variants such as GO-CFAR and SO-CFAR.

The conceptual flow is:

1. compute magnitude for each cell
2. estimate local noise from neighboring training cells
3. scale that estimate with a configurable factor
4. compare the cell under test against the adaptive threshold

CFAR matters because the radar is not operating in one fixed noise condition.
It needs a thresholding method that responds to local variation rather than assuming the same background everywhere.

## Why these blocks are arranged this way

The sequence of blocks is part of the engineering story:

- pulse compression improves range definition
- range reduction keeps later work affordable
- MTI removes clutter before velocity estimation
- Doppler processing extracts slow-time frequency information
- CFAR converts a map of values into actual detections

Each stage prepares the data for the next one.
That is why changing one block often changes the assumptions of the downstream blocks as well.

## Practical concepts

### Modular algorithm boundaries help

Each major block has a distinct role and a relatively clean interface.
That makes it easier to study and debug one algorithm at a time without losing track of the wider signal chain.

### Fixed-point arithmetic shapes the design

These algorithms are not implemented with floating-point math.
Bit width, scaling, saturation, and overflow matter throughout the design.

A mathematically correct algorithm can still fail in hardware if fixed-point choices are poor.

### Resource tradeoffs are part of the algorithm story

Resolution, latency, memory use, FFT size, and configurability are all linked.
In FPGA radar work, algorithm design and hardware architecture are rarely separable.

## Tradeoffs to notice

- better resolution usually costs more compute, more memory, or more latency
- stronger clutter suppression can also weaken slow-target response
- smaller FFTs are cheaper, but they limit Doppler detail
- adaptive detection is more robust than fixed thresholds, but harder to verify and tune

These are system tradeoffs, not just implementation details.

## Known caveats

- This chapter explains the implemented logic, but it is not yet a full performance-analysis document with measured sidelobes, latency, dynamic range, or false-alarm statistics.
- Several modules are written in a way that emphasizes architectural readability, so a future optimized implementation could look different while preserving the same high-level algorithm flow.

## Future questions to explore

- Could some blocks be replaced with IP cores or more optimized FFT structures?
- How should these algorithms change for different waveform plans or PRF strategies?
- Which thresholds and parameters should be host-tunable, and which should remain fixed in RTL?
- How should algorithm quality be validated against real measured scenes rather than only simulation data?

## Future improvement ideas

- Replace behavioral FFT structures with more optimized pipelined FFT implementations where justified.
- Add more explicit documentation of windowing, scaling, and latency through each stage.
- Explore more advanced CFAR variants such as OS-CFAR.
- Extend the processing chain toward richer range-Doppler mapping or tracking modules.
