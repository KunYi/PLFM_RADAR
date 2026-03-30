# System Architecture Guide

## Overview

This document explains how the radar system works end to end, from host GUI to MCU to FPGA.
Its main purpose is to show how the repository splits responsibility across control, sequencing, and high-speed data processing.

The simplest mental model is:

- the host chooses settings and displays results
- the MCU turns those settings into safe hardware actions
- the FPGA performs the timing-critical and data-intensive radar pipeline

## At a glance

- Main system split: host control/display, MCU sequencing/control, FPGA real-time data path
- Main control contract: GUI to MCU over USB CDC
- Main data contract: FPGA to host over FT601
- Common source of confusion: many apparent algorithm failures are really boundary mismatches between subsystems

## Companion references

This document is the end-to-end narrative view.
When a question becomes more table-oriented, the most useful companion references are:

- `device_connection_matrix.csv` for subsystem ownership and external-device mapping
- `system_signal_flow_matrix.csv` for following control and data paths step by step
- `protocol_field_reference.csv` for exact packet and command layout
- `Protocol_Field_Reference.md` for a readable protocol index before opening the CSV

The markdown chapters explain architecture and intent.
The CSV files are better when the task is filtering, tracing, or cross-checking.

## Main roles

- `system_guide/GUI/` explains the host-side control flow, packet formatting, FT601 parsing, and display path.
- `system_guide/MCU/` explains STM32 startup, USB CDC settings intake, RF/control sequencing, and system-status reporting.
- `system_guide/FPGA/` explains ADC capture, DDC, pulse compression, Doppler/CFAR processing, and FT601 transport.

This split is not arbitrary.
It reflects a common radar-system tradeoff:

- low-rate configuration benefits from explicit control and recoverable startup logic
- high-rate radar data benefits from dedicated hardware pipelines and wider transport paths

## End-to-end flow

The main system flow is:

1. `GUI_V6.py` sends a 4-byte start flag to the MCU over USB CDC.
2. The GUI sends a binary settings packet framed by `SET` and `END`.
3. `USBHandler` on the MCU detects the start flag, buffers the packet, and passes it to `RadarSettings::parseFromUSB()`.
4. If parsing succeeds, the MCU reaches its internal `READY_FOR_DATA` state and continues with FPGA/RF startup actions.
5. The FPGA runs the high-speed radar path and streams processed packets to the host through FT601.
6. The host receives FT601 packets, parses data and status frames, and updates the display or engineering dashboard.

This flow is the backbone of the repository.
Most practical bugs or extension ideas touch one of these six steps.

## Why the control/data split matters

The current implementation separates transport by purpose:

- STM32 CDC is used for host-to-MCU startup and settings control.
- FT601 USB3 is used for FPGA-to-host high-speed radar data and status packets.

That separation keeps the embedded control path simple while allowing the radar data path to remain wide and deterministic.
In other words, the architecture treats "set up the radar" and "move radar data" as different engineering problems.

## Main processing motivations

The FPGA path exists because several radar operations are both repetitive and timing-sensitive:

- pulse compression improves range resolution while preserving chirp energy
- range-bin reduction prevents later processing from being overwhelmed
- MTI cancellation suppresses stationary clutter
- staggered-PRF Doppler processing helps with velocity ambiguity
- CFAR detection adapts thresholds to local noise and clutter conditions

These choices are not only algorithmic.
They are architectural choices about where the system should spend bandwidth, latency, memory, and logic resources.

## Shared protocol contracts

The system depends on two main cross-module contracts:

- **GUI ↔ MCU**: start flag + settings packet over USB CDC
- **Host ↔ FPGA**: packetized radar data and status packets over FT601

For ordinary operation, `GUI_V6.py` is the main host application.
`radar_dashboard.py`, `radar_protocol.py`, and `smoke_test.py` are support-side engineering tools rather than replacements for the main GUI workflow.

For the STM32 settings path:

- `GUI_V6.py` is the active GUI baseline for the current MCU packet field order

Keeping these contracts aligned matters because many apparent "algorithm problems" are really interface mismatches.

## Integration checkpoints

- `RadarSettings` in the GUI must match the parser order in the MCU.
- `radar_protocol.py` must match the packet structure emitted by `usb_data_interface.v`.
- `radar_system_top.v` must keep command, status, and timing ownership clear between host, MCU, and FPGA.
- `Protocol_Contracts_and_Known_Gaps.md` is the quickest place to cross-check these dependencies.

## Known caveats

- The GUI/MCU settings packet field order is clear, but the repository still contains an `82` versus `74` byte-count inconsistency in MCU-side comments and guards.
- The MCU startup loop strongly prefers a fully parsed settings packet before progressing, but its blocking condition is still keyed to start-flag receipt rather than strictly to `READY_FOR_DATA`.
- The repository mixes main operational flows with bring-up and engineering support tools, so file role matters when describing the "main system".

These caveats are worth calling out because real systems often drift at subsystem boundaries before the internal documentation catches up.

## Common system-level debugging paths

### When the system does not get past startup

Check these layers in order:

1. Confirm the GUI is actually sending the 4-byte start flag.
2. Confirm the GUI still sends a `SET ... END` settings packet with the expected field order.
3. Confirm the MCU sees and parses the packet rather than only seeing the start flag.
4. Confirm the MCU completes the expected RF/control and FPGA enable sequence after parsing.

At this level, many failures that look like "the radar never starts" are really contract mismatches between the host and MCU.

### When control looks correct but no radar data appears

Check these layers in order:

1. Confirm the MCU has progressed far enough to release or enable the FPGA path.
2. Confirm the FPGA is producing FT601 traffic rather than staying idle.
3. Confirm the host is looking for the correct FT601 packet markers and frame size.
4. Confirm the dashboard or GUI is attached to the expected data path rather than a support-only tool flow.

This is a classic boundary problem between startup sequencing and data-plane visibility.

### When packets arrive but the display or interpretation looks wrong

Check these layers in order:

1. Confirm the packet framing markers still match the host parser expectations.
2. Confirm the command word or status word bit layout still matches the current parser.
3. Confirm the host and FPGA still agree on packet size, word ordering, and reserved fields.
4. Confirm the issue is not caused by stale assumptions in comments, guards, or helper code rather than the active `GUI_V6.py` path.

At this point, the most useful companion files are `Protocol_Field_Reference.md`, `protocol_field_reference.csv`, and `system_signal_flow_matrix.csv`.

## Future directions

Natural follow-up questions include:

- Could the GUI/MCU settings protocol move from marker-based framing to a versioned packet with explicit length and CRC?
- How much radar processing should remain in FPGA versus moving to host-side post-processing?
- What additional metadata should travel with each radar frame: timestamps, beam index, GPS, temperature, self-test state?
- How should the architecture change if the system grows from a bring-up platform into a field-ready product?
- Which diagnostics would most reduce time spent debugging subsystem interactions?

These questions move the discussion from understanding the current implementation to thinking about how the architecture should evolve.
