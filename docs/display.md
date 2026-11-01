# Display

240x320 TFT with a GC9307 controller on the SoC's parallel LCM (DBI) bus. The SoC's LCDC composes
up to five layers from RAM and pushes the result to the panel. Kernel addresses are real VAs (file
offset + 0x80A05E00).

## Panel

- Controller: **GC9307**. FPLinux read ID 0x009307 from the panel (hw).
- Bus: LCM/DBI, chip select 0, 16-bit transfers; FPLinux runs the LCM clock at 128 MHz.
- FPLinux drives it with the GC9307 init sequence from fpdoom plus MADCTL (0x36) = 0x48; without
  that MADCTL value the image is mirrored.
- The stock kernel does not hard-code a panel. At boot it resets the panel, reads its ID and picks
  a driver from a catalogue of 18 controllers (ILI9225, ILI9328, ILI9340X, ILI9341, ILI9342,
  GC9106, GC9108, GC9300, GC9304, GC9305, GC9306, GC9307, ST7735S, ST7789, JD9851, NV3029S,
  NV3029G, RM61526). Each driver's init sequence is inline code (one call per command or
  parameter byte), not a data table.
- Window commands are sent in the byte-wise form: 0x2A then four parameter bytes, 0x2B then four,
  then 0x2C and the pixel data.

My emulator assumed an ST7789 because the panel ID read had nothing behind it; that was a guess
and is wrong for this phone.

## LCM bus (0x20A00000 and the CPU-push window)

| address | role |
|---|---|
| 0x20A00000 | clock/divider configuration |
| 0x20A00008 | status; bit 1 = busy, polled before every bus write |
| 0x20A00010 | mode configuration (init writes 1) |
| 0x20A00014 | timing, init value 0x00A50100 |
| 0x20A00020 / 30 / 40 | per-bus-type control |
| 0x60000000 + (slot << 26) | command port for chip select `slot` (16-bit stores) |
| 0x60020000 + (slot << 26) | data port |

Stock writes a command as: wait for 0x20A00008 bit 1 clear, `strh cmd` to the command port. Data the
same way to the data port. 8-bit panels use the same 16-bit store with byte values.

Other bits stock sets during LCM init: AP AHB 0x20E01000 bit 12 (the set alias of 0x20E00000; LCM
enable, guess) and AON clock gate 0x402E060C via its set/clear aliases 0x402E160C / 0x402E260C.

A second LCM engine at 0x70B00000 is used by some catalogue drivers (status at +0x34, bit 25 busy;
start at +0x64 bit 0). I decoded it from about ten accessor functions only; the details are
unverified.

Stock kernel functions:

| address | function |
|---|---|
| 0x8100457E | LCD driver init: probe the panel, register the bus ops |
| 0x80A498F2 | set the bus windows for a slot |
| 0x80A498AC / 0x80A498CE | write one command / data word (busy-poll then store) |
| 0x8100538C / 0x81005396 / 0x810053A4 | panel-driver API: command, data, delay in ms |

## LCDC (0x20800000)

The LCDC fetches layers from RAM, blends them and sends the frame over the LCM bus.

| offset | role |
|---|---|
| +0x00 | control; bit 3 = run (self-clears when the frame is done) |
| +0x04 | output size: height in 27:16, width in 11:0 |
| +0x20, +0x50, +0x80, +0xB0 | OSD layers 1-4: control (bit 0 enable) and the registers after it |
| +0xE0 | image layer |
| +0x110 | interrupt enable |
| +0x114 | interrupt clear (write 1) |
| +0x118 | masked status |
| +0x11C | raw status: bit 0 frame done, bit 1 panel TE |

Layer address registers hold `address >> 2`. Frames that show the UI enable the image layer plus
OSD1 and OSD2 (control 0x0B in the emulator). OSD2 is a 240x18 ARGB8888 strip over the status bar
and is usually transparent; the status bar itself is drawn into OSD1. Menus and the dialler use
partial updates: the LCDC output size shrinks to the dirty rectangle, and the rectangle's origin is
only in the kernel's LCD state (`{u16 left, top, w, h}` at 0x817CE248 + 0x15C), not in any LCDC
register.

The frame-done interrupt is GIC SPI 19. A high-priority display task busy-polls the TE bit with
the interrupt masked, which starves the MMI task if TE never toggles; an emulator has to model TE.

## Screenshots

On the phone the LCDC registers read 0 between frames, so the hardware registers can't be used
to find the current image. The persistent state is the MMI layer manager in RAM (hw, matched pixel
for pixel against the emulator):

| address | content |
|---|---|
| 0x818AC6B4 | 16 z-order slots of 16 bytes, bottom to top: `u32 in_use, u32 ?, u32 layer_id (0xFFFF = empty), u32 ?` |
| 0x818AC874 | layer 0 (the full-screen base layer); layer `id` (6 and up) at 0x818AC874 + 0x60 * (id - 5) |

Layer object, 0x60 bytes:

| offset | field |
|---|---|
| +0x18 | source buffer (what the UI draws into) |
| +0x1C | buffer size in bytes |
| +0x20 / +0x22 | x / y (u16) |
| +0x24 / +0x26 | width / height (u16); pitch = width |
| +0x28 | format: 3 = RGB565, 0 = ARGB8888 |
| +0x2C | flags; bit 1 = shown |
| +0x34 | raw pointer to the hardware copy buffer (aligned up to 256 bytes) |
| +0x40 | layer alpha |

To compose a screenshot: take the slots in order, keep layers that are in use, shown, have a source
buffer and a sane size, and blend them bottom to top (RGB565 opaque, ARGB8888 with pixel alpha
times layer alpha). A 240x320 RGB565 layer is 150 KB; over the USB agent a shot takes 3-8 s.

When the backlight goes off, the MMI releases overlay layers, so a shot then only shows the base
layer. Wake the screen with a real key event first.

## Backlight

The display backlight is the SC2720 WLED block (PMIC 0x180-0x1D8, module enable 0xC08 bit 9); the
keypad backlight is KPLED at PMIC 0xDF8/0xDFC ([pmic-watchdog.md](pmic-watchdog.md)). The stock
brightness setter takes 0-100 %.
