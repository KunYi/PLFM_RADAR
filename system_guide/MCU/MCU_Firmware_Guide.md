# MCU Firmware Guide

## Overview

This guide covers the STM32 F7 firmware in `9_Firmware/9_1_Microcontroller`.
The MCU is the control-plane coordinator of the system: it receives startup settings from the host, performs hardware sequencing, manages RF-support devices, resets and enables the FPGA path, and reports system status.

It does not carry the heavy radar DSP.
Its main value is converting configuration intent into safe, ordered hardware behavior.

## At a glance

- Main runtime file: `main.cpp`
- Main responsibilities: startup sequencing, settings parsing, peripheral control, early status reporting
- Main interfaces: USB CDC from host, GPIO/SPI/I2C/UART to hardware, startup control toward FPGA
- Common source of confusion: the parser field order is clear, but packet-length bookkeeping in source still contains a known inconsistency

## Companion references

This document is the MCU-side runtime and control-sequencing view.
For lookup-heavy tasks, use these companion references alongside it:

- `MCU_Pin_Definitions.csv` for the full MCU pin, peripheral, connected-device, and ownership inventory
- `../protocol_field_reference.csv` for GUI-to-MCU settings fields and related protocol details
- `../system_signal_flow_matrix.csv` for tracing start, settings, status, and control paths through the larger system
- `../device_connection_matrix.csv` for cross-checking which external devices are owned by MCU-controlled peripherals

The markdown chapter explains startup logic, sequencing intent, and firmware behavior.
The CSV tables are better when the task is filtering by interface, device, or protocol field.

## Main file and support files

- `9_1_3_C_Cpp_Code/main.cpp` — primary firmware entrypoint and overall system orchestration
- `9_1_1_C_Cpp_Libraries/USBHandler.cpp` — USB packet state machine for start/settings intake
- `9_1_1_C_Cpp_Libraries/RadarSettings.cpp` — settings parsing and validation
- `9_1_1_C_Cpp_Libraries/ADAR1000_Manager.cpp` / `adar1000.c` — RF front-end management
- `9_1_1_C_Cpp_Libraries/adf4382a_manager.c` / `adf4382.c` — LO frequency synthesis
- `9_1_1_C_Cpp_Libraries/ADS7830.c` — analog sensor and current-sense ADC support

`main.cpp` should be treated as the primary file.
The library files support its startup, parsing, and hardware-control sequence.

## MCU subsystem summary

| Subsystem | Main files | Main interfaces | External devices or paths | Main caution point |
|---|---|---|---|---|
| Host startup and settings intake | `main.cpp`, `USBHandler.cpp`, `RadarSettings.cpp` | USB CDC | GUI settings path | packet field order is stable, but packet-length handling is still inconsistent in source |
| RF front-end control | `ADAR1000_Manager.cpp`, `adar1000.c`, `adf4382a_manager.c`, `adf4382.c` | SPI, GPIO | ADAR1000 array, ADF4382 TX/RX | sequencing and ownership are spread across manager code and startup flow |
| Analog monitoring and PA support | `ADS7830.c`, `DAC5578` driver usage in `main.cpp` | I2C, GPIO | ADS7830 monitors, DAC5578 bias DACs | mixed control and measurement logic increases startup-path complexity |
| Platform sensors and diagnostics | `TinyGPSPlus`, `GY_85_HAL`, `BMP180`, `diag_log` usage in `main.cpp` | UART, I2C, GPIO interrupts, USART3 | GPS, IMU, barometer, debug console | diagnostics are useful but mixed into the same broad runtime file as bring-up logic |
| FPGA and platform startup control | `main.cpp` | GPIO, timer-style sequencing | FPGA reset, TX mixer enable, stepper stage | direct GPIO actions are important to system safety but are still relatively raw in structure |

## What this layer is solving

The MCU solves a different class of problems than the FPGA:

