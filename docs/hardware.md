# Hardware

What is inside the F286 and where it sits in the address space. Facts tagged **hw** were checked on
the phone (stock firmware or FPLinux), **static** come from the stock firmware's code, **guess** are
inferred.

## Parts

| part | what | notes |
|---|---|---|
| SoC | Unisoc UMS9117, sold as T107 | single Cortex-A7, GIC-400; FPLinux's OPP table has 768 MHz and 1 GHz |
| baseband | WTL CP subsystem ("whale" in the firmware): two CEVA DSPs (LTE and GSM, called LDSP and TGDSP), two small XDSP cores used by WCDMA L1, an RF transceiver | no separate Cortex-R modem CPU: the whole protocol stack runs on the A7 inside the MocorOS kernel (static). See [radio.md](radio.md) |
| BT/FM | Cortex-M4 subsystem ("CM4", WCN) with 66 KB IRAM | runs its own firmware from the `wcn_fw_*` partitions |
| PMIC | Spreadtrum SC2720 | on the ADI bus; regulators, RTC, watchdog, charger, fuel gauge, ADC, keypad backlight, vibrator. See [pmic-watchdog.md](pmic-watchdog.md) |
| RAM | 48 MB, at 0x80000000-0x82FFFFFF | the bus window is 64 MB and wraps; the stock kernel puts its mode stacks just below 0x83000000 (static) |
| flash | 128 MiB SPI-NAND, READ_ID `b0 15` | 2048 + 64 bytes per page, 64 pages per block, 1024 blocks (hw). Features after boot: A0 = 0x3e, B0 = 0x10, C0 = 0x00 |
| display | 240x320, GC9307 controller on the parallel LCM/DBI bus | panel ID 0x009307 read from FPLinux (hw). See [display.md](display.md) |
| keypad | 5x5 matrix on the SoC keypad controller at 0x40250000 | power/red key is not in the matrix |
| storage | microSD slot (SDIO0); internal FAT drive on NAND | stock calls the SD card `E:` and internal storage `D:` (68 MB, the `userdata` partition) |
| USB | MUSB-style controller at 0x20200000, USB 2.0 device | see [usb.md](usb.md) |
| security | no secure boot seen | DHTB hash fields are zero and a patched `mmi_app` boots (hw) |

## Register blocks

The AON APB (0x402E0000) and PMU (0x402B0000) blocks have set and clear aliases: writing a mask to
base + 0x1000 sets those bits, writing it to base + 0x2000 clears them. Stock code uses the
aliases almost everywhere, so a write to 0x402B2034 means "clear bits in PMU 0x034". Other blocks
are not known to have them.

## Memory map

From the stock kernel's own accesses, FPLinux's device tree and the SoC map. Bases only; the
pages linked in the last column have register detail.

| base | size | block | notes |
|---|---|---|---|
| 0x00000000 | 0x12C00 | on-chip shared RAM (75 KB) | AP view. Sub-areas below. Stock copies its vectors to 0x0 at start |
| 0x0000DC00 | 2 KB | DSP boot RAM | |
| 0x0000E400 | 10 KB | LTE DSP share | stock LTE DSP command/response rings live here (hw), see [radio.md](radio.md) |
| 0x00010C00 | 4 KB | GSM DSP share | |
| 0x00011C00 | 4 KB | multimode share, includes BT/WTL comms | |
| 0x12001000 | 0x2000 | GIC-400 distributor and CPU interface | |
| 0x20000000 | | mirror of the shared RAM (AON side) | |
| 0x20100000 | | AP DMA | |
| 0x20200000 | | USB controller | |
| 0x20300000 | | SDIO0 (SD card) | |
| 0x20800000 | | LCDC (display controller, layers) | [display.md](display.md) |
| 0x20900000 | | rotation engine | |
| 0x20A00000 | | LCM (panel bus) control | [display.md](display.md) |
| 0x20C00000 | | JPEG codec | |
| 0x20E00000 | | AP AHB registers (clock/enable bits) | |
| 0x21500000 | | AP clock control | |
| 0x21A00000 | | NAND controller | |
| 0x33000000 | | hardware spinlock | |
| 0x40000000 | | audio | |
| 0x40050000 | | AON timer | |
| 0x400A0000 | | mailbox (AP, CM4, GSM DSP, LTE DSP) | [radio.md](radio.md) |
| 0x40220000 | | AP timer 0 | stock OS tick (32 kHz one-shots, tickless) |
| 0x40230000 | | system counter | |
| 0x40240000 | | eFuse | |
| 0x40250000 | | keypad | below |
| 0x40280000 | | GPIO | |
| 0x402A0000 | | pin mux and pad config | |
| 0x402B0000 | | PMU (power domains, sleep) | set/clear aliases |
| 0x402C0000 | | EIC2 | |
| 0x402D0000 | | AON clock control | |
| 0x402E0000 | | AON APB (resets, DSP boot vectors, clock gates) | set/clear aliases |
| 0x40300000 | | thermal sensor | |
| 0x40310000 | | AP CPU watchdog (keyed) | [pmic-watchdog.md](pmic-watchdog.md) |
| 0x404D0000-0x404F0000 | | WTL CP / GSM DSP / LTE DSP interrupt controllers | |
| 0x40600000 | 0x400 | ADI master | [pmic-watchdog.md](pmic-watchdog.md) |
| 0x40608000 | 0x1000 | SC2720 register window over ADI | [pmic-watchdog.md](pmic-watchdog.md) |
| 0x50800000 | 66 KB | CM4 IRAM | stock writes the CM4 reset vector at +0 |
| 0x50820000 | | CM4 AHB registers | any read from 0x5082_0000 upward hangs the stock phone (hw) |
| 0x60000000 | | LCM CPU-push window (command), data at +0x20000 | [display.md](display.md) |
| 0x60830000 | | WTL CP AHB registers | |
| 0x70000000, 0x70100000 | | UART0, UART1 | boot1 prints its log on UART1 at 115200 |
| 0x70500000 | | I2C0 | |
| 0x70F00000, 0x71000000, 0x71100000 | | SIM controllers 0-2 | |
| 0x71300000 | | AP APB registers | |
| 0x80000000 | 48 MB | DRAM | layout below |
| 0x88000000 | 15 MB | `mmi_app`, demand-paged from flash | [boot.md](boot.md) |
| 0xD0000000-0xEFFFFFFF | | WTL CP die window | XDSP core A at 0xE0400000, core B at 0xE0480000 |
| 0xFFFF0000 | | BootROM | |

