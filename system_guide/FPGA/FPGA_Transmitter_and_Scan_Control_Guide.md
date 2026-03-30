# FPGA Transmitter and Scan Control Guide

## Overview

The transmitter and scan-control blocks define how the radar sends chirps and steps through its scan pattern.
In the FPGA design, these responsibilities live mainly in `radar_transmitter.v` and `radar_mode_controller.v`.

The core question behind this chapter is:

- how does a high-level request to "transmit and scan" become exact digital timing events that the rest of the hardware can trust?

## At a glance

- Main control modules: `radar_transmitter.v` and `radar_mode_controller.v`
- Main focus: chirp timing, scan-event handling, and transmit-side control ownership
- Best companion page: `FPGA_Interfaces_and_Verification_Guide.md`
- Common source of confusion: bring-up behavior, MCU-driven timing, and FPGA-driven scan logic all meet in this subsystem

## Main jobs of this subsystem

In this design, the FPGA does not directly synthesize the RF waveform at the antenna.
Instead, it coordinates the digital side of transmission:

- generate DAC-side chirp sample flow
- manage chirp, elevation, and azimuth event timing
- control mixer and RF switch signals
- bridge STM32 beamformer SPI signals through the FPGA fabric
- support both MCU-driven and FPGA-driven scan behavior

This makes the transmitter chapter less about "RF generation" and more about deterministic timing control.

## `radar_transmitter.v`

`radar_transmitter.v` is the main transmit-side coordination module.
Its major functions include:

- detecting STM32 timing toggles
- moving those events across clock domains safely
- driving the DAC sample interface
- controlling RX/TX mixer enables and RF switch outputs
- forwarding beamformer-related SPI signals

This module is the bridge between slow control intent and timing-accurate digital action.

## Passive SPI pass-through

The FPGA does not implement the ADAR1000 beamformer protocol itself.
Instead, it passes STM32 SPI signals through to the 1.8 V beamformer-side interface.

That design choice keeps protocol ownership in software while allowing the FPGA to provide voltage-domain bridging and timing coexistence with the rest of the radar path.

This is a useful architectural choice to notice:

- interpret the protocol in one place only
- keep the FPGA focused on timing and transport rather than duplicating control logic unnecessarily

## Timing signal flow

The STM32 provides three important event signals:

- `stm32_new_chirp`
- `stm32_new_elevation`
- `stm32_new_azimuth`

These events originate outside the main FPGA timing domains, so the design converts them into internal pulses safely.

The key challenge is that the transmitter uses both:

- `clk_100m` for system/control logic
- `clk_120m_dac` for DAC-domain timing

Single-cycle pulses do not cross clock domains safely on their own.
The design therefore uses toggle-based CDC:

1. convert the event pulse into a toggle in the source domain
2. synchronize the toggle into the destination domain
3. recover the pulse by edge detection

This is one of the most important implementation ideas in the transmitter path.

## Scan pattern and auto-scan logic

`radar_mode_controller.v` determines whether scan timing comes from the MCU or from internal FPGA sequencing.
It supports three main modes:

- STM32 pass-through mode
- auto-scan mode
- single-chirp trigger mode

The auto-scan structure follows a fixed sequence that includes:

- 32 chirps per elevation
- 31 elevations per azimuth
- 50 azimuth positions per scan
- long-chirp and short-chirp phases separated by listen and guard intervals

This sequencing is important because the radar timing plan and the later Doppler interpretation are linked.

## Runtime tuning

The controller exposes timing-related parameters such as:

- long chirp duration
- long listen duration
- guard interval duration
- short chirp duration
- short listen duration
- chirps per elevation

That runtime configurability matters because it allows experimentation without requiring FPGA resynthesis for every timing adjustment.

## Common problems this stage is solving

- turning cross-domain or asynchronous control events into clean timing pulses
- keeping chirp, elevation, and azimuth counters synchronized
- allowing both MCU-driven operation and FPGA-driven autonomous scan behavior
- preserving deterministic timing while still permitting runtime changes

These are easy problems to underestimate because they do not look like "signal processing", yet they directly affect measurement integrity.

## Why this chapter matters

It is common to focus on radar performance only in the receive and processing chain.
But transmit timing and scan sequencing are just as important:

- bad chirp timing can corrupt Doppler interpretation
- bad scan indexing can make displayed target positions meaningless
- unsafe CDC can create intermittent failures that look like random radar instability

This chapter therefore connects digital control design to actual measurement quality.

## Practical concepts

### Separate timing control from protocol ownership

Passing SPI through the FPGA while keeping protocol interpretation in the MCU is a useful example of dividing responsibility cleanly.

### Toggle CDC is more than a coding pattern

It is the mechanism that turns a fragile one-shot event into something that can survive a clock-domain boundary.

### Configurable timing changes system behavior

Changing chirp and listen timing does not only affect one module.
It can change integration time, scan cadence, and the assumptions later processing makes about the measurement sequence.

## Known caveats

- This chapter explains the sequencing logic clearly, but a future revision could still benefit from a timing diagram that ties state-machine phases to physical RF events.
- Runtime programmability is a strength here, but it increases the importance of documenting who owns each timing parameter: host, MCU, or FPGA.

## Future questions to explore

- Which scan behaviors should remain in the MCU, and which should migrate fully into FPGA automation?
- How should the timing controller evolve if more waveform types are added?
- Would a schedule-table model be clearer than a hand-written state machine?
- What telemetry would best verify that real hardware scan motion matches intended digital timing?

## Future improvement directions

- Add stronger instrumentation for jitter and synchronization debugging.
- Expand auto-scan modes toward adaptive or target-aware scanning behavior.
- Add clearer mapping from scan state to external telemetry/status reporting.
- Evaluate whether some beam-management tasks should remain pure pass-through or become more structured inside the FPGA.
