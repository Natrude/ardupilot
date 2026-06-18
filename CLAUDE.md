# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Fork purpose

This is a custom fork of ArduPilot adding support for a tricopter with a bidirectional rear motor. The long term goal is to enable flight without control surfaces. The key local change is in `libraries/AP_Motors/`:

- `AP_Motors_Class.h`: Added `TRI_REAR_BIDIR` to the `MotorOptions` bitmask enum.
- `AP_MotorsTri.cpp`: When `TRI_REAR_BIDIR` is set, the rear motor (MOT_4) maps 0 thrust to PWM neutral (1500) rather than PWM min (1000), enabling reverse thrust.
- `AP_MotorsMulticopter.cpp`: Related throttle change.

## Build system

ArduPilot uses Waf. Always run `./waf` from the repo root; never with `sudo`.

```sh
# Configure for SITL (used for local development)
./waf configure --board sitl

# Build a vehicle
./waf copter        # also: plane, rover, sub, heli, antennatracker

# Build a specific target
./waf --targets bin/arducopter
./waf --targets tests/test_vectors

# List supported boards
./waf list_boards

# Clean
./waf clean         # current board only
./waf distclean     # everything
```

Configure only needs to run once per board, or when switching boards.

## Testing

### SITL integration tests
```sh
# Run all tests for a vehicle
Tools/autotest/autotest.py build.Copter test.Copter

# Run a single test
Tools/autotest/autotest.py build.Copter test.Copter.RTLYaw
```

Test suites live in `Tools/autotest/` (`arducopter.py`, `arduplane.py`, etc.).

### C++ unit tests (GTest)
```sh
./waf --targets tests/test_math
./waf check           # build everything + run relevant tests
./waf check --alltests
```

Unit tests live in `libraries/<lib>/tests/` and use `#include <AP_gtest.h>`.

### Python linting (for files marked `AP_FLAKE8_CLEAN`)
```sh
flake8 <file.py>
```

## Repository structure

```
ArduCopter/        # Copter vehicle — modes, GCS, parameters
ArduPlane/         # Plane vehicle
ArduSub/           # Sub/ROV vehicle
Rover/             # Rover/boat vehicle
libraries/
  AP_<Name>/       # Shared ArduPilot libraries (AP_GPS, AP_Baro, AP_Motors...)
  AC_<Name>/       # Copter/QuadPlane controls (AC_PID, AC_WPNav...)
  AR_<Name>/       # Rover-specific (AR_Motors, AR_WPNav)
  AP_HAL/          # Hardware Abstraction Layer interface
  AP_HAL_ChibiOS/  # HAL for STM32/ChibiOS hardware
  AP_HAL_SITL/     # HAL for software-in-the-loop simulation
  GCS_MAVLink/     # MAVLink ground control station interface
  SITL/            # SITL physics model backends
Tools/
  autotest/        # SITL integration test framework
  CodeStyle/       # astylerc formatting config
modules/           # Git submodules — do not modify directly
```

Each library typically has: main class `.h`/`.cpp`, backend interface (`*_Backend.*`), a `*_config.h` for compile-time flags, and optional `tests/` and `examples/` subdirectories.

Each vehicle directory has: mode implementations (`mode_*.cpp`), parameters (`Parameters.cpp`/`.h`), GCS interface (`GCS_*.cpp`/`.h`), and a `wscript` listing required libraries.

## Coding style

Enforced via `astyle` using `Tools/CodeStyle/astylerc`. Format only the lines you change.

| Rule | Convention |
|---|---|
| Indentation | 4 spaces, no tabs |
| Braces | Opening brace on same line (K&R) |
| Line endings | LF only |
| Header guards | `#pragma once` |
| Classes | `AP_`/`AC_` prefix, PascalCase |
| Methods | `snake_case` |
| Constants | `UPPER_SNAKE_CASE` |
| Compile-time flags | `AP_<NAME>_ENABLED` |

## Parameters

Document parameters with `@` annotations above `AP_GROUPINFO` macros. Required annotations: `@Param`, `@DisplayName`, `@Description`, `@Values` or `@Range`, `@User`. Parameter full names are limited to 16 characters. Never change existing `AP_GROUPINFO` index numbers — they are baked into user configurations.

## Commit messages

```
Subsystem: short description under ~72 chars

Optional longer explanation.
```

The first line must contain a colon. Use the library or vehicle name as the subsystem (e.g. `AP_Motors:`, `ArduCopter:`). No merge commits; rebase onto target branch. No `fixup!` commits.

## Key constraints

- Do not modify `modules/` — those are upstream git submodules.
- Do not bypass `#if AP_<FEATURE>_ENABLED` compile-time guards.
- Do not introduce platform-specific code in shared libraries; use the HAL abstraction.
- RAM and flash are scarce on embedded targets — avoid unnecessary allocations or dependencies.
- AI-assisted contributions must be disclosed in PR descriptions.
