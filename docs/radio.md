# Radio: baseband DSPs and CM4

How the stock firmware powers up and talks to the baseband, and how far FPLinux has got
reproducing it. This is the least finished page. Every statement carries a tag:

- `hw-stock`: seen on the real phone running stock MocorOS (register samples and memory read
  over USB by an on-phone logger).
- `hw-linux`: seen on the real phone running FPLinux with our radio driver.
- `emu`: measured running the stock firmware from reset in the emulator. The AP code is real;
  peripheral read-backs are modelled, so values of read-modify-writes may differ on silicon.
- `static`: read from the kernel code.
- `guess`: inferred.

Kernel addresses are real VAs (file offset + 0x80A05E00). Registers in the PMU (0x402B0000) and
AON APB (0x402E0000) blocks are often written through their set (+0x1000) and clear (+0x2000)
aliases; `0x402B2034 = 0x03000000` means "clear bits 25:24 of PMU 0x034".

## What is in there

| core | firmware | DRAM image | entry | rings / shared memory |
|---|---|---|---|---|
| LTE DSP (CEVA, "LDSP") | `fw_dsp_enc`, 0x2E4000 bytes copied | 0x80448000 | 0x80448400 | on-chip 0xE400-0x10BFF (static, hw-stock) |
| GSM DSP (CEVA, "TGDSP") | `fw_modem_enc` | 0x80100000 | 0x80120080 | on-chip share at 0x10C00 (static) |
| XDSP core A, B | four small images inside the kernel | WTL die, 0xE0400000 / 0xE0480000 | | used by WCDMA layer 1 (static) |
| CM4 (BT/FM) | `wcn_fw_*` | 0x80020000 | reset vector copied to 0x50800000 | shared buffers in 0x80000000-0x8001FFFF |

There is no separate modem CPU and no encryption: the full protocol stack (NAS, RRC, L2, L1
control) runs on the A7 inside the MocorOS kernel, and the DSP partitions are plain images
(static). The AP loads all firmware, writes boot vectors and releases resets.

## Status

| step | stock | FPLinux |
|---|---|---|
| WTL CP power domain on | yes (hw-stock) | yes, with the "stock early" sequence: PMU 0x0C4 = 0, power-ok (hw-linux) |
| XDSP cores A and B loaded, ready 1 then 2 | yes (emu, static) | yes (hw-linux) |
| GSM DSP released | yes, 0.96 s (emu) | yes |
| LTE DSP released | yes, ~5.0 s after power-on (hw-stock) | yes |
| LTE DSP answers VERSION | 57 ms after release (hw-stock) | **no**: the DSP writes nothing, and about 1.1 s after the release the whole AP hangs (hw-linux) |
| camped on a cell | about 0.5 s after the release (hw-stock) | no |

The hang after the release is the current wall. No IRQ storm, mailbox interrupts at 0, PMU status
static, no panic; the USB gadget stays enumerated, so the AP is hung, not reset (hw-linux). The
fixed ~1.0-1.2 s delay looks like a watchdog; PMU 0x402B3080 bit 4 (an LTE watchdog reset flag) is
the next thing to read (guess).

## Stock start-up order

From the emulator run of stock firmware (times = instructions / 2e8, emu) with the LTE release
time taken from the phone (hw-stock).

