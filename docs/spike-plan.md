# Prototype Spike Plan

Before full implementation, build a narrow technical spike.

## Goals

1. Start hidden in tray.
2. Open egui config window from tray double-click.
3. Hide config window on close without exiting process.
4. Enumerate Windows built-in VPN profiles.
5. Enumerate physical Ethernet adapters.
6. Exclude obvious virtual adapters.
7. Read `MIB_IF_ROW2` counters.
8. Detect Ethernet link present.
9. Inspect active RAS VPN status/projection.
10. Run `GetBestRoute2` for `8.8.8.8`.
11. Show one upper-right alert without stealing focus.
12. Play one default alert sound.
13. Write one CSV event row.

## Pass Criteria

- No admin prompt.
- Tray remains stable.
- Config window can be opened, hidden, and reopened.
- Closing config does not exit the process.
- VPN profiles appear.
- Physical adapter list is sane.
- Virtual adapters are excluded.
- Route lookup returns usable interface data.
- Alert does not steal focus in normal desktop use.
- CSV log lands under `%LOCALAPPDATA%`.

## Fallback Decisions

If egui alert windows are unsafe:

```text
keep egui config
use raw Win32 popup windows for alerts
```

If eframe tray lifecycle is unsafe:

```text
move to custom winit + egui shell
```

If native lifecycle remains a blocker:

```text
consider Tauri
```

