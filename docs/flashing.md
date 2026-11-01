# Backing up and flashing

The F286 can always be reached through the BootROM's USB download mode, whatever is in flash. That
is what makes experiments on it reasonably safe: as long as you have a good backup of a partition,
you can put it back. Back up everything before writing anything.

I have only written one partition (`mmi_app`) on my phone, many times. I have never written
boot0, boot1, the kernel or any NV partition, and I don't recommend it.

## What you need

- A PC with a client for the Spreadtrum/Unisoc download protocol. I used `spd_dump`, an
  open-source command-line client. The vendor's Windows flash tools also speak it, but they want
  a full PAC package and push you towards repartitioning.
- An FDL1/FDL2 pair for the UMS9117. FDL1 runs from internal SRAM at 0x6200 and brings up DDR;
  FDL2 runs at 0x80100000 and does the flash work. These are not distributed here.
  - The FDL2 has to know this phone's SPI-NAND (READ_ID `b0 15`). The UMS9117 FDL2 I started
    with did not; it worked after its NAND table entry was changed to this chip's ID.
  - An FDL1 built from source works fine as the first stage.
- A USB cable to a port directly on the PC. Through a hub, reads dropped out after 2.5-3.25 MB.

## Entering download mode

1. Start the PC client first, waiting for the device (`spd_dump --wait 300 ...`).
2. Phone off, battery out.
3. Hold the left soft key (top left), plug in USB. With the battery in it also works.

The BootROM shows up as USB 1782:4D00. `keep_charge 1` keeps it powered while the FDLs run.

## Commands

Partition IDs the FDL2 understands (from the UMS9117 partition XML in the vendor packages and
checked against this phone):

| ID | partition |
|---|---|
| 0x80000001 | boot0 |
| 0x80000002 | boot1 |
| 0x80000003 | kernel |
| 0x80000004 | mmi_app ("UserImg") |
| 0x90000001 | nvcal (NV items) |
| 0x90000002 | prodnv |
| 0x90000004 | mmi_res |
| 0x90000005 | LTE DSP firmware |
| 0x90000006 | GSM DSP firmware |
| 0x90000013 | CM4 firmware A |
| 0x90000018 | CM4 firmware B |
| 0x90000019 | hw_ver |

A read of the four image partitions and an `mmi_app` write with read-back, in `spd_dump` syntax:

```
spd_dump --wait 300 keep_charge 1 \
    fdl fdl1.bin 0x6200 fdl fdl2.bin 0x80100000 blk_size 0x1000 \
    read_flash 0x80000001 0 auto boot0.bin \
    read_flash 0x80000002 0 auto boot1.bin \
    read_flash 0x80000003 0 auto kernel.bin \
    read_flash 0x80000004 0 auto mmi_app.bin

spd_dump --wait 300 keep_charge 1 \
    fdl fdl1.bin 0x6200 fdl fdl2.bin 0x80100000 blk_size 0x1000 \
    write_data 0x80000004 0 0x920000 mmi_app_patched.bin \
    read_flash 0x80000004 0 0x920000 mmi_app_readback.bin
```

`auto` takes the size from the DHTB header. For a write, give the full partition size
(0x920000 for `mmi_app`) and compare the read-back's sha256 with the file you wrote.
`read_part nvcal 0 0x100000 nvcal.bin` reads by name where the ID route is awkward.

## What is normal

- **"FDL2: incompatible partition"** right after FDL2 starts. FDL2 returns this code
  unconditionally. Vendor tools react by insisting on a repartition; don't. Reads and
  `write_data` work regardless.
- Without a `power_off` command at the end, the phone stays in the FDL; pull the battery to leave.

## Backup advice

- Read every partition, plus a whole-NAND image if your FDL2 can do it. Record sha256 sums and
  keep two copies off the PC.
- Read through FDL2, not through raw NAND access from Linux. Raw reads from FPLinux had random
  0-to-1 bit flips (about 400 bytes per 16 MB per pass) and some pages that need the chip's
  read-retry, which the Linux driver doesn't implement. The FDL2 read matched boot0, boot1 and
  nvcal byte for byte.
- `mmi_res` read raw, without ECC, had about 2,500 flipped bits, enough to break font glyphs.
  Read it through the 0x9000000x ID so ECC applies.
- `prodnv` and `nvcal` hold per-unit data (calibration, serial numbers). Never write another
  phone's copy, and don't share yours.

## What to write

| partition | safe to experiment with? |
|---|---|
| mmi_app | yes, with the stock image kept for restore. The kernel boots whatever is there; nothing checks a signature |
| mmi_res | probably, same reasoning; not tried |
| kernel, boot0, boot1 | not tried; a bad boot0/boot1 should still leave the BootROM usable, but I haven't tested that |
| nvcal, nvcal_bak, prodnv | no: per-unit calibration, no way to regenerate it |
| userdata | it is the `D:` FAT drive; easier to change from the phone |

Restoring stock `mmi_app` is the same `write_data` with the backup. Download mode doesn't depend on
anything in `mmi_app`, so a broken patch can always be undone.
