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

## Installing and running RT809H.exe under Wine

Confirmed working setup: Wine 9.0, Linux Mint 22.2 (Ubuntu 24.04 base).

### 1. Install the software

Copy the official RT809H software folder as-is into a Wine prefix (32-bit,
since `RT809H.exe` is a 32-bit Windows app) — `RT809H.exe`, its bundled
`FTD2XX.dll`, and the `LIB/` folder of `.DAT` algorithm files all need to
stay together in the same directory, exactly as they ship from the vendor.
No installer is required; it runs directly from the folder.

### 2. Give Wine raw access to the FTDI chip

The RT809H programmer identifies on USB as vendor `0403` / product `6010`
(an FTDI FT2232-series dual-channel chip). By default, Linux's kernel
`ftdi_sio` driver claims this device automatically and exposes it as a
serial (`/dev/ttyUSB*`) port — but `FTD2XX.dll` (running under Wine) needs
raw, exclusive USB access instead, so the kernel driver has to be kept off
the device, and the device node itself needs to be readable/writable by a
normal user (not just root).

Create `/etc/udev/rules.d/99-rt809h.rules`:

```
SUBSYSTEM=="usb", ATTR{idVendor}=="0403", ATTR{idProduct}=="6010", MODE="0666"
SUBSYSTEM=="usb", DRIVER=="ftdi_sio", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6010", RUN+="/bin/sh -c 'echo $kernel > /sys/bus/usb/drivers/ftdi_sio/unbind'"
```

- Line 1 makes the raw USB device world-accessible (`0666`), so libusb can
  open it without root.
- Line 2 fires whenever the kernel's `ftdi_sio` driver binds to this
  specific VID:PID and immediately unbinds it, so it doesn't hold the
  device before Wine gets a chance to claim it.

Apply it:

```
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Then unplug and replug the RT809H (or power-cycle it) so the new rule
applies to a fresh device attach.

### 3. Run it

```
wine RT809H.exe
```

launched from inside the software's folder. With the device plugged in and
the udev rule applied, the software detects the programmer and can
read/write chips normally — confirmed end-to-end by reading a full eMMC
dump from a real TV mainboard.

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