- receiving and validating settings from the host
- making sure hardware powers up in a safe order
- managing devices such as beamformers, synthesizers, sensors, and power-control elements
- providing system-status feedback before the high-speed data path is fully active

This is one of the most important system-design lessons in the repository:
real radar firmware is not only about "starting the radar".
It is also about sequencing, supervision, and recoverable interaction with real hardware.

## Main firmware flow

The firmware boots, initializes clocks and peripherals, and then proceeds through a long hardware bring-up sequence.
That path includes USB, UART, SPI, I2C, GPS, IMU/barometer setup, RF power control, and FPGA reset/enable actions.

In practical terms, the MCU stands between an abstract settings packet and a real mixed-signal system that can be damaged or misconfigured if startup ordering is wrong.

## USB startup handshake

The MCU waits for an explicit host startup sequence before entering the radar control path:

1. The GUI sends the 4-byte start flag `[23, 46, 158, 237]`.
2. `USBHandler` scans incoming CDC data while in `WAITING_FOR_START`.
3. After matching the flag, it switches to `RECEIVING_SETTINGS`.
4. The GUI sends a settings packet framed by `SET` and `END`.
5. `USBHandler` buffers the packet and passes it to `RadarSettings::parseFromUSB()`.
6. If parsing succeeds, the MCU reaches `READY_FOR_DATA`.

This handshake is the main contract between the host GUI and the embedded control path.

## USB CDC receive path

The CDC receive callback in `main.cpp` forwards all received USB data to `USBHandler`:

```c++
void CDC_Receive_FS(uint8_t* Buf, uint32_t *Len) {
    DIAG("USB", "CDC_Receive_FS callback: %lu bytes received", *Len);
    usbHandler.processUSBData(Buf, *Len);
    USBD_CDC_SetRxBuffer(&hUsbDeviceFS, &usb_rx_buffer[0]);
    USBD_CDC_ReceivePacket(&hUsbDeviceFS);
}
```

This makes `USBHandler` the central point for start-flag detection, packet buffering, and settings parsing.

## `USBHandler` state machine

`USBHandler.cpp` uses three main states:

- `WAITING_FOR_START`
- `RECEIVING_SETTINGS`
- `READY_FOR_DATA`

Its job is simple but important:

- scan for the start flag
- buffer settings bytes safely
- detect `SET` and `END`
- reject incomplete or malformed packets

If the settings buffer fills without a valid `END`, the buffer is reset to avoid overflow and accidental misconfiguration.

## Settings packet contract

The current MCU parser expects this field order:

- `"SET"` prefix
- `system_frequency` as big-endian double
- `chirp_duration_1` as big-endian double
- `chirp_duration_2` as big-endian double
- `chirps_per_position` as big-endian uint32
- `freq_min` as big-endian double
- `freq_max` as big-endian double
- `prf1` as big-endian double
- `prf2` as big-endian double
- `max_distance` as big-endian double
- `map_size` as big-endian double
- `"END"` suffix

By literal field count, this layout is 82 bytes:

- `SET` = 3 bytes
- 9 doubles = 72 bytes
- 1 uint32 = 4 bytes
- `END` = 3 bytes

That field order matches the current `GUI_V6.py` packet builder and should be treated as the active baseline.

## Parser behavior and current caveat

`RadarSettings::parseFromUSB()` extracts fields in the order listed above and validates their ranges before accepting them.
That makes the packet structure understandable and traceable.

However, the source still contains a real inconsistency:

- the actual field layout is 82 bytes
- MCU-side comments still mention 74 bytes
- the minimum-length guard also checks against 74 bytes

So the parser field order is trustworthy, but the MCU-side byte-count bookkeeping should still be treated as needing cleanup.

## Settings packet length mismatch summary

