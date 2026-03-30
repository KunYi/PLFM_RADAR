# Protocol Field Reference

## Overview

This page is the readable companion to `protocol_field_reference.csv`.
It reorganizes the most important protocol fields into a compact index that is easier to scan in markdown before moving to spreadsheet-style lookup.

Together with `protocol_field_reference.csv`, this page should be treated as the main protocol reference for the current codebase.
Use this markdown page as the readable entry point, and use the CSV when exact offsets, bit ranges, or spreadsheet-style comparison are needed.

Use this page when the question is:

- "Which fields are in this packet?"
- "Which offsets or bit ranges matter most?"
- "Which parser owns this contract?"

Use `protocol_field_reference.csv` when the task requires filtering, sorting, or comparing many fields at once.

## At a glance

- GUI to MCU path: start flag plus `SET ... END` framed settings packet
- Host to FPGA path: 32-bit command word over FT601
- FPGA to host path: `0xAA` data packets and `0xBB` status packets, both ending in `0x55`
- Common source of confusion: field order is stable, but some source comments, guards, and symbolic names still reflect older assumptions

## How to use this page

This page is grouped by message family:

1. GUI to MCU startup and settings
2. MCU to host status text
3. Host to FPGA command words
4. FPGA to host data packets
5. FPGA to host status packets

For each group, the goal is to show:

- the stable framing markers
- the important field order
- the parser or handler that depends on it
- the known caveats that are easy to miss

The wording in this page uses three different meanings:

- `Strict contract` means the implementation clearly depends on this value or layout
- `Current parser interpretation` means this is how the present host or MCU code interprets the payload
- `Implementation note` means this is useful context, but not necessarily something every parser enforces equally strongly

Maintenance rule:

- if code comments, host code, or firmware code are updated later, align them to this page and to `protocol_field_reference.csv`
- treat older comments or helper names as secondary to the current field layout documented here

## GUI to MCU

### Start flag

The MCU control path starts from a fixed 4-byte sentinel:

| Field | Offset | Size | Encoding | Consumer |
|---|---|---|---|---|
| `byte0` | `0` | `1` | `uint8` | `USBHandler::checkForStartFlag` |
| `byte1` | `1` | `1` | `uint8` | `USBHandler::checkForStartFlag` |
| `byte2` | `2` | `1` | `uint8` | `USBHandler::checkForStartFlag` |
| `byte3` | `3` | `1` | `uint8` | `USBHandler::checkForStartFlag` |

Value sequence:

- `23, 46, 158, 237`

Strict contract:

- the GUI must send this exact 4-byte sequence
- the MCU start detector explicitly searches for this pattern

### Settings packet

The newer GUI-compatible settings packet is framed by `SET` and `END`.

| Field | Offset | Size | Encoding | Consumer |
|---|---|---|---|---|
| `prefix_SET` | `0..2` | `3` bytes | `ASCII` | `RadarSettings::parseFromUSB()` |
| `system_frequency` | `3..10` | `8` bytes | `float64` | `RadarSettings::parseFromUSB()` |
| `chirp_duration_1` | `11..18` | `8` bytes | `float64` | `RadarSettings::parseFromUSB()` |
| `chirp_duration_2` | `19..26` | `8` bytes | `float64` | `RadarSettings::parseFromUSB()` |
| `chirps_per_position` | `27..30` | `4` bytes | `uint32` | `RadarSettings::parseFromUSB()` |
| `freq_min` | `31..38` | `8` bytes | `float64` | `RadarSettings::parseFromUSB()` |
| `freq_max` | `39..46` | `8` bytes | `float64` | `RadarSettings::parseFromUSB()` |
| `prf1` | `47..54` | `8` bytes | `float64` | `RadarSettings::parseFromUSB()` |
| `prf2` | `55..62` | `8` bytes | `float64` | `RadarSettings::parseFromUSB()` |
| `max_distance` | `63..70` | `8` bytes | `float64` | `RadarSettings::parseFromUSB()` |
| `map_size` | `71..78` | `8` bytes | `float64` | `RadarSettings::parseFromUSB()` |
| `suffix_END` | `79..81` | `3` bytes | `ASCII` | `RadarSettings::parseFromUSB()` |

Practical note:

- by literal field count, this packet spans `82` bytes
- MCU-side comments and one minimum-length guard still reference `74`, which should be treated as a known implementation gap rather than the intended field layout

Strict contract:

- the packet must begin with `SET`
- the packet must end with `END`
- the field order must match the parser order shown above

Implementation note:

- the field layout is stable
- packet-length bookkeeping in MCU code is not yet canonical, because some comments and guards still reflect the older `74`-byte assumption

## MCU to Host

### Status text

