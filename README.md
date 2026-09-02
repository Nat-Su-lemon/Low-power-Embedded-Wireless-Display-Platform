# DINO Test Setup Firmware Architecture

This document gives a practical overview of the `DINO_TEST_SETUP` firmware codebase. It is meant to answer:

- Where does the program start?
- What owns each peripheral?
- How do the main subsystems communicate?
- Which files should be edited for application logic versus CubeMX-generated setup?
- How do sensor reads, dashboard rendering, logging, sleep, WiFi, USB, and debug commands fit together?

For the detailed USB CDC VCP and DFU implementation, see:

```text
Documentation/USB_CDC_DFU.md
```

## Big Picture

This firmware is a cooperative, bare-metal STM32 application. There is no RTOS. The main loop calls each subsystem regularly, and interrupt callbacks set flags or move bytes into buffers.

The firmware is organized into these layers:

```text
Core/
  CubeMX-generated startup, clocks, GPIO, peripheral init, interrupts, main loop.

Bsp/
  Board support wrappers around HAL peripherals.
  UART, SPI, I2C, GPIO interrupt handling, bootloader jump helpers.

Common/
  Shared utilities and project-wide config.
  Debug command parser, ring buffer, global settings.

Control/
  Application-level managers.
  Sensors, dashboard, sleep, WiFi, logging, GPIO button behavior, ESP flash mode.

Modules/
  Device drivers and third-party/vendor code.
  TinyUSB, BME680, MAX17048, OPT4001, e-paper, FatFs/SD.

Drivers/
  STM32 HAL and CMSIS.
  Mostly generated/vendor code.

Documentation/
  Human-written architecture and integration notes.
```

The application shape is:

```text
main()
  |
  +-- HAL/CubeMX initialization
  +-- application init
  |
  +-- forever loop:
        usb_vcp_task()
        bspUARTTask()
        checkUARTRX(debug_dev, vcp_dev, main_dev)
        GPIOCheck()
        wifiTask()
        sensorHubPoll()
        dashRender()
        SDLogDashboard()
        checkSleep()
```

Most runtime behavior is therefore driven by:

```text
main loop polling + interrupt callbacks + shared flags/ring buffers
```

## Startup Flow

The main entry point is `Core/Src/main.c`.

Current startup order:

```c
HAL_Init();
system_bootloader_jump_if_requested();

SystemPower_Config();
SystemClock_Config();

MX_GPIO_Init();
system_bootloader_rescue_check();

MX_GPDMA1_Init();
MX_RTC_Init();
MX_USART2_UART_Init();
MX_I2C1_Init();
MX_SPI2_Init();
MX_ICACHE_Init();
MX_USART3_UART_Init();
MX_USB_OTG_FS_PCD_Init();

DelayUs_Init();
usb_vcp_init();
bspUARTInit();
checkRstCondition();
sensorInit();
dashInit();
sleepInit();
SDLogInit();
```

Important details:

- `system_bootloader_jump_if_requested()` runs very early so a reset-to-DFU request can jump to ROM before normal app peripherals are initialized.
- `system_bootloader_rescue_check()` runs after GPIO init so UI3/STAT pins can be read for boot-time DFU rescue.
- `MX_USB_OTG_FS_PCD_Init()` is still generated/called, but the function returns immediately in its USER CODE section. TinyUSB owns USB instead.
- Application-level init happens after generated peripherals are initialized.

## Main Loop

The main loop is cooperative:

```c
while (1)
{
    usb_vcp_task();
    bspUARTTask();

    checkUARTRX(&debug_dev);
    checkUARTRX(&vcp_dev);
    checkUARTRX(&main_dev);
    GPIOCheck();
    wifiTask();
    sensorHubPoll(&g_data);
    dashRender();
    SDLogDashboard(&g_data);
    checkSleep();
}
```

There is no scheduler. Each function should either:

- return quickly,
- run only when armed by a flag,
- or intentionally perform a blocking hardware operation such as an e-paper refresh, SPI transfer wait, or sleep entry.

