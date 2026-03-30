# MCU Pin Reference

## Overview

Sources:
- `9_Firmware/9_1_Microcontroller/9_1_3_C_Cpp_Code/main.h`
- `9_Firmware/9_1_Microcontroller/9_1_1_C_Cpp_Libraries/stm32f7xx_hal_msp.c`

This file summarizes the STM32 MCU pin definitions visible through the generated `main.h` mapping, the active peripheral alternate-function mappings configured in `stm32f7xx_hal_msp.c`, and a few important direct GPIO uses observed in `main.cpp`.

Its purpose is to connect firmware identifiers to real MCU pins, peripheral instances, and board functions.

## At a glance

- Main purpose: map MCU pins and peripherals to external radar hardware and board functions
- Best for: tracing ownership from GPIO or peripheral name to connected device
- Best companion files: `MCU_Pin_Definitions.csv`, `../device_connection_matrix.csv`
- Common source of confusion: some important startup GPIO actions appear in `main.cpp` directly rather than only through `main.h` macros

## Companion references

This markdown file explains the MCU-side pin and peripheral map in grouped form.
For spreadsheet-style lookup and filtering, use these companion tables:

- `MCU_Pin_Definitions.csv` for the full MCU pin inventory with peripheral, connected-device, subsystem, and ownership columns
- `../device_connection_matrix.csv` for cross-subsystem device ownership and external-hardware mapping
- `../system_signal_flow_matrix.csv` for following MCU-controlled messages and signals through the larger system

The markdown version is better for reading the grouped hardware story.
The CSV versions are better for sorting by peripheral, device, subsystem, or firmware owner.

## Power and board control

| Pin Macro | GPIO | Pin | Function |
|---|---|---|---|
| `EN_P_5V0_PA1_Pin` | `GPIOG` | `GPIO_PIN_0` | 5V PA rail enable 1 |
| `EN_P_5V0_PA2_Pin` | `GPIOG` | `GPIO_PIN_1` | 5V PA rail enable 2 |
| `EN_P_5V0_PA3_Pin` | `GPIOG` | `GPIO_PIN_2` | 5V PA rail enable 3 |
| `EN_P_5V5_PA_Pin` | `GPIOG` | `GPIO_PIN_3` | 5.5V PA rail enable |
| `EN_P_1V0_FPGA_Pin` | `GPIOE` | `GPIO_PIN_7` | FPGA 1.0V rail enable |
| `EN_P_1V8_FPGA_Pin` | `GPIOE` | `GPIO_PIN_8` | FPGA 1.8V rail enable |
| `EN_P_3V3_FPGA_Pin` | `GPIOE` | `GPIO_PIN_9` | FPGA 3.3V rail enable |
| `EN_P_5V0_ADAR_Pin` | `GPIOE` | `GPIO_PIN_10` | ADAR 5V rail enable |
| `EN_P_3V3_ADAR12_Pin` | `GPIOE` | `GPIO_PIN_11` | ADAR1/2 3.3V rail enable |
| `EN_P_3V3_ADAR34_Pin` | `GPIOE` | `GPIO_PIN_12` | ADAR3/4 3.3V rail enable |
| `EN_P_3V3_ADTR_Pin` | `GPIOE` | `GPIO_PIN_13` | ADTR 3.3V rail enable |
| `EN_P_3V3_SW_Pin` | `GPIOE` | `GPIO_PIN_14` | RF switch 3.3V rail enable |
| `EN_P_3V3_VDD_SW_Pin` | `GPIOE` | `GPIO_PIN_15` | switch VDD rail enable |
| `EN_P_1V8_CLOCK_Pin` | `GPIOG` | `GPIO_PIN_4` | clock-tree 1.8V rail enable |
| `EN_P_3V3_CLOCK_Pin` | `GPIOG` | `GPIO_PIN_5` | clock-tree 3.3V rail enable |
| `EN_DIS_RFPA_VDD_Pin` | `GPIOD` | `GPIO_PIN_6` | RF PA supply enable/disable |
| `EN_DIS_COOLING_Pin` | `GPIOD` | `GPIO_PIN_7` | cooling enable/disable |

