# Quick Start System Guide

## Overview

This page is the fastest way to build a mental model of the repository before diving into module-level details.

## At a glance

- Main split: host GUI, MCU control path, FPGA data path
- Main startup flow: start flag, settings packet, hardware sequencing, FT601 data stream
- Main risk area: subsystem boundaries, especially packet layout and startup expectations
- Best next page after this one: `System_Architecture_Guide.md`

The project can be understood as three cooperating layers:

- the host GUI chooses operating parameters and shows results
- the MCU configures hardware and manages startup
- the FPGA performs the real-time radar data path

If those three roles are clear, the rest of the repository becomes much easier to read.

## What this system is trying to do

At a high level, the system sends radar chirps, receives echoes, digitizes them, processes them, and sends useful results back to the PC.

In practice, the repository splits that work into separate responsibilities:

- `GUI_V6.py` provides the main operational front panel
- the STM32 firmware receives settings and sequences the hardware
- the FPGA handles the fast sample path and packetized output data

This separation is common in real radar platforms because control traffic and signal-processing traffic have very different timing requirements.

## Fast mental model

### Host side

The main user-facing program is `GUI_V6.py`.
This is where operating parameters such as frequency range, PRF, chirp timing, and display-related settings are chosen.

There are also support-side tools:

- `radar_dashboard.py` for engineering visualization and FT601-side inspection
- `radar_protocol.py` for packet parsing, acquisition, and host-side command formatting
- `smoke_test.py` for quick bring-up and scripted checks

The key idea is that not every host-side file serves the same purpose.

### MCU side

The MCU is the control-plane coordinator.
It receives settings from the host, configures external devices, controls startup ordering, and helps the hardware enter a usable state.

This is why the MCU documentation centers `main.cpp`, `USBHandler`, and `RadarSettings`.

### FPGA side

The FPGA is the data-plane engine.
It captures and moves samples, applies radar processing blocks, and packages results for high-speed transfer back to the host.

If the GUI is the operator view and the MCU is the system manager, the FPGA is the part that keeps the real-time signal chain alive.

## What happens when the system starts

The simplest end-to-end startup story is:

1. The GUI sends a 4-byte start flag to the MCU.
2. The GUI sends a binary settings packet framed by `SET` and `END`.
3. The MCU parses the settings and configures the RF/control side.
4. The FPGA runs the high-speed radar processing path.
5. The FPGA streams processed packets to the host through FT601.
6. The host parses those packets and updates the display.

Almost every bug or extension idea in the repository touches one of these steps.

## Why the interfaces are split

A common early question is why the project needs both MCU USB CDC and FT601.

The short answer is:

- MCU USB CDC is good for low-rate control and configuration
- FT601 is used because radar output data is much larger and faster

Once the control-path versus data-path split is clear, the file structure starts making more sense.

## What the FPGA processing is trying to achieve

The FPGA chapters describe several common radar processing goals:

- pulse compression improves range resolution while keeping signal energy
- range-bin reduction keeps later processing manageable
- MTI helps suppress stationary clutter
- Doppler processing helps estimate target velocity
- CFAR helps detect targets under changing noise or clutter conditions

The first goal is not to master all of the mathematics.
It is to understand what engineering problem each block solves.

## Recommended reading order

1. `Quick_Start_System_Guide.md`
2. `System_Architecture_Guide.md`
3. `GUI/GUI_Host_Guide.md`
4. `MCU/MCU_Firmware_Guide.md`
5. `FPGA/FPGA_Architecture_Overview.md`
6. `Protocol_Contracts_and_Known_Gaps.md`
7. `CSV_Reference_Guide.md`

This order moves from system view to implementation view, which is usually easier than starting from code details.

## Main contracts to remember

There are a few cross-module facts worth keeping in mind early:

- the GUI starts MCU configuration with a 4-byte start flag
- the MCU settings packet uses `SET ... END` framing
- the current main operational GUI is `GUI_V6.py`
- the FPGA host packet flow uses FT601 markers such as `0xAA`, `0xBB`, and `0x55`
- the host, MCU, and FPGA must agree on packet layout or the system fails in confusing ways

Many practical failures are not algorithm bugs.
They are interface mismatch bugs.

## Known caution points

The repository is also valuable because it contains real engineering rough edges.
The most important current examples are:

- the GUI/MCU settings packet byte count is not described consistently everywhere
- the MCU startup loop is slightly weaker than a strict "validated settings before proceed" design
- the repository contains both operational tools and engineering support tools, so file purpose must be interpreted carefully

The most concrete example is the settings packet:

- the active field layout is `82` bytes
- some MCU-side comments and guards still refer to `74` bytes
- this should be read as implementation debt around packet-length bookkeeping, not as evidence of two equally valid packet formats

Real systems often contain exactly this kind of partial drift between implementation, assumptions, and documentation.

## Fast debugging checklist

When something looks wrong, a quick first-pass checklist is:

1. Check whether the problem is on the control path or the data path.
2. If it is on the control path, verify the start flag, `SET ... END` framing, and settings field order.
3. If it is on the data path, verify the FT601 markers, packet size, and host parser expectations.
4. If the system appears half-alive, check whether the issue is really a boundary mismatch between GUI, MCU, and FPGA rather than a single broken module.

Good companion references for this step are:

- `Protocol_Field_Reference.md`
- `Protocol_Contracts_and_Known_Gaps.md`
- `system_signal_flow_matrix.csv`

## Good next questions

Once the basic system flow is clear, useful follow-up questions include:

- How should a radar control packet be versioned so GUI and MCU do not drift apart?
- What information belongs in FPGA packets beyond processed data alone?
- Which radar processing stages must stay in FPGA, and which could move to the host?
- How should the design change if the platform grows from research bring-up to a more product-like system?
- What additional calibration, synchronization, or self-test features would be needed for more serious experiments?

These questions help turn the repository from something to read into something to reason with.

## Where to go next

- Read `System_Architecture_Guide.md` for the full end-to-end story.
- Read `GUI/GUI_Host_Guide.md` to understand user interaction and host transport.
- Read `MCU/MCU_Firmware_Guide.md` to understand startup, settings parsing, and control logic.
- Read `FPGA/FPGA_Architecture_Overview.md` and related FPGA chapters to understand the signal path.
- Read `Protocol_Contracts_and_Known_Gaps.md` whenever a cross-module interface needs to be checked.
- Read `CSV_Reference_Guide.md` before using the CSV tables for pin, protocol, or system-level lookup work.

The goal is not only to understand what the repository does today, but also to learn how to reason about radar systems through a working open-source example.
