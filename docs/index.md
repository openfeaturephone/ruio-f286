# Ruio F286 notes

The Ruio F286 is a 4G feature phone: Unisoc UMS9117 (sold as T107) SoC, SC2720 PMIC,
48 MB of RAM, 128 MiB SPI-NAND and a 240x320 screen. It ships with MocorOS, Unisoc's ThreadX-based
feature phone OS. I studied it to add hardware support to FPLinux (an open Linux port for these
chips) and to run my own apps on the stock OS. These pages are what I found.

All of it comes from one phone with one firmware build. Addresses are only valid for that build;
the stock `mmi_app` partition has sha256 `5d534329...`. Where something is guessed and not
measured, the page says so.

## Pages

| page | contents |
|---|---|
| [hardware.md](hardware.md) | SoC, PMIC, RAM and flash, display, keypad, peripherals, physical memory map |
| [boot.md](boot.md) | BootROM to idle screen, load addresses, address frames for the kernel and `mmi_app` |
| [flashing.md](flashing.md) | Download mode, backing up and restoring partitions |
| [firmware-format.md](firmware-format.md) | Partition table, DHTB headers, PAC packages |
| [mmi-api.md](mmi-api.md) | Addresses of MMI functions in `mmi_app` and how each was identified |
| [rtc-alarm.md](rtc-alarm.md) | The PMIC RTC alarm, how it reaches the MMI, a silent periodic wake for apps |
| [usb.md](usb.md) | USB modes and IDs, the UCOM bulk channels, talking to the phone from a PC |
| [pmic-watchdog.md](pmic-watchdog.md) | ADI bus, SC2720 registers and regulators, the watchdogs, rebooting remotely |
| [display.md](display.md) | Panel, LCM bus, LCDC layers, taking screenshots |
| [radio.md](radio.md) | Baseband DSP and CM4 power-on, LTE DSP rings, what works under Linux and what doesn't |

## Related projects

All under [github.com/openfeaturephone](https://github.com/openfeaturephone/):

- [mocor-sdk](https://github.com/openfeaturephone/mocor-sdk): run your own native apps on stock
  MocorOS. A patcher for your own `mmi_app` dump, the API headers from [mmi-api.md](mmi-api.md)
  and example apps.
- [f286dbg](https://github.com/openfeaturephone/f286dbg): a debug agent that runs on the phone and
  a PC client. Read and write memory, call firmware functions, move files, take screenshots over
  USB ([usb.md](usb.md)).
- [mocor-emu](https://github.com/openfeaturephone/mocor-emu): a Unicorn-based emulator that boots a
  stock MocorOS dump to the idle screen. Most "emu" results on these pages came from it.
- [tinymtproto](https://github.com/openfeaturephone/tinymtproto): a small Telegram (MTProto 2.0)
  client library in C.
- [f286-telegram](https://github.com/openfeaturephone/f286-telegram): the Telegram app for the
  F286, built on tinymtproto.
- [fplinux](https://github.com/openfeaturephone/fplinux): the Linux port the hardware notes feed.

## Conventions

- Addresses are hexadecimal physical or runtime virtual addresses as the stock OS sees them. On
  this SoC the stock kernel maps most I/O 1:1, so a register address is the same in both.
- "Real VA" for the kernel means `kernel.bin` file offset + `0x80a05e00`. Some older notes used an
  offset that was 0x38 lower; [boot.md](boot.md) explains where that came from.
- Thumb functions are called with bit 0 set. Where a function is ARM code, the page says so.
- Register names in capitals are the SoC's own names where I know them; lower-case names are
  mine.
- Tags used on several pages: **hw** = seen on the real phone, **emu** = seen running the real
  firmware in the emulator, **static** = read from the code, **guess** = inferred.
