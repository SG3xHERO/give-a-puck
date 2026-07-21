# Steam Controller 2026 - 2.4GHz "Puck" dongle fix

## Summary

The `steam2026` driver (`SteamController2026.cpp`/`.h`) already had correct button/axis
mapping for this hardware, but never actually parsed any input on the 2.4GHz wireless
dongle variant (VID `0x28de` PID `0x1304`, "Steam Controller (2026) Wireless (Puck)").
The controller would be detected and successfully attached as a virtual HDLS pad, but
no button or stick presses would ever register.

## Root cause

`SteamController2026::ParseData()` only accepted input reports with `report_id ==
REPORT_INPUT` (`0x45`). Live USB capture against real hardware (both a direct Windows
HID capture and an on-device trace log from the Switch itself) showed the wireless
puck consistently sends its input reports with `report_id == 0x42`, never `0x45`. As a
result every single report was silently dropped in the `else` branch of `ParseData()`
and the controller appeared completely dead in-game and in the Switch's own
"Test Input Devices" screen, despite `sys-con`'s log showing a successful
initialization.

`0x45` may be correct for the **wired** Steam Controller 2026 (VID `0x28de` PID
`0x1302`) - this wasn't tested here, only the wireless puck. The fix accepts both IDs
so both variants should keep working:

```c
// SteamController2026.h
#define REPORT_INPUT             0x45
#define REPORT_INPUT_WIRELESS    0x42

// SteamController2026.cpp
if (controllerData->report_id == REPORT_INPUT || controllerData->report_id == REPORT_INPUT_WIRELESS)
```

The rest of the `Steam2026InputReport` struct layout (buttons, triggers, sticks,
trackpads, IMU) was verified against the same captures and lines up correctly - no
other changes were needed.

## How this was found

1. Captured raw HID reports directly from the dongle on a PC (Windows `ReadFile` on
   the vendor-defined HID collection), isolating button/axis bytes by asking for a
   scripted sequence of presses and diffing which bytes changed and when.
2. Deployed a debug build of `sys-con` to the Switch with `log_level=0` (Trace) to
   confirm the exact same `report_id` (0x42) was arriving on-device, and to confirm
   `ParseData()` was hitting the "unhandled report id" log line on every single frame.
3. Patched the report ID check, rebuilt, and confirmed both digital and analog input
   now work correctly in the Switch's "Test Input Devices" screen.

## Config notes specific to this unit

The default `[steam2026]` face button mapping in `config.ini` (`B=1 A=2 Y=4 X=3`) does
**not** match the button labels printed on this controller - pressing the button
labeled `A` triggers the Switch's `Y`, etc. If you want the labels to match, use:

```ini
[steam2026]
X=1
Y=2
B=3
A=4
```

This is a personal-preference remap (label-matching vs. Xbox-style position-matching),
not a bug fix - leave the defaults if you prefer position-matching instead.

## Known gaps

The following inputs were not individually re-verified against this specific unit
after the report ID fix (they come from the existing `steam2026` driver/config and are
expected to work based on the struct layout, but weren't exhaustively pressed one by
one): back paddles (`l4`/`l5`/`r4`/`r5`), grips, and trackpad click vs. touch
distinction. If something doesn't respond as expected, use
`Settings -> Controllers & Sensors -> Test Input Devices` on the Switch with
`log_level=1` (Debug) in `config.ini` to see the live `B1..B18`/`DPAD` state dump per
`doc/Troubleshooting.md`, and adjust `[steam2026]` in `config.ini` accordingly.

**Important**: leave `log_level` at `3` (Info) or higher for normal use. Levels `0`
(Trace) and `1` (Debug) write a full hex dump of every USB report to the SD card and
make input noticeably laggy - only use them while actively debugging.
