# FPGA Interfaces and Verification Guide

## Overview

This chapter explains how the FPGA communicates with the outside world and how the design is validated.
It focuses on FT601 transport, host command handling, clock-domain crossing, reset synchronization, and the repository's verification support.

The key idea is that a radar FPGA is not only a processing engine.
It is also a boundary device that must move data out reliably and accept control input safely.

## At a glance

- Main interface module: `usb_data_interface.v`
- Main focus: FT601 transport, command intake, CDC safety, and status visibility
- Best companion pages: `Protocol_Field_Reference.md` and `FPGA_Architecture_Overview.md`
- Common source of confusion: transport framing, runtime contract, and bring-up-oriented behavior are related but not identical

## Main files and roles

- `usb_data_interface.v` — primary host-facing RTL interface
- `radar_system_top.v` — main integration point where host interface logic is wired into the wider design
- `tb/` — simulation assets for engineering validation
- `scripts/` and `run_regression.sh` — build, regression, programming, and debug-support tools

For architecture understanding, `usb_data_interface.v` and `radar_system_top.v` are the main files.
For engineering workflow understanding, the testbench and script assets matter next.

## USB and host command interface

The FPGA uses an FT601 chip in Slave FIFO mode for high-speed host transfer.
`usb_data_interface.v` manages:

- the 32-bit data bus
- byte-enable signals
- read/write strobes
- FT601 status flags
- packet framing for radar data and status packets

The write path is implemented as a state machine that inserts headers, payload groups, and footers while respecting FT601 flow control.
That structure matters because the host must receive packets in a form it can trust, not just a stream of words.

One subtle point is that the current FT601 data-packet shape is not the same thing as the current logical payload.
The transport emits range and Doppler word groups that fit the FT601-facing write pattern, while the host currently treats only the first packed range word, first packed Doppler word, and the detection flag as the logically significant decoded values.

The read path accepts 32-bit command words from the host and decodes them into:

- opcode
- address
- value

This forms the main host-to-FPGA runtime control path.

## Packet and command framing

The host/FPGA interface uses three especially important markers:

- `0xAA` for data packets
- `0xBB` for status packets
- `0x55` for packet footer

The command word format is:

- `{opcode[31:24], addr[23:16], value[15:0]}`

This command path is simple enough to inspect directly in RTL and in the host tooling, which makes it one of the cleanest cross-module contracts in the repository.

## Bring-up defaults versus runtime contract

Some FPGA-side behaviors are easiest to understand if they are separated into two categories:

- bring-up defaults that help the design avoid deadlock or provide an immediately observable path during early integration
- runtime contract items that host tools and other modules must keep aligned during normal operation

Bring-up defaults usually answer questions like:

- which streams are enabled after reset
- whether status can be requested even before a full control session has been exercised
- which internal values are exposed first because they are useful for first-pass validation

Runtime contract items are the things other modules must treat as stable:

- packet markers such as `0xAA`, `0xBB`, and `0x55`
- command-word bit layout
- the meaning of active control bits and decoded status fields
- CDC-safe transfer of control and data-valid events between domains

Practical reading rule:

- if a behavior mainly exists to make first hardware bring-up easier, do not assume it is the long-term architectural contract
- if a behavior must remain aligned between host parsing, top-level decode, and transport framing, treat it as a true runtime contract

## Clock domain crossing and reset synchronization

This part of the design crosses several domains:

- `clk_100m` for main system logic
- `clk_120m_dac` for DAC-domain timing
- `ft601_clk` for USB transport
- `clk_400m` for ADC capture and DDC

Those domains cannot safely share events directly.
The design therefore uses:

- toggle-based CDC for one-shot events
- multi-flop synchronization for level signals
- per-domain reset synchronization

This is one of the most important implementation patterns in the FPGA tree because subsystem-boundary bugs often appear here first.

## CDC summary table