| t | step |
|---|---|
| 0.044 s | PMIC rail defaults ([pmic-watchdog.md](pmic-watchdog.md)) |
| 0.89 s | CM4 start (below) |
| 0.898 s | power-domain init: `0x402B2044 = 0x02000000`, `0x402B20B0 = 5` (release WTL CP system and DSP system resets), `0x402B20CC = 0x20000`, `0x402B2038 = 0x03000000` (GSM DSP domain on), then for each of PMU 0x034, 0x028, 0x02C, 0x04C, 0x050, 0x03C: set bit 25 (0x402B1xxx = 0x02000000), clear bit 24 (0x402B2xxx = 0x01000000) |
| 0.899 s | radio-on: WTL die power request (0x60830044 bit 6), die clock and reset block, 0x402E0244 = 0x10100, 0x404D0008 = 0x8C |
| 0.906 s | PMIC: dcdcwpa on, 0xC5C bit 7 set / bit 5 clear, 0xE88 = (v & ~8) \| 0x20 |
| 0.906-0.925 s | RF init from NV, transceiver programmed over its SPI engine |
| 0.94 s | mailbox init |
| 0.964 s | GSM DSP release |
| 0.966-0.998 s | XDSP core A, then core B, then their follow-up configuration |
| ~5.0 s | LTE DSP release (phone; ~21 s in the emulator) |

Between 1.0 s and the LTE release the AP makes no further writes to the WTL die and no PMIC writes
related to the radio (emu).

## Power and reset registers

| register | role | notes |
|---|---|---|
| PMU 0x0B0 | CP soft reset | bit 0 WTL CP, bit 2 DSP system, bit 8 BT system. Reset value 0x507; stock running reads 0x2 (hw-stock) |
| PMU 0x0C0 | power status | 5-bit state per domain, 0 = on, 7 = off. Bits 4:0 HU3G, 14:10 LTE DSP, 29:25 LTE_P1 |
| PMU 0x0C4 | WTL CP power status | 0 = on, 7 = off |
| PMU 0x0CC | sleep control | bit 17 DSP, bit 21 CM4; stock clears bit 21 before bit 17 |
| PMU 0x0D4 | sleep status | bits 7:4 = 6 when the WTL island is ready |
| PMU 0x028-0x050 | per-domain power config | bit 25 FORCE_SHUTDOWN, bit 24 AUTO_SHUTDOWN_EN, then on/sequence/isolation delays. 0x034 = LTE DSP, 0x038 = GSM DSP, 0x03C = HU3G, 0x028/0x02C/0x04C/0x050 = LTE_P1-P4, 0x044 = WTL CP system |
| AON 0x06C | DSP subsystem clocks | stock does `\|= 0x9620` before each DSP release |
| AON 0x130 | DSP core resets | bit 16 LTE DSP, bit 17 GSM DSP; 1 = held |
| AON 0x248 | LTE DSP boot vector | entry >> 1 (0x40224200); reset value 0x18000000 |
| AON 0x24C | LTE DSP boot control | bit 0 |
| AON 0x250 / 0x254 | GSM DSP boot vector / control | 0x40090040, bit 0 |
| AON 0x114 | CM4 system soft reset | bits 4 and 0 |
| AON 0x0FC | chip ID | 0x98180001 on this phone |

## LTE DSP boot

Stock sequence (static; the result matches a 1 ms register sample on the phone, hw-stock):

1. Register the mailbox callbacks for core 3 (the LTE DSP).
2. First boot only: boot control low (0x24C bit 0 = 0), hold reset (0x130 bit 16 = 1), then
   `0x402B2044 = 0x02000000`, `0x402B2044 = 0x01000000`, `0x402B20CC = 0x00020000`,
   `0x402B2034 = 0x03000000`. On a later re-boot it instead waits for the sleep sync word
   0x80447E14 to read 0x22222222.
3. Calibration download: NV item 0x4C into AP RAM, then about 30 NV items copied back to back
   into DSP memory from 0x80678000, each with an exact size the code asserts. About 320 KB in
   total. The data at 0x80678000 was byte-identical across two stock boots (hw-stock).
4. `AON 0x06C |= 0x9620`.
5. 0x24C bit 0 = 0, 0x130 bit 16 = 1, `0x248 = 0x40224200`, 0x24C bit 0 = 1, 0x130 bit 16 = 0.
   The DSP runs.