The general data flow is:

```text
Sensors / WiFi / USB / sleep state
  -> dashboard_data_t g_data
  -> Dashboard_GuiRefresh()
  -> SDLogDashboard()
```

## CubeMX Core Layer

`Core/` contains mostly CubeMX-generated files:

```text
Core/Src/main.c
Core/Src/gpio.c
Core/Src/i2c.c
Core/Src/spi.c
Core/Src/usart.c
Core/Src/rtc.c
Core/Src/gpdma.c
Core/Src/icache.c
Core/Src/usb_otg.c
Core/Src/stm32u5xx_it.c
Core/Src/stm32u5xx_hal_msp.c
Core/Src/system_stm32u5xx.c
Core/Startup/startup_stm32u585ciuxq.s
```

Use CubeMX USER CODE sections for custom code in these files. Avoid editing generated sections directly.

Key generated responsibilities:

- Clock tree and power config.
- GPIO modes and EXTI setup.
- UART, SPI, I2C, RTC, DMA peripheral init.
- Interrupt handler shells.
- HAL MSP pin/clock setup.

Custom behavior currently inserted into Core:

- Early ROM bootloader jump call in `main.c`.
- Boot-time UI3 rescue check in `main.c`.
- TinyUSB task call in the main loop.
- TinyUSB IRQ dispatch in `stm32u5xx_it.c`.
- HAL USB PCD init bypass in `usb_otg.c`.

## BSP Layer

`Bsp/` wraps low-level peripherals in project-specific APIs. This is the layer that makes HAL peripherals usable by application managers.

### UART BSP

Files:

```text
Bsp/Src/uart_bsp.c
Bsp/Inc/uart_bsp.h
```

The UART BSP owns three UART-like devices:

```c
debug_dev  // physical debug UART, DEBUG_UART / USART3
main_dev   // physical main UART, MAIN_UART / USART2, used for ESP32
vcp_dev    // USB CDC virtual COM port
```

`uart_dev_t` hides whether a device is:

```text
HAL UART
USB VCP
```

Physical UART behavior:

- TX uses `HAL_UART_Transmit_IT()`.
- RX uses `HAL_UART_Receive_IT()`.
- Bytes are stored in ring buffers.
- Complete lines are processed by `checkUARTRX()`.
- `debug_dev` echoes typed characters for terminal usability.
- `main_dev` does not echo RX bytes back to the ESP32. This avoids reflecting ESP responses back into the ESP protocol stream.
- Physical UART RX treats either CR or LF as a complete line, so ESP firmware may send `\r\n`, `\r`, or `\n`.

VCP behavior:

- TX bytes are buffered in `vcp_dev.tx_rb`.
- `vcpTxPump()` moves buffered bytes to TinyUSB when CDC is connected.
- RX bytes arrive through `usb_vcp_rx_cb()` and are placed in `vcp_dev.rx_rb`.

Routing:

```text
debug_dev and vcp_dev RX -> debugProcess()
main_dev RX              -> wifiProcess()
```

The debug output path is controlled by:

```c
#define DEBUG_CONSOLE_MODE DEBUG_CONSOLE_BOTH
```

in `Common/Inc/config.h`.

### SPI BSP

Files:

```text
Bsp/Src/spi_bsp.c
Bsp/Inc/spi_bsp.h
```

SPI devices:

```c
spi_eink_dev
spi_sd_dev
```

The SPI BSP:

- controls chip select,
- tracks the active SPI target,
- starts blocking waits around interrupt/DMA transfers,
- uses `spi_transfer_done` set by HAL SPI callbacks.

Typical flow:

```text
spiStart(device)
  -> CS low
  -> spiSendData/spiReadData/spiTxRx
  -> wait for completion callback
  -> spiFinish(device)
  -> CS high
```

This SPI layer is used by e-paper and SD/FatFs code.

### I2C BSP

Files:

```text
Bsp/Src/i2c_bsp.c
Bsp/Inc/i2c_bsp.h
```

The I2C BSP wraps HAL I2C calls with:

- device ready probe,
- raw read/write,
- 8-bit register read/write,
- 16-bit register read/write,
- byte helpers,
- masked register update,
- simple bus recovery by deinit/init.

It returns project-level `i2c_status_t` values instead of raw HAL status.

This layer supports BME680, MAX17048, OPT4001, and other I2C drivers.

### GPIO BSP

Files:

```text
Bsp/Src/gpio_bsp.c
Bsp/Inc/gpio_bsp.h
```

This layer handles HAL GPIO EXTI callbacks and converts them into simple flags:

```c
ui1_button_flag
ui2_button_flag
ui3_button_flag
opt_int_flag
```

It also handles wake behavior:

```c
if ((GPIO_Pin == STAT1_Pin) || (GPIO_Pin == STAT2_Pin)) {
    usb_vcp_request_reprobe();
}

if (sleeping) {
    sleeping = 0;
    wake_reason = WAKEUP_GPIO;
}
```

Important: heavy work is not done in the EXTI callback. The callback sets flags, and `GPIOCheck()` later performs application actions.

### System Bootloader BSP

Files:

```text
Bsp/Src/system_bootloader.c
Bsp/Inc/system_bootloader.h
```

This module owns:

- DFU magic flag in TAMP backup register,
- software reset request into DFU,
- early boot magic check,
- jump to STM32 ROM bootloader,
- boot-time UI3 rescue.

The USB-specific details are documented in `Documentation/USB_CDC_DFU.md`.

## Common Layer

`Common/` contains shared code that is not tied to one application manager.

### Config

File:

```text
Common/Inc/config.h
```

This is the central project configuration file for:

- buffer sizes,
- UART assignments,
- debug enable flags,
- debug console routing,
- sleep behavior,
- SPI/I2C aliases,
- sensor addresses/settings,
- e-paper dimensions,
- sleep timing,
- WiFi protocol timing.

Examples:

```c
#define TX_BUFFER_SIZE          4096
#define RX_BUFFER_SIZE          4096
#define DEBUG_UART              huart3
#define MAIN_UART               huart2
#define DEBUG_CONSOLE_MODE      DEBUG_CONSOLE_BOTH
#define SLEEP_BLOCK_WHEN_USB_ACTIVE 1U
#define PANEL_W                 400
#define PANEL_H                 300
```

### Debug

Files:

```text
Common/Src/debug.c
Common/Inc/debug.h
```

Responsibilities:

- `debugMsg()` formatted debug printing,
- optional debug categories: display, SPI, sensor,
- command parser for debug UART/VCP,
- commands for SPI/I2C inspection,
- `dfu` command to reset into STM32 ROM DFU,
- help text.

Debug command input can come from:

```text
physical debug UART
USB VCP
```

because both are routed into `debugProcess()` by `checkUARTRX()`.

### Ring Buffer

Files:

```text
Common/Src/ring_buffer.c
Common/Inc/ring_buffer.h
```

Used heavily by the UART BSP. Provides:

- init,
- put,
- get,
- peek,
- empty/full checks,
- reset.

The UART and VCP devices each get RX/TX storage arrays and ring-buffer state.

## Control Layer

`Control/` contains application managers. These modules coordinate device drivers, BSPs, and shared dashboard state.

### Sensor Manager

Files:

```text
Control/Sensors/Src/Sensor_Manager.c
Control/Sensors/Inc/Sensor_Manager.h
```

Responsibilities:

- initialize/deinitialize sensors,
- wake sensors after sleep,
- arm sensor reads,
- read light, environment, battery, time, WiFi status, and charging status,
- write values into `dashboard_data_t`,
- mark dashboard fields ready/stale.

Main functions:

```c
sensorInit()
sensorDeinit()
sensorWake()
sensorEnable()
sensorHubPoll(&g_data)
updateTime(&g_data)
updateWifi(&g_data)
```

The normal poll flow:

