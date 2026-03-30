# FPGA Pin Reference

## Overview

Sources:
- `9_Firmware/9_2_FPGA/constraints/xc7a200t_fbg484.xdc`
- `9_Firmware/9_2_FPGA/constraints/te0713_te0701_umft601x.xdc`
- `9_Firmware/9_2_FPGA/constraints/te0712_te0701_minimal.xdc`
- `9_Firmware/9_2_FPGA/constraints/README.md`

This file summarizes FPGA I/O mappings used in both production and development flows.
Unlike the algorithm chapters, it is mainly a tracing aid: its purpose is to connect RTL port names to physical pins and target boards.

## At a glance

- Main purpose: map RTL ports to physical FPGA pins and board contexts
- Best for: tracing ADC, FT601, STM32 bridge, and DAC-related interfaces to package pins
- Best companion files: `FPGA_Pin_Definitions.csv`, `../device_connection_matrix.csv`
- Common source of confusion: production mappings, dev-board mappings, and minimal bring-up mappings live side by side in the same broader FPGA tree

## Companion references

This markdown file gives the grouped FPGA I/O view.
For filtering and cross-checking in spreadsheet tools, use these companion tables:

- `FPGA_Pin_Definitions.csv` for the full FPGA pin inventory grouped by interface and board context
- `../device_connection_matrix.csv` for seeing how FPGA-facing interfaces relate to system devices and ownership
- `../system_signal_flow_matrix.csv` for tracing where FPGA-side signals enter or leave the larger data path

The markdown view is better for understanding interface groups.
The CSV tables are better for sorting by signal name, package pin, board context, or subsystem role.

## How this file is organized

- production mappings come first because they are the main hardware reference
- FT601 development mappings are separated because they belong to a bring-up path rather than the main production map
- the notes at the end explain how to use the tables without treating them as a replacement for the constraint files

## Production board (XC7A200T FBG484)

### System clock and reset

| Port | Package Pin | Function |
|---|---|---|
| `clk_100m` | `J19` | 100 MHz system clock input |
| `clk_120m_dac` | `K18` | 120 MHz DAC clock output |
| `adc_dco_p` | `W19` | ADC 400 MHz LVDS clock positive |
| `adc_dco_n` | `W20` | ADC 400 MHz LVDS clock negative |
| `reset_n` | `J16` | FPGA active-low reset |

### ADC interface (AD9484)

| Port | Package Pin | Function |
|---|---|---|
| `adc_d_p[0]` | `P22` | ADC data bit 0 positive |
| `adc_d_n[0]` | `R22` | ADC data bit 0 negative |
| `adc_d_p[1]` | `P21` | ADC data bit 1 positive |
| `adc_d_n[1]` | `R21` | ADC data bit 1 negative |
| `adc_d_p[2]` | `U22` | ADC data bit 2 positive |
| `adc_d_n[2]` | `V22` | ADC data bit 2 negative |
| `adc_d_p[3]` | `T21` | ADC data bit 3 positive |
| `adc_d_n[3]` | `U21` | ADC data bit 3 negative |
| `adc_d_p[4]` | `P19` | ADC data bit 4 positive |
| `adc_d_n[4]` | `R19` | ADC data bit 4 negative |
| `adc_d_p[5]` | `W21` | ADC data bit 5 positive |
| `adc_d_n[5]` | `W22` | ADC data bit 5 negative |
| `adc_d_p[6]` | `AA20` | ADC data bit 6 positive |
| `adc_d_n[6]` | `AA21` | ADC data bit 6 negative |
| `adc_d_p[7]` | `Y21` | ADC data bit 7 positive |
| `adc_d_n[7]` | `Y22` | ADC data bit 7 negative |
| `adc_pwdn` | `P20` | ADC power-down control |

### DAC interface

| Port | Package Pin | Function |
|---|---|---|
| `dac_data[0]` | `H13` | DAC data bit 0 |
| `dac_data[1]` | `G13` | DAC data bit 1 |
| `dac_data[2]` | `G15` | DAC data bit 2 |
| `dac_data[3]` | `G16` | DAC data bit 3 |
| `dac_data[4]` | `G17` | DAC data bit 4 |
| `dac_data[5]` | `G18` | DAC data bit 5 |
| `dac_data[6]` | `J15` | DAC data bit 6 |
| `dac_data[7]` | `H15` | DAC data bit 7 |
| `dac_clk` | `H17` | DAC clock output |
| `dac_sleep` | `H18` | DAC sleep control |

