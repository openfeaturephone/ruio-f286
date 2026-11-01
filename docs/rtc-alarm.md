# RTC alarm

The SC2720 PMIC has the only real-time clock, and its alarm is the one timer that reliably wakes
the phone. This page covers the hardware, the kernel time layer, how the alarm interrupt reaches
the MMI, and how an app can use it for a silent periodic wake. Kernel addresses are real VAs
(file offset + 0x80A05E00).

Short version: don't program the RTC alarm yourself. Register an event with the stock alarm
service as type 1 or 2; it calls your callback with no UI and keeps stock alarms working. This ran
on the phone: 13 wakes out of 13, three minutes apart, each at second 0 of the planned minute, no
sound, no screen.

## RTC registers

SC2720 RTC block at PMIC offset 0x200, so ADI address 0x40608200 ([pmic-watchdog.md](pmic-watchdog.md)).
The layout matches the mainline Linux `rtc-sc27xx` driver.

| offset | register |
|---|---|
| 0x00-0x0C | counters: sec, min, hour, day |
| 0x10-0x1C | time update registers |
| 0x20-0x2C | alarm: sec, min, hour, day |
| 0x30 | INT_EN (bit 4 = alarm) |
| 0x34 | INT_RAW |
| 0x38 | INT_CLR |
| 0x54 | SPG_UPD; write 0xA5 to latch an alarm update |

The day counter counts days since 1980-01-01. The alarm is one-shot: the ISR disables it, so it
has to be re-armed for every wake. With the phone off, an armed alarm powers it on (restart reason
2). The RTC interrupt is sub-interrupt 1 of the PMIC interrupt (GIC SPI 38).

## Kernel time layer

| function | kernel | `mmi_app` veneer (ARM) | signature and behaviour |
|---|---|---|---|
| TM_SetAlarm | 0x80C2FD02 | 0x880A8F90 | `uint32 (SCI_ALARM_T)` by value in r0, r1. Validates; asserts on a bad date or time (returns 4). Returns 5 and disables the alarm if the time is not in the future. Otherwise stores it, writes an NV item and calls RTC_SetAlarm; returns 0 |
| TM_DisableAlarm | 0x80C2FCE0 | 0x880A8170 | clears the stored alarm, turns the RTC alarm interrupt off |
| TM_GetSysDate | 0x80C2FAC6 | 0x880A9B88 | `(SCI_DATE_T *out)`: u8 mday, u8 mon, u16 year, u8 wday |
| TM_GetSysTime | 0x80C2FB3C | 0x880A9B90 | `(SCI_TIME_T *out)`: u8 sec, u8 min, u8 hour |
| TM_GetTotalSeconds | 0x80C2FC5A | 0x880A97A8 | `uint32 (void)`: seconds since 1980-01-01 |
| TM_SetSysTime | 0x80C2F78A | 0x880A9340 | `(sec \| min << 8 \| hour << 16)` |
| TM_SetSysDate | 0x80C2F936 | 0x880A9348 | `SCI_DATE_T` by value; also writes NV |
| RTC_SetAlarm | 0x80C50708 | | `(sec \| min << 8 \| hour << 16, u16 day)`: waits for update idle, clears INT_EN bit 4, writes 0x220-0x22C, writes 0xA5 to 0x254, sets INT_EN bit 4. Only caller: TM_SetAlarm |
| RTC alarm interrupt off | 0x80C507E2 | | INT_EN &= ~0x10 |
| RTC ISR | 0x80C50306 | | bit 4 alarm, bit 3 day, bit 1 minute |

`SCI_ALARM_T` is 8 bytes: `{u8 sec, u8 min, u8 hour, u8 mday, u8 mon, u8 unused, u16 year}`, so
`r0 = sec | min << 8 | hour << 16 | mday << 24` and `r1 = mon | year << 16`.

## From interrupt to MMI

1. The PMIC interrupt dispatcher calls the RTC ISR, which queues code 0x72 for an alarm.
2. The misc task turns it into system signal 0x905 (0x902 = minute, 0x903 = hour, 0x904 = day)
   and sends it to registered clients through the kernel's event service.