On the phone all of step 2 and step 5 land within one 1 ms sample. The boot vector reads back for
about 2 ms, then returns to 0x18000000. The DSP powers LTE_P1 itself about 1 ms after the release;
the AP never writes PMU 0x028 for it (hw-stock).

Stock's LTE boot function is at 0x80E2D9E6 and the four-write power function at 0x80D0FAB4.

## LTE DSP rings

Both rings are in **on-chip shared RAM**, AP physical 0xE400-0x10BFF, not in the DRAM window at
0x8040E400. The live pointers on a running stock phone are 0xE400 (commands) and 0xE478
(responses), and the DRAM window there still holds the 0xAA fill pattern (hw-stock).

Response ring (DSP to AP) at 0xE478:

```
+0       u16 in     DSP: next entry it will fill (0..11)
+2       u16 out    AP: written after consuming an entry, out = (out + 1) % 12
+4 + 8i  entry i:   u16 type, u16 len, u32 addr     (12 entries, 0xE47C..0xE4DB)
body     0xFBEC + addr, len bytes, addr + len <= 0x100C
```

| type | meaning |
|---|---|
| 0 | downlink L1 message: `u16 card, u16 0, u32 msgid, payload` |
| 2 | calibration announce |
| 5 | VERSION: `u16 branch, u16 svn, u16 chip id, u16 chip version, string`; len <= 0x80 |
| 7 | DSP assert |
| 0xA | calibration signature forward |
| 0xB | status |
| 0xF | ignored |

The DSP reuses the body area from offset 0 as soon as `out` catches up, so copy the body before
writing `out`.

Command ring (AP to DSP) at 0xE400:

```
+0       u16 wr     AP: next slot to fill (0..7)
+2       u16 rd     DSP: consumed index
+4 + 8i  slot i:    u16 cmd, u16 len, u32 off       (8 slots)
body     0xE4E0 + off, len bytes, off + len < 0xC80
```

To put: body at 0xE4E0 if `wr == rd`, else at the end watermark stored in `slot[wr].off`; fill
`slot[wr]`; set `slot[(wr+1) & 7].off` to the 4-byte-aligned end of this body plus 4 (0 on wrap);
then `wr = (wr + 1) & 7`. Message commands use cmd 0 and a body header `u16 card, u16 0x5555
(0xAAAA for big payloads held in the slot ring), u32 msgid`.

Kick: a mailbox message with zero content to core 3 (mailbox 0x400A0000; cores 1 = CM4, 2 = GSM
DSP, 3 = LTE DSP). If the DSP is awake (0x80447C40 == 0) stock always kicks; if it sleeps, only
when two or more commands are pending. The DSP side signals new responses by raising `in` and a
mailbox interrupt; a polling driver can just compare `in` with its own cursor. (static)

Stock sends nothing before VERSION arrives (static, hw-stock). On the phone the first exchange was
VERSION 57 ms after the release, then commands 0x87, 0x77, a burst of 0x13, 0x6E, and the cell was
camped about 0.5 s after the release (hw-stock). The VERSION reply on this phone: branch 0x19A0,
svn 0xFC78, chip id 0, chip version 1. In idle the DSP sends a burst of downlink messages about
every 300 ms, and the LTE DSP and CM4 start sleeping about 1.7 s after the release (hw-stock).

A driver that sees no type-5 entry at 0xE478 within ~100 ms of the release has a power, clock or
reset problem, not a protocol one.

## GSM DSP boot

Static, matches the emulator: hold reset (AON 0x130 bit 17), write parameters into the image while
it is held (first boot only: a 0x6F8-byte block at 0x800F9670 with clock, tick and mode values,
plus a few flags at image + 0x44/0x48), boot control low (0x254 bit 0), `AON 0x06C |= 0x9620`,
`0x250 = (0x80100000 + 0x20080) >> 1`, short delay, control high, release bit 17. No PMU writes
here; the domain was powered at 0.898 s. Stock's function is at 0x810B33FE.

## XDSP cores