## RF synthesizer and clock control

### AD9523 clock distribution control

| Pin Macro | GPIO | Pin | Function |
|---|---|---|---|
| `AD9523_PD_Pin` | `GPIOF` | `GPIO_PIN_3` | AD9523 power-down |
| `AD9523_REF_SEL_Pin` | `GPIOF` | `GPIO_PIN_4` | AD9523 reference select |
| `AD9523_SYNC_Pin` | `GPIOF` | `GPIO_PIN_5` | AD9523 sync |
| `AD9523_RESET_Pin` | `GPIOF` | `GPIO_PIN_6` | AD9523 reset |
| `AD9523_CS_Pin` | `GPIOF` | `GPIO_PIN_7` | AD9523 chip select |
| `AD9523_STATUS0_Pin` | `GPIOF` | `GPIO_PIN_8` | AD9523 status 0 |
| `AD9523_STATUS1_Pin` | `GPIOF` | `GPIO_PIN_9` | AD9523 status 1 |
| `AD9523_EEPROM_SEL_Pin` | `GPIOF` | `GPIO_PIN_10` | AD9523 EEPROM select |

### ADF4382 synthesizer control and monitor

| Pin Macro | GPIO | Pin | Function |
|---|---|---|---|
| `ADF4382_RX_LKDET_Pin` | `GPIOG` | `GPIO_PIN_6` | RX PLL lock detect |
| `ADF4382_RX_DELADJ_Pin` | `GPIOG` | `GPIO_PIN_7` | RX delay adjust |
| `ADF4382_RX_DELSTR_Pin` | `GPIOG` | `GPIO_PIN_8` | RX delay strobe |
| `ADF4382_RX_CE_Pin` | `GPIOG` | `GPIO_PIN_9` | RX synthesizer chip enable |
| `ADF4382_RX_CS_Pin` | `GPIOG` | `GPIO_PIN_10` | RX synthesizer chip select |
| `ADF4382_TX_LKDET_Pin` | `GPIOG` | `GPIO_PIN_11` | TX PLL lock detect |
| `ADF4382_TX_DELSTR_Pin` | `GPIOG` | `GPIO_PIN_12` | TX delay strobe |
| `ADF4382_TX_DELADJ_Pin` | `GPIOG` | `GPIO_PIN_13` | TX delay adjust |
| `ADF4382_TX_CS_Pin` | `GPIOG` | `GPIO_PIN_14` | TX synthesizer chip select |
| `ADF4382_TX_CE_Pin` | `GPIOG` | `GPIO_PIN_15` | TX synthesizer chip enable |

## ADAR1000 beamformer chip select signals

| Pin Macro | GPIO | Pin | Function |
|---|---|---|---|
| `ADAR_1_CS_3V3_Pin` | `GPIOA` | `GPIO_PIN_0` | Chip select for ADAR1000 #1 |
| `ADAR_2_CS_3V3_Pin` | `GPIOA` | `GPIO_PIN_1` | Chip select for ADAR1000 #2 |
| `ADAR_3_CS_3V3_Pin` | `GPIOA` | `GPIO_PIN_2` | Chip select for ADAR1000 #3 |
| `ADAR_4_CS_3V3_Pin` | `GPIOA` | `GPIO_PIN_3` | Chip select for ADAR1000 #4 |

## Peripheral interface inventory

This section keeps communication and mixed-signal interfaces explicit instead of hiding them inside broader GPIO groups.

### SPI interfaces

| Peripheral | MCU Pin | AF | Role |
|---|---|---|---|
| `SPI1` | `PA5` | `GPIO_AF5_SPI1` | SPI1 SCK |
| `SPI1` | `PA6` | `GPIO_AF5_SPI1` | SPI1 MISO |
| `SPI1` | `PA7` | `GPIO_AF5_SPI1` | SPI1 MOSI |
| `SPI4` | `PE2` | `GPIO_AF5_SPI4` | SPI4 SCK |
| `SPI4` | `PE5` | `GPIO_AF5_SPI4` | SPI4 MISO |
| `SPI4` | `PE6` | `GPIO_AF5_SPI4` | SPI4 MOSI |