| Item | Current value | Where it appears | What to trust today |
|---|---|---|---|
| Intended field layout | `82` bytes | GUI builders and parser field order | trust this as the active packet shape |
| Legacy source comment | `74` bytes | MCU-side parser comments | treat this as stale documentation |
| Current minimum-length guard | `74` bytes | MCU-side guard and buffering threshold | treat this as implementation debt, not the canonical contract |

Practical reading rule:

- if the question is "what packet does the system mean to exchange?", use the `82`-byte field layout
- if the question is "why might code behavior still look inconsistent?", check for the older `74`-byte assumptions in MCU-side comments and guards

## Main loop and run-state nuance

`main.cpp` contains a blocking loop that waits for host startup before progressing into radar bring-up.
The important detail is:

- the loop definitely waits for the start flag
- it also checks `READY_FOR_DATA` inside the loop body
- but the final `do ... while` exit condition is based on `isStartFlagReceived()` rather than strictly on `READY_FOR_DATA`

That means the current behavior strongly prefers a complete settings packet before proceeding, but it is slightly weaker than a strict "validated settings required before continue" contract.

## Power and hardware sequencing

The MCU also performs dedicated power-up and power-down actions for the RF front end.

- `systemPowerUpSequence()` configures ADTR1107 power behavior, initializes ADAR1000 devices, calibrates the system, and moves the chain into TX-related operating states.
- `systemPowerDownSequence()` returns devices to safer states, lowers PA bias, and disables supplies as needed.

This is one of the clearest examples in the repository of why embedded control logic matters in radar systems.
The sequence is not just "turn things on".
It is "turn things on in the right order, with the right checks, and with a recoverable path back down".

## FPGA reset and RF enable

After peripheral configuration, the MCU resets the FPGA and enables TX mixers through GPIO:

```c++
HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
HAL_Delay(10);
HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);
HAL_GPIO_WritePin(GPIOD, GPIO_PIN_11, GPIO_PIN_SET);
```

This sequencing matters because the FPGA and RF path are not independent.
The MCU is responsible for deciding when the hardware is allowed to enter an active state.

## System status reporting

The MCU sends initial status back to the host over USB CDC:

```c++
getSystemStatusForGUI(initial_status, sizeof(initial_status));
CDC_Transmit_FS((uint8_t*)initial_status, strlen(initial_status));
```

That status path provides early visibility into GPS, sensor, and radar-subsystem readiness.
It is especially useful because the high-speed FT601 data path is not the only signal of system health.

## Sensor and support subsystems

The firmware also integrates:

- GPS via `TinyGPSPlus`
- IMU via `GY_85_HAL`
- barometer via `BMP180`
- stepper motor control for azimuth positioning
- RF power amplifier control with `DAC5578` and current sensing via `ADS7830`

These subsystems make it clear that the MCU is handling more than packet parsing.
It is also the repository's main bridge between radar electronics and platform-level state.

## Why this layer matters

- It shows how host commands become safe hardware actions.
- It highlights the difference between "configuration arrived" and "the system is now safe to operate".
- It demonstrates that many radar-platform issues are really sequencing and supervisory-control issues rather than signal-processing issues.

## Known caveats

- The parser field order is clear, but the source still contains an inconsistent `82` versus `74` byte-count assumption.
- The startup loop is not as strict as the higher-level architectural intent may suggest.
- `main.cpp` combines bring-up logic, control logic, and system integration concerns, which makes it useful but dense.

## Questions for deeper study

- Should the MCU remain mainly a control-plane device, or should it grow into a stronger supervisory processor?
- Which failures should stop startup immediately, and which should only degrade capability?
- How should settings packets be versioned as the radar evolves?
- How should sensor metadata be attached to radar frames for later offline study or review?

## Notes for future expansion

- Inspect `main.cpp` further to separate steady-state operation from bring-up behavior.
- Trace `USBHandler` through `usbd_cdc_if.c` and `usb_device.c` to complete the USB control-plane picture.
- Compare MCU status reporting with the FT601-side status path to clarify subsystem ownership.
