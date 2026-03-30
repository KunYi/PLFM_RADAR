# Improvement Recommendations

## Overview

This document collects improvement ideas discovered while reviewing the current `system_guide`, MCU firmware, and FPGA RTL.

It does not propose immediate code edits.
Instead, it records where the current implementation or documentation could be made clearer, safer, easier to maintain, or easier to extend.

## Current state

After the latest deep scan, the overall state is good:

- the `system_guide` tree is internally consistent
- the major cross-references are aligned
- the main known protocol caveats are already documented
- CSV files remain structurally consistent

No new high-severity structural documentation problems were found during the latest verification pass.

## Priority scale

- High: worth addressing early because it affects correctness, integration reliability, or maintenance cost
- Medium: improves robustness or clarity, but does not immediately block understanding
- Lower: useful polish or longer-term evolution ideas

## MCU firmware improvement ideas

### High: strengthen startup gating semantics

Observed behavior:

- `main.cpp` checks for both `isStartFlagReceived()` and `READY_FOR_DATA` inside the loop body
- the final blocking condition still exits on `isStartFlagReceived()` alone

Files:

- `9_Firmware/9_1_Microcontroller/9_1_3_C_Cpp_Code/main.cpp`
- `9_Firmware/9_1_Microcontroller/9_1_1_C_Cpp_Libraries/USBHandler.cpp`

Recommended direction:

- make the exit condition reflect the intended system contract directly
- if valid settings are required before startup continues, gate on parsed-and-accepted settings rather than only on the start flag

Why it matters:

- this is a real correctness boundary between host control and hardware sequencing
- the current behavior is explainable, but weaker than the likely intent

### High: make packet-length handling canonical in one place

Observed behavior:

- `RadarSettings.cpp` still uses `74` in the minimum-size comment and guard
- `USBHandler.cpp` starts packet-complete checks once `buffer_index >= 74`
- actual field order now implies `82` bytes

Files:

- `9_Firmware/9_1_Microcontroller/9_1_1_C_Cpp_Libraries/RadarSettings.cpp`
- `9_Firmware/9_1_Microcontroller/9_1_1_C_Cpp_Libraries/USBHandler.cpp`

Recommended direction:

- define one canonical packet-size constant
- define one canonical field layout comment near the parser
- avoid duplicated magic numbers across parser and buffering logic

Why it matters:

- it reduces protocol drift
- it makes host and firmware compatibility easier to reason about

### Medium: separate bring-up, steady-state control, and diagnostics more clearly

Observed behavior:

- `main.cpp` combines long startup sequencing, hardware initialization, GPS/status handling, and later runtime actions in one large file

Recommended direction:

- gradually split into clearer sections or modules such as:
  - startup sequencing
  - RF/control initialization
  - platform sensors and diagnostics
  - runtime service loop

Why it matters:

- the firmware is understandable today, but maintenance cost will rise quickly as more features are added

### Medium: reduce raw GPIO writes where the action is system-critical

Observed behavior:

- some highly important actions still appear as direct `HAL_GPIO_WritePin(...)` calls
- examples include FPGA reset and TX mixer enable

Recommended direction:

- introduce small named helpers for high-impact actions
- examples:
  - `reset_fpga_path()`
  - `enable_tx_mixer_path()`
  - `apply_rf_power_sequence()`

Why it matters:

- this improves readability
- it also makes sequencing intent more visible in logs and future reviews

### Medium: make parser and transport states more explicit in diagnostics

Observed behavior:

- the USB path already has useful diagnostics
- however, system-level distinction between "start flag received", "settings buffered", "settings validated", and "startup allowed to proceed" could still be clearer

Recommended direction:

- preserve the current logs, but consider a more explicit state transition vocabulary

Why it matters:

- many integration failures happen at this exact boundary
- sharper logging often saves more time than deeper postmortem analysis

## FPGA improvement ideas

### High: separate protocol contract comments from historical gap comments

Observed behavior:

- `usb_data_interface.v` contains useful commentary, including `Gap 2`, `Gap 4`, `Fix 3`, `Fix 7`, and similar labels
- these comments preserve history, but they mix:
  - current contract
  - rationale
  - historical repair notes

Files:

- `9_Firmware/9_2_FPGA/usb_data_interface.v`
- `9_Firmware/9_2_FPGA/radar_system_top.v`

Recommended direction:

- keep the rationale, but separate historical repair labels from current interface definitions
- one approach is:
  - `Current contract`
  - `CDC rationale`
  - `Implementation note`
  - `Historical note`

Why it matters:

- the current comments are valuable, but they are harder to scan than they need to be
- this especially affects new maintainers trying to distinguish "what is true now" from "how it got here"

### High: centralize command decoding and contract metadata

Observed behavior:

- host command structure is stable
- symbolic meaning is spread across host code, top-level decode, and comments

Recommended direction:

- centralize command opcode documentation in one canonical block
- keep bit layout, opcode names, and affected registers together

Why it matters:

- command drift is one of the most likely long-term cross-module failure modes
- a single source of truth reduces ambiguity between host and FPGA

## Cross-module improvement ideas

### High: define one canonical protocol specification source

Right now the protocol is understandable, but it is still reconstructed from:

- host builders/parsers
- MCU parser code
- FPGA packet generation and command decode
- narrative guide pages

Recommended direction:

- treat `Protocol_Field_Reference.md` plus `protocol_field_reference.csv` as the canonical human-readable source
- keep the code comments aligned to that reference

Why it matters:

- this is the cleanest way to reduce future host/MCU/FPGA drift

### Medium: consider a compact known-gaps summary page

The existing guide already mentions the important gaps in the right places.
A very short single-page summary could still be useful later if the project grows.

Recommended direction:

- only do this if the number of active known gaps increases
- otherwise the current distributed approach is adequate

## Suggested next steps

If only a few follow-up tasks should be done, the highest-value sequence is:

1. make MCU packet-length handling canonical in one place
2. reorganize FPGA contract comments so historical fix notes do not compete with current interface meaning
3. consider a compact known-gaps summary page only if the number of active gaps grows
4. keep code comments aligned to `Protocol_Field_Reference.md` and `protocol_field_reference.csv` as the canonical protocol description

## Closing note

The important result of this review is that the folder no longer looks like a loose set of reverse-engineering notes.
It now behaves much more like a real system guide.

The remaining work is mostly refinement:

- reducing ambiguity
- reducing maintenance risk
- making long-term evolution easier

That is a good place for the folder to be.