### RF and mixer control

| Port | Package Pin | Function |
|---|---|---|
| `fpga_rf_switch` | `J22` | RF switch control |
| `rx_mixer_en` | `H22` | RX mixer enable |
| `tx_mixer_en` | `H20` | TX mixer enable |

### STM32 / SPI interface signals

| Port | Package Pin | Function |
|---|---|---|
| `stm32_sclk_3v3` | `K21` | STM32 3.3V SPI clock |
| `stm32_mosi_3v3` | `K22` | STM32 3.3V SPI MOSI |
| `stm32_miso_3v3` | `M21` | STM32 3.3V SPI MISO |
| `stm32_cs_adar1_3v3` | `L21` | STM32 CS for ADAR1 3.3V |
| `stm32_cs_adar2_3v3` | `N22` | STM32 CS for ADAR2 3.3V |
| `stm32_cs_adar3_3v3` | `M22` | STM32 CS for ADAR3 3.3V |
| `stm32_cs_adar4_3v3` | `M18` | STM32 CS for ADAR4 3.3V |
| `stm32_sclk_1v8` | `Y3` | STM32 1.8V SPI clock |
| `stm32_mosi_1v8` | `AA3` | STM32 1.8V SPI MOSI |
| `stm32_miso_1v8` | `AA5` | STM32 1.8V SPI MISO |
| `stm32_cs_adar1_1v8` | `AB5` | STM32 CS for ADAR1 1.8V |
| `stm32_cs_adar2_1v8` | `W6` | STM32 CS for ADAR2 1.8V |
| `stm32_cs_adar3_1v8` | `W5` | STM32 CS for ADAR3 1.8V |
| `stm32_cs_adar4_1v8` | `U6` | STM32 CS for ADAR4 1.8V |

### Chirp and beam control signals

| Port | Package Pin | Function |
|---|---|---|
| `stm32_new_chirp` | `L18` | New chirp trigger signal from STM32 |
| `stm32_new_elevation` | `N18` | Beam elevation command |
| `stm32_new_azimuth` | `N19` | Beam azimuth command |
| `stm32_mixers_enable` | `N20` | Enables mixer chains |

### ADAR1000 beamformer load / transmit control

| Port | Package Pin | Function |
|---|---|---|
| `adar_tx_load_1` | `T1` | ADAR TX load line 1 |
| `adar_tx_load_2` | `U1` | ADAR TX load line 2 |
| `adar_tx_load_3` | `U2` | ADAR TX load line 3 |
| `adar_tx_load_4` | `V2` | ADAR TX load line 4 |
| `adar_rx_load_1` | `W2` | ADAR RX load line 1 |
| `adar_rx_load_2` | `Y2` | ADAR RX load line 2 |
| `adar_rx_load_3` | `W1` | ADAR RX load line 3 |
| `adar_rx_load_4` | `Y1` | ADAR RX load line 4 |
| `adar_tr_1` | `AA1` | ADAR TX/RX enable 1 |
| `adar_tr_2` | `AB1` | ADAR TX/RX enable 2 |
| `adar_tr_3` | `AB3` | ADAR TX/RX enable 3 |
| `adar_tr_4` | `AB2` | ADAR TX/RX enable 4 |

## FT601 USB interface (TE0713 dev + UMFT601X)

This mapping is used by the Trenz TE0713 / UMFT601X development flow.

### FT601 data and control bus

