# PMIC, ADI and watchdogs

The SC2720 PMIC sits behind the SoC's ADI bus controller. This page covers the bus, the PMIC
registers that matter, the regulators, and the two watchdogs, including the remote reboot that
works on the phone. Kernel addresses are real VAs (file offset + 0x80A05E00).

Tags: **hw** = seen on the phone, **emu** = measured running stock firmware in the emulator,
**static** = read from code, **guess** = inferred.

## ADI bus

ADI master at 0x40600000. The PMIC's 16-bit registers appear in a window at 0x40608000, so PMIC
register R is at ADI address `0x40608000 + R`. This matches mainline Linux `spi-sprd-adi`.

- Read: write the ADI address to 0x40600028, poll 0x4060002C until bit 31 is set, then
  bits 15:0 are the data and bits 31:16 echo the address.
- Write: wait until the write FIFO has room (0x40600030 bit 11), then a plain store into the
  window.
- Stock read and write wrappers: 0x81218546 and 0x81218598. All PMIC writes the kernel makes go
  through the second one.

ADI master setup done by boot0, boot1 and the kernel (static): `0x40600008 = 0`, `0x4060000C = 0`,
`0x40600020 &= ~0x40000000` (CLK_ALL_ON off), `0x40600024 |= 0x80000000`.

The kernel also maps hardware channels, letting other blocks write PMIC registers without the
CPU (emu + static):

| channel | PMIC target | probable use (guess) |
|---|---|---|
| 2 | 0xC68 | PA supply voltage, written by the RF hardware |
| 0x1D, 0x1E | 0xC20 | PMU sleep / power-down handshake |
| 0x20, 0x21 | 0xCC0, 0xCCC | SIM 0/1 supply |
| 0x23, 0x26 | 0xD0C, 0xD00 | SD card supplies |

Channel addresses go in 0x40600044 (channel 2), 0x406000B0-0x406000D4 (channels 0x1D-0x26), with
enables in 0x40600040 and 0x4060020C. Mainline exposes the same thing as the `sprd,hw-channels`
property. FPLinux doesn't set any (hw: its log says so).

## SC2720 register blocks

| offset | block |
|---|---|
| 0x040 | watchdog (below) |
| 0x0C0 | interrupt controller; 0x0C8 = INT_EN (bit 1 RTC, bit 4 EIC) |
| 0x180-0x1D8 | backlight (WLED) and RGB LED |
| 0x200 | RTC ([rtc-alarm.md](rtc-alarm.md)) |
| 0x280 | EIC (PMIC GPIO/edge inputs) |
| 0x300 | eFuse |
| 0x400 | AUXADC (registers at +4 offsets) |
| 0x600-0x640 | PMIC pin config |
| 0x700-0x764, 0x800 | audio codec |
| 0xA00 | fuel gauge |
| 0xC00 / 0xC04 | chip ID, reads 0xA003 / 0x2720 |
| 0xC08 | MODULE_EN0: bit 1 RTC, bit 2 watchdog, bit 3 EIC, bit 5 ADC, bit 7 fuel gauge, bit 9 WLED |
| 0xC0C | ARM_CLK_EN0: bits 5, 6 ADC clocks |
| 0xC10 | RTC_CLK_EN0: bit 1 RTC, bit 2 watchdog, bit 3 EIC, bit 6 fuel gauge, bit 7 WLED |
| 0xC1C | power-down bits for the DC-DC converters; bit 7 = dcdcgen |
| 0xC54 | dcdcgen voltage, stock 0x2E |
| 0xC5C-0xD34 | regulator power-down and voltage pairs (table below) |
| 0xD68-0xDD8 | what powers down in deep sleep |
| 0xDE8 | XTL_WAIT_CTRL0; bit 8 forces the 26 MHz crystal buffer on |
| 0xDF8 / 0xDFC | keypad backlight |
| 0xE00 | vibrator |
| 0xE0C, 0xE10, 0xE14, 0xE18 | charger |
| 0xE24 | reset monitor / mode field ([boot.md](boot.md)) |
| 0xE30, 0xE38 | power-on reason and long-press reset flags |
| 0xE40 | ARCH_EN; bit 0 = arch timer clock |
| 0xE88 | RF control bits; stock sets bit 5, clears bit 3 during radio-on |

## Regulators

Bit 0 of each regulator's control register is a **power-down** bit: 1 = off. FPLinux's SC2720
regulator driver treats it as inverted for that reason, and stock's default table writes 0 to
exactly the rails it keeps on (static + hw). Some older notes of mine had this backwards.