| Crossing | Source domain | Destination domain | Signal type | Technique used | Why it matters |
|---|---|---|---|---|---|
| STM32 GPIO events into transmitter control | async STM32 GPIO treated into `clk_100m` path | `clk_100m` | event-like control inputs | single-bit synchronization plus edge detection | prevents metastability and false chirp/elevation/azimuth events |
| `new_chirp_pulse` into DAC-domain control | `clk_100m` | `clk_120m_dac` | one-shot pulse | toggle CDC plus edge recovery | a plain level synchronizer could miss narrow pulses across the 100/120 MHz boundary |
| `stm32_mixers_enable` into DAC-domain control | async or `clk_100m`-side control | `clk_120m_dac` | level signal | synchronized level crossing | keeps runtime mixer-enable behavior stable in the transmit timing domain |
| transmitter chirp counter into system domain | `clk_120m_dac` | `clk_100m` | multi-bit state | dedicated multi-bit CDC path | allows the receiver and top-level logic to consume transmitter progress safely |
| `new_chirp_frame` into system domain | `clk_120m_dac` | `clk_100m` | one-shot pulse | toggle CDC plus edge recovery | preserves frame-boundary events that could otherwise be missed |
| range/doppler/CFAR valid signals into FT601 interface | `clk_100m` | `ft601_clk_in` | valid strobes with associated held data | 2-stage valid synchronization plus destination-side capture from holding registers | lets the FT601 write FSM consume stable samples even though the transport logic is in a different domain |
| stream-control register into FT601 interface | `clk_100m` | `ft601_clk_in` | slowly changing control bits | per-bit 2-stage synchronizers | avoids write-FSM interpretation drift between host control and FT601-domain transport |
| status-request pulse into FT601 interface | `clk_100m` | `ft601_clk_in` | pulse | toggle CDC plus edge detection | allows status snapshots to be generated in the FT601 transport domain |
| FT601 command-valid pulse into system command decode | `ft601_clk_in` | `clk_100m` | pulse | toggle CDC plus edge recovery | safely transfers host command arrival into the main control domain |
| reset release into each local domain | async external reset | `clk_100m`, `clk_120m_dac`, `ft601_clk_in` | reset deassertion | per-domain reset synchronizers | prevents recovery/removal issues and keeps each domain's local state release predictable |

This table is not a replacement for reading the RTL.
Its purpose is to make the main CDC intent visible before diving into implementation details.

## Status readback and observability

When the host requests status, the FPGA captures runtime information and sends a dedicated status packet back over FT601.
This matters for two reasons:

- the host needs more than raw radar samples
- debugging requires some visibility into mode, thresholds, timing, and health state

Even a relatively small status path can greatly reduce bring-up time if it captures the right state.

One useful distinction is that status readback sits across both categories above:

- the existence of a status-request path is part of the runtime interface contract
- the exact amount of debug detail currently returned is closer to a bring-up-oriented implementation choice that could grow later

## Top-level integration

`radar_system_top.v` ties the interface layer into the wider radar design.
It is responsible for:

- clock buffering and reset synchronization
- instantiating transmitter, receiver, and USB interface modules
- routing host commands into runtime control registers
- collecting status information for readback
- managing CDC between subsystems

This makes the top level the place where transport logic stops being "just an interface" and becomes part of the system architecture.

## Verification and build support

The repository includes:

- testbench assets in `tb/`
- automated regression through `run_regression.sh`
- supporting build/programming scripts in `scripts/`

These assets matter because radar and CDC bugs often depend on timing, ordering, or corner cases that are hard to catch by inspection alone.

## Why this chapter matters

- It explains how processed radar data actually leaves the FPGA in a usable form.
- It shows that synchronization, transport, and reset design are part of the radar architecture, not afterthoughts.
- It provides a practical debugging map for problems that appear only at subsystem boundaries.

## Known caveats

- The packet and command handling are understandable from source, but the repository would still benefit from a single canonical register-map or command-map page.
- Verification assets exist, but this chapter is still more structural than coverage-oriented; a future pass could describe what is simulated and how deeply.

## Questions for further development

- Is the current FT601 packet structure a long-term interface or mainly a bring-up-friendly format?
- Should more internal debug signals be exportable on demand?
- How should verification grow from module-level simulation toward tighter hardware correlation?
- Which failures are still difficult to observe because the status path remains relatively small?

## Future improvement directions

- Add a clearer documented register or command map shared by host and FPGA docs.
- Expand the status packet with health, timing, and error indicators where useful.
- Add a lightweight debug mode for selected internal pipeline values.
- Connect regression assets more explicitly to the subsystem boundaries they are meant to protect.
