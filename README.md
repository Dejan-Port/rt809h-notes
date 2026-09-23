# RT809H notes

Notes and reverse-engineering findings about the RT809H universal programmer
(the hardware used to read/write eMMC, SPI NOR, NAND, and EEPROM chips —
commonly used by TV/phone repair technicians).

This is **not** a driver replacement, an official tool, or a clone of RT809H's
device database. It's a running log of what's been confirmed about how the
official Windows software talks to the hardware, gathered while exploring the
feasibility of a Linux-native tool for the same programmer.

## Wine compatibility

The official `RT809H.exe` software **runs correctly under Wine 9.0** on Linux
Mint 22.2 (Ubuntu 24.04 base). Confirmed by successfully reading a full eMMC
dump from a real TV mainboard through a physical RT809H programmer connected
via USB — no crashes, no missing driver errors.

## How the software talks to the hardware

Static analysis of `RT809H.exe` (17MB, packed/obfuscated — ~27,000 strings,
no readable device identifiers) shows it statically imports `FTD2XX.dll` and
specifically uses:

- `FT_SetBitMode` / `FT_GetBitMode`
- `FT_Write` / `FT_Read`

(confirmed via `objdump -p` against the ordinal-mapped export table of the
bundled `ftd2xx.dll`).

**This means the RT809H hardware is an FTDI chip running in bit-bang mode** —
the software directly writes/reads individual GPIO pin states as raw bytes
over the FTDI interface. FTDI bit-bang mode is publicly documented (FTDI
Application Note AN232R-01), and [libftdi](https://www.intra2net.com/en/developer/libftdi/)
(open source, Linux) speaks the same wire protocol as the closed-source
`FTD2XX.dll`.

## What's still unknown

The software side is understood; the **hardware pinout is not**. Specifically:
which physical pin of the FTDI chip is wired to which programmer signal
(CLK / CMD / DAT0-3 for eMMC, CS/CLK/MOSI/MISO for SPI NOR, etc.) on the
RT809H PCB itself. Determining this requires continuity testing with a
multimeter against a real unit.

## Other findings

- `LIB/ALGO_EMMC*.DAT` (8 variants: 1BIT / 4BIT / ISP / BOOT_ISP / 1.8V /
  SYNC / SYNC_2 / MST_DEBUG_ON) — the first 644 bytes are a shared header,
  everything after that is encrypted (>60% byte-level difference even
  between the 1BIT and 4BIT variants). Not readable bytecode; cracking this
  would require reversing the `RT809H.exe` loader itself — not attempted,
  and not the goal here.
- `DEVICE.INI` is **not** the main eMMC device database — it's a template
  for community-contributed SPI NOR flash chip definitions.

## Goal

The eventual goal (separate, ongoing project) is a Linux-native, open tool
for reading/writing eMMC chips in-circuit, built around an ESP32-S3's
hardware SDMMC peripheral rather than reverse-engineered RT809H hardware —
this repo just documents what was learned about the RT809H side along the
way, in case it's useful to others.

## License

Findings documented here are shared as-is for reference. No RT809H
proprietary code, device database, or encrypted `.DAT` content is included
or reproduced in this repository.