### I2C interfaces

| Peripheral | MCU Pin | AF | Role |
|---|---|---|---|
| `I2C1` | `PB6` | `GPIO_AF4_I2C1` | I2C1 SCL |
| `I2C1` | `PB7` | `GPIO_AF4_I2C1` | I2C1 SDA |
| `I2C2` | `PF0` | `GPIO_AF4_I2C2` | I2C2 SDA |
| `I2C2` | `PF1` | `GPIO_AF4_I2C2` | I2C2 SCL |
| `I2C3` | `PA8` | `GPIO_AF4_I2C3` | I2C3 SCL |
| `I2C3` | `PC9` | `GPIO_AF4_I2C3` | I2C3 SDA |

### UART / USART interfaces

| Peripheral | MCU Pin | AF | Role |
|---|---|---|---|
| `UART5` | `PC12` | `GPIO_AF8_UART5` | UART5 TX |
| `UART5` | `PD2` | `GPIO_AF8_UART5` | UART5 RX |
| `USART3` | `PB10` | `GPIO_AF7_USART3` | USART3 TX |
| `USART3` | `PB11` | `GPIO_AF7_USART3` | USART3 RX |

### Timer support

| Peripheral | Note |
|---|---|
| `TIM1` | Base-timer clock is explicitly initialized in `HAL_TIM_Base_MspInit()` |

### DAC and analog-support signals

| Signal | GPIO | Pin | Function |
|---|---|---|---|
| `DAC_1_VG_CLR_Pin` | `GPIOB` | `GPIO_PIN_4` | DAC1 clear |
| `DAC_1_VG_LDAC_Pin` | `GPIOB` | `GPIO_PIN_5` | DAC1 LDAC |
| `DAC_2_VG_CLR_Pin` | `GPIOB` | `GPIO_PIN_8` | DAC2 clear |
| `DAC_2_VG_LDAC_Pin` | `GPIOB` | `GPIO_PIN_9` | DAC2 LDAC |

## Peripheral-to-device mapping

This section ties MCU peripheral instances to the external devices and subsystem roles visible in `main.cpp` and the driver layer.

| Peripheral / handle | Connected device(s) or path | Evidence in firmware | System role |
|---|---|---|---|
| `SPI1` / `hspi1` | General SPI master path for board RF/control devices | `MX_SPI1_Init()` in `main.cpp` | General-purpose SPI path kept available for RF/control integration |
| `SPI4` / `hspi4` | `ADF4382A_Manager` and low-level ADF4382 path | `init_param.spi_init.extra = &hspi4`, `ADF4382A_SPI_DEVICE_ID 4` | TX/RX LO synthesizer configuration and control |
| GPIO chip selects `ADAR_1..4_CS_3V3` | Four `ADAR1000` beamformer ICs | `ADAR1000Manager adarManager`, beamformer init/use in `main.cpp` | Beam steering, TX/RX phase/amplitude control |
| `I2C1` / `hi2c1` | `DAC5578` devices at `0x48` and `0x49` | `DAC5578_Init(&hdac1, &hi2c1, 0x48, ...)`, `DAC5578_Init(&hdac2, &hi2c1, 0x49, ...)` | RF PA gate-bias DAC control |
| `I2C2` / `hi2c2` | `ADS7830` devices at `0x48`, `0x4A`, and `0x49` | `ADS7830_Init(&hadc1, &hi2c2, 0x48, ...)`, `&hadc2, 0x4A`, `&hadc3, 0x49` | Analog monitoring and temperature/current-related sensing |
| `I2C3` / `hi2c3` | Platform sensors and support devices on the third I2C bus | `MX_I2C3_Init()` plus sensor stack in `main.cpp` | Reserved/available sensor and support-device bus, useful to keep explicit in the inventory |
| `UART5` / `huart5` | GPS receiver / NMEA data path | `HAL_UART_Receive(&huart5, ...)`, `TinyGPSPlus gps` | GPS input and location/time parsing |
| `USART3` / `huart3` | Diagnostic and status text output | many `HAL_UART_Transmit(&huart3, ...)`, `diag_log.h` notes USART3 logging | Bring-up logs, status, and debug telemetry |
| `TIM1` | Timed control/synchronization support | `HAL_TIM_Base_MspInit()` | Base-timer support for timing-sensitive control |