3. The MMI registered for those events at start. Its signal router walks a table of
   (code range, app) pairs and offers the signal to each app until one takes it.
4. The phone app's message handler maps 0x905 to the alarm service's handler (0x880E0268 in
   `mmi_app`).

The alarm service handler:

1. Reads the current date and time and collects every enabled event whose year, month, day, hour
   and minute equal now. Seconds are ignored.
2. Type 0: calls only the first matching type-0 event's callback. Stock alarm clock, calendar
   events and automatic power on/off all register as type 0 with the same callback, which rings.
3. Types 1 and 2: for every match, sets the event to off, then calls `cb(id, type)`. No UI.
4. Re-programs the RTC for the soonest future event.

## Alarm service API

```c
typedef struct {               /* 16 bytes */
    uint8_t  sec;              /* stored, ignored when matching; use 0 */
    uint8_t  min, hour;
    uint8_t  mday, mon;        /* mon 1..12 */
    uint8_t  pad;
    uint16_t year;             /* 1980..2099 */
    uint32_t type;             /* 0 = stock alarm class, 1 or 2 = silent; > 2 is rejected */
    uint32_t (*cb)(uint32_t id, uint32_t type);   /* Thumb pointer */
} almsvc_info_t;

/* mmi_app, Thumb (call with bit 0 set) */
uint32_t MMIALMSVC_RegService(const almsvc_info_t *, uint32_t is_on, uint32_t set_rtc);           /* 0x880E0476, returns id, 0 = fail */
uint32_t MMIALMSVC_UpdateService(const almsvc_info_t *, uint32_t id, uint32_t is_on, uint32_t set_rtc); /* 0x880E03EC */
uint32_t MMIALMSVC_UnRegService(uint32_t id);       /* 0x880E0574 */
uint32_t MMIALMSVC_IsValidSvc(uint32_t id);         /* 0x880E03A0 */
void     MMIALMSVC_SetAlarm(void);                  /* 0x880E015C: RTC := soonest future event */
```

## Periodic wake recipe

1. `now = TM_GetTotalSeconds()`.
2. Add the period, convert to a date and time yourself.
3. `id = RegService(&info /* type 1, cb */, 1, 1)`.
4. In the callback, do the work, then `UpdateService(&next, id, 1, 0)`. The handler re-programs
   the RTC after all callbacks return, so `set_rtc = 0` is enough here.
5. On exit, `UnRegService(id)`.

The callback runs in the MMI task and may call `TM_GetTotalSeconds`, the SFS file functions and
`UpdateService`.

## Gotchas

- Calling TM_SetAlarm directly doesn't work for long. Any alarm edit, every alarm firing, boot
  and any service update with `set_rtc = 1` reprograms the RTC from the service's list. If your
  direct alarm does fire, the handler finds no matching event: the phone wakes, nothing runs.
- Keep the callback resident. If the code that holds `cb` is freed while the event is
  registered, the next alarm jumps into freed memory.
- Don't register or unregister inside the callback. The handler is walking a temporary list;
  unregistering frees a node it still points at. Updating your own event is fine.
- SetServiceStatus (0x880E0442) re-programs the RTC before storing the new status. Use
  UpdateService instead.
- Exact minutes. Matching is to the minute and only future events are considered. If the
  clock jumps past your event (network time, user edit), it never fires. Re-arm whenever the app
  runs, and never register for the current minute.
- Power-off. If the phone is switched off while your event is the soonest, the PMIC alarm
  still powers it on at that time. Your RAM-only event is gone by then, the handler returns 0 and
  the phone probably powers off again (not verified). Unregister before power-off.
- NV writes. Every TM_SetAlarm and TM_DisableAlarm writes a 0x130-byte NV item (guess, from
  the call it makes). A short period means a lot of NV writes per day; I have not checked
  whether that path caches.

## Open questions

- Whether the phone was really in deep sleep between the measured wakes (no sleep-state readout
  yet). The RTC alarm is a hardware wake source either way.
- The kernel's deep-sleep enable/disable API was not located.