| PD register | VOL register | rail | stock state before the LTE DSP starts | supplies (guess where marked) |
|---|---|---|---|---|
| 0xC1C bit 7 | 0xC54 | dcdcgen | on, 0x2E | 1.8 V buck feeding the 1.8 V LDOs (guess) |
| 0xC60 | 0xC64 | dcdcwpa | on | RF power amplifier supply |
| 0xC8C | 0xC90 | vddcamio | off | camera IO |
| 0xC94 | 0xC98 | vddrfa1v8 | on | RF transceiver analogue 1.8 V |
| 0xC9C | | unnamed | off | unknown |
| 0xCA4 | 0xCA8 | vddcamcore | on, 0x24 (1.25 V) | camera core |
| 0xCAC | 0xCB0 | vddwcn | off until BT is used | CM4 / BT / FM |
| 0xCC0 | 0xCC4 | vddsim0 | on once the SIM driver starts | SIM 0 |
| 0xCCC | | vddsim1 | off | SIM 1 |
| 0xCD8 | 0xCDC | vddsim2 | on, 0x5F | named SIM 2; use unknown |
| 0xCE0 | 0xCE4 | vddcama | off | camera analogue |
| 0xCE8 | | unnamed | off | unknown |
| 0xCF4 | | vddemmccore | off | unused (NAND phone) |
| 0xD00 | | vddsdcore | on | SD card |
| 0xD0C | 0xD10 | vddsdio | on | SD card IO |
| 0xD1C | | vddwifipa | off | unused |
| 0xD30 | 0xD34 | vddusb | off | USB 3.3 V |

The core, ARM and memory supplies are hardware-managed; the stock software never writes them.
Stock sets the rail defaults in one table walk about 44 ms after the kernel starts (emu). The radio
power-on at about 0.9 s turns dcdcwpa on again and sets 0xC5C bit 7 / clears bit 5.

On FPLinux dcdcwpa read as off (0xC60 = 0x0079) (hw). That is one of the known differences on
the radio path ([radio.md](radio.md)).

## PMIC watchdog and remote reboot

SC2720 watchdog at PMIC 0x040. The layout matches mainline `sprd_wdt` with base 0x40; LOCK and
CTRL are confirmed in the stock driver, the rest follows mainline.

| PMIC offset | ADI address | register |
|---|---|---|
| 0x040 / 0x044 | 0x40608040 / 44 | load, low / high |
| 0x048 | 0x40608048 | CTRL: bit 0 interrupt enable, bit 1 run, bit 2 new mode, bit 3 reset enable |
| 0x04C | 0x4060804C | interrupt clear |
| 0x058 / 0x05C | | counter, low / high |
| 0x060 | 0x40608060 | LOCK: write 0xE551 to open, 0x1AAE to close |
| 0x06C / 0x070 | | interrupt load, low / high |

The module and clock enable (0xC08 and 0xC10 bit 2) are already set by boot0/boot1.

The stock kernel driver's configuration function is at 0x8136F3F0 (Thumb):

```c
uint32_t wdg_config(const uint32_t cfg[3]);   /* {run, mode, load in 32 kHz ticks} */
```

`{1, 0, 0x666}` (mode 0 = interrupt + reset, 50 ms) resets the whole phone: the PMIC reset brings
it back through the BootROM and a normal boot. This is the same sequence as the Linux `sprd-adi`
restart handler. On the phone it worked three times out of three, and the debug agent answered
again 18 s later each time. A dry run with run = 0 sets CTRL to 0xD and writes no load. After a
watchdog reset, CTRL reads 0.

What did not work: a "dead-man" timer that re-armed this watchdog every second with a 30 s load.
Left unattended, the phone ended up dropping off USB and needed a battery pull; the cause is not
established. My suspicion is mode 0's interrupt, which fires shortly before the reset and may make
the stock OS record an abnormal restart. A redesign would use reset only, without INT_EN.

Don't try to reset the phone by faulting it. A store to a read-only `mmi_app` page livelocks in the
pager instead of resetting ([boot.md](boot.md)).

## AP CPU watchdog

The SoC's own CPU watchdog at 0x40310000 is keyed the same way: 0xE551 to 0x40310020 opens it,
0xFFFF1AAE closes it. CTRL is at 0x40310008. The stock assert handler clears bits 2:0 of CTRL to
stop it, which is why an assert leaves a frozen screen with no reset. The stock code that touches
this block is ARM code. Vendor FDL2 builds also configure it. (static)

## Not done

- POR values of most PMIC registers on the real chip: the emulator returns the last write, so the
  "stock state" above is the set/clear masks stock applies, not a full read-back.
- What the bits of 0xE38 do. Stock boot sets bits 0 and 9; the FPLinux FDL1 clears bits 0 and 8
  instead. Probably a reset or power-key control (guess).