```text
sensorEnable()
  -> sensorHubPoll()
      -> read_light()
      -> read_environment()
      -> read_battery()
      -> read_time()
      -> read_wifi()
      -> read_charging_status()
      -> mark ready/stale flags
      -> sensor_data_ready = 1
      -> sensor_enable = 0
```

The dashboard can render partial data because each field has its own ready flag.

Current sensors:

```text
OPT4001 light sensor
BME680 environment sensor
MAX17048 fuel gauge
RTC time/date
BQ25185 charger status through STAT1/STAT2 GPIOs
ESP/WiFi state from Wifi_Manager
```

### Dashboard Manager

Files:

```text
Control/Dashboard/Src/Dashboard_Manager.c
Control/Dashboard/Src/dashboard_gui.c
Control/Dashboard/Inc/Dashboard_Manager.h
Control/Dashboard/Inc/dashboard_gui.h
Control/Dashboard/Inc/dashboard_data.h
```

`dashboard_data_t g_data` is the shared display/logging data model.

`dashboard_data.h` defines:

- data fields,
- ready flags,
- render done flag helpers,
- initialization helper.

`Dashboard_Manager.c` controls when the display should render:

```c
dashInit()
dashEnable()
dashRender()
dashSetUsbConnected()
```

`dashRender()`:

```text
if dashboard not armed and no new sensor data, return
EINKStart()
Dashboard_GuiInit()
Dashboard_GuiRefresh(&g_data)
Dashboard_GuiSleep()
EINKExit()
dash_enable = 0
```

`dashboard_gui.c` owns the actual layout and rendering calls into the e-paper paint/display driver.

USB mount/unmount callbacks update the dashboard USB indicator through:

```c
usb_vcp_mounted_cb()
usb_vcp_unmounted_cb()
```

### Sleep Manager

Files:

```text
Control/Sleep/Src/Sleep_Manager.c
Control/Sleep/Inc/Sleep_Manager.h
```

Responsibilities:

- sleep enable/disable/toggle,
- configure minute/hour RTC alarms,
- decide when to sleep,
- shut down sensors before STOP2,
- shut down USB before STOP2,
- enter STOP2,
- restore clocks/peripherals after wake,
- refresh sensor/dashboard state after wake,
- track wake reason.

The normal sleep decision:

```text
checkSleep()
  -> if sleep disabled, return
  -> if USB active and sleep block enabled, return
  -> choose minute/hour alarm behavior based on light/night status
  -> startSleep()
```

`startSleep()`:

```text
sensorDeinit()
configure RTC alarms
optional OPT night interrupt setup
usb_vcp_shutdown()
HAL_SuspendTick()
HAL_PWREx_EnterSTOP2Mode()
restoreAfterSleep()
sensorWake() when full refresh is due
```

USB is deliberately stopped before STOP2 because USB FS needs clocks and because of the board's backpowering concerns.

### GPIO Manager

Files:

```text
Control/GPIO/Src/GPIO_Manager.c
Control/GPIO/Inc/GPIO_Manager.h
```

This module converts button flags into application actions.

Current behavior:

```text
UI1:
  if ESP flash mode active, exit flash mode
  else if awake, enter ESP flash mode
  else start WiFi

UI2:
  arm sensor read
  immediately poll sensors

UI3:
  short press toggles sleep
  while USB is mounted, active-low 1 second hold requests ROM DFU
```

The UI3 runtime DFU hold timer is nonblocking and only polls UI3 while USB is mounted.

### WiFi Manager

Files:

```text
Control/Wifi/Src/Wifi_Manager.c
Control/Wifi/Inc/Wifi_Manager.h
```

This module coordinates an ESP device over `main_dev` UART.

State machine:

```text
WIFI_DISCONNECTED
WIFI_ESP_CONNECTING
WIFI_ESP_CONNECTED
WIFI_QUERYING
WIFI_PASS_ACQUIRED
WIFI_DISCONNECTING
```

`startWifi()`:

- clears last messages/password/IP,
- powers ESP path with `ESP_SWITCH`,
- disables sleep.

`wifiTask()`:

