# FPGA Architecture Overview

## Overview

This overview explains the high-level structure of the FPGA implementation in `9_Firmware/9_2_FPGA`, connects the main functional blocks, and points to the more detailed FPGA guide chapters.

The FPGA is the part of the system that turns raw sampled radar data into processed outputs while also handling timing-sensitive control functions such as chirp generation, scan coordination, and host-side packet transport.

## At a glance

- Main integration file: `radar_system_top.v`
- Main responsibilities: ADC capture, radar DSP pipeline, packetization, timing-side runtime control
- Main interfaces: ADC input, DAC/transmit control, FT601 host transport, MCU-facing trigger/control signals
- Common source of confusion: the design mixes production RTL, dev wrappers, and verification assets in the same tree

## Companion references

This document is the FPGA-side narrative overview.
For lookup-oriented work, the most useful companion references are:

- `FPGA_Pin_Definitions.csv` for board-level FPGA signal inventory and interface grouping
- `../system_signal_flow_matrix.csv` for tracing how samples, commands, and packets move through the system
- `../protocol_field_reference.csv` for FT601 packet and command-word layout
- `../device_connection_matrix.csv` for connecting FPGA-facing interfaces to larger system ownership and device roles

The markdown chapter is best for understanding block structure and processing intent.
The CSV tables are better for filtering by signal, interface, packet field, or subsystem relationship.

## What this subsystem does

The FPGA subsystem is responsible for:

- capturing high-speed ADC data from the radar front end
- converting raw samples into complex baseband I/Q signals
- performing pulse compression and range processing
- extracting range bins for later Doppler and detection stages
- running MTI, Doppler FFT, and CFAR logic
- streaming results over USB and responding to host commands

This is where abstract radar ideas become timing-constrained hardware structures.

## Main organization

The top-level FPGA design can be understood as five cooperating domains:

- transmitter and scan control
- receiver and ADC pipeline
- signal-processing algorithms
- data and control interfaces
- verification and build support

Each of those domains has a dedicated guide chapter.

## Main RTL versus support files

From an architecture-reading viewpoint, the FPGA tree also has a clear main/support split:

- `radar_system_top.v` is the main integrated RTL view.
- `radar_transmitter.v`, `radar_receiver_final.v`, `radar_mode_controller.v`, and `usb_data_interface.v` are the core functional modules.
- `radar_system_top_te0712_dev.v`, `radar_system_top_te0713_dev.v`, and `radar_system_top_te0713_umft601x_dev.v` are development-target wrappers.
- `tb/`, `scripts/`, and `run_regression.sh` are support infrastructure for simulation, build, and regression.
- `.mem` waveform files and FFT tables are data assets consumed by the RTL.

## FPGA subsystem summary

| Subsystem | Main files | Main interfaces | Main data or control role | Main caution point |
|---|---|---|---|---|
| Top-level integration | `radar_system_top.v` | clocks, resets, internal routing, host command/status links | ties transmitter, receiver, processing, and FT601 transport together | design intent, defaults, and host-configurable behavior meet here, so drift is easiest to introduce at this layer |
| Transmitter and scan control | `radar_transmitter.v`, `radar_mode_controller.v` | DAC path, mixer enables, STM32 trigger/control lines, ADAR bridge | chirp generation, scan sequencing, runtime control ownership | timing behavior and bring-up defaults can be easy to confuse with long-term architectural intent |
| Receiver and ADC pipeline | `ad9484_interface_400m.v`, `ddc_400m.v`, `radar_receiver_final.v` | ADC LVDS interface, internal sample streams | safe sample capture, baseband conversion, handoff to later DSP | capture or CDC issues here can look like algorithm failures later in the chain |
| Signal processing | `matched_filter_processing_chain.v`, `range_bin_decimator.v`, `mti_canceller.v`, `doppler_processor.v`, `cfar_ca.v` | internal FPGA processing streams | pulse compression, range reduction, clutter suppression, Doppler, detection | fixed-point behavior and stage-by-stage assumptions matter more than the block names alone suggest |
| Host and verification support | `usb_data_interface.v`, dev top wrappers, `tb/`, `scripts/` | FT601, host command path, simulation/build tooling | packet transport, command decode, status readback, regression support | transport rules, parser expectations, and historical fix notes are spread across several layers |

## Main documents

- `FPGA_Transmitter_and_Scan_Control_Guide.md`
- `FPGA_Receiver_and_ADC_Pipeline_Guide.md`
- `FPGA_Signal_Processing_Guide.md`
- `FPGA_Interfaces_and_Verification_Guide.md`

## Suggested reading order

1. `FPGA_Transmitter_and_Scan_Control_Guide.md` — understand how timing, triggers, and scan sequencing are generated.
2. `FPGA_Receiver_and_ADC_Pipeline_Guide.md` — see how raw ADC data becomes stable baseband samples.
3. `FPGA_Signal_Processing_Guide.md` — follow how those samples become range-Doppler and detection information.
4. `FPGA_Interfaces_and_Verification_Guide.md` — see how the FPGA communicates with the host and how the design is validated.

## What these chapters make easier to understand

These chapters make it easier to:

- describe the role of each major FPGA module
- understand why multiple clock domains are unavoidable in this design
- see how matched filtering, Doppler FFT, and CFAR are realized in RTL
- distinguish the data plane from the control plane
- identify natural places for future optimization or instrumentation

They also help separate two ideas that are easy to mix together in hardware projects:

- bring-up defaults that make early validation practical
- runtime contracts that other modules must continue to obey

That distinction matters most around host transport, command decode, status readback, and reset-time behavior.

## Future questions

- Which algorithms truly need to remain in FPGA, and which could move to software?
- How should the design change if waveform complexity, range bins, or Doppler bins increase?
- Which parts of the current RTL emphasize clarity, and which emphasize timing closure?
- What instrumentation would most help debug a weak target, false alarm, or timing failure?