Reading a peripheral whose clock is gated resets the stock phone (hw). On FPLinux, stacked
accesses to 0x515D0000/0x515E0000 have wedged the SoC.

## DRAM layout under the stock OS

| address | size | user |
|---|---|---|
| 0x80000000 | 0x20000 | zeroed at start; shared buffers between AP and CM4 |
| 0x80020000 | 0xC0000 | CM4 firmware |
| 0x800E0000 | | GSM DSP window |
| 0x80100000 | 0x300000 | GSM DSP image, entry +0x20080. boot1 also runs from here before the kernel |
| 0x80400000 | | LTE DSP window |
| 0x80447000 | 0x1000 | LTE DSP handshake cells (sleep sync word at 0x80447E14, awake flag 0x80447C40) |
| 0x80448000 | 0x2E4000 | LTE DSP image, entry 0x80448400 |
| 0x80678000 | ~0x4F000 | LTE RF calibration, copied from NV by the AP before release |
| 0x8073B000 | 4 x 0x2800 | LTE split-message slots |
| 0x80745000 | 10 x 0x2800 | LTE uplink data slots (ends 0x8075E000) |
| 0x80A05E00 | ~12 MB | kernel image, DHTB header included; code from 0x80A06000 |
| 0x81700000 | 0x5E1B4 | kernel RW data, decompressed at start |
| 0x82FE0000 | 0x20000 | zeroed at start |

A Linux system that wants to boot the DSPs has to keep its RAM clear of 0x80000000-0x80760000.

## Keypad

The keypad controller is at 0x40250000; the key status register at 0x4025002C holds one byte per key slot.
It never reads 0 at rest on the real phone (0x77777731 was typical); bit 7 of a slot byte means
"held", so test `& 0x80808080`. (hw)

Matrix (column, row), decoded from the stock key map and checked with key presses under FPLinux:

| position | key | position | key |
|---|---|---|---|
| (0,0) | left soft | (1,0) | right soft |
| (1,1) | call | (2,1) | up |
| (3,1) | OK | (3,0) | left |
| (4,0) | down | (4,1) | right |
| column 2 | 1, 4, 7, * | column 3 | 2, 5, 8, 0 |
| column 4 | 3, 6, 9, # | (2,0) | unused |

The red/power key is outside the matrix. The left soft key (top left) is the download-mode key.
The FPLinux keypad driver only scans the rows and columns present in the DT key map, and the
column count must be 5 or the cells shift.

## Interrupts

GIC SPI numbers (add 32 for the GIC interrupt ID). The first group is from FPLinux's device tree
and works on the phone; the second from the stock kernel's registrations.

| SPI | source |
|---|---|
| 11 | I2C0 |
| 28 | AON timer |
| 35 | GPIO |
| 36 | keypad |
| 37 | EIC2 |
| 38 | SC2720 PMIC (RTC, EIC, ADC are sub-interrupts 1, 4, 0) |
| 45 | JPEG |
| 50 | AP DMA |
| 55 | USB |
| 57 | SDIO0 |
| 68 | mailbox |
| 19 | LCDC frame done (stock) |
| 29 | AP timer 0, the stock OS tick (stock) |

## Unknowns

- Camera: stock probes and toggles camera rails at boot; whether a sensor is fitted I have not
  checked. In the emulator all I2C probes NACK, which says nothing about the real board.
- Exact RAM chip and its timing: not read.
- Battery and charger calibration: FPLinux leaves the fuel gauge off because it needs board
  values I have not extracted.