- advances the state machine,
- sends text commands to ESP through `main_dev`,
- updates dashboard WiFi fields,
- handles timeout/disconnect.
- rate-limits STM-to-ESP requests with elapsed `HAL_GetTick()` timing instead of blocking in `HAL_Delay()`.
- keeps the connection timeout duration separate from the PING transmit interval.

`wifiProcess()`:

- receives lines from `main_dev`,
- strips `ESP: `,
- detects `FOUND: ` password messages,
- detects `IP: ` address messages,
- updates acquired password/IP state.

The STM32-to-ESP protocol is text based:

```text
STM: PING
STM: GET WIFI
STM: ACQUIRED

ESP: ACKNOWLEDGED
ESP: IP: ...
ESP: FOUND: ...
```

WiFi timing is configured in `Common/Inc/config.h`:

```c
#define ESP_CONNECTION_DELAY 100U
#define ESP_TIMEOUT_COUNTER 100U
#define ESP_PING_INTERVAL_MS 1000U
#define ESP_QUERY_DELAY 1000U
```

The connection timeout is still:

```text
ESP_CONNECTION_DELAY * ESP_TIMEOUT_COUNTER
```

With the current values, that is 10 seconds. `ESP_PING_INTERVAL_MS` controls how often `STM: PING` is sent inside that timeout window. This prevents the STM32 from flooding the ESP32 with PING messages while preserving the same timeout duration.

Important UART behavior for this protocol:

```text
ESP responses must start with "ESP: "
line endings may be CR, LF, or CRLF
main_dev RX is not echoed back to main_dev
```

### Logging Manager

Files:

```text
Control/Logging/Src/Logging_Manager.c
Control/Logging/Inc/Logging_Manager.h
```

Responsibilities:

- initialize SD card and FatFs,
- create/validate `log.csv`,
- write CSV header,
- append dashboard samples after display render,
- restore RTC time from the last valid log row,
- provide `SDCardTest()` debug helper.

Startup:

```text
SDLogInit()
  -> disk_initialize()
  -> f_mount()
  -> open/create log.csv
  -> validate header
  -> recreate if needed
  -> SDRestoreRTCFromLog()
```

Runtime:

```text
SDLogDashboard(&g_data)
  -> if SD fault, try SDLogInit()
  -> if dashboard GUI has not completed render, return
  -> append CSV row from dashboard_data_t
```

Logging waits for `Dashboard_GuiIsReady()` so it logs values after the dashboard render has completed.

### Flash Controller

Files:

```text
Control/Flash/Src/Flash_Controller.c
Control/Flash/Inc/Flash_Controller.h
```

This controls ESP flashing/pass-through mode.

`flashStart()`:

- powers ESP path,
- drives BOOT/EN pins to put ESP into bootloader mode,
- sets `pass_through = true`,
- routes bytes between debug UART and main UART in `HAL_UART_RxCpltCallback()`.

`flashEnd()`:

- powers/resets ESP back to normal,
- clears pass-through mode.

UI1 is currently the main user control for this mode.

## Modules Layer

`Modules/` contains device drivers and third-party stacks.

Important module groups:

```text
Modules/BME680/
  Bosch BME680 driver and app wrapper.

Modules/MAX17048/
  Fuel gauge driver and app wrapper.

Modules/OPT4001/
  Light sensor driver and app wrapper.

Modules/Eink/
  E-paper panel drivers, drawing/paint utilities, fonts, display app wrappers.

Modules/SD/fat/
  FatFs and SD block/media glue.

Modules/tinyusb/
  TinyUSB stack, CDC, DFU runtime, DWC2 backend, descriptors.
```

The `Control/` layer should call module app wrappers rather than reaching directly into low-level register drivers when possible.

Examples:

```text
Sensor_Manager -> BMEInit/BMESampleN/BMEDeinit
Sensor_Manager -> MAXInit/MAXRead
Sensor_Manager -> OPTInit/OPTRead/OPTNightInit
Dashboard GUI  -> EPD/Paint APIs
Logging        -> FatFs diskio/f_open/f_write
USB VCP        -> TinyUSB tud_* APIs
```

