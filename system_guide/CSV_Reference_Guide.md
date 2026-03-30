# CSV Reference Guide

## Why these CSV files exist

The markdown documents explain the system in narrative form.
The CSV files serve a different purpose: they make it easy to filter, sort, and cross-check system information in spreadsheet tools such as Excel, LibreOffice Calc, or Google Sheets.

## At a glance

- Start with `device_connection_matrix.csv` for hardware ownership
- Use `system_signal_flow_matrix.csv` for end-to-end tracing
- Use `protocol_field_reference.csv` for field and bit layout
- Use the MCU and FPGA pin CSVs for exact interface inventory

This is useful when a question is easier to answer as a table lookup than as a long document read.

Typical examples include:

- finding every MCU pin connected to a specific external device
- checking which FPGA pins belong to ADC, FT601, or clocking paths
- tracing how one signal moves from source to parser to display
- comparing protocol field offsets without scanning several code files

## Recommended way to use them

If the repository is still unfamiliar, it helps to use the CSV files in this order:

1. `device_connection_matrix.csv`
2. `system_signal_flow_matrix.csv`
3. `protocol_field_reference.csv`
4. `MCU/MCU_Pin_Definitions.csv`
5. `FPGA/FPGA_Pin_Definitions.csv`

This order moves from system-level structure to implementation detail.

## What each CSV is for

### `device_connection_matrix.csv`

This is the fastest system-level hardware map.
It connects devices and subsystems across the host, MCU, FPGA, and RF/control layers.

Use it when the question sounds like:

- "Which controller or module talks to this chip?"
- "Is this device handled by MCU code, FPGA logic, or both?"
- "Where should I look first if this component is failing?"

Helpful columns to filter:

- `device_name`
- `mcu_peripheral_or_gpio`
- `fpga_port_or_interface`
- `firmware_owner_module`
- `fpga_owner_module`
- `system_role`

### `system_signal_flow_matrix.csv`

This table focuses on data movement and control movement.
It shows where a signal or message starts, how it is transported, what parses it, and where it ends up being used.

Use it when the question sounds like:

- "How does a GUI action reach the hardware?"
- "Where is this FT601 packet parsed?"
- "What happens after ADC data enters the FPGA?"

Helpful columns to filter:

- `source`
- `transport_or_interface`
- `intermediate_handler`
- `destination`
- `system_purpose`
- `known_caveats`

### `protocol_field_reference.csv`

This is the field-by-field contract table.
It flattens packet bytes and command bit ranges into a spreadsheet format.

If a readable markdown summary is more useful before opening a spreadsheet, pair it with `Protocol_Field_Reference.md`.
If the task is to understand which parts are strict contracts versus current parser interpretation, also pair it with `Protocol_Contracts_and_Known_Gaps.md`.

Use it when the question sounds like:

- "What byte offset is `map_size`?"
- "How is the FPGA command word packed?"
- "Which parser owns this packet field?"

Helpful columns to filter:

- `message_name`
- `field_name`
- `offset_or_bit_range`
- `encoding`
- `handler_or_parser`
- `known_caveats`

This file is especially useful during host/MCU/FPGA integration work, because interface bugs often come from one side changing a field without the other sides noticing.

For maintenance, treat this CSV together with `Protocol_Field_Reference.md` as the current protocol reference pair:

- `Protocol_Field_Reference.md` is the readable protocol index
- `protocol_field_reference.csv` is the exact table for offsets, ranges, encodings, and parser ownership
- `Protocol_Contracts_and_Known_Gaps.md` records the remaining cleanup debt, not an alternate field layout

### `MCU/MCU_Pin_Definitions.csv`

This table is the detailed MCU-side pin and peripheral inventory.
It includes GPIO names, alternate functions, connected devices, subsystem roles, and firmware ownership.

Use it when the question sounds like:

- "Which MCU pins belong to `SPI4` or `I2C2`?"
- "Which firmware module owns this connected device?"
- "Which GPIO resets FPGA or enables the TX path?"

Helpful columns to filter:

- `peripheral`
- `connected_device`
- `firmware_owner_module`
- `subsystem`
- `gpio_port`
- `alternate_function`

### `FPGA/FPGA_Pin_Definitions.csv`

This table is the FPGA-side pin and interface inventory.
It is useful for checking board-level wiring, interface grouping, and bring-up scope.

Use it when the question sounds like:

- "Which FPGA pins belong to FT601?"
- "Which signals are part of the ADC path?"
- "What is included in the minimal bring-up target?"

Helpful columns to filter:

- `interface_group`
- `signal_name`
- `package_pin`
- `io_standard`
- `board_context`
- `notes`

## A simple workflow for common tasks

### To debug a hardware connection

1. Start with `device_connection_matrix.csv` to identify the device owner and subsystem role.
2. Open `MCU/MCU_Pin_Definitions.csv` or `FPGA/FPGA_Pin_Definitions.csv` to inspect exact pins and interfaces.
3. Open the related markdown chapter to understand how that block is supposed to behave.

### To debug a protocol mismatch

1. Start with `protocol_field_reference.csv`.
2. Filter by the packet or command name.
3. Check `handler_or_parser` and `source_of_truth`.
4. Then compare with `Protocol_Contracts_and_Known_Gaps.md` to see whether the mismatch is already known.
5. Use `Protocol_Field_Reference.md` when a quick readable summary is enough and spreadsheet filtering is not necessary.

### To understand end-to-end behavior

1. Start with `system_signal_flow_matrix.csv`.
2. Follow one flow from `source` to `destination`.
3. Use the linked markdown chapters for the subsystem details behind each stage.

## Practical spreadsheet tips

- Freeze the top row so column names stay visible while scrolling.
- Turn on filters for every column.
- Create temporary filtered views for one subsystem at a time, such as `FT601`, `SPI4`, `GPS`, or `CFAR`.
- When a table feels too large, filter by `system_role`, `message_name`, or `connected_device` first.

## Relationship to the markdown files

The CSV files are not meant to replace the main documents.
They are companion references.

Use the markdown files when the question is about:

- why a subsystem exists
- how the architecture is organized
- what tradeoffs the current design has
- what extensions or open questions follow from the current implementation

Use the CSV files when the question is about:

- exact fields
- exact pins
- exact ownership
- exact source-to-destination paths

Together, they make it easier to move between concept-level understanding and implementation-level inspection.