| Port | Package Pin | Function |
|---|---|---|
| `ft601_clk_in` | `J20` | FT601 reference clock input |
| `ft601_data[0]` | `L21` | USB FIFO data bit 0 |
| `ft601_data[1]` | `N20` | USB FIFO data bit 1 |
| `ft601_data[2]` | `M21` | USB FIFO data bit 2 |
| `ft601_data[3]` | `M20` | USB FIFO data bit 3 |
| `ft601_data[4]` | `M13` | USB FIFO data bit 4 |
| `ft601_data[5]` | `N22` | USB FIFO data bit 5 |
| `ft601_data[6]` | `L13` | USB FIFO data bit 6 |
| `ft601_data[7]` | `M22` | USB FIFO data bit 7 |
| `ft601_data[8]` | `H18` | USB FIFO data bit 8 |
| `ft601_data[9]` | `K17` | USB FIFO data bit 9 |
| `ft601_data[10]` | `H17` | USB FIFO data bit 10 |
| `ft601_data[11]` | `J17` | USB FIFO data bit 11 |
| `ft601_data[12]` | `L15` | USB FIFO data bit 12 |
| `ft601_data[13]` | `J15` | USB FIFO data bit 13 |
| `ft601_data[14]` | `L14` | USB FIFO data bit 14 |
| `ft601_data[15]` | `H15` | USB FIFO data bit 15 |
| `ft601_data[16]` | `G13` | USB FIFO data bit 16 |
| `ft601_data[17]` | `G15` | USB FIFO data bit 17 |
| `ft601_data[18]` | `H13` | USB FIFO data bit 18 |
| `ft601_data[19]` | `G16` | USB FIFO data bit 19 |
| `ft601_data[20]` | `G18` | USB FIFO data bit 20 |
| `ft601_data[21]` | `L18` | USB FIFO data bit 21 |
| `ft601_data[22]` | `G17` | USB FIFO data bit 22 |
| `ft601_data[23]` | `M18` | USB FIFO data bit 23 |
| `ft601_data[24]` | `H14` | USB FIFO data bit 24 |
| `ft601_data[25]` | `J14` | USB FIFO data bit 25 |
| `ft601_data[26]` | `N18` | USB FIFO data bit 26 |
| `ft601_data[27]` | `N19` | USB FIFO data bit 27 |
| `ft601_data[28]` | `K14` | USB FIFO data bit 28 |
| `ft601_data[29]` | `K13` | USB FIFO data bit 29 |
| `ft601_data[30]` | `H19` | USB FIFO data bit 30 |
| `ft601_data[31]` | `J19` | USB FIFO data bit 31 |
| `ft601_be[0]` | `B20` | Byte enable 0 |
| `ft601_be[1]` | `A20` | Byte enable 1 |
| `ft601_be[2]` | `D16` | Byte enable 2 |
| `ft601_be[3]` | `E16` | Byte enable 3 |
| `ft601_oe_n` | `C17` | Output enable (active low) |
| `ft601_rd_n` | `D17` | Read enable (active low) |
| `ft601_wr_n` | `E13` | Write enable (active low) |
| `ft601_siwu_n` | `E14` | Send immediate / wake-up (active low) |
| `ft601_txe` | `E17` | Transmit enable |
| `ft601_rxf` | `F16` | Receive full indicator |
| `ft601_chip_reset_n` | `A14` | FT601 reset (active low) |
| `ft601_wakeup_n` | `A13` | FT601 wake-up (active low) |
| `ft601_gpio0` | `A18` | FT601 GPIO 0 (heartbeat/monitor) |
| `ft601_gpio1` | `A19` | FT601 GPIO 1 |

## Development target notes

- `te0712_te0701_minimal.xdc` maps a minimal bring-up interface for TE0712/TE0701 development with only `clk_100m`, `reset_n`, `user_led[]`, and `system_status[]`.
- For production-level pin checking, use `xc7a200t_fbg484.xdc` as the canonical source.
- For FT601-based USB data capture, use `te0713_te0701_umft601x.xdc`.

## How to use this file

1. Start from the functional group you are analyzing: ADC, DAC, FT601, or STM32 interface.
2. Cross-check the port name with the top-level Verilog module and the relevant constraint file.
3. Use the package pin to trace the signal into the board schematic or package view if needed.

## Why this matters

A common difficulty is understanding the RTL while still not being able to answer questions like:

- Which physical pin carries this signal?
- Why does the same logical port move between development targets?
- Which interfaces belong to production hardware and which belong only to bring-up boards?

This file connects code-level signal names, target-board differences, and the physical I/O view in one place.

## Future questions to explore

- Which signals are truly essential on a minimal bring-up board?
- Which debug signals deserve dedicated pins versus internal logic analyzers?
- How should the pin plan change if bandwidth, debug visibility, or board complexity increases?

> Note: This document is a tracing aid, not the authoritative hardware schematic. The constraint files remain the ground truth for exact FPGA pin assignments.
