# IntelliSat — system design

> How a reset vector becomes a **mission**.
>
> IntelliSat is the flight software for REALOP-1, a 3U CubeSat built by Space
> and Satellite Systems at UC Davis. It runs on the Orbital Platform flight
> computer (STM32L476ZG, ARM Cortex-M4F) with no vendor HAL — every driver
> talks to hardware registers directly. On boot it brings up clocks, buses,
> sensors, and watchdogs, then idles in `while(1)`. Handing control to a
> FreeRTOS scheduler whose mission modes call into a fully separable ADCS
> subsystem is the planned next step (the start call is still commented out
> in `main.c`). The **north star** is the autonomous mission loop — the
> scheduler picks a mode, ADCS points the satellite, the 100 ms logger records
> the experiment, and the radio downlinks it to the ground.

This document is the developer-facing map of the whole system — every
component, **built or planned**, and how data moves between them. Read the
flowchart top-to-bottom; the dashed boxes and links are the roadmap.

---

## End-to-end flowchart

<p align="center"><img src="docs/system-design-flowchart.svg" alt="IntelliSat end-to-end flowchart. Reset_Handler calls main(), which runs init_init() (clocks to 80 MHz, RTC on LSI) and then init_platform() (FPU, console, both IMUs, both magnetometers, sun sensors, LEDs, DMA, watchdogs, TIM7 heartbeat), then takes one compile-time path: a hardware test, the unit tests, or by default while(1). The branch_main() call that would start FreeRTOS is commented out, so the kernel (heap_4, 1 kHz configured tick, tickless idle via PWR and the RTC wakeup) and the empty mission-mode tasks are not reached, and nothing calls ADCS_MAIN yet. ADCS control (B-dot, PID, ramp) and determination (TRIAD with IGRF, SGP4, SPA) reach hardware only through the vi_* bridge: gyro reads, timing and HDD PWM are wired; magnetometer, sun sensor, coil, epoch and TLE calls are stubs. Register-level system drivers serve the device drivers for the IMUs (SPI), magnetometers (soft I2C, reads always use MAG1), sun sensors (ADC2/ADC3 and soft I2C), HDD ESCs (PWM), the magnetorquer MCU (USART2) and the radio board (USART1), both two-way at 9600 baud with CRC; the radio board carries downlink and commands to and from the ground station. The TIM6 experiment logger is set for 100 ms and started only by its test, no log callback is registered, and logging records, flash and FRAM storage are declared only. The power-distribution driver has no caller and the battery monitor is commented out." width="100%"></p>

---

## How to read it: the three flows that matter

1. **Two headers decouple the flight software from ADCS.** `ADCS.h` is the
   downcall — a scheduler mode is meant to command `ADCS_MAIN(mode)` and block
   until it returns a status (no caller exists yet). `virtual_intellisat.h` is
   the upcall — ADCS reads sensors and commands actuators *only* through `vi_*`
   functions that IntelliSat implements (gyro, millisecond clock and HDD PWM
   so far; mag, sun sensors and coils are still stubs). ADCS never touches a
   hardware register, so its sources also compile on a host into `libADCS.a`
   (`Src/ADCS/makefile`, gcc). `virtual_rtos` is the ADCS-side wrapper over
   FreeRTOS critical sections and task notifications; nothing calls it yet.
2. **The experiment logger is a callback, not a subsystem** (the
   `TIM6 → callback` edge). TIM6 is configured for an update interrupt every
   ≈100 ms (99.88 ms); whatever an experiment needs recorded is bound at
   runtime with `logger_registerLogFunction()`, and the ISR dispatches it.
   Timer hardware stays decoupled from application logic. Today only hardware
   test 3 starts the timer, and no experiment registers a callback yet.
3. **Redundancy and board revisions are gates, not forks.** The `OP_REV` macro
   selects pin maps and bus assignments per Orbital Platform revision at
   compile time. On Rev 3 the platform carries two IMUs on separate SPI buses
   and two magnetometers — `set_IMU()` selects which of IMU0 and IMU1 every
   driver call addresses, so switching IMUs is one call. Automatic failover is
   designed in ADCS `selectSensor()` but waits on `vi_get_sensor_status()`,
   which is still a stub.

The **Mission modes** node is the north star. The diagram makes the
punchline visible: most of what feeds it already exists — drivers, the
FreeRTOS kernel, ADCS algorithms, and the TIM6 logger timer are built; what is
missing is the wiring (the scheduler start in `main()`, routing SysTick to the
FreeRTOS tick handler — `xPortSysTickHandler` is commented out in
`FreeRTOSConfig.h`, so `SysTick_Handler` is still the startup default loop —
creating the mode tasks, a caller for `ADCS_MAIN`, the stubbed `vi_*` calls, a
registered log callback), and the mode functions (`*_time` / `config_*` / run /
`clean_*`) are scaffolded and waiting for flight logic.

---

## Component inventory

