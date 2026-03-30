# GUI and Host Software Guide

## Overview

This guide explains how the host-side software in `9_Firmware/9_3_GUI` operates and why it matters.
The host layer is responsible for two different jobs:

- sending settings and startup intent to the MCU
- receiving and interpreting high-speed data from the FPGA

Because of that, the host layer is not only a user interface.
It is also a protocol boundary and an integration boundary.

## At a glance

- Main operational program: `GUI_V6.py`
- Main responsibilities: settings entry, MCU startup handshake, FT601-side data and status consumption
- Main interfaces: USB CDC to MCU, FT601 to FPGA
- Common source of confusion: the host tree mixes the main GUI with bring-up and protocol-inspection tools

## Companion references

This document is the host-side narrative view.
When the task becomes more field-oriented or flow-oriented, the most useful companion references are:

- `../protocol_field_reference.csv` for exact byte offsets, command fields, parser ownership, and known caveats
- `../system_signal_flow_matrix.csv` for tracing settings, commands, status, and data from host to MCU or FPGA
- `../device_connection_matrix.csv` for seeing which host-visible paths connect to which subsystems and devices

The markdown version explains why the host layer is structured this way.
The CSV references are better for quick lookup during integration and debugging.

## Main program and support files

- `GUI_V6.py` — primary user-facing GUI currently used for normal operation
- `radar_dashboard.py` — engineering dashboard for FT601 data inspection and visualization
- `radar_protocol.py` — shared host transport, parsing, acquisition, and command-building layer
- `smoke_test.py` — quick verification path for board bring-up and self-test
- `GUI_versions.txt` — version history for GUI iterations
- `requirements_dashboard.txt` — Python dependencies for the dashboard/tooling path

The main operational path is `GUI_V6.py`.
The other files are still important, but they mainly support validation, bring-up, and protocol inspection.

## What this layer is solving

At first glance the GUI can look like "just a control panel", but in this system the host layer solves several deeper problems:

- it expresses radar settings in physical terms such as frequency, PRF, chirp timing, and map size
- it translates those settings into a binary packet the MCU can parse
- it receives high-speed processed packets from the FPGA
- it gives engineers a place to inspect whether the system is alive, synchronized, and producing plausible outputs

This makes the host layer one of the best places to study the connection between user intent and system behavior.

## Startup and control flow

`GUI_V6.py` performs the MCU startup handshake over USB CDC:

1. Open the STM32 CDC device.
2. Send the 4-byte start flag `[23, 46, 158, 237]`.
3. Send a binary settings packet framed by `SET` and `END`.
4. Wait for the MCU to accept and apply the configuration path.

This startup flow matches the MCU `USBHandler` state machine.
It is the main control-plane path for ordinary operation.

## Settings packet contract

`GUI_V6.py` builds the MCU-compatible settings packet in this order:

- `b'SET'` (3 bytes)
- `system_frequency` as big-endian double
- `chirp_duration_1` as big-endian double
- `chirp_duration_2` as big-endian double
- `chirps_per_position` as big-endian unsigned int
- `freq_min` as big-endian double
- `freq_max` as big-endian double
- `prf1` as big-endian double
- `prf2` as big-endian double
- `max_distance` as big-endian double
- `map_size` as big-endian double
- `b'END'` (3 bytes)

By literal field count, the current `GUI_V6.py` packet layout is 82 bytes:

- `SET` = 3 bytes
- 9 doubles = 72 bytes
- 1 uint32 = 4 bytes
- `END` = 3 bytes

The exact offsets are:

- `0..2`: `SET`
- `3..10`: `system_frequency`
- `11..18`: `chirp_duration_1`
- `19..26`: `chirp_duration_2`
- `27..30`: `chirps_per_position`
- `31..38`: `freq_min`
- `39..46`: `freq_max`
- `47..54`: `prf1`
- `55..62`: `prf2`
- `63..70`: `max_distance`
- `71..78`: `map_size`
- `79..81`: `END`

This field order is the real contract the MCU must match for the current operational GUI path.

## Transport paths

The host layer uses two different transport channels:

- USB CDC for settings and startup control between GUI and MCU
- FT601 for high-speed FPGA-to-host data streaming

`GUI_V6.py` can use both paths.
`radar_dashboard.py` and `radar_protocol.py` are especially useful for the FT601 side because they separate packet handling from the front-end GUI logic.

The GUI also pads control transfers to 64-byte CDC packet boundaries, which is a practical transport detail rather than a radar algorithm detail.

## Host-side FPGA packet handling

`radar_protocol.py` defines the main host/FPGA packet constants:

- `HEADER_BYTE = 0xAA`
- `STATUS_HEADER_BYTE = 0xBB`
- `FOOTER_BYTE = 0x55`

It also defines the 32-bit host-to-FPGA command word format:

- `{opcode[31:24], addr[23:16], value[15:0]}`

This module is the shared protocol layer behind engineering tools, tests, and FT601 acquisition logic.

The important point is that the packed bit layout is the stable contract.
Human-readable command names are still useful, but they are a descriptive layer built on top of the bit positions rather than a stronger source of truth than the packed format itself.

## Acquisition and frame assembly

`RadarAcquisition` reads raw FT601 bytes, parses packets, and assembles a complete `RadarFrame` after `NUM_RANGE_BINS * NUM_DOPPLER_BINS` samples.
In the current tooling, that corresponds to `64 * 32 = 2048` samples.

The acquisition path also handles status packets separately and forwards them through callbacks.
This separation matters because data packets and health/status packets serve different debugging purposes.

## Why this layer matters

- It is the only layer that directly connects user settings, embedded configuration, and displayed results.
- It reveals how protocol framing and packet interpretation can be just as important as the radar algorithms behind the data.
- It provides the clearest place to compare what the system was asked to do against what it actually returned.

## Known caveats

- The GUI packet builders and MCU parser agree on field order, but MCU-side comments and guards still contain inconsistent byte-count assumptions.
- The repository contains both main operational tools and engineering support tools, so "host software" should not be treated as one single workflow.
- Packet parsing, transport handling, and display logic are distributed across several files, which makes the host layer flexible but easy to misread if only one file is inspected.

## Related files

- `test_radar_dashboard.py` — tests for dashboard behavior
- `smoke_test.py` — quick validation of GUI data flow
- `GUI_versions.txt` — version history for GUI iterations

## Future directions

- Add clearer protocol-state feedback so the GUI can distinguish "MCU accepted config" from "FPGA data path alive".
- Add replay and annotation tools so captured sessions can be reused for study, debugging, and comparison.
- Compare GUI-displayed outputs with offline golden references to catch subtle protocol or scaling errors.
- Expose richer engineering metadata without overwhelming the main operational interface.

## Notes for future expansion

- Inspect `radar_dashboard.py` in more detail to map packet fields into plots and detection layers.
- Confirm how partial FT601 packet boundaries are reconstructed under sustained streaming conditions.
- Trace how host-side detections, status words, and display queues are synchronized under load.
