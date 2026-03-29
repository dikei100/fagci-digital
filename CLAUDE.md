# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bare-metal embedded firmware for the Quansheng UV-K5/K6 amateur radio. Fork of fagci/uvk5-fagci-reborn with digital mode (M17/9600 GFSK) support. Targets the DP32G030 ARM Cortex-M0 MCU.

## Build

```bash
git submodule update --init --recursive   # first time only (CMSIS_5, printf)
pip install crcmod                         # needed by fw-pack.py
make                                       # build firmware
make clean && make                         # full rebuild
arm-none-eabi-size bin/firmware             # check flash usage (text+data ≤ 61440)
```

Toolchain: `arm-none-eabi-gcc` (CI uses 13.3.Rel1). Output: `bin/firmware.bin` (raw), `bin/firmware.packed.bin` (flashable via [uvtools](https://egzumer.github.io/uvtools/)).

Compiler flags: `-Os -mcpu=cortex-m0 -fshort-enums -flto=auto -std=c2x`. No floating point (Cortex-M0). CI runs on release via `.github/workflows/main.yml`.

## Critical Constraint: Flash Budget

Flash is 60KB (61439 bytes per flasher). The firmware currently uses **61384 bytes (56 bytes free)** after swapping the About app for the restored Generator app. Every byte counts. When adding code:
- Avoid new string literals, use existing ones where possible
- Prefer direct register writes over `BK4819_SetRegValue()` with `RegisterSpec` (the string name field costs flash)
- Combine code paths, use lookup tables instead of switch/if chains
RAM is 16KB. All allocation is static (no malloc). Structures in `settings.h` are heavily bit-packed.

Size-saving techniques that were used and should be continued:
- Reuse existing functions with small overlays instead of duplicating (e.g. `BK4819_PrepareTransmit()` + `BK4819_DigitalTxSetup()`)
- Precomputed `static const` lookup tables instead of runtime bit-shifting with ternaries
- Direct `BK4819_WriteRegister()` calls instead of `BK4819_SetRegValue(RegisterSpec)` when the `RegisterSpec` name string isn't needed for UI display

## Architecture

### Three-Layer Runtime Stack

1. **Scheduler** (`src/scheduler.h/c`) — Priority-based task system. Max 32 tasks. 1ms SysTick drives countdown. Lower priority number = runs first.

2. **Services** (`src/svc.h/c`) — 9 background services with Init/Update/Deinit lifecycle, registered as continuous scheduler tasks:
   - `SVC_KEYBOARD` (pri 0) → `SVC_LISTEN` (50) → `SVC_SCAN` (55) → `SVC_FC` (57) → `SVC_BEACON` (58) → `SVC_BAT_SAVE` (60) → `SVC_APPS` (100) → `SVC_SYS` (150) → `SVC_RENDER` (255)
   - Toggled with `SVC_Toggle(svc, on, interval_ms)`

3. **Apps** (`src/apps/apps.h/c`) — 18 apps with LIFO stack (8 deep). Each app implements: `init()`, `update()`, `render()`, `key()`, `deinit()`. Run with `APPS_run(APP_TYPE)`, exit with `APPS_exit()`. About app was removed to free flash for the restored Generator app.

### Radio Control (`src/radio.h/c`)

Central radio abstraction over three RF chips:
- **BK4819** (`src/driver/bk4819.c`) — Primary transceiver, 87 registers via 3-wire SPI. Handles FM/AM/USB/LSB/BYP/RAW/WFM/DIG modes. Note: `BK4819_WriteRegister()` reads-before-write and skips if unchanged — this is intentional for SPI efficiency but means the call is not free
- **SI4732** (`src/driver/si473x.c`) — Multi-band receiver (AM/FM/SSB), used below `SI_BORDER` (3 MHz)
- **BK1080** (`src/driver/bk1080.c`) — FM broadcast receiver

`RADIO_Selector()` auto-selects chip by frequency and modulation. Key frequency boundaries in `src/config.h`:
- `BEKEN_BORDER` (1.588 MHz), `SI_BORDER` (3 MHz), `WFM_LOW/HI` (64-108 MHz)

### Modulation System

`ModulationType` enum in `src/driver/bk4819.h`: FM(0), AM(1), LSB(2), USB(3), BYP(4), RAW(5), WFM(6), PRST(7), DIG(8).

Mode arrays in `radio.c` (`MODS_BK4819`, `MODS_BOTH`, etc.) control which modes appear per radio chip. `getNextModulation()` cycles through available modes. DIG mode only available on BK4819.

### EEPROM Layout (`src/settings.h`)

Offset 0: `Settings` → `VFO[2]` → `Preset[29]` → Channels (from end). Total patch size: 15832 bytes. All structs are `__packed__` with bit-fields. `ModulationType` fields are 4 bits (supports values 0-15).

### Display

128×64 monochrome LCD via ST7565 driver. 8-row page buffer. `UI_*` functions in `src/ui/components.c`. `gRedrawScreen` flag triggers refresh.

### Key Input

16 keys (0-9, MENU, UP, DOWN, EXIT, STAR, F + PTT, SIDE1/2). Apps receive key events via `key(KEY_Code_t, bKeyPressed, bKeyHeld)`.

## Digital Mode (MOD_DIG)

Flat audio passthrough for external TNC (M17/9600 GFSK). No on-device codec — the BK4819 is configured with disabled speech filters (pre-emphasis, 300Hz HPF, 3kHz LPF, AGC, ALC). Requires mobilinkd hardware modification. MIC ADC bias initialized lazily on first DIG TX via `BK4819_InitMicBias()`.

### DIG Register Management

DIG mode modifies shared BK4819 registers that must be properly saved/restored:

- **REG_7E bits [5:0]** (speech filters) and **REG_2B** (sub-audio/DC filters): Saved on first DIG filter entry in `BK4819_SetFilterBandwidth()`, restored when switching to non-DIG bandwidth. DIG path uses live reads (not saved values) to preserve AGC bits set by `BK4819_SetAGC()`.
- **REG_7D** (MIC sensitivity): Saved/restored by `BK4819_DigitalTxSetup()` / `BK4819_DigitalTxCleanup()`.
- **`gIsListening` state**: Must be set to `false` after DIG TX so `SVC_LISTEN` triggers the full audio re-enable path (`toggleBK4819(true)` → AF DAC + AF bit + speaker). Without this, `RADIO_ToggleRX(true)` is a no-op because `gIsListening` was never cleared during DIG TX entry.
- **`RADIO_SetupBandParams()` order**: `SetFilterBandwidth` must run before `SetGain` so filter save/restore happens before `SetAGC` does read-modify-write on REG_7E upper bits.

## Adding a New Modulation Mode

1. Add enum value to `ModulationType` in `bk4819.h` (after MOD_PRST to preserve EEPROM compat)
2. Add AF type mapping in `modTypeReg47Values[]` in `bk4819.c`
3. Add label string to `modulationTypeOptions[]` in `radio.c` (update array size in `radio.h`)
4. Add to appropriate `MODS_*` arrays in `radio.c`
5. Handle in `BK4819_SetModulation()` if special register config needed
6. Adjust `presetcfg.c` menu size if the new mode shouldn't appear in preset config
