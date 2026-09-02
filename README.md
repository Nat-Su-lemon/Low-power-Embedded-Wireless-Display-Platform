# Low-Power Embedded Wireless Display Platform

A low-power embedded system for driving an E-paper display, environmental sensors/dlogging, built around an energy-efficient STM32U5 microcontroller and ESP32 IoT.

## Hardware

<p align="center">
  <img src="https://raw.githubusercontent.com/Nat-Su-lemon/Low-power-Embedded-Wireless-Display-Platform/main/assets/Screenshot%202026-09-02%20015859.png" height="250">
  <img src="https://raw.githubusercontent.com/Nat-Su-lemon/Low-power-Embedded-Wireless-Display-Platform/main/assets/Screenshot%202026-09-02%20015914.png" height="250">
</p>

## Schematics

<p align="center">
  <img src="https://raw.githubusercontent.com/Nat-Su-lemon/Low-power-Embedded-Wireless-Display-Platform/main/assets/MAIN.png" width="100%">
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/Nat-Su-lemon/Low-power-Embedded-Wireless-Display-Platform/main/assets/PERIPHERALS.png" width="100%">
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/Nat-Su-lemon/Low-power-Embedded-Wireless-Display-Platform/main/assets/POWER.png" width="100%">
</p>

📄 **[Download the full schematic PDF](https://github.com/Nat-Su-lemon/Low-power-Embedded-Wireless-Display-Platform/blob/main/assets/DINO.pdf)**
# Firmware Overview

This firmware is an embedded test platform for a low-power STM32U585-based device with environmental sensing, battery monitoring, e-paper display output, SD card logging, ESP32 WiFi coordination, USB serial debug access, and firmware update support through the STM32 ROM DFU bootloader.

The project is written in C using STM32CubeIDE/HAL with a cooperative bare-metal architecture. The main loop services each subsystem in sequence, while interrupts are used for UART byte handling, GPIO wake events, SPI/DMA completion, RTC alarms, and USB events.

## System Architecture

The codebase is organized into clear firmware layers:

```text
Core/
  STM32CubeMX-generated startup, clocks, GPIO, peripherals, interrupts, and main loop.

Bsp/
  Board support wrappers for UART, SPI, I2C, GPIO interrupts, and ROM bootloader entry.

Common/
  Shared configuration, debug command parsing, and ring-buffer utilities.

Control/
  Application managers for sensors, dashboard rendering, sleep, WiFi, logging, GPIO buttons, and ESP flash mode.

Modules/
  Device drivers and middleware: TinyUSB, BME680, MAX17048, OPT4001, e-paper display, and FatFs/SD.

Drivers/
  STM32 HAL and CMSIS vendor libraries.
```

The runtime flow is cooperative:

```text
USB task
UART task
debug/ESP command processing
GPIO button handling
WiFi state machine
sensor polling
dashboard rendering
SD logging
sleep decision
```

This structure keeps hardware-specific code separated from application logic while still being lightweight enough for a bare-metal embedded target.

## Major Firmware Features

### Sensor and Dashboard Pipeline

The firmware samples multiple device subsystems and publishes the results into a shared `dashboard_data_t` model:

- BME680 for temperature, humidity, and pressure.
- OPT4001 for ambient light.
- MAX17048 for battery voltage and state of charge.
- RTC for date/time.
- Charger status pins for charging state.
- ESP32 communication state for WiFi provisioning/status.
- USB connection state.

Each field has an associated ready/stale flag, allowing the dashboard to render partial data without treating a failed sensor read as a full-system failure. The e-paper dashboard is updated only when new data is available or a full refresh is explicitly requested.

### Low-Power Operation

The sleep manager controls STOP2 entry using RTC alarms and GPIO wake sources. Before entering STOP2, the firmware deinitializes sensors and shuts down USB so the device can reduce power consumption and avoid USB backpowering issues.

USB activity blocks sleep while enabled, because USB full-speed operation requires the 48 MHz clock domain and active peripheral state. When USB is disconnected, the firmware avoids unnecessary USB polling or long probe delays so normal refresh timing is not affected.

### UART and Debug Architecture

The UART BSP abstracts both physical UARTs and the USB virtual COM port behind the same `uart_dev_t` style interface.

Current UART-like devices:

```text
debug_dev  - physical debug UART
main_dev   - ESP32 communication UART
vcp_dev    - TinyUSB CDC virtual COM port
```

Debug output can be routed to:

```text
physical UART only
USB virtual COM port only
both UART and USB
```

The same debug command parser works over both physical UART and USB VCP. This includes commands for hardware debugging and entering DFU mode.

### ESP32 WiFi Coordination

The WiFi manager controls a simple text protocol between the STM32 and ESP32:

```text
STM: PING
STM: GET WIFI
STM: ACQUIRED

ESP: ACKNOWLEDGED
ESP: IP: ...
ESP: FOUND: ...
```

The WiFi state machine is paced using elapsed `HAL_GetTick()` timing instead of blocking delays. PING traffic is rate-limited separately from the connection timeout, which prevents UART TX flooding while preserving the same timeout behavior.

ESP RX handling was also adjusted so the STM32 does not echo received ESP bytes back to the ESP32, preventing protocol noise on the main UART.

### SD Card Logging

The logging manager uses FatFs to maintain a CSV log on the SD card. It validates or recreates the log header at startup, appends dashboard samples after a display refresh, and can restore RTC time from the latest valid log entry.

Logged data includes:

- date/time,
- environmental readings,
- light level,
- battery state,
- WiFi state,
- charging state,
- sleep/wake state,
- SD fault status.

### USB CDC and Firmware Update Path

TinyUSB provides a composite USB device with:

- CDC ACM virtual COM port for debug and command access.
- DFU Runtime interface used to request a reboot into the STM32 factory ROM bootloader.

The firmware does not implement flash programming directly. Instead, it uses the STM32U585 built-in ROM DFU bootloader for firmware updates.

DFU request flow:

```text
debug command, DFU_DETACH, or UI3 hold
  -> write magic value to backup register
  -> software reset
  -> early boot checks magic value
  -> jump to STM32 ROM bootloader
  -> PC tools flash firmware over USB DFU
```

This design gives a cleaner and more reliable transition than jumping directly from a running USB application into the ROM bootloader.

### Manual DFU Rescue

The firmware includes a user-accessible DFU rescue mechanism:

- Hold UI3 active-low for 1 second while USB is mounted to request DFU during normal operation.
- Hold UI3 during boot with USB/charger present to request DFU before normal application startup.

The runtime hold detector is nonblocking and only polls UI3 while USB is mounted, minimizing power and refresh impact when USB is disconnected.

## Engineering Highlights

Key implementation work in this firmware includes:

- Integrated TinyUSB CDC VCP without taking over CubeMX-generated HAL files.
- Routed USB interrupts to TinyUSB while keeping generated HAL USB code intact.
- Built a portable USB configuration layer for descriptors, VID/PID, MCU family, and ROM bootloader address.
- Added CDC VCP as a UART-like debug device in the existing UART BSP.
- Implemented buffered USB TX/RX with DTR-based connection detection.
- Added DFU Runtime support and a software jump into the STM32 ROM DFU bootloader.
- Added a magic-register reset flow using TAMP backup registers.
- Implemented active-low UI3 long-press DFU rescue.
- Improved USB unplug/suspend behavior to shut down USB pins, clocks, CRS, and VDDUSB.
- Reduced USB impact on refresh/sleep when disconnected.
- Rate-limited STM32-to-ESP WiFi messages to prevent UART flooding.
- Fixed ESP UART handling so ESP responses are not echoed back into the ESP protocol stream.
- Documented firmware architecture, USB/DFU behavior, and hardware debugging notes.

## Technical Skills Demonstrated

This project demonstrates experience with:

- STM32CubeIDE and STM32 HAL.
- STM32U5 low-power firmware development.
- Bare-metal cooperative firmware architecture.
- UART, SPI, I2C, RTC, DMA, EXTI, and USB peripheral integration.
- TinyUSB CDC ACM and DFU Runtime integration.
- STM32 ROM bootloader handoff.
- Embedded state machines and nonblocking timing.
- Ring-buffer based serial IO.
- E-paper dashboard rendering.
- FatFs SD card logging.
- Sensor driver integration.
- ESP32 co-processor communication.
- Hardware-aware debugging of USB-C, VBUS, D+/D-, TVS, and backpowering behavior.

## Summary

The DINO test setup firmware is a practical embedded systems project that combines low-level STM32 peripheral work with higher-level application management. It coordinates sensors, display output, SD logging, WiFi communication, USB debug access, low-power sleep, and firmware-update recovery paths in a compact bare-metal architecture.

The implementation emphasizes maintainability by keeping CubeMX-generated code separate from board support and application logic, while still preserving direct control over timing, power, and hardware behavior.

