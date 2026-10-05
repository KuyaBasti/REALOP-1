# IntelliSat — REALOP-1 Flight Software

<p align="center"><img src="docs/system-overview.svg" alt="IntelliSat (REALOP-1) system overview. On the Orbital Platform's STM32L476ZG (Cortex-M4F, 80 MHz), main() initialises clocks, the RTC, sensors and watchdogs through register-level drivers with no HAL, then idles in while(1) or runs a selected test. Sensors: two ASM330LHH IMUs on SPI3 and SPI2 chosen with set_IMU(), two QMC5883L magnetometers on bit-banged I2C, and sun sensors (12 photodiodes on ADC2/3, INA226 and TMP275 over I2C). The FreeRTOS scheduler is scaffolded with empty mode stubs and its start is commented out, and nothing calls ADCS_MAIN(mode) yet. ADCS (B-dot, TRIAD, PID, ramp; SGP4, IGRF and SPA models) reaches hardware only through virtual_intellisat, which wires the gyro, a millisecond clock and PWM duty; mag, sun and coils are still stubs. Only hardware tests drive the magnetorquer MCU link (USART2) and the radio link (USART1, relaying to the ground station), both 9600 baud with CRC, and the PWM to two HDDs via ESCs. The GPIO-switched power-distribution driver has no caller yet." width="100%"></p>

Flight software for REALOP-1, the first CubeSat mission of Space and Satellite
Systems at UC Davis — bare-metal, register-level C firmware (no HAL) for the
Orbital Platform flight computer (STM32L476ZG, ARM Cortex-M4F) that boots the
satellite and brings up its sensors and watchdogs. Drivers for the HDD PWM and
the radio and magnetorquer intercoms are exercised by on-board hardware tests,
and the FreeRTOS mission-mode scheduler is scaffolded but not yet started from
`main()`.

See [SYSTEM-DESIGN.md](./SYSTEM-DESIGN.md) for the full architecture — every
component, built or planned, and how data moves between them.

## Repository layout

| Dir | What |
|-----|------|
| `Src/system_config/` | Register-level MCU drivers — clocks, GPIO/EXTI, UART, I2C, SPI, QSPI, ADC+DMA, RTC, timers, low-power sleep, watchdogs, fault handlers |
| `Src/peripherals/` | Board peripherals — dual ASM330LHH IMUs (SPI), QMC5883L magnetometers, sun sensors (photodiodes + TMP275 + INA226), power distribution, radio & magnetorquer intercoms |
| `Src/scheduler/` | FreeRTOS mission scheduler scaffolding — mode-task stubs (detumble, experiment, comms, ecc, idle), tickless-idle hook (`low_pwr.c`), and kernel hooks |
| `Src/ADCS/` | Attitude Determination & Control — B-dot detumble, PID/ramp control, TRIAD, IGRF · SGP4 · SPA models |
| `Src/unit_tests/` | On-target unit test framework — suites, registry, assertions |
| `Src/tools/` | Debug print/scan utilities over the debug UART |
| `FreeRTOS/`, `Drivers/` | Vendored FreeRTOS kernel and CMSIS / STM32L4 device headers |
| `Startup/`, `*.ld` | Cortex-M4 startup assembly and FLASH/RAM linker scripts |
| `Manuals/` | Toolchain setup, hardware reference, and onboarding guides |
| `docs/system-overview.svg` | System overview diagram (top of this README) |
| `docs/system-design-flowchart.svg` | End-to-end flowchart (SYSTEM-DESIGN.md) |

## Quickstart

Building and flashing goes through STM32CubeIDE with an ST-Link probe — see
[Manuals/Getting_Started.md](./Manuals/Getting_Started.md) for the full setup.

```c
// Src/inc/globals.h — set the Orbital Platform board revision first
#define OP_REV 3

// Src/main.c — compile-time switches select what runs
#define RUN_TEST        0   // 0 = flight path (currently: init, then idle in while(1)), 1 = run one hardware test
#define TEST_ID         0   // which test to run when RUN_TEST = 1
#define RTOS_TEST       0   // 0 = branch_main, 1 = RTOS test_main (both calls are commented out for now)
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