Where the device mapping is directly visible in source, it is stated plainly.
Where the ownership is broader or more platform-level, the description is intentionally conservative.

## On-board status LEDs

| Pin Macro | GPIO | Pin | Function |
|---|---|---|---|
| `LED_1_Pin` | `GPIOF` | `GPIO_PIN_12` | Status LED 1 |
| `LED_2_Pin` | `GPIOF` | `GPIO_PIN_13` | Status LED 2 |
| `LED_3_Pin` | `GPIOF` | `GPIO_PIN_14` | Status LED 3 |
| `LED_4_Pin` | `GPIOF` | `GPIO_PIN_15` | Status LED 4 |

## RF frontend control and monitor

| Pin Macro | GPIO | Pin | Function |
|---|---|---|---|
| `MAG_DRDY_Pin` | `GPIOC` | `GPIO_PIN_6` | Magnetometer data ready |
| `ACC_INT_Pin` | `GPIOC` | `GPIO_PIN_7` | Accelerometer interrupt |
| `GYR_INT_Pin` | `GPIOC` | `GPIO_PIN_8` | Gyroscope interrupt |

Important direct `main.cpp` usage not defined as named macros:

| GPIO | Pin | Function in firmware |
|---|---|---|
| `GPIOD` | `GPIO_PIN_12` | FPGA reset control |
| `GPIOD` | `GPIO_PIN_11` | TX mixer enable during startup |

## Motion, thermal, and platform control

| Pin Macro | GPIO | Pin | Function |
|---|---|---|---|
| `STEPPER_CW_P_Pin` | `GPIOD` | `GPIO_PIN_4` | Stepper direction / CW control |
| `STEPPER_CLK_P_Pin` | `GPIOD` | `GPIO_PIN_5` | Stepper pulse / clock |

## Notes

- `main.h` is the main source for symbolic GPIO naming.
- `stm32f7xx_hal_msp.c` is the main source for active peripheral alternate-function mapping.
- Some firmware-relevant GPIO uses appear directly in `main.cpp` without a matching macro in `main.h`; those are included here when they affect system understanding.
- This file now keeps SPI, I2C, UART/USART, timer, and analog-support interfaces explicit so the peripheral inventory remains complete rather than overly compressed.
- This file also keeps a peripheral-to-device view so the reader can move from MCU resources to external radar hardware more directly.

## Why this matters

One recurring difficulty in embedded-system work is turning code like `HAL_GPIO_WritePin(...)`, `HAL_SPI_Init(...)`, or `HAL_UART_Init(...)` into an actual picture of the hardware.

This file helps answer questions such as:

- Which pin actually controls the FPGA reset?
- Which communication peripherals are active in the HAL MSP layer?
- Which peripheral instance talks to which external device?
- Which rails and RF paths are MCU-controlled?
- Which interfaces are dedicated and which are multiplexed through alternate functions?

It is therefore a useful bridge between firmware flow and board-level behavior.

## Future questions to explore

- Which pins are safety-critical and should have stronger software-state tracking?
- Which GPIO actions would benefit from a named abstraction instead of direct pin writes?
- Should some direct `main.cpp` GPIO uses be pulled back into `main.h` macros for clarity and maintainability?
- Should the system guide later add a separate table that maps each peripheral instance to the firmware module that owns it?
