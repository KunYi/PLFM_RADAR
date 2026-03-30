# System Guide

This folder contains a guided system reference for the radar platform.
It started from structured analysis, and has grown into a combined set of architecture notes, protocol references, pin maps, CSV lookup tables, and debugging-oriented guidance.

Recommended first read:

- `Quick_Start_System_Guide.md` — one-page onboarding summary for understanding the radar stack and repository structure quickly
- `CSV_Reference_Guide.md` — guide for using the spreadsheet-friendly reference tables in this folder

Suggested reading path:

1. Start with `Quick_Start_System_Guide.md` for the first system-wide mental model.
2. Continue to `System_Architecture_Guide.md` for the end-to-end role split and main data/control boundaries.
3. Read `GUI/GUI_Host_Guide.md`, `MCU/MCU_Firmware_Guide.md`, and `FPGA/FPGA_Architecture_Overview.md` to understand each major subsystem.
4. Use `Protocol_Contracts_and_Known_Gaps.md` and `Protocol_Field_Reference.md` when cross-module interfaces need to be checked.
5. Use `CSV_Reference_Guide.md` and the CSV files when the task becomes lookup-heavy.

Protocol source-of-truth rule:

- `Protocol_Field_Reference.md` is the main readable protocol index
- `protocol_field_reference.csv` is the exact field-and-bit lookup table to keep aligned with code
- `Protocol_Contracts_and_Known_Gaps.md` explains why those fields matter and where real cleanup debt still exists

If protocol wording is ever updated later, these three files should be kept aligned in that order.

Read the documents with this role split in mind:

- User-facing main host program: `GUI/GUI_Host_Guide.md` centers `GUI_V6.py`
- MCU main runtime flow: `MCU/MCU_Firmware_Guide.md` centers `main.cpp`
- FPGA main RTL integration view: `FPGA/FPGA_Architecture_Overview.md` and `FPGA/FPGA_Architecture_Guide.md` center `radar_system_top.v`
- Testbenches, dashboards, regression scripts, and board-specific wrappers are described as support tools or development variants rather than the primary operational path

The overall writing goal is not only to describe what the current code does, but also to:

- connect implementation details to system-level radar concepts
- understand why each subsystem exists
- see what engineering problems each module is trying to solve
- identify natural extension points for future development or study

Whenever possible, each document should therefore answer four questions:

1. What problem does this part of the system solve?
2. How does this repository solve it today?
3. What tradeoffs or limitations does the current solution have?
4. What future questions or extensions naturally follow from it?

- `FPGA/` — FPGA RTL architecture, data paths, USB interface, and signal processing.
  - `FPGA_Architecture_Overview.md`
  - `FPGA_Transmitter_and_Scan_Control_Guide.md`
  - `FPGA_Receiver_and_ADC_Pipeline_Guide.md`
  - `FPGA_Signal_Processing_Guide.md`
  - `FPGA_Interfaces_and_Verification_Guide.md`
- `MCU/` — microcontroller firmware behavior, commands, and system integration.
  - `MCU_Firmware_Guide.md`
- `GUI/` — host/PC software behavior, USB/FT601 communication, and user interface flow.
  - `GUI_Host_Guide.md`
- `System_Architecture_Guide.md` — end-to-end system summary linking GUI, MCU, and FPGA.
- `Protocol_Contracts_and_Known_Gaps.md` — cross-module contract summary for GUI, MCU, and FPGA interfaces, including currently known inconsistencies.
- `Protocol_Field_Reference.md` — readable protocol index for start flags, settings fields, command words, and FT601 packet fields.
- `Improvement_Recommendations.md` — non-invasive improvement ideas for the guide, MCU firmware, and FPGA RTL.
- `Quick_Start_System_Guide.md` — one-page first-stop summary of system roles, startup flow, interfaces, and learning directions.
- `CSV_Reference_Guide.md` — how to use the CSV reference tables and when each one is the right starting point.
- `device_connection_matrix.csv` — cross-subsystem map of devices, ownership, and system roles.
- `system_signal_flow_matrix.csv` — end-to-end source-to-destination signal and control flow table.
- `protocol_field_reference.csv` — packet fields, command bit ranges, parsers, and known caveats.
- `MCU/MCU_Pin_Reference.md` — grouped MCU pin, peripheral, and connected-device reference.
- `MCU/MCU_Pin_Definitions.csv` — MCU pin, peripheral, and connected-device inventory.
- `FPGA/FPGA_Pin_Reference.md` — grouped FPGA pin and interface reference.
- `FPGA/FPGA_Pin_Definitions.csv` — FPGA pin and board-interface inventory.

Use this space as a system-level guide for architecture understanding, protocol tracing, pin lookup, and cross-module debugging.