The MCU also emits human-readable status and debug text.
This is not a rigid binary packet like the settings or FT601 data paths.

| Field | Offset | Size | Encoding | Producer / Consumer |
|---|---|---|---|---|
| `ascii_payload` | `variable` | `variable` | `ASCII string` | `CDC_Transmit_FS` to GUI or serial monitor |

This path is useful for bring-up and diagnostics, but it should not be confused with a fixed schema contract.

Implementation note:

- this path is intentionally flexible and human-readable
- it is useful for status visibility, but it is not a substitute for a versioned telemetry packet

## Host to FPGA

### Command word

The host-side command layer packs commands into one 32-bit word:

| Field | Bit Range | Size | Encoding | Consumer |
|---|---|---|---|---|
| `opcode` | `31..24` | `8` bits | `uint8` | FPGA command decode |
| `address` | `23..16` | `8` bits | `uint8` | FPGA command decode |
| `value` | `15..0` | `16` bits | `uint16` | FPGA command decode |

Full packed form:

- `{opcode[31:24], addr[23:16], value[15:0]}`

Representative opcode regions visible in the current host/FPGA code:

- `0x01`
- `0x10..0x15`
- `0x20..0x27`
- `0x30..0x31`
- `0xFF`

Name layer versus bit-level contract:

- names such as "radar mode", "timing control", "self-test", or "status request" are useful summaries of the current code paths
- the stable cross-module contract is still the packed placement of `opcode`, `address`, and `value`
- if comments, helper names, or UI labels drift slightly, the bit ranges remain the source of truth

The exact symbolic naming can drift over time.
The stable contract is the bit layout.

Strict contract:

- the packed bit layout is the part that must remain aligned between host and FPGA

Implementation note:

- opcode naming is easier to change than field position
- comments and enum names should therefore be treated as secondary to the packed layout itself

## FPGA to Host

### Data packet

The current host parser treats the data packet as a `35`-byte structure:

| Field | Offset | Size | Encoding | Consumer |
|---|---|---|---|---|
| `header` | `0` | `1` byte | `0xAA` marker | `parse_data_packet()` |
| `range_word0` | `1..4` | `4` bytes | packed `uint32` | `parse_data_packet()` |
| `range_word1` | `5..8` | `4` bytes | packed `uint32` | `parse_data_packet()` |
| `range_word2` | `9..12` | `4` bytes | packed `uint32` | `parse_data_packet()` |
| `range_word3` | `13..16` | `4` bytes | packed `uint32` | `parse_data_packet()` |
| `doppler_word0` | `17..20` | `4` bytes | packed `uint32` | `parse_data_packet()` |
| `doppler_word1` | `21..24` | `4` bytes | packed `uint32` | `parse_data_packet()` |
| `doppler_word2` | `25..28` | `4` bytes | packed `uint32` | `parse_data_packet()` |
| `doppler_word3` | `29..32` | `4` bytes | packed `uint32` | `parse_data_packet()` |
| `detection_flag` | `33` | `1` byte | `uint8` | `parse_data_packet()` |
| `footer` | `34` | `1` byte | `0x55` marker | `parse_data_packet()` |

Logical payload versus transfer shape:

- the current logical payload used by the host is the first range I/Q word, the first Doppler I/Q word, and the detection flag
- the remaining range and Doppler words are part of the current FT601-facing transfer shape
- they should not be read as eight equally independent logical measurement fields

Current host-side interpretation:

- `range_word0` carries the meaningful range I/Q pair
- `doppler_word0` carries the meaningful Doppler I/Q pair
- the repeated shifted words exist to fit FT601 transfer behavior and current packet construction

Strict contract:

- the data packet starts with `0xAA`
- the transport shape reserves `35` bytes for the packet form currently used by the host tooling

Current parser interpretation:

- the host parser extracts meaning mainly from `range_word0`, `doppler_word0`, and `detection_flag`
- later repeated words are currently treated as transport-driven structure rather than independent logical payload fields

Implementation note:

- `RadarProtocol.parse_data_packet()` checks the header strongly
- footer handling is looser than the status parser, because the current host flow already depends on packet boundary finding before the per-packet parse step

### Status packet

The current host parser treats the status packet as a `26`-byte structure:

