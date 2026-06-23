# DC Motor Cascade PI Control System

Low-speed DC motor speed control based on STM32F407 and AX309 FPGA.

## Overview

A rotary potentiometer sets the target speed (15–100 RPM). The STM32 reads the potentiometer, encoder, and motor current, runs a cascade PI controller, and sends a 12-bit duty-cycle command to the FPGA over SPI. The FPGA generates a 20 kHz PWM signal that drives the motor through a TB6612FNG H-bridge.

```
Potentiometer → ADC → STM32 (Cascade PI) → SPI → FPGA → PWM → TB6612 → Motor
                              ↑                                          |
                       Encoder + Current ←──────────────────────────────┘
```

## Hardware

| Component | Part |
|-----------|------|
| MCU | STM32F407 (STM32F4-DISC1) |
| FPGA | Xilinx Spartan-6 (AX309) |
| Motor | JGA25-370, 12 V, 280 RPM, with encoder |
| Driver | TB6612FNG |
| Current sense | INA240A2 + 0.1 Ω shunt |
| Display | SSD1306 OLED (I2C) |
| Flash | W25Q64 (SPI) |

## Repository Structure

```
KELI_MOTOR/
├── KELI_MOTOR_REBUILD/   # STM32 Keil project (register-level C)
│   ├── BSP/              # Peripheral drivers: ADC, SPI, I2C, UART, encoder, OLED, Flash, PID
│   ├── Inc/              # Headers including motor_fsm.h
│   └── Src/              # main.c, stm32f4xx_it.c (1 kHz control loop)
└── MOTOR_TEST/           # FPGA VHDL source
    ├── spi_slave.vhd     # SPI slave receiver (SCK-clocked, 2-byte 12-bit frame)
    ├── pwm_gen.vhd       # 12-bit PWM generator (50 MHz / 4096 ≈ 12.2 kHz)
    └── top.vhd           # Top-level wiring
```

## Key Design Points

- **Register-level STM32**: no HAL, all peripherals configured via direct register writes
- **Cascade PI**: outer speed loop (100 Hz) → inner current loop (1 kHz)
- **State machine**: IDLE → STARTING → RUNNING → STOPPING, with startup current limits to prevent overshoot
- **Flash storage**: PI parameters saved to W25Q64 and reloaded on power-on
- **FPGA PWM**: VHDL SPI slave reconstructs 12-bit duty from two SPI bytes; PWM updated each frame

## Build

- STM32: open `KELI_MOTOR_REBUILD/MDK-ARM/KELI_MOTOR_REBUILD.uvprojx` in Keil MDK
- FPGA: synthesise with Xilinx ISE 14.7, target XC6SLX9-2FTG256
