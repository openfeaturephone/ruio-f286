# USB

USB device modes of the stock firmware, and how to get a two-way byte pipe between a PC and code
running on the phone. The pipe is what the f286dbg debug agent uses. Kernel addresses are real VAs
(file offset + 0x80A05E00).

## IDs

All use vendor ID 0x1782 (Spreadtrum).

| PID | when | interfaces |
|---|---|---|
| 0x4D00 | BootROM / FDL download mode | vendor bulk |
| 0x4D01 | stock "U Disk" choice | mass storage (class 08/06/50), exposes `D:` (68 MB FAT) |
| 0x000B | USB service type 4 (UCOM) | 4 vendor-class (0xFF) interfaces, one bulk IN + OUT each |
| 0x000C | USB service type 2 | RNDIS (IAD + CDC comm + CDC data) plus 2 vendor bulk interfaces |
| 0x000A, 0x4D03 | descriptors exist in the kernel | not seen in use |

When plugged in while running, the stock OS offers Charging, U Disk and "PC Connecting" (USB
tethering over RNDIS). No serial port is offered in the menu.

## Kernel pieces

| function | address | notes |
|---|---|---|
| USB_START | 0x81329DE2 | `(service type)`, Thumb |
| USB_STOP | 0x81329E5E | `(service type)`, Thumb |
| USB_CURRENT | 0x81329DDC | returns the running service type, Thumb |
| UCOM ops table | 0x814C8154 | array of function pointers, below |
| UCOM mode byte | 0x817040AC | 2 = connected UCOM; set by the detect handler |

UCOM ops used so far:

| index | call | meaning |
|---|---|---|
| 0 | `open(chan, 0, cb)` | register a channel and its event callback |
| 2 | `ctl(chan, 1)` | pump: calls `cb(0)` if data is waiting |
| 3 | `read(chan, buf, len)` | |
| 4 | `write(chan, buf, len)` | |

Service type 4 registers five plain UCOM channels. Type 2 also registers two extra channels that
look like diag/log; what they speak is not decoded.

## Getting a working pipe

This sequence worked on the phone from the MMI task:

1. `USB_STOP(USB_CURRENT())`, then `USB_START(4)`.
2. Straight away, in the same handler: `open(ch, 0, cb_ch)` for channels 0, 1, 2 and 3, each with
   its own callback.
3. A one-shot 5 s detect timer in the kernel then connects all four (event 1 in the callback).
4. Every 200 ms call `ctl(ch, 1)`; on `cb(0)` read with `read`. Write with `write`.
5. Close all four channels before your code goes away. The kernel keeps the callback pointers.

The traps, each of which cost a phone reset or a dead link:

- Open all four channels before the 5 s timer fires. The detect handler calls the callback of
  every channel; a missing one is a jump to nothing and the phone resets.
- Don't open again later without restarting the service. A late `open` sets the mode back to 0
  and no second timer fires to recover it.
- Write is zero-copy. `write` stores your pointer in the transfer descriptor and returns; the
  controller reads the buffer afterwards. A write while the previous transfer is still busy is
  dropped silently (and still returns `len`). Keep one transfer in flight, wait for the busy flag
  at descriptor + 0x20 to clear (pipe = `[0x817040D0 + 4 * ch]`, descriptor = `[0x818FC800 + pipe *
  0x20]`), and D-cache clean each buffer after filling it.
- Service type 2 crashed the Windows PC, not the phone: type 2 brings up an RNDIS adapter and
  the Windows LLDP protocol driver (`mslldp.sys`) bugchecked on it, twice. I disabled that driver
  (`sc.exe config mslldp start= disabled`, then reboot; `start= auto` reverts it). Type 2 is no
  use for the pipe anyway: its detect path never reaches the connected mode. Use type 4.

## PC side

Type 4 enumerates as 1782:000B with four vendor interfaces:

| interface | endpoints | UCOM channel |
|---|---|---|
| 0 | OUT 0x02, IN 0x81 | 0 |
| 1 | OUT 0x04, IN 0x83 | probably 1 |
| 2 | OUT 0x06, IN 0x85 | probably 2 |
| 3 | OUT 0x08, IN 0x87 | probably 3 |

The interfaces are vendor class, not CDC-ACM, so Windows binds no driver by itself. I attached
WinUSB to the device with Zadig (once per PID) and used libusb from Python. Before data flowed, the
client sent the CDC requests SET_LINE_CODING (0x20) and SET_CONTROL_LINE_STATE (0x22) with DTR
set; the kernel answers them on the control pipe. Echo tests confirmed that interface 0 is UCOM
channel 0. Interfaces 1-3 accept OUT transfers but my test code didn't echo on them, so their
mapping is inferred.

Measured throughput through the agent: about 51 KB/s for memory reads.

After the phone reboots, a libusb process that was already running often misses the
re-enumerated device. Start a fresh process for each connection attempt.

I have only done this from Windows. On Linux the device should work with libusb directly; not
tested.

## Debug agent

The [f286dbg](https://github.com/openfeaturephone/f286dbg) agent runs as an MMI app, uses UCOM
channel 0 with the sequence above, and polls every 20 ms from a window timer (once a second when
idle, to save battery). It supports ping, memory read and write, function calls (r0-r3, and an
8-argument variant), heap alloc and free, cache maintenance, key injection, file list/get/put/rm,
screenshots and a remote reboot through the PMIC watchdog ([pmic-watchdog.md](pmic-watchdog.md)).
Protocol details are in that repository.

## Diag protocol

The kernel has a full Unisoc diag command dispatcher, including memory read and write subtypes,
but I found no path that attaches it to USB in normal use, and the stock "U Disk"/"PC Connecting"
modes don't expose it. Not pursued.