| Field | Offset or Bits | Size | Encoding | Consumer |
|---|---|---|---|---|
| `header` | `0` | `1` byte | `0xBB` marker | `parse_status_packet()` |
| `word0_cfar_threshold` | `word0[15:0]` | `16` bits | `uint16` | `parse_status_packet()` |
| `word0_stream_ctrl` | `word0[18:16]` | `3` bits | `uint3` | `parse_status_packet()` |
| `word0_radar_mode` | `word0[22:21]` | `2` bits | `uint2` | `parse_status_packet()` |
| `word1_long_chirp` | `word1[31:16]` | `16` bits | `uint16` | `parse_status_packet()` |
| `word1_long_listen` | `word1[15:0]` | `16` bits | `uint16` | `parse_status_packet()` |
| `word2_guard` | `word2[31:16]` | `16` bits | `uint16` | `parse_status_packet()` |
| `word2_short_chirp` | `word2[15:0]` | `16` bits | `uint16` | `parse_status_packet()` |
| `word3_short_listen` | `word3[31:16]` | `16` bits | `uint16` | `parse_status_packet()` |
| `word3_chirps_per_elev` | `word3[5:0]` | `6` bits | `uint6` | `parse_status_packet()` |
| `word4_range_mode` | `word4[1:0]` | `2` bits | `uint2` | `parse_status_packet()` |
| `word5_self_test_flags` | `word5[4:0]` | `5` bits | `uint5` | `parse_status_packet()` |
| `word5_self_test_detail` | `word5[15:8]` | `8` bits | `uint8` | `parse_status_packet()` |
| `word5_self_test_busy` | `word5[24]` | `1` bit | `bit` | `parse_status_packet()` |
| `footer` | `25` | `1` byte | `0x55` marker | `parse_status_packet()` |

Practical note:

- several upper or middle bits are treated as reserved by the current host parser
- the exact meaning of self-test detail fields should still be checked against FPGA self-test logic

Strict contract:

- the status packet starts with `0xBB`
- the current host parser expects `6` status words and a trailing `0x55`

Current parser interpretation:

- the listed fields are the parts the current host parser extracts and names explicitly
- several other bits are currently treated as reserved rather than decoded state

## Markers to remember

The most important framing markers across the repository are:

- GUI to MCU start flag: `23, 46, 158, 237`
- GUI to MCU settings prefix: `SET`
- GUI to MCU settings suffix: `END`
- FPGA data header: `0xAA`
- FPGA status header: `0xBB`
- FPGA footer: `0x55`

These markers are small details, but many integration failures start here.

Implementation note:

- start flag and settings framing are enforced on the MCU side
- status footer checking is stricter than data-footer checking in the current host parser
- packet markers are therefore still the right first thing to verify, but parser strictness is not identical for every packet family

## Common debugging paths

### When the MCU never leaves startup wait

Check these items in order:

1. Confirm the GUI actually sends the 4-byte start flag `23, 46, 158, 237`.
2. Confirm the settings packet still starts with `SET` and ends with `END`.
3. Confirm the field order still matches `RadarSettings::parseFromUSB()`.
4. If packet-length checks behave strangely, remember that the implemented field layout is `82` bytes while MCU-side comments and one guard still mention `74`.

The most common failure in this path is not "USB is broken".
It is usually that the host and MCU disagree about framing, field order, or total packet length.

### When FT601 data parses incorrectly

Check these items in order:

1. Confirm the packet starts with `0xAA` and ends with `0x55`.
2. Confirm the host parser still expects a `35`-byte data packet.
3. Confirm the first range and Doppler words are unpacked with the expected I/Q half-word ordering.
4. If later repeated words look strange, remember that not every packed word is equally meaningful in the current host interpretation.

The most common failure in this path is not the DSP chain itself.
It is often packet framing drift or word unpacking drift between FPGA output and host parsing.

### When status words look wrong

Check these items in order:

1. Confirm the packet starts with `0xBB` and ends with `0x55`.
2. Confirm the host parser still expects `6` status words.
3. Check whether the unexpected value is a real field or a bit range currently treated as reserved.
4. For self-test fields, cross-check the FPGA-side self-test logic before assuming the host interpretation is complete.

Status parsing failures are often subtle because the packet can still look structurally valid while individual bit meanings drift.

### When host-to-FPGA control does not match expectation

Check these items in order:

1. Confirm the command still uses `{opcode[31:24], addr[23:16], value[15:0]}`.
2. Confirm the opcode family still targets the expected FPGA block.
3. Confirm the address and value are interpreted under the same opcode meaning on both sides.

The stable part of this contract is the bit layout.
Symbolic command names are more likely to drift than the packed field positions.

## Best companion files

- `Protocol_Contracts_and_Known_Gaps.md` for the narrative explanation of what must stay aligned
- `protocol_field_reference.csv` for full filtering and spreadsheet lookup
- `system_signal_flow_matrix.csv` for tracing where each message goes next
- `GUI/GUI_Host_Guide.md` and `MCU/MCU_Firmware_Guide.md` for the control-path context
- `FPGA/FPGA_Interfaces_and_Verification_Guide.md` for the FT601 transport context
