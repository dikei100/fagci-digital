# Reborn firmware - Digital Mode Fork

Fork of [fagci/uvk5-fagci-reborn](https://github.com/fagci/uvk5-fagci-reborn) adding digital mode (MOD_DIG) support for the Quansheng UV-K5/K6 amateur radio.

## What this fork adds

**MOD_DIG** - a flat audio passthrough mode for 9600 baud GFSK / M17 4-FSK digital communications via an external TNC (e.g. Mobilinkd TNC4).

The BK4819 transceiver is configured with all speech-optimized DSP disabled:
- Pre-emphasis / de-emphasis off
- 300 Hz HPF and 3 kHz LPF bypassed
- AGC, ALC, and compander disabled
- DC and sub-audio filters disabled during TX
- Squelch forced open, DTMF/tone detection off
- AFC enabled (FM demodulator path)

This gives a clean baseband path suitable for external modems. No on-device codec is involved.

### Other changes from upstream

- Generator app removed to free flash space for DIG mode register management code
- DIG mode added to CHIRP module

## Required hardware modification

DIG mode requires a hardware mod to route audio between the radio and an external TNC. This is the same modification used by the Mobilinkd TNC4 for M17 digital voice on HTs:

1. **TX audio input** - Tap the microphone ADC line (before the BK4819 MIC input) to inject baseband TX audio from the TNC
2. **RX audio output** - Tap the speaker/AF output line (after the BK4819 AF DAC) to feed received baseband audio to the TNC
3. **PTT control** - The TNC needs to key the radio via the PTT line

Refer to [Mobilinkd's documentation](https://mobilinkd.com/) for detailed wiring instructions for the UV-K5 hardware mod. The modification typically involves soldering to test pads or component legs on the radio's PCB.

> **Note:** Without this hardware modification, DIG mode will not produce useful results. The mode is designed for direct baseband I/O with external hardware.

## How to use DIG mode

1. Flash this firmware to your UV-K5/K6 (see Build section below)
2. Connect a Mobilinkd TNC4 (or compatible TNC) via the hardware mod
3. Cycle modulation modes until you reach **DIG**
4. Select bandwidth: **Wide** (25 kHz, for 9600 GFSK) or **Narrow** (12.5 kHz)
5. Use M17 software on your phone/computer connected to the TNC (e.g. M17 client apps)

### TX behavior

- On first DIG mode TX, the MIC ADC bias is initialized (one-time 250 ms delay)
- TX filters, sub-audio injection, and scrambler are automatically disabled
- Audio path stays active during TX for fast turnaround (~79 ms vs ~378 ms stock)
- All modified BK4819 registers are saved and restored on TX exit

## Build

```bash
git submodule update --init --recursive   # first time (CMSIS_5, printf)
pip install crcmod                         # needed by fw-pack.py
make                                       # build firmware
```

Toolchain: `arm-none-eabi-gcc` (CI uses 13.3.Rel1). Output artifacts:
- `bin/firmware.bin` - raw binary
- `bin/firmware.packed.bin` - flashable via [uvtools](https://egzumer.github.io/uvtools/)

Flash budget: 60 KB (61439 bytes max). Currently ~444 bytes free.

## Original project

Based on [DualTachyon/uv-k5-firmware](https://github.com/DualTachyon/uv-k5-firmware).

Upstream firmware: [fagci/uvk5-fagci-reborn](https://github.com/fagci/uvk5-fagci-reborn) (maintainer has moved to [s0v4](https://github.com/fagci/s0v4)).

[Telegram group](https://t.me/uvk5_spectrum_talk) | [Donations](https://t.me/uvk5_spectrum_talk/6032/6035)

Chirp module by [Giorgio IU0WQJ](https://github.com/gpillon/)

## License

Apache License. Main author: [Dual Tachyon](https://github.com/DualTachyon).
