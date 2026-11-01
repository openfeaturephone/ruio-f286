# Boot

How the F286 gets from power-on to the idle screen, and the address arithmetic you need to read
the images. Mostly static analysis of boot1 and the kernel, checked against the emulator and
against what the phone does.

## Stages

| stage | from | runs at | does |
|---|---|---|---|
| BootROM | mask ROM at 0xFFFF0000 | | checks the download key and USB; loads boot0 |
| boot0 | `boot0` partition (DHTB, 0x7240-byte payload) | internal SRAM, code at 0x6200 | CPU and DDR init, early PMU pokes, PMIC watchdog clock on |
| boot1 | `boot1` partition (DHTB, 0xE0A0-byte payload) | 0x80100000 (header), entry 0x80100200 | UART log, reset-mode checks, MMU, SPI-NAND and FTL init, loads the kernel |
| kernel | `kernel` partition | 0x80A05E00 (header), entry 0x80A06000 | the whole MocorOS: ThreadX, drivers, protocol stack |
| MMI | `mmi_app` partition | 0x88000000, paged in on demand | the user interface and apps |
| resources | `mmi_res` partition | read by the kernel through a resource service | fonts, images, strings |

### boot1 in order

1. Clear AP AHB 0x20E00000 bit 4 (USB) first, then set up UART1 (0x70100000, 115200) and print
   its banner with the timer value.
2. `[0x402E1000] = 0x10000` and the ADI master setup (see [pmic-watchdog.md](pmic-watchdog.md)).
3. Read the PMIC reset-monitor register 0x40608E24. Bits 8:4 are a reset-mode field the kernel
   writes before a reboot. If it is 0x17, boot1 clears it and enters USB download ("auto
   download"). This is the only boot1 path that starts USB.
4. If 0x40608E30 bit 12 is set and bit 7 clear (the 7 s long-press reset), boot1 pulses
   0x40608E38 bit 15, writes 0xAAAAAAAA to address 0, and jumps into the kernel already in RAM
   without reloading it.
5. Build a flat L1 page table, turn on MMU and caches, set up NAND clocks and pins, initialise
   the SPI-NAND (SET FEATURE 0xA0 = 0, 0xB0 = 0) and the FTL.
6. Read partition 3 (kernel): check the `DHTB` magic and version 1, take the payload size from
   header +0x30, copy header plus payload to 0x80A05E00.
7. Clean the D-cache, turn MMU and caches off, jump to 0x80A06000.

The kernel is entered in ARM state, SVC mode, IRQ and FIQ masked, MMU off, with r0 = 0x80A06000.
There is no parameter block: the kernel ignores r0-r3.

### What the kernel does first

- Reads 0x40608E24 directly. If bits 15:8 match the image default (0x1F), it goes straight to the
  assert handler in post-assert dump mode.
- Sets up mode stacks below 0x83000000, SVC stack at 0x80100000.
- Reads word 0. 0xAAAAAAAA (the 7 s marker) or 0x54464E53 runs a small dump agent copied to
  0x80010000. A particular key pair held at boot takes the same path.
- Scatter-load: copies 0x18DC bytes of vectors and handlers to SRAM 0x0, decompresses RW data to
  0x81700000-0x8175E1B4, zeroes several areas (see [hardware.md](hardware.md)).
- Later, a power task picks the start-up kind from the restart reason: power key or software
  restart = full start; alarm (reason 2) or charger = reduced start; assert = full start plus
  protocol stack; anything else powers off.

In the emulator the stock firmware reaches the idle window at about 5.4 G instructions, roughly
27 s of virtual time. On the phone, a hook placed at idle-open runs 16-25 s after power-on.

## Kernel addresses

```
kernel VA = kernel.bin file offset + 0x80a05e00
```

The file offset counts from the start of the partition image, DHTB header included. This was
proven by the kernel veneers in `mmi_app`: their targets land on function prologues with this
base, and calls made on the phone with it work.

An older note gave `+0x80a05dc8`. That is wrong by 0x38. The cause: an early "payload" extract
of the kernel started at file offset 0x38 instead of 0x200, and every address read from that
file with base 0x80A05E00 came out 0x38 too low. Using one of those addresses on the phone put a
call 0x38 bytes into the wrong place and reset it. If you find a kernel address in old notes,
check that it lands on a prologue before trusting it; `real = old + 0x38` fixes the ones from
that extract.

Data inside the kernel image: data at real VA V is at file offset V - 0x80A05E00 as well, except
for the RW section, which is stored compressed and only exists at 0x81700000+ at run time.

## mmi_app: demand paging at 0x88000000

`mmi_app` is not copied into RAM at boot. The kernel maps a 15 MB window at 0x88000000 and its
page-fault handler fills a page by reading the partition at `VA - 0x88000000`. So:

```
mmi_app VA = mmi_app.bin file offset + 0x88000000
```

File offset 0 is the DHTB header, and it is mapped too: the header occupies 0x88000000-0x880001FF
and code starts at 0x88000200. An older note said "file offset = VA - 0x88000000 + 0x200"; that
is wrong, and an emulator that loaded the image that way never got the MMI running.

Consequences that matter when patching:

- The header's size field (+0x30, 0x903EE0) is never consulted. Anything written into the
  partition's 0xFF padding after the payload (up to 0x920000) pages in like stock code. The
  app loader sits at 0x88904100 this way.
- Pages are read-only at run time. Code there cannot have writable globals.
- A permission fault on a present page (a store to a read-only page) does not reset the phone:
  the pager sees the page present, returns, and the store retries forever. The phone freezes
  until the battery is pulled.
- The kernel sets SCTLR.A. An unaligned access is an alignment fault, which does reset the phone.
- `mmi_app` RW data is also compressed, so function-pointer tables and window tables are not
  visible as literals in the file.
- `mmi_app` calls the kernel through 8-byte ARM veneers (`ldr pc, [pc, #-4]`) near 0x880A8000-
  0x880AA000.

## Reset mode register

0x40608E24 (SC2720, reset monitor):

| bits | meaning |
|---|---|
| 15:8 | marker; equal to the image default means "came back from an assert" |
| 11 | flag folded into the reset mode by the kernel |
| 8:4 | reset mode written before a reboot; 0x17 = enter download mode |

The kernel latches and clears the field early in boot. 0x40608E30 bits 12 and 7 are the 7 s
long-press reset flag and its disable (guess, from how boot1 and the kernel test them).

## FPLinux boot notes

FPLinux is booted from RAM through download mode: an FDL1 loads the image and jumps to it, so
nothing is written to flash. Starting the stock kernel the same way (to compare register state)
stalled after the splash screen. Comparing the FDL1 jump with boot1's gave this list; the first
item is the likely cause, none of them is confirmed yet:

- FDL1 leaves USB enumerated with a receive DMA queued into low SRAM. boot1 disables USB before
  the jump. The kernel then copies its vectors to 0x0 and any host traffic can DMA over them.
  Stop USB (mask its IRQ, stop the receive DMA, disconnect) before jumping.
- Write 0 to address 0 so the kernel does not take the dump-agent path.
- Clear the reset-mode field in 0x40608E24 unless a mode is wanted.
- The power-on cause is "USB plugged with a key held", which a normal boot never sees.
