# Firmware format

Layout of the NAND, the partition images and the vendor PAC packages. All of it was read from a
backup of my own phone and from public UMS9117 packages for other phones.

## NAND layout

128 MiB SPI-NAND, 128 KiB erase blocks (64 pages of 2048 bytes + 64 bytes OOB). The partition
table as FDL2 reports it, with what each partition holds:

| ID | name | blocks | offset | size | content |
|---|---|---|---|---|---|
| 0x00000001 | boot0 | 0-1 | 0x00000000 | 0x40000 | DHTB, first-stage loader |
| 0x00000002 | boot1 | 2-3 | 0x00040000 | 0x40000 | DHTB, kernel loader |
| 0x10000001 | nvcal | 4-10 | 0x00080000 | 0xE0000 | NV item database (calibration, settings) |
| 0x10000002 | prodnv | 11 | 0x00160000 | 0x20000 | product NV: serial numbers, factory flags |
| 0x1000000F | nvcal_bak | 12-18 | 0x00180000 | 0xE0000 | copy of nvcal |
| 0x10000006 | fw_modem_enc | 19-42 | 0x00260000 | 0x300000 | GSM DSP firmware |
| 0x10000005 | fw_dsp_enc | 43-78 | 0x00560000 | 0x480000 | LTE DSP firmware |
| 0x10000013 | wcn_fw_a | 79-86 | 0x009E0000 | 0x100000 | CM4 (BT/FM) firmware, variant A |
| 0x10000018 | wcn_fw_b | 87-94 | 0x00AE0000 | 0x100000 | CM4 firmware, variant B |
| 0x10000019 | hw_ver | 95 | 0x00BE0000 | 0x20000 | small hardware-version records |
| 0x1000001A | (unnamed) | 96-127 | 0x00C00000 | 0x400000 | read/write log-like pages, purpose unknown |
| 0x10000003 | (unnamed) | 128-159 | 0x01000000 | 0x400000 | read/write data pages, purpose unknown |
| 0x00000003 | kernel | 160-255 | 0x01400000 | 0xC00000 | DHTB, MocorOS kernel |
| 0x00000004 | mmi_app | 256-328 | 0x02000000 | 0x920000 | DHTB, MMI (user interface) |
| 0x10000004 | mmi_res | 329-401 | 0x02920000 | 0x920000 | resource container |
| 0x10000007 | fs_a | 402-433 | 0x03240000 | 0x400000 | contains a small FAT12 image at +0x3C6800 |
| 0x00000008 | userdata | 434-980 | 0x03640000 | 0x4460000 | the `D:` drive, FAT, 68 MB |

The IDs in this table are the kernel's own. Download-mode tools use a different set
(0x8000000x, 0x9000000x), listed in [flashing.md](flashing.md).

The `_enc` partitions are not encrypted. The kernel copies `fw_dsp_enc` (0x2E4000 bytes) to
0x80448000 and `fw_modem_enc` to 0x80100000 as they are, and the DSPs run them. For the CM4 the
kernel picks one of two images from the chip ID at 0x402E00FC: 0x98180000 selects the plain one,
0x98180001 the "AB" one, and this phone is 0x98180001. Which partition holds the "AB" image is not
settled: the vendor packages put it in the slot that is `wcn_fw_b` here, while the kernel's own
partition table names ID 0x1000001A for it.

## DHTB header

boot0, boot1, kernel and mmi_app start with a 0x200-byte header; the payload follows at +0x200.

| offset | size | field | value on this phone |
|---|---|---|---|
| 0x00 | 4 | magic | `DHTB` |
| 0x04 | 4 | version | 1 |
| 0x08-0x2F | | hash/signature area | all zero |
| 0x30 | 4 | payload size | boot0 0x7240, boot1 0xE0A0, kernel 0xBFD3D0, mmi_app 0x903EE0 |
| 0x34-0x1FF | | reserved | zero, except boot0 has some non-zero bytes from 0x14D (meaning unknown) |

Everything past the payload, up to the partition size, is 0xFF padding. boot1 trusts the size
field when it loads the kernel. Nothing reads the size field of `mmi_app`, because it is paged in
straight from the partition (see [boot.md](boot.md)). With the hash fields zero and no signature
trailer, nothing checks the content; a patched `mmi_app` boots.

Load addresses (header included):

| image | header at | code starts |
|---|---|---|
| boot0 | 0x6000 (guess) | 0x6200 |
| boot1 | 0x80100000 | 0x80100200 |
| kernel | 0x80A05E00 | 0x80A06000 |
| mmi_app | 0x88000000 | 0x88000200 |

## Inside the kernel image

- Code is Thumb-2 with some ARM (the exception vectors, cache maintenance, the watchdog gates).
- RW data is compressed in the image and decompressed to 0x81700000 at start. Anything in RW
  (task tables, function-pointer tables) only exists in RAM.
- The kernel registers a fixed table of flash partitions by ID with names, which is where the
  roles in the table above come from.
- The LTE DSP calibration is not in the DSP partition: the AP copies about 30 NV items from
  nvcal into DSP memory at 0x80678000 before releasing the DSP ([radio.md](radio.md)).
- Four small XDSP images used by WCDMA layer 1 live only inside the kernel image, not in any
  partition.

## mmi_res

Starts with the magic `sres0001`; it contains sub-containers marked `tres0001`. The MMI reads
resources (fonts, bitmaps, strings by text ID) through a kernel resource service. I have not
written a full decoder description here. Read this partition with ECC on: a raw copy has enough
bit flips to corrupt glyphs.

## PAC packages

Vendor packages for UMS9117 phones (I looked at Nokia 2660 Flip, Nokia 3210 4G, Nokia 220 4G and
an itel model) use the newer PAC layout, not the old `PCS\0` one:

| offset | content |
|---|---|
| 0x000 | UTF-16LE version string, `BP_R1.0.0` |
| 0x030 | u32 package size |
| 0x034 | UTF-16 product name (`UMS9117`) |
| 0x234 | UTF-16 firmware version |
| 0x434 | file count and directory offset |
| 0x84C | first file entry |

Each file entry is 0xA14 bytes: u32 struct size (0xA14), UTF-16 ID[256], UTF-16 name[256], then at
+0x5FC: size high, offset high, size, type (1 = file, 2 = XML, 0x101 = FDL), flash-use flag, offset
in the package, omit flag, address count, addresses[5]. The open-source `unpac` tools for
Spreadtrum packages read this format. The package's partition XML gives the 0x8000000x/0x9000000x
IDs.

No package for the F286 itself was available. The other phones' images differ from this one
everywhere (different builds), so their `mmi_app` and kernel cannot be swapped in. Their value was
as a naming reference, see [mmi-api.md](mmi-api.md). The vendor packages also carry an LTE-A DSP
partition that this phone doesn't have, and their LTE DSP image is 0x288400 bytes; this phone's is
0x2E4000.