| Component | Layer | Tech | Status | Where |
|---|---|---|---|---|
| Startup + linker scripts | Boot | ARM asm · LD | ✅ built | `Startup/`, `STM32L476ZGTX_*.ld` |
| Clock tree + core config | System | C (registers) | ✅ built | `Src/system_config/core_config.c` |
| GPIO + EXTI interrupts | System | C (registers) | ✅ built | `Src/system_config/GPIO/` |
| UART + CRC | System | C (registers) | ✅ built | `Src/system_config/UART/` |
| Software I2C (bit-bang) | System | C (registers) | ✅ built | `Src/system_config/I2C/` |
| SPI | System | C (registers) | ✅ built | `Src/system_config/SPI/` |
| QSPI flash driver | System | C (registers) | ✅ built | `Src/system_config/QSPI/` |
| ADC + DMA | System | C (registers) | ✅ built | `Src/system_config/ADC/`, `DMA/` |
| RTC (calendar, alarms) | System | C (registers) | ✅ built | `Src/system_config/RTC/` |
| PWM timers | System | C (registers) | ✅ built | `Src/system_config/Timers/pwm_timer.c` |
| Experiment log timer (TIM6 + callback) | System | C (registers) | ✅ built | `Src/system_config/Timers/logger_timer.c` |
| Startup wait timer (TIM5, 30 min) | System | C (registers) | ✅ built | `Src/system_config/Timers/startup_timer.c` |
| Watchdogs (IWDG + WWDG) | System | C (registers) | ✅ built | `Src/system_config/WDG/` |
| Fault handlers | System | C (registers) | ✅ built | `Src/system_config/Faults/` |
| Low-power run + sleep | System | C (registers) | ✅ built | `Src/system_config/PWR/` |
| Dual IMU driver (ASM330LHH) | Peripheral | C · SPI | ✅ built | `Src/peripherals/IMU/` |
| Magnetometer driver (QMC5883L) | Peripheral | C · I2C | ✅ built | `Src/peripherals/MAG/` |
| Sun sensors (diodes + TMP275 + INA226) | Peripheral | C · ADC/I2C | ✅ built | `Src/peripherals/SunSensors/`, `PWRMON/` |
| Power distribution (pyro / MGT / HDD) | Peripheral | C · GPIO | ✅ built | `Src/peripherals/PDB/` |
| MGT intercom | Peripheral | C · UART | ✅ built | `Src/peripherals/MGTINTERCOM/` |
| Radio intercom | Peripheral | C · UART+CRC | 🟡 protocol defined | `Src/peripherals/Radio/` |
| Battery monitor | Peripheral | C · ADC | ⬜ planned (stubbed) | `Src/peripherals/BAT/` |
| FreeRTOS kernel + hooks | Scheduler | FreeRTOS | ✅ built | `FreeRTOS/`, `Src/scheduler/hooks/` |
| Tickless low-power idle | Scheduler | FreeRTOS + RTC | ✅ built | `Src/scheduler/intelliTasks/low_pwr.c` |
| Mission mode tasks | Scheduler | FreeRTOS | ⬜ scaffolded | `Src/scheduler/intelliTasks/` |
| Flight main loop | Scheduler | C | 🟡 boots, logic pending | `Src/main.c` |
| ADCS math (vector · matrix · quaternion) | ADCS | C | ✅ built | `Src/ADCS/adcs_math/` |
| Attitude determination (TRIAD + models) | ADCS | C · IGRF/SGP4/SPA/NOVAS | ✅ built | `Src/ADCS/determination/` |
| Attitude control (B-dot · PID · ramp) | ADCS | C | ✅ built | `Src/ADCS/control/` |
| `virtual_intellisat` bridge | ADCS | C | 🟡 gyro, clock, HDD PWM wired; mag, sun, coils, epoch, TLE stubbed | `Src/ADCS/virtual_intellisat.h`, `.c` |
| On-target unit test framework | Verification | C | ✅ built (IMU suite) | `Src/unit_tests/` |
| Per-driver hardware testers | Verification | C | ✅ built | `*_tester.c`, `*_test.c` |

> Mission mode tasks follow a fixed contract — `*_time()` (should it run?),
> `config_*`, a run function (`run_detumble`, `comms`, `experiment`, `ecc`,
> `idle`), `clean_*` — declared in
> `Src/scheduler/intelliTasks/intelliTasks_proto.h`. The pattern is in place;
> the bodies are stubs awaiting flight logic, and the `low_pwr` set is
> declared but not yet defined.

---

## Repository layout

| Path | Role |
|---|---|
| `Src/system_config/` | Register-level MCU drivers: clocks, buses, timers, RTC, power, watchdogs, faults |
| `Src/peripherals/` | Sensor / actuator / intercom drivers built on the system layer |
| `Src/scheduler/` | FreeRTOS mission scheduler: mode tasks, kernel hooks, tickless idle |
| `Src/ADCS/` | Attitude determination & control — separable subsystem with its own math and third-party models |
| `Src/unit_tests/` | On-target unit test framework (suites, registry, assertions) |
| `Src/tools/` | `printMsg` debug print/scan over the debug UART |
| `FreeRTOS/`, `Drivers/` | Vendored FreeRTOS kernel and CMSIS / STM32L4 device headers |
| `Startup/`, `*.ld` | Cortex-M4 reset vector and FLASH/RAM linker scripts |
| `Manuals/`, `img/` | Onboarding, toolchain, and hardware documentation |

---

## Bring-up stages

The build follows the classic firmware bring-up ladder (Stage 0 board
bring-up → Stage 8 flight readiness):

| Stage | Name | Status |
|---|---|---|
| 0 | Board bring-up — clocks, GPIO, debug UART | ✅ done |
| 1 | Bus + storage drivers — I2C, SPI, QSPI, ADC+DMA | ✅ done |
| 2 | Sensor + actuator peripherals — dual IMU, mag, sun sensors, PDB | ✅ done |
| 3 | Timekeeping + protection — RTC, timers, watchdogs, fault handlers | ✅ done |
| 4 | RTOS integration — kernel, hooks, tickless low-power idle | 🟡 queue/task demos; heap tuning in progress |
| 5 | Mission scheduler — mode tasks with time/config/run/clean contract | ⬜ scaffolded, bodies pending |
| 6 | ADCS integration — algorithms built, `vi_*` bridge implementations | 🟡 in progress |
| 7 | Comms — radio + magnetorquer intercom protocols, ground loop | 🟡 protocols defined |
| 8 | Flight readiness — antenna burnwire, flash logging, boot counters | ⬜ not started |
