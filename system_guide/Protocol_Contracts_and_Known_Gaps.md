# Protocol Contracts and Known Gaps

## Overview

This document collects the most important cross-module contracts in the repository:

- GUI ↔ MCU startup and settings transport
- host ↔ FPGA FT601 packet transport
- command and status word formats
- current inconsistencies or cleanup debt

Its purpose is to answer two practical questions:

1. What must match across subsystems for the system to work?
2. Where does the repository still contain ambiguity, inconsistency, or historical drift?

## Companion references

This document explains the contracts in narrative form.
For field-by-field lookup, use the CSV references alongside it:

- `Protocol_Field_Reference.md` for a readable packet and command-field index
- `protocol_field_reference.csv` for byte offsets, bit ranges, encodings, parser ownership, and known caveats
- `system_signal_flow_matrix.csv` for seeing how control and data messages move from source to destination

The markdown view is best for understanding why the contracts matter.
The protocol index is best for quick reading.
The CSV view is best for filtering and checking exact field layout during integration work.

Practical maintenance rule:

- use `Protocol_Field_Reference.md` plus `protocol_field_reference.csv` as the canonical protocol reference
- use this page to explain contract meaning, risk, and cleanup debt around that reference

This document also distinguishes between:

- `strict contract` items that the current implementation clearly depends on
- `current parser interpretation` items that describe how the present host or MCU code reads the payload
- `known gaps` where comments, guards, or symbolic names have drifted from the clearest intended contract

## GUI ↔ MCU control contract

### Start flag

The host GUI starts the STM32 control path with a 4-byte start flag:

- `23, 46, 158, 237`

Current contract:

- the GUI must send this exact sequence
- the MCU scans CDC traffic for it while in `WAITING_FOR_START`
- once detected, the MCU moves into `RECEIVING_SETTINGS`

### Settings framing

The newer GUI path frames the settings packet as:

- prefix: `SET`
- payload: binary fields
- suffix: `END`

Current contract:

- `USBHandler` expects `SET` at the start of the buffered packet
- `USBHandler` searches for `END` to detect packet completion
- `RadarSettings::parseFromUSB()` expects both framing markers

### Active field order

The active layout used by `GUI_V6.py` is:

1. `system_frequency` — double
2. `chirp_duration_1` — double
3. `chirp_duration_2` — double
4. `chirps_per_position` — uint32
5. `freq_min` — double
6. `freq_max` — double
7. `prf1` — double
8. `prf2` — double
9. `max_distance` — double
10. `map_size` — double

## Known GUI ↔ MCU inconsistencies

### Packet length inconsistency

The repository currently contains a real byte-count inconsistency.

Observed facts:

- `GUI_V6.py` emits a packet with `9` doubles plus `1` uint32 between `SET` and `END`
- that field count gives `3 + 72 + 4 + 3 = 82` bytes
- `RadarSettings.cpp` comments the minimum packet size as `74` bytes
- `RadarSettings.cpp` checks `length < 74`
- `USBHandler.cpp` begins packet-complete checking once `buffer_index >= 74`

Interpretation:

- the field order is clear
- the packet-length accounting is not
- this is a cleanup bug, not merely a documentation issue

Quick reference:

| Aspect | Current value | Best interpretation |
|---|---|---|
| Active field layout | `82` bytes | canonical intended packet shape |
| Older MCU comment and guard threshold | `74` bytes | stale or incomplete packet-length bookkeeping |
| Integration expectation | use `82` bytes for field reasoning | rely on `74` only when explaining current MCU-side implementation debt |

### Startup loop semantics

The higher-level architecture suggests startup should wait for valid settings, but the current firmware loop exits on start-flag receipt alone.

Interpretation:

- `READY_FOR_DATA` is checked inside the loop body
- the final `do ... while` exit condition is still based on `isStartFlagReceived()`

So the current startup contract is slightly weaker than a strict "validated config required before proceed" design.

## Host ↔ FPGA FT601 packet contract

### Packet markers

The host-side protocol layer and FPGA USB interface agree on these markers:

- data header: `0xAA`
- status header: `0xBB`
- footer: `0x55`

### Data packet structure

The current expected data packet structure is:

- header `0xAA`
- range word group: 4 x 32-bit transfers
- doppler word group: 4 x 32-bit transfers
- detection byte
- footer `0x55`

Host-side interpretation:

- packet length is treated as 35 bytes when all streams are enabled
- only the first 32-bit range word and first 32-bit doppler word are logically significant
- repeated shifted words exist to satisfy FT601 byte-enable behavior

Logical payload versus transfer shape:

- the current logical payload is smaller than the on-wire transfer shape
- readers should treat the later repeated words as transport-shaped redundancy, not as additional independently decoded radar values
- this distinction matters whenever host parsing is rewritten or the packet contract is redesigned

Important distinction:

- packet framing markers are part of the transport contract
- the fact that only the first range and Doppler words are treated as logically significant is a current host-parser interpretation, not a statement that every transferred word carries equal semantic weight

### Status packet structure

The current expected status packet structure is:

- header `0xBB`
- 6 status words
- footer `0x55`

Important distinction:

- the current host parser enforces the status footer more strictly than the data-packet parser does
- readers should therefore not assume every packet family is validated with identical parser strictness

### Frame assembly contract

The host acquisition logic assembles a frame after:

- `NUM_RANGE_BINS * NUM_DOPPLER_BINS`
- currently `64 * 32 = 2048` samples

## Host → FPGA command word contract

The current command-word format is:

- `{opcode[31:24], addr[23:16], value[15:0]}`

Representative opcode regions visible in host and FPGA code include:

- `0x01` — trigger or radar-mode related path depending on naming layer
- `0x10..0x15` — chirp timing / chirps-per-elevation controls
- `0x20..0x27` — range mode, CFAR, MTI, notch controls
- `0x30..0x31` — self-test
- `0xFF` — status request

The bit layout is the stable contract.
Symbolic naming is useful, but the field positions are the part that must stay aligned.

Practical reading rule:

- trust the packed fields `opcode[31:24]`, `addr[23:16]`, and `value[15:0]` first
- treat symbolic command names as the current interpretation layer built on top of those fields
- if host comments and FPGA comments ever feel slightly different, the bit layout is the canonical contract to preserve

## Current stable contracts

These items are stable enough to teach with confidence:

- GUI start flag value
- `SET ... END` framing concept
- field order used by the active `GUI_V6.py` path
- FT601 packet markers `0xAA`, `0xBB`, `0x55`
- host command-word bit layout
- `64 x 32` frame assembly expectation in current host tooling

## Known gaps and cleanup targets

These items should be called out explicitly:

- GUI/MCU settings packet byte count is inconsistent in source comments and guards
- MCU startup-loop behavior is slightly weaker than the likely intended strict validated-settings contract
- the repository mixes main operational host flow and engineering FT601 tooling
- some symbolic opcode naming is descriptive rather than canonical, while bit layout is the true contract

## Why this document matters

In a multi-part radar project it is easy to understand each file separately but still miss what absolutely must match across modules.
This document exists to solve that problem.

It is also the best place to separate:

- historical differences that are understandable
- active contracts that must remain aligned
- real inconsistencies that should be cleaned up

## Future extensions

Natural next improvements include:

- introduce explicit protocol version fields
- add packet length fields and CRC protection
- define one canonical shared protocol specification
- keep the active `GUI_V6.py` contract clearly separated from source-code cleanup debt
- include timestamps and richer metadata in high-speed data packets