Two small cores on the WTL die, loaded by the WCDMA layer-1 code (static, emu; hw-linux confirms
the load sequence works):

| core | window | control register | ready word |
|---|---|---|---|
| A | 0xE0400000 | 0xE047F000 | 0xE0419908 |
| B | 0xE0480000 | 0xE04FF000 | 0xE0499540 |

Per core: pulse the control register (0, 1, write 0xF2370000 to window + 0x40, 0, 0x1E, with ~1 us
waits); clear the ready word; zero 0x7FF words at window + 0x40000 and + 0x48000; copy the boot
loader into the window; pulse again; wait for ready = 1; parameter pulse; copy the main image;
pulse; wait for ready = 2. Each core has its own boot loader and main image, and all four exist only
inside the kernel image. Stock then writes some configuration to the running cores (0xE041B2F4,
0xE041AF80, 0xE041B328/32C, 0xE0418CF4, 0xE0498D08) that FPLinux doesn't yet.

## CM4 start

Stock (static, emu): `0x402B20CC = 0x00200000` (clear CM4 sleep bit 21), copy the 8-byte initial
SP and reset PC to CM4 IRAM 0x50800000, `0x402B20B0 = 0x100` (release BT system), then
`0x402E2114 = 0x10` and `= 0x1` (release CM4). The CM4 image itself is loaded to 0x80020000
earlier. Function at 0x80C2EE9C. On a stock phone in idle the CM4 is up and its HCI traffic to
the BT controller can be read from the shared buffers (hw-stock).

Never read the CM4 AHB registers (0x50820000 and up) from the AP on a running stock phone: it
hangs (hw-stock).

## WTL die radio-on block

Stock order, written while the die is coming up (static, emu): `E010102C |= 0x2000000`, then
`&= ~0x1000000`; `E010100C &= ~0x1000000`; `E0101014 = 0xCC`; `D15E0068 = 0xDD00DD`;
`0x60831044 = 0x80`; `D15E1028 = 0x380`; `D15E2000 = 0x380`; `0x60831044 = 0x80`;
`D15E2000 = 0x300000`; later `D15D0020 = 2` if it reads 0. The same radio-on wrapper also writes
`0x60831034 = 0x410`. From
Linux, accesses to the same blocks through 0x515D0000/0x515E0000 have wedged the bus, and the
die domain powers on without them (hw-linux).

## Differences still open (FPLinux vs stock)

Ranked by how likely I think each is to explain the silent DSP:

1. No LTE calibration at 0x80678000. The firmware file FPLinux loads is almost all zero there;
   stock copies ~320 KB of NV data in right before the release. Test: put this phone's own
   calibration block (from a stock DRAM image) into the file at offset 0x230000.
2. Rings watched in the wrong place. Earlier driver versions used the DRAM window at
   0x8040E400; stock uses on-chip 0xE400. An assert reply would have gone unseen.
3. CM4 not started (vector at 0x50800000, BT system and CM4 resets released).
4. XDSP follow-up configuration not written.
5. GSM DSP not running first.
6. dcdcwpa (PA supply) off, 0xC5C not set, no ADI hardware channels ([pmic-watchdog.md](pmic-watchdog.md)).
   Probably not what stops DSP code running, but cheap to fix.

Ruled out on the phone (hw-linux): CP soft reset (after the power sequence it reads 0x502 versus
stock 0x2, only the BT bits differ) and the stock AON clock enables (`0x402E0004 |= 0x16BE0000`,
`0x402E0010 |= 0x30400`): no change, same silence and the same hang.

## Hazards when poking at the radio

- Reading a peripheral whose clock is off resets the stock phone (hw-stock).
- On a stock phone with the radio powered down, the WTL die pages read nothing useful and may hang.
- Every hang under Linux costs a battery pull. Stream the kernel log (`cat /dev/kmsg` over SSH)
  to the PC during experiments so the last lines survive.
