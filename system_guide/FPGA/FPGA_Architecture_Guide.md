# FPGA Architecture Guide

## Overview

This document provides a structural guide to the FPGA subsystem under `9_Firmware/9_2_FPGA`.
It is the broadest FPGA chapter in the set: its job is to show how transmitter logic, receiver logic, interfaces, and support assets fit together into one design.

The central question is:

- how is the FPGA partitioned so that timing control, signal processing, and host communication can coexist cleanly?

## At a glance

- Main integration view: `radar_system_top.v`
- Main focus: how the FPGA tree is partitioned into timing, processing, transport, and support layers
- Best companion page: `FPGA_Architecture_Overview.md`
- Common source of confusion: support wrappers and verification assets live alongside production RTL, so file role matters

## Scope

This chapter covers:

- top-level FPGA integration
- transmitter and scan control
- receiver signal path
- core processing blocks
- host/data interfaces
- verification and build support

It does not attempt to replace the MCU or GUI guide chapters, and it does not serve as the authoritative hardware schematic.

## Main files and support assets

Primary RTL modules include:

- `radar_system_top.v`
- `radar_receiver_final.v`
- `radar_transmitter.v`
- `radar_mode_controller.v`
- `matched_filter_processing_chain.v`
- `doppler_processor.v`
- `cfar_ca.v`
- `range_bin_decimator.v`
- `mti_canceller.v`
- `usb_data_interface.v`
- `ad9484_interface_400m.v`
- `ddc_400m.v`

Support assets include:

- `tb/`
- `scripts/`
- `run_regression.sh`
- waveform and twiddle `.mem` files

From an architecture-reading viewpoint:

- `radar_system_top.v` is the main integrated RTL view
- the main functional RTL lives in transmitter, receiver, mode-control, and USB-interface modules
- board-specific top wrappers and build/test assets are support-side infrastructure

## Top-level architecture

`radar_system_top.v` is the best single file for understanding the FPGA as a whole.
It integrates:

- transmitter timing and DAC-facing logic
- receiver capture and DSP path
- mode-control logic
- FT601 USB transport
- runtime command and status wiring
- clock buffering and reset synchronization

The top level is not just a container.
It is the place where subsystem boundaries become explicit and where cross-domain assumptions have to be managed carefully.

## Architectural partitioning

The design is easiest to understand as five domains:

- transmitter control and waveform timing
- receiver capture and baseband conversion
- core radar processing
- host/data interfaces
- verification and build support

This partitioning is one of the most useful lessons in the repository.
Real radar FPGA design is not one large algorithm block.
It is a coordinated system of timing, transport, signal processing, and hardware control.

## Transmitter and scan subsystem

The transmit-side structure centers on:

- `radar_transmitter.v`
- `radar_mode_controller.v`

Key behaviors include:

- DAC-side chirp coordination
- STM32 timing-toggle handling
- cross-domain transfer of chirp-related events
- RF switch and mixer control
- support for both MCU-driven and FPGA-driven scan timing

This part of the design is where control intent becomes deterministic measurement timing.

## Receiver signal chain

The receive-side structure centers on `radar_receiver_final.v`, which integrates:

- ADC LVDS reception via `ad9484_interface_400m.v`
- digital down-conversion via `ddc_400m.v`
- matched filtering and range-profile generation
- range bin decimation
- MTI cancellation
- Doppler processing
- CFAR detection

This is the data plane of the radar.
It is where raw sampled information becomes structured outputs that the host can display and interpret.

## Core signal-processing structure

The main processing blocks represent a practical range-to-detection pipeline:

- pulse compression using matched filtering
- range reduction to keep later work manageable
- MTI to suppress stationary clutter
- staggered-PRF Doppler processing for velocity information
- CFAR for adaptive threshold-based detection

The important architectural point is not only what each algorithm does.
It is also why the pipeline is ordered this way:

- each stage reduces uncertainty or volume for the next stage
- each stage carries assumptions that downstream logic depends on

## Data and control interfaces

The FPGA also contains a clear separation between:

- data-plane movement of radar outputs
- control-plane movement of runtime settings and triggers

`usb_data_interface.v` is the main host-facing transport block.
It handles:

- FT601 packet framing
- host command intake
- status readback
- packetized output of radar data

This interface layer is where the FPGA stops being "just DSP" and becomes part of a larger system contract.

## Clock domains and synchronization

Several domains exist for good reasons:

- `clk_100m` for general system logic
- `clk_120m_dac` for DAC/transmit timing
- `ft601_clk` for USB transport
- `clk_400m` for ADC capture and DDC

That means the architecture depends heavily on:

- CDC for one-shot events
- synchronized resets
- clean ownership of where signals are generated and consumed

A large share of integration risk in the FPGA lives at these boundaries rather than inside the arithmetic itself.

## Verification and build support

The repository includes:

- `tb/` for simulation assets
- `scripts/` for build/programming flow support
- `run_regression.sh` for automated regression

These support files matter because they show how the design is meant to be exercised, checked, and evolved rather than merely read.

## Why this chapter matters

- It provides the highest-level FPGA map in the system guide set.
- It helps identify where a new feature or bug likely belongs: timing, transport, processing, or integration.
- It shows how radar FPGA work is as much about subsystem partitioning as about algorithm correctness.

## Known caveats

- This chapter is intentionally structural, so it does not provide a full latency budget, resource budget, or verification-coverage analysis.
- Some module descriptions prioritize architectural clarity over low-level implementation detail; the companion FPGA chapters should be used for deeper study of each area.

## Future directions

- Add a more explicit signal-flow or block-connectivity diagram for `radar_system_top.v`.
- Document latency and buffering assumptions across the main processing stages.
- Expand verification discussion toward subsystem coverage and hardware-correlation strategy.
- Connect architectural descriptions more directly to measured performance and observed system behavior.