## Shared Data Model

The main shared application state is:

```c
dashboard_data_t g_data;
```

defined in `Dashboard_Manager.c`.

This structure contains:

- time/date,
- temperature,
- pressure,
- humidity,
- light,
- battery,
- WiFi status/pass/IP,
- charging state,
- sleep state,
- USB state,
- SD fault status,
- ready/stale flags.

The ready/stale flags let the GUI distinguish:

```text
data is valid and should be shown
data is missing/stale and should be blank or treated carefully
```

Producers:

```text
Sensor_Manager
Wifi_Manager via updateWifi()
Sleep_Manager
Dashboard_Manager USB callbacks
Logging_Manager SD fault handling
```

Consumers:

```text
dashboard_gui.c
Logging_Manager.c
debug output
sleep decisions
```

## Interrupt Model

The firmware uses interrupts for:

- SysTick/HAL tick,
- RTC alarms,
- GPIO EXTI,
- UART RX/TX,
- SPI DMA/IT completion,
- USB OTG FS through TinyUSB,
- DMA channels.

General pattern:

```text
ISR or HAL callback
  -> set a flag, move one byte, or mark transfer complete
  -> main loop performs larger work later
```

Examples:

```text
GPIO EXTI
  -> ui button flag / wake flag / USB reprobe flag

UART RX callback
  -> put byte into ring buffer
  -> mark rx_ready on CR/LF

SPI completion callback
  -> spi_transfer_done = 1

USB IRQ
  -> tud_int_handler(0)
  -> TinyUSB events are processed by usb_vcp_task()
```

Avoid doing slow work in callbacks. Keep callbacks short unless there is a deliberate reason.

## Power and Wake Model

Sleep is controlled by `Sleep_Manager`.

Important globals:

```c
volatile wake_reason_t wake_reason;
volatile uint8_t sleeping;
```

Wake sources include:

- RTC alarm,
- GPIO interrupts,
- charger STAT pin changes,
- optical interrupt behavior for night/light mode.

USB interaction:

- If USB is mounted/connected/probing, sleep is blocked when `SLEEP_BLOCK_WHEN_USB_ACTIVE` is enabled.
- Before STOP2, `usb_vcp_shutdown()` is called.
- STAT1/STAT2 GPIO edges can wake the board and request USB reprobe.

This design tries to keep normal refresh/sleep behavior fast when USB is disconnected.

## Application Feature Flows

### Normal sensor/display/log cycle

```text
Wake or UI2 arms sensors
  -> sensorEnable()
  -> sensorHubPoll(&g_data)
  -> each sensor field marks ready/stale
  -> sensor_data_ready = 1
  -> dashRender()
  -> Dashboard_GuiRefresh(&g_data)
  -> Dashboard_GuiSleep()
  -> SDLogDashboard(&g_data)
  -> checkSleep()
```

### Debug command flow

```text
physical UART or USB VCP receives characters
  -> UART/VCP RX ring
  -> checkUARTRX()
  -> debugProcess()
  -> command handler
```

Examples:

```text
help
dfu
spi ...
i2c ...
```

### ESP/WiFi flow

```text
UI1 while sleeping/idle
  -> startWifi()
  -> ESP_SWITCH on
  -> sleepOff()
  -> wifiTask state machine
  -> STM text commands sent to ESP at the configured interval
  -> ESP responses parsed by wifiProcess()
  -> g_data updated through updateWifi()
  -> dashboard render/log
```

### ESP flash/pass-through flow

```text
UI1 while awake and not sleeping
  -> flashStart()
  -> drive ESP BOOT/EN pins
  -> pass_through = true
  -> UART RX callback forwards bytes between debug_dev and main_dev
  -> UI1 again exits flash mode
```

### USB CDC debug flow

```text
usb_vcp_task()
  -> TinyUSB events/RX
  -> usb_vcp_rx_cb()
  -> vcp_dev ring buffer
  -> checkUARTRX(&vcp_dev)
  -> debugProcess()
```

