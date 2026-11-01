# MMI function addresses

Addresses of MocorOS MMI functions in this phone's `mmi_app`, and a few kernel functions apps need.
There is no SDK and the image has almost no symbol strings, so every name here was assigned by
matching code. The names follow MocorOS conventions (`MMK_*`, `GUI*`, `SFS_*`), which are public
through Mocor documentation and other phones' debug output.

Valid only for `mmi_app` sha256 `5d534329...` and the kernel built with it. `mmi_app` addresses are
`file offset + 0x88000000` ([boot.md](boot.md)).

## How the addresses were found

1. A reference build with names. The Nokia 3210 4G (TA-1618) uses the same chip and screen
   size, and its `mmi_app` build kept the trace and assert messages that ours had stripped. Those
   messages name the function they sit in, so a large part of the Nokia image could be named.
2. Function matching between the two images. Byte similarity between the builds is
   effectively zero, so matching works at function level. Anchors are functions that use the same
   unique strings or source-file references in both images; about 4,600 of those. Propagating
   through the call graph (a matched caller's callees in the same order are probably matched too)
   gave about 14,300 pairs. The 8-byte kernel veneers in `mmi_app` were paired the same way, by
   aligning the call lists of matched functions.
3. Counting branch references. A raw scan for every BL/BLX in the image gives the true number
   of call sites per function; the disassembler misses many because of gaps in its coverage. A
   real match has similar counts in both images (e.g. 1843 vs 2784 callers for a function called
   from everywhere, 1 vs 1 for an internal helper).
4. Reading the code by hand. Each function used by apps was checked in disassembly or
   decompiled output against what the name says it does: argument use, the structure it reads,
   what it calls.
5. Running it. Most were then called in the emulator, and the ones apps depend on were used on
   the phone.

Function-pointer tables and window tables live in compressed RW data, so a function reached only
through a pointer has no reference in the file. Those were placed by the set of functions they
call ("callee fingerprint"), not by cross-reference.

Evidence column: **M** = matched to the reference build with identical instruction structure and
similar call counts; **D** = checked by hand in disassembly or decompile; **E** = called in the
emulator with the expected result; **P** = used on the phone. "(likely)" marks a name inferred from
behaviour, not matched.

## Calling convention

- `mmi_app` functions are Thumb: call `address | 1`.
- Kernel veneers in `mmi_app` (0x880A8xxx-0x880A9xxx) are ARM `ldr pc, [pc, #-4]` stubs: call the
  even address (BLX, or a pointer without bit 0).
- Kernel functions listed here are Thumb unless marked ARM.
- Everything runs in the MMI task. Blocking stalls the UI and eventually trips the watchdog; use
  window timers for long work.

## Window manager

| function | address | signature | evidence |
|---|---|---|---|
| MMK_CreateWin | 0x883FBA64 | `handle (const uint32 *win_table, void *add_data)` | M D E P |
| MMK_CreateWindow | 0x883F989E | `handle (const MMI_WINDOW_CREATE_T *)`, 0x2C-byte struct | D E P |
| MMK_GetCurApplet | 0x883F9966 | `handle (void)` | D E P |
| MMK_CloseWin | 0x883F9AB2 | `bool (handle win)` | M D E P |
| MMK_IsOpenWin | 0x883FC3A4 | `bool (win)`; reads a byte at win+0x1E | M D P |
| MMK_IsFocusWin (likely) | 0x883FC752 | `bool (win)` | M |
| MMK_ConvertIdToHandle | 0x883FBB0A | `handle (id)` | M D |
| MMK_GetCtrlHandleByWin (likely) | 0x8840113A | `(win, ctrl_id)` | M |
| MMK_SetAtvCtrl | 0x8840167E | `(win, ctrl_id)`: focus a control | D E P |
| MMK_SendMsg | 0x883F8EBA | `bool (handle, msg, void *param)`, synchronous | M D E |
| MMK_PostMsg | 0x883F8F4A | `bool (handle, msg, void *param, size)`, queued; param copied | M D E P |

`MMI_WINDOW_CREATE_T` (0x2C bytes, from the idle window's creation and about 30 other callers):
+0x00 applet handle, +0x04 parent (0 = none), +0x08 window ID, +0x0C handler (Thumb pointer),
+0x10 add_data, +0x18 priority (0 = lowest, 1 = normal), +0x20 style (0 = plain), +0x24 move
style; the other fields are zero in every caller I looked at.

## Timers

| function | address | signature | evidence |
|---|---|---|---|
| MMK_CreateWinTimer | 0x882630F8 | `uint8 id (win, ms, is_period, 0, 0)` | M D E P |
| MMK_CreateTimer | 0x88263088 | binds to the focused window | M |
| MMK_CreateTimerCallback (likely) | 0x882630E0 | no window; argument order not verified | M |
| MMK_StopTimer | 0x88263136 | `bool (uint8 id)` | M D P |

Use one-shot window timers (`is_period = 0`) and re-arm from the timer message. The periodic path
is never used by stock and resets the phone. Timer IDs below 14 are reserved. MMK timers are
ordinary OS timers and do not fire while the phone sleeps; the RTC alarm does
([rtc-alarm.md](rtc-alarm.md)).

## Memory and kernel helpers

| function | address | notes | evidence |
|---|---|---|---|
| SCI_ALLOC_APP (veneer, ARM) | 0x880A9BC8 | `void * (size, 0x44444444, const char *file, int line)` | M D P |
| SCI_FREE (veneer, ARM) | 0x880A9C98 | `(ptr, file, line)` | M D P |
| memset to 0 (veneer, ARM) | 0x880A9CA8 | `(ptr, len)`; over 9,000 call sites | M |
| memcpy (veneer, ARM) | 0x880A9CB8 | `(dst, src, len)` | M |
| trace (veneer, ARM) | 0x880A9C60 | `(fmt, ...)` | M |
| assert (veneer, ARM) | 0x880A9BE0 | lands on the kernel assert entry 0x80A09414 | M D |
| D-cache clean range (kernel, ARM) | 0x814C3F88 | `(start, end)` | D P |
| I-cache invalidate range (kernel, ARM) | 0x814C4120 | `(addr, len)` | D P |

Clean the D-cache and invalidate the I-cache over any code you load before jumping to it. The MMI
task runs privileged, so these work from an app.

## Files (SFS)

Paths are UCS-2. `E:\` is the SD card, `D:\` internal storage.

| function | address | signature | evidence |
|---|---|---|---|
| SFS_CreateFile | 0x88085B16 | `handle (path, mode, share, attr)`; mode 0x31 = open existing for read, 0x22 = create/overwrite for write | M D E P |
| SFS_ReadFile | 0x88086562 | `err (h, buf, len, uint32 *read, overlapped /*NULL*/)` | M D E P |
| SFS_WriteFile | 0x880865BA | same shape as ReadFile | M D P |
| SFS_CloseFile | 0x88086530 | `err (h)` | M D E P |
| SFS_GetFileSize | 0x8808671A | `err (h, uint32 *size)` | M D E P |
| SFS_SetFilePointer | 0x8808664E | `err (h, offset, origin)`; check the offset width before use | M |
| SFS_FindFirstFile | 0x88086754 | `(pattern, SFS_FIND_DATA_T *)` | M P |
| SFS_FindNextFile | 0x88086A54 | | M P |
| SFS_FindClose | 0x88086A92 | | M P |
| delete file | 0x88085DAA | `(path, 0)` | D P |
| create directory | 0x88085CC8 | not tested | D |

`SFS_FIND_DATA_T`: attributes u16 at +0x14 (0x10 = directory), size u32 at +0x18, UCS-2 name at
+0x1C. Allocate at least 0x400 bytes for it; a 0x220-byte buffer overflowed on a long name,
corrupted the heap and reset the phone.

## Drawing

| function | address | signature | evidence |
|---|---|---|---|
| GUI_FillRect | 0x8824ACAC | `(const GUI_LCD_DEV_INFO *dev, GUI_RECT_T rect /*by value, r1-r2*/, uint16 rgb565)` | M D E P |
| GUISTR_DrawTextToLCDInRect | 0x882615A8 | `(dev, const GUI_RECT_T *disp, const GUI_RECT_T *clip, const MMI_STRING_T *str, style *, state, int8 dir)` | M D E P |
| GUI_IntersectRect | 0x887EAB16 | `bool (GUI_RECT_T *out, GUI_RECT_T a, GUI_RECT_T b)` | M P |

The main LCD is `GUI_LCD_DEV_INFO {lcd_id 0, block_id 0}`. The text style struct is 0x2C bytes;
only +0x00 font size, +0x04 colour (RGB565) and +0x08 alignment are known.

## Stock controls

No matches existed for these in the reference build; they were found from stock call sites and
decompiles, then exercised in the emulator.

| function | address | signature | evidence |
|---|---|---|---|
| GUILIST_SetMaxItem | 0x88802852 | `(ctrl, max, 0)` | D E P |
| GUILIST_AppendItem | 0x888028B0 | `(ctrl, const GUILIST_ITEM_T *)` | D E P |
| GUILIST_GetCurItemIndex | 0x88802C2A | `(ctrl)` | D E |
| GUILIST_SetCurItemIndex | 0x887FEF34 | `(ctrl, idx)` | D |
| GUILIST_RemoveAllItems | 0x888028FE | `(ctrl)` | D |
| list style lookup | 0x883C8C36 | `(enum)` returns a style pointer; 6 = one-line text | D E P |
| GUIWIN_SetSoftkeyTextId | 0x887F7DE2 | `(win, text_id, button 0/1/2, update)` | D E |
| GUIWIN_SetTitleTextId | 0x887F8512 | `(win, text_id)` | D E |
| GUIWIN_SetSubTitleText | 0x887F842C | `(win, has, wstr, len, update)`: short text in the title's right tab | E |
| GUILABEL_SetText | 0x887F77BA | `(ctrl, MMI_STRING_T *, update)` | D |
| GUITEXT_SetString | 0x882AAB10 | `(ctrl, wstr, len, update)` | D E |
| GUIEDIT_GetString | 0x8825ECD4 | `(ctrl, MMI_STRING_T *out)`, points into the control | D E |
| GUIEDIT_SetString | 0x8825ED2E | `(ctrl, wstr, len)` | D E |
| GUIEDIT_ClearAllStr | 0x8825EE20 | `(ctrl)` | D |
| MMIPUB_OpenAlertWinByTextPtr | 0x88219C44 | `(time*, text1*, text2*, image, win_id*, prio*, softkey_style, handler)`; four stack arguments | D E |
| MMIPUB_OpenAlertWinByTextId | 0x88219C16 | same with text IDs | D |
| MMI_GetLabelText | 0x883DBCF0 | `(text_id, applet, MMI_STRING_T *out)` | D E |

Window table codes in this build differ from the published Mocor ones:

| code | entry |
|---|---|
| 0xF001 | handler function |
| 0xF002 | window ID |
| 0xF008 | text control (id) |
| 0xF009 | list box (type 3 = text list, id) |
| 0xF00A | edit text (max length, id) |
| 0xF00B | edit phone number (max length, id) |
| 0xF00C | edit digits (max length, id) |
| 0xF01F | status bar |
| 0xF020 | softkeys (left, middle, right text IDs) |
| 0xF021 | title (text id) |
| 0xF030 | end of table |

`GUILIST_ITEM_T` is 0x1C bytes: +0x00 style 0, +0x04 style pointer from the lookup above, +0x08 data
pointer, +0x0C flags, three u16 at +0x10, +0x18 user data. Item data is 0x6C bytes: six 16-byte
content slots `{ptr or text id, u16 len, ..., +0x0C type: 1 = text id, 2 = buffer}` and three softkey
text IDs at +0x60.

Stock text IDs (0x400000xx): 0x02 Reset, 0x0D Exit, 0x0E Menu, 0x10 Select, 0x11 OK, 0x12 Back,
0x15 Cancel, 0x16 Done, 0x17 Save, 0x19 Options, 0x1C Delete, 0x1D Search, 0xC1 Edit, 0xC7 Add,
0xC8 View.

## Messages and keys

Window messages: 0xF020 timer, 0xF021 open window, 0xF022 close window (likely), 0xF023 lose
focus, 0xF024 get focus (likely), 0xF025 full paint. A table window's handler gets 0xE001 for the
left softkey, 0xE002 for back/right softkey and 0xE004 for the centre key.

Key messages are `type | key`:

| type | base |
|---|---|
| press | 0xFA00 |
| release | 0xFB00 |
| long press | 0xFC00 |
| long release | 0xFD00 |
| repeat | 0xFE00 |

| key | code | key | code |
|---|---|---|---|
| up, down, left, right | 0x01-0x04 | call (green) | 0x05 |
| 1-9 | 0x08-0x10 | *, 0, # | 0x11, 0x12, 0x13 |
| left soft, right soft | 0x16, 0x17 | red/power | 0x18 |
| OK | 0x1B | | |

The keypad scan code is converted by `MMIDEFAULT_ConvertKeyCode` (0x88258C20, M), which searches a
0x69-byte table at 0x882F9C6C. The idle window handler `IdleWin_HandleMsg` is at 0x8828A37A (M:
52 of 56 callees agree with the reference). At idle, OK and every long press except `*` are not
handled.

## Hook points

| what | address | notes |
|---|---|---|
| Apps menu "Facebook" action | 0x880EC994 | 20 bytes, one caller (the menu dispatch). It only opens the browser at a fixed URL. Replacing its first instruction with `b.w <loader>` turns that menu item into an app launcher with no menu or resource change (P) |
| free space in the partition | 0x88904100-0x8891FFFF | 0xFF padding after the payload; pages in like stock code, read-only at run time |

## Alarm service and time

See [rtc-alarm.md](rtc-alarm.md) for signatures. `mmi_app` addresses (call with bit 0 set):

| function | address | evidence |
|---|---|---|
| MMIALMSVC_RegService | 0x880E0476 | D E P |
| MMIALMSVC_UpdateService | 0x880E03EC | D P |
| MMIALMSVC_UnRegService | 0x880E0574 (re-programs the RTC), 0x880E051C (doesn't) | D P |
| MMIALMSVC_IsValidSvc | 0x880E03A0 | D E P |
| MMIALMSVC_SetAlarm | 0x880E015C | D E |
| TM_GetTotalSeconds (veneer, ARM) | 0x880A97A8 | D P |
| TM_GetSysDate (veneer, ARM) | 0x880A9B88 | D E |
| TM_GetSysTime (veneer, ARM) | 0x880A9B90 | D |
