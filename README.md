# IntelliSat — REALOP-1 Flight Software

Flight software for REALOP-1, the first CubeSat mission of Space and Satellite
Systems at UC Davis — bare-metal, register-level C firmware (no HAL) for the
Orbital Platform flight computer (STM32L476ZG, ARM Cortex-M4F) that boots the
satellite, drives its sensors and actuators, and schedules mission modes on
FreeRTOS.

See [SYSTEM-DESIGN.md](./SYSTEM-DESIGN.md) for the full architecture — every
component, built or planned, and how data moves between them.

## Repository layout

| Dir | What |
|-----|------|
| `Src/system_config/` | Register-level MCU drivers — clocks, GPIO/EXTI, UART, I2C, SPI, QSPI, ADC+DMA, RTC, timers, low-power sleep, watchdogs, fault handlers |
| `Src/peripherals/` | Board peripherals — dual ASM330LHH IMUs (SPI), QMC5883L magnetometers, sun sensors (photodiodes + TMP275 + INA226), power distribution, radio & magnetorquer intercoms |
| `Src/scheduler/` | FreeRTOS mission scheduler — mode tasks (detumble, experiment, comms, low-power, …) and kernel hooks |
| `Src/ADCS/` | Attitude Determination & Control — B-dot detumble, PID/ramp control, TRIAD, IGRF · SGP4 · SPA models |
| `Src/unit_tests/` | On-target unit test framework — suites, registry, assertions |
| `Src/tools/` | Debug print/scan utilities over the debug UART |
| `FreeRTOS/`, `Drivers/` | Vendored FreeRTOS kernel and CMSIS / STM32L4 device headers |
| `Startup/`, `*.ld` | Cortex-M4 startup assembly and FLASH/RAM linker scripts |
| `Manuals/` | Toolchain setup, hardware reference, and onboarding guides |

## Quickstart

Building and flashing goes through STM32CubeIDE with an ST-Link probe — see
[Manuals/Getting_Started.md](./Manuals/Getting_Started.md) for the full setup.

```c
// Src/inc/globals.h — set the Orbital Platform board revision first
#define OP_REV 3

// Src/main.c — compile-time switches select what runs
#define RUN_TEST        0   // 0 = flight software, 1 = run one hardware test
#define TEST_ID         0   // which test to run when RUN_TEST = 1
#define RUN_UNIT_TESTS  0   // 1 = run the on-target unit test suites
```

> The Orbital Platform goes through hardware revisions; pin maps and peripheral
> assignments are gated on `OP_REV` throughout the codebase. Revision
> differences are documented in
> [Manuals/OrbitalPlatform_Hardware/OP_Hardware.md](./Manuals/OrbitalPlatform_Hardware/OP_Hardware.md).

## Hardware

The Orbital Platform is the mission's custom flight computer:
[uwu64/orbital-platform](https://github.com/uwu64/orbital-platform).

## License

MIT — see [LICENSE](./LICENSE). The ADCS subsystem vendors third-party
components under their own licenses; see
[Src/ADCS/THIRD_PARTY_NOTICES.md](./Src/ADCS/THIRD_PARTY_NOTICES.md).