TX:

```text
debugMsg()
  -> debugTransmitMsg()
  -> transmitMsg(&vcp_dev)
  -> VCP TX ring
  -> bspUARTTask()
  -> vcpTxPump()
  -> TinyUSB CDC write
```

### ROM DFU flow

```text
dfu command / DFU_DETACH / UI3 hold
  -> system_bootloader_request_reset()
  -> write magic to TAMP backup register
  -> NVIC_SystemReset()
  -> early main()
  -> system_bootloader_jump_if_requested()
  -> jump to STM32 ROM bootloader
```

## Where To Add New Code

Use this as a rough guide:

```text
New board-level peripheral wrapper:
  Bsp/Src and Bsp/Inc

New application behavior/state machine:
  Control/<Feature>/Src and Control/<Feature>/Inc

New sensor/device driver:
  Modules/<Device>/Src and Modules/<Device>/Inc

New debug command:
  Common/Src/debug.c

New project setting:
  Common/Inc/config.h

New dashboard field:
  Control/Dashboard/Inc/dashboard_data.h
  Control/Dashboard/Src/dashboard_gui.c
  producer manager that fills the field

New low-level CubeMX peripheral:
  DINO_TEST_SETUP.ioc / CubeMX
  Core generated files
  then wrap in BSP or Control as needed
```

For generated files, prefer USER CODE sections.

## Current Design Tradeoffs

The code is intentionally practical and test-setup oriented. A few current tradeoffs are worth remembering:

- Main loop is cooperative, so long HAL delays or e-paper refreshes block other tasks.
- Some SPI operations wait using `__WFI()` until transfer complete.
- WiFi PING/query pacing is nonblocking, but other parts of the firmware still contain intentional blocking waits.
- Dashboard rendering powers/initializes the e-paper path on demand.
- Logging waits until GUI refresh completes.
- USB is manually owned by TinyUSB even though CubeMX still generates HAL USB files.
- Several managers share `g_data`, so ready/stale flags are important for correctness.
- Some board-specific assumptions are still spread across config, BSP, and Control code.

These tradeoffs are acceptable for a test setup, but if the firmware grows, the next architecture improvements would be:

- continue removing remaining blocking waits from long-running managers,
- isolate board-specific USB hardware init into a board USB file,
- reduce direct cross-includes between managers,
- formalize event flags instead of ad hoc globals,
- split dashboard data producers from rendering/logging consumers more cleanly,
- add small unit tests for ring buffer and pure parsing code,
- document pin ownership in one board-level file.

## Quick Navigation

Most common questions:

```text
Where is main loop?
  Core/Src/main.c

Where are project settings?
  Common/Inc/config.h

Where does debugMsg go?
  Common/Src/debug.c -> Bsp/Src/uart_bsp.c

Where are debug commands parsed?
  Common/Src/debug.c

Where does USB run?
  Core/Src/usb_vcp.c
  Modules/tinyusb/usb_descriptors.c

Where does UART RX/TX happen?
  Bsp/Src/uart_bsp.c

Where are UI buttons handled?
  Bsp/Src/gpio_bsp.c sets flags
  Control/GPIO/Src/GPIO_Manager.c acts on flags

Where are sensors read?
  Control/Sensors/Src/Sensor_Manager.c

Where is display rendering?
  Control/Dashboard/Src/Dashboard_Manager.c
  Control/Dashboard/Src/dashboard_gui.c

Where is shared dashboard data defined?
  Control/Dashboard/Inc/dashboard_data.h
  Control/Dashboard/Src/Dashboard_Manager.c

Where is sleep handled?
  Control/Sleep/Src/Sleep_Manager.c

Where is SD logging?
  Control/Logging/Src/Logging_Manager.c

Where is ESP WiFi flow?
  Control/Wifi/Src/Wifi_Manager.c

Where is ESP flash/pass-through mode?
  Control/Flash/Src/Flash_Controller.c

Where is STM32 ROM DFU jump?
  Bsp/Src/system_bootloader.c
```
