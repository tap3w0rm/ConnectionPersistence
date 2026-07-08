# ConnectionPersistence Plan

## Purpose

ConnectionPersistence is a Windows-only Rust tray application that keeps a selected Windows built-in VPN connection alive whenever a real physical Ethernet link is present.

The app is intended to be quiet most of the time. It lives in the system tray, watches network state, reconnects the configured VPN aggressively when needed, alerts without disrupting fullscreen/game sessions, and writes verbose CSV logs for later review.

The core product goal is reliability:

- Detect physical Ethernet link changes quickly.
- Ignore virtual adapters completely.
- Detect when the selected Windows VPN drops or is not routing correctly.
- Reconnect the selected VPN with no retry limit.
- Prove that traffic to a configured route target uses the VPN interface.
- Record exactly what happened, when it happened, and how many attempts recovery required.

## Product Name

Use the correct spelling:

```text
ConnectionPersistence
```

Use this spelling consistently for:

- executable name,
- app title,
- tray tooltip,
- config directory,
- log directory,
- startup registry entry,
- documentation,
- UI labels.

## Operating System Scope

Windows only.

No cross-platform abstraction is required for this version. Prefer direct Windows APIs where they produce safer or more reliable behavior.

Target normal user execution. The app should not require administrator elevation for its planned baseline behavior.

## High-Level Behavior

On launch:

```text
if no config exists:
  open configuration window
  require explicit Save before VPN persistence becomes active
else:
  start watching automatically
  remain primarily in the tray
```

The app starts with Windows and starts watching automatically after configuration exists.

The configuration window is secondary to the app lifecycle. Closing the configuration window hides it. It does not exit the app.

The tray is the primary control surface.

## Tray Behavior

Tray icon right-click menu:

```text
Start Watching
Stop Watching
Open Configuration
Exit
```

Tray double-click:

```text
Open Configuration
```

Tray menu behavior:

- `Start Watching` enables network monitoring and persistence logic.
- `Stop Watching` cancels monitoring and any active reconnect loop.
- `Open Configuration` shows the configuration window.
- `Exit` cancels active work, flushes logs/config as needed, removes the tray icon, and exits.

The tray icon should reflect state:

- stopped,
- watching/healthy,
- reconnecting,
- uplink unavailable,
- error.

The tray tooltip should expose current state, for example:

```text
ConnectionPersistence - Watching
ConnectionPersistence - Reconnecting Work VPN, attempt 12
ConnectionPersistence - Ethernet unavailable
ConnectionPersistence - Stopped
```

## Configuration Window

The configuration window should use a native compact utility style:

- native Rust `egui`/`eframe`,
- compact panels,
- small labels,
- dense but readable operational layout,
- simple bordered sections,
- explicit light/dark colors,
- no marketing layout,
- no oversized hero content.

The configuration window should include these sections:

- Connection Persistence
- Valid Uplinks
- Alerts
- Sounds
- Logging
- Startup
- Advanced

### Theme

Theme options:

```text
System
Light
Dark
```

Default:

```text
System
```

Theme applies to:

- configuration window,
- alert popups,
- any secondary event-log windows.

### Connection Persistence Section

Controls:

- Enable connection persistence checkbox.
- Dropdown of Windows built-in VPN profiles.
- Route test target field.
- Connect and Test button.

Defaults:

```text
enabled = true after first saved config
route_test_target = 8.8.8.8
```

First run requires explicit Save. Persistence should not start until a VPN profile has been selected and the configuration has been saved.

If no VPN is selected:

- monitoring may run,
- network changes may be logged/alerted,
- VPN reconnect enforcement is inactive.

### Connect and Test Button

The button should be labeled:

```text
Connect and Test
```

Expected behavior:

1. Confirm selected VPN profile still exists.
2. Attempt to connect it using the normal app VPN path.
3. Confirm Windows reports the selected VPN connected.
4. Resolve the VPN interface.
5. Verify the configured route target routes through that VPN interface.
6. Show pass/fail details in the config window.
7. Write verbose CSV events.

### Valid Uplinks Section

Default rule:

```text
Any physical non-virtual Ethernet adapter with link present counts as a valid uplink.
```

USB Ethernet and docking-station Ethernet count.

Virtual adapters never count.

Optional checkbox:

```text
Include Wi-Fi as valid uplink
```

Default:

```text
false
```

The persistence gate is Ethernet link present, not internet reachability.

This means:

- DNS failure does not prevent persistence.
- Internet check failure does not prevent persistence.
- Physical Ethernet link present is enough to begin reconnect attempts.

### Alerts Section

Visual alerts are custom app-owned popups, not native Windows toast notifications.

Reason:

- Windows toast placement is controlled by the OS.
- The app needs upper-right placement.
- Toasts may conflict with game HUD areas.

Default alert behavior:

```text
normal desktop / safe borderless fullscreen:
  show upper-right custom popup

risky exclusive fullscreen / game mode:
  suppress visual popup
  log event
  play sound if enabled
  queue summary popup for later
```

Popups:

- appear on the Windows primary monitor,
- use monitor work area, not raw full-screen bounds,
- anchor to upper-right,
- animate downward,
- stack downward,
- auto-dismiss,
- use the configured theme,
- do not steal focus.

The popup implementation must be conservative:

- no `SetForegroundWindow`,
- no forced focus,
- no keyboard activation,
- no taskbar presence,
- no Alt+Tab presence,
- no display mode changes.

If egui viewports cannot provide this safely, implement alert popups using raw Win32 windows.

### Alert Frequency

Network connection status changes:

- alert every valid non-virtual network status change.
- virtual adapters never alert.

VPN reconnect alerts:

- popup on attempt 1,
- popup every 10 attempts after that,
- final recovery popup when VPN is restored.

Do not popup on every reconnect failure.

Example reconnect popup cadence:

```text
attempt 1: show "Reconnecting VPN"
attempt 10: show "Still reconnecting"
attempt 20: show "Still reconnecting"
attempt 30: show "Still reconnecting"
...
restored: show recovery summary
```

Final recovery alert should include:

- VPN name,
- time VPN went down,
- time VPN came back up,
- outage duration,
- attempt count,
- route target,
- route validation result,
- last error if relevant.

### Sounds Section

Support distinct sounds for:

- connection up,
- connection down,
- VPN went down,
- VPN came back up.

Default:

```text
standard Windows alert sound
```

Configuration supports selecting any `.wav` file on the system for each event.

If a custom WAV file is missing or fails to play:

- fall back to the standard alert sound,
- log `SOUND_FAILED`,
- continue normal operation.

Sound playback must not block the watcher or reconnect loop.

### Logging Section

Use verbose CSV logs.

Retention:

- configurable in 30-day increments,
- minimum 30 days,
- maximum 420 days,
- default 30 days.

Main event logs should be monthly CSV files:

```text
%LOCALAPPDATA%\ConnectionPersistence\logs\events-YYYY-MM.csv
```

Per-physical-Ethernet adapter bad-counter logs should be separate monthly CSV files:

```text
%LOCALAPPDATA%\ConnectionPersistence\logs\ethernet\<sanitized-adapter-name>-YYYY-MM.csv
```

Config should not be CSV. Use JSON for settings:

```text
%APPDATA%\ConnectionPersistence\config.json
```

Reason:

- CSV is good for logs.
- JSON is better for structured settings.

### Startup Section

The app starts with Windows by default after first valid configuration.

Use the per-user HKCU Run key:

```text
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
```

This avoids requiring administrator privileges.

### Advanced Section

Include an option for Always On / managed VPN profile support.

Default:

```text
best-effort / disabled unless enabled by user
```

Managed Always On VPN profiles may have policy quirks. Treat this as an explicit config option rather than silently assuming full support.

## VPN Behavior

Scope:

```text
Windows built-in VPN profiles only.
```

Third-party VPN clients are out of scope:

- Cisco AnyConnect,
- GlobalProtect,
- OpenVPN GUI,
- WireGuard app,
- FortiClient,
- vendor-specific clients.

The app does not store VPN credentials. Windows must already have saved credentials or a usable authentication method.

### VPN Profile Enumeration

Use RAS APIs as the primary implementation path.

Likely APIs:

- `RasEnumEntries`
- `RasGetEntryProperties`

The UI dropdown should show only Windows built-in VPN profiles, not arbitrary virtual adapters.

PowerShell `Get-VpnConnection` may be used only as a diagnostic fallback during development.

### VPN Connection

Use RAS APIs as primary:

- `RasDial`
- `RasEnumConnections`
- `RasGetConnectStatus`
- `RasHangUp`
- `RasGetErrorString`
- `RasGetProjectionInfoEx`

Do not use Settings UI automation.

Do not use `rasdial.exe` as the primary implementation.

`rasdial.exe` is acceptable only as:

- a diagnostic fallback,
- a manual troubleshooting comparison,
- a last-resort recovery option if direct API behavior is blocked.

### VPN Health

VPN is healthy only when both are true:

1. Windows/RAS reports the selected VPN is connected.
2. The configured route target routes through the selected VPN interface.

Default route target:

```text
8.8.8.8
```

Ping success alone is not enough.

The route to the target must specifically use the VPN interface.

### Route Validation

Use IP Helper / NetIO route APIs.

Primary route API:

```text
GetBestRoute2
```

Validation flow:

```text
selected VPN profile
  -> active RAS connection
  -> projection/interface details
  -> route lookup for configured target
  -> compare returned route interface to selected VPN interface
```

`RasGetProjectionInfoEx` is likely the cleanest RAS-side API for PPP/IKEv2 projection information.

If Windows reports the VPN connected but route validation fails:

```text
treat VPN as unhealthy
log route failure
disconnect/reset selected VPN if needed
reconnect selected VPN
increment attempt count
```

### Reconnect Loop

Only the selected default VPN is forcefully reconnected.

Manual disconnect is overridden if:

- watching is enabled,
- connection persistence is enabled,
- Ethernet link is present,
- selected VPN is unhealthy.

Reconnect rules:

```text
one active connection attempt at a time
no overlapping attempts
no retry limit
immediate retry after each completed failure
pause when Ethernet link disappears
resume immediately when Ethernet link returns
```

No startup delay.

No startup delay is used. Ethernet and VPN changes should be handled immediately.

### Ethernet Gate

Persistence state machine:

```text
Watching disabled:
  do nothing

Watching enabled + no physical Ethernet link:
  pause VPN reconnect attempts
  continue monitoring adapters
  log state change

Watching enabled + physical Ethernet link present + VPN healthy:
  normal healthy state

Watching enabled + physical Ethernet link present + selected VPN unhealthy:
  reconnect selected VPN immediately
  retry until healthy

Ethernet returns:
  immediately resume VPN persistence if selected VPN is unhealthy
```

## Network Adapter Monitoring

Virtual adapters are excluded from:

- valid uplink decisions,
- network status alerts,
- per-adapter packet/stat logs.

Examples of excluded adapters:

- VMware virtual adapters,
- Hyper-V virtual switches,
- loopback,
- tunnel adapters,
- VPN adapters,
- WAN miniports,
- Bluetooth PAN,
- other virtual NICs.

### Physical Ethernet Detection

Use Windows IP Helper / NetIO APIs.

Primary source:

```text
GetIfTable2 / MIB_IF_ROW2
```

Useful fields:

- `InterfaceAndOperStatusFlags.ConnectorPresent`
- `Type`
- `MediaConnectState`
- `OperStatus`
- interface alias/name
- interface LUID/index

Physical Ethernet rule should combine:

```text
ConnectorPresent == true
Type == Ethernet-like
not known virtual/tunnel/VPN/WAN type
not known virtual name/description/vendor pattern
```

Link present should be based on media/connect state, not internet reachability.

### Adapter Change Events

Use:

```text
NotifyIpInterfaceChange
```

This handles interface up/down and IP interface changes.

The watcher may still poll periodically for counters and sanity checks.

## Bad Packet / Adapter Counter Logging

The app does not inspect packets.

The app does not capture traffic.

The app does not require Npcap/WinPcap.

The app watches Windows-reported adapter statistics.

For each non-virtual physical Ethernet adapter:

- maintain last-seen counters,
- poll or refresh counters,
- when error/discard counters increase, write a row to that adapter's CSV file.

Counters of interest from `MIB_IF_ROW2`:

- `InErrors`
- `OutErrors`
- `InDiscards`
- `OutDiscards`
- `InUnknownProtos`

Example adapter log row:

```csv
timestamp,adapter_name,adapter_id,counter,previous_value,current_value,delta
2026-07-07T14:22:31-07:00,Intel(R) Ethernet Controller,I226-V,InErrors,120,123,3
```

Counter reset behavior:

- if current counter is less than previous counter, treat as adapter reset/driver reload,
- log counter reset,
- do not emit fake bad-packet delta,
- establish a new baseline.

## Fullscreen / Game Safety

The app must avoid causing:

- focus steals,
- game minimization,
- resolution changes,
- display mode flicker,
- Alt+Tab disruptions.

Default behavior:

```text
if fullscreen/game state appears risky:
  suppress visual popup
  log event
  play sound if enabled
  queue a summary popup for later
```

Signals to use:

- `SHQueryUserNotificationState`
- foreground window checks,
- monitor bounds vs foreground window bounds,
- foreground process/window style heuristics.

Do not rely on `SHQueryUserNotificationState` alone. It can report broad states such as busy instead of a precise fullscreen game state.

## Alert Popup Implementation

Target behavior:

- primary monitor only,
- top-right work area,
- slides down,
- stacks downward,
- does not activate,
- no taskbar icon,
- no Alt+Tab entry,
- themed light/dark/system,
- safe suppression in fullscreen.

Placement formula:

```text
primary_monitor = Windows primary monitor
work_area = primary_monitor work area

popup_width = 360
popup_height = 96
margin = 16
stack_gap = 10

final_x = work_area.x + work_area.width - popup_width - margin
final_y = work_area.y + margin + stack_index * (popup_height + stack_gap)

start_x = final_x
start_y = work_area.y - popup_height - margin
```

Implementation path:

1. Try egui viewport windows.
2. If egui cannot guarantee non-activating behavior, use raw Win32 popup windows.

Relevant Win32 behavior:

- `WS_EX_NOACTIVATE`
- tool-window/no taskbar style
- topmost positioning when safe
- no foreground activation calls

## Sounds

Use Windows `PlaySound`.

Requirements:

- support system sound alias for default alert,
- support user-selected `.wav` files,
- play asynchronously,
- fall back to default sound if custom path fails,
- log sound failures.

Sound events:

- connection up,
- connection down,
- VPN down,
- VPN restored.

Do not make sound playback block the reconnect engine.

## Storage

Config:

```text
%APPDATA%\ConnectionPersistence\config.json
```

Main logs:

```text
%LOCALAPPDATA%\ConnectionPersistence\logs\events-YYYY-MM.csv
```

Ethernet adapter counter logs:

```text
%LOCALAPPDATA%\ConnectionPersistence\logs\ethernet\<sanitized-adapter-name>-YYYY-MM.csv
```

Startup/early failure log if needed:

```text
%LOCALAPPDATA%\ConnectionPersistence\startup_error.log
```

## CSV Logging

Be verbose.

Main event log candidate columns:

```csv
timestamp,event_type,severity,message,vpn_name,uplink_type,uplink_name,adapter_id,route_target,down_time,up_time,duration_seconds,attempt,status,last_error,details
```

Event types should include at least:

- `APP_STARTED`
- `APP_EXITING`
- `CONFIG_OPENED`
- `CONFIG_SAVED`
- `WATCHING_STARTED`
- `WATCHING_STOPPED`
- `ETHERNET_LINK_UP`
- `ETHERNET_LINK_DOWN`
- `WIFI_LINK_UP`
- `WIFI_LINK_DOWN`
- `VPN_PROFILE_SELECTED`
- `VPN_DOWN`
- `VPN_CONNECTED`
- `VPN_ROUTE_TEST_STARTED`
- `VPN_ROUTE_TEST_PASSED`
- `VPN_ROUTE_TEST_FAILED`
- `VPN_RECONNECT_ATTEMPT`
- `VPN_RECONNECT_FAILED`
- `VPN_RESTORED`
- `VPN_RESET_REQUESTED`
- `ALERT_SHOWN`
- `ALERT_SUPPRESSED_FULLSCREEN`
- `ALERT_QUEUED`
- `ALERT_SUMMARY_SHOWN`
- `SOUND_PLAYED`
- `SOUND_FAILED`
- `ADAPTER_COUNTER_DELTA`
- `ADAPTER_COUNTER_RESET`
- `ERROR`

Reconnect attempts should be logged individually.

Popups should be less noisy than logs.

## Internal Architecture

Keep the UI separate from the core logic.

Suggested modules:

```text
app/
  state
  commands
  lifecycle

ui/
  egui_config_window
  theme
  panels
  event_log_view

tray/
  tray_controller
  tray_menu
  tray_icons

alerts/
  alert_manager
  popup_window
  fullscreen_guard
  sound_manager

config/
  config_model
  config_store
  migration

logging/
  csv_logger
  retention
  adapter_log_writer

network/
  adapter_inventory
  adapter_monitor
  adapter_stats
  physical_filter

vpn/
  ras_profiles
  ras_connection
  ras_status
  route_validator
  reconnect_engine

win32/
  ras_bindings
  ip_helper
  shell_state
  startup_registry
  sound
  windowing
```

Core logic should not depend on egui.

This keeps the fallback path clean:

```text
egui config + raw Win32 popups
```

or, if needed:

```text
winit/egui shell
```

without rewriting VPN/reconnect/logging behavior.

## Chosen Technology Stack

Primary:

- Rust
- `egui` / `eframe`
- `tray-icon`
- `windows` or `windows-sys`
- `serde`
- `serde_json`
- `csv`
- `chrono` or `time`

Windows APIs:

- RAS APIs for VPN
- IP Helper / NetIO APIs for adapters, counters, routes
- Shell APIs for fullscreen notification suitability
- WinMM `PlaySound`
- Registry APIs for HKCU Run startup

Fallbacks:

- raw Win32 alert windows if egui alerts are not safe enough,
- `rasdial.exe` only as diagnostic fallback,
- PowerShell only as development/debugging aid.

Do not start with Tauri.

Only consider Tauri if both egui/eframe and lower-level winit/Win32 lifecycle handling become unreasonable.

## Technical Research Findings

### Tray

`tray-icon` supports Windows tray icons and menus. On Windows it requires a running Win32 event loop, and the tray icon must be created on the same thread as that loop.

This makes integration with egui/eframe plausible but important to spike early.

### egui Viewports

egui viewport APIs expose useful window controls:

- visibility,
- decorations,
- transparency,
- taskbar visibility,
- topmost window level,
- mouse passthrough.

This may be enough for alert popups, but raw Win32 provides better control over non-activation.

### RAS

RAS APIs are the correct native Windows path for built-in VPN:

- `RasEnumEntries`
- `RasDial`
- `RasEnumConnections`
- `RasGetConnectStatus`
- `RasGetProjectionInfoEx`
- `RasHangUp`
- `RasGetErrorString`

`RasGetProjectionInfoEx` is important because it returns projection information for PPP/IKEv2 RAS connections and may help map active VPN connection state to interface/routing details.

### Adapters and Counters

`MIB_IF_ROW2` exposes:

- physical connector presence,
- type,
- media connect state,
- operational status,
- error counters,
- discard counters,
- unknown protocol counters.

This supports physical Ethernet filtering and bad-counter logging without packet capture.

### Route Validation

`GetBestRoute2` returns the best route for a destination. Use it to validate the configured route target uses the selected VPN interface.

### Fullscreen

`SHQueryUserNotificationState` helps determine whether notifications are appropriate, but it is not perfect. Use it with foreground-window heuristics.

### Startup

HKCU Run key is the no-admin path for start-with-Windows.

### Sound

`PlaySound` supports system event sounds and WAV file paths.

## Build Risk Ranking

Highest risk first:

1. Tray + egui hidden-window lifecycle.
2. Non-activating upper-right alert windows.
3. Mapping selected RAS VPN to exact interface used for route validation.
4. Physical/virtual adapter filtering across odd drivers.
5. Fullscreen/game suppression heuristics.
6. No-admin verification across all APIs.
7. Counter reset behavior after sleep/resume/driver reload.

## Prototype Spike Plan

Before building the full app, create a technical spike that proves the risky surfaces.

Spike goals:

1. Start app hidden in tray.
2. Show/hide egui config window from tray double-click/menu.
3. Keep process alive when config window closes.
4. Enumerate Windows built-in VPN profiles.
5. Enumerate physical Ethernet adapters.
6. Exclude obvious virtual adapters.
7. Read `MIB_IF_ROW2` counters.
8. Detect Ethernet link present.
9. If a VPN is active, inspect RAS status/projection.
10. Run `GetBestRoute2` against `8.8.8.8`.
11. Show one upper-right test alert without stealing focus.
12. Play default alert sound.
13. Write one CSV event log row.

Spike pass criteria:

- tray remains stable,
- config window opens/closes without exiting app,
- no admin prompt,
- VPN profiles appear,
- physical Ethernet adapter list is sane,
- virtual adapters are excluded,
- route lookup returns usable interface data,
- alert popup does not steal focus in normal desktop use,
- logs are written to expected local app data path.

If alert popup is unsafe in egui:

```text
keep egui config
switch alert popup implementation to raw Win32
```

If eframe tray lifecycle is unsafe:

```text
evaluate custom winit + egui shell
```

Only if native lifecycle remains a blocker:

```text
consider Tauri as final fallback
```

## MVP Definition

MVP includes:

- tray app,
- first-run config,
- VPN dropdown,
- Save-required first-run behavior,
- start/stop watching,
- physical Ethernet detection,
- virtual adapter exclusion,
- selected VPN reconnect loop,
- route validation to configured target,
- reconnect attempt counting,
- upper-right alert popups when safe,
- fullscreen suppression/summary behavior,
- WAV/default sounds,
- verbose CSV main event log,
- per-adapter counter CSV logs,
- retention cleanup,
- light/dark/system theme.

MVP excludes:

- third-party VPN clients,
- packet capture,
- storing VPN credentials,
- admin-only features,
- cross-platform support,
- snooze mode,
- custom network reachability rules beyond Ethernet link present,
- fancy installer polish unless needed to run startup cleanly.

## Requirements

- Windows only.
- Use Windows built-in VPN only.
- Use native Windows connection methods.
- VPN health requires route to `8.8.8.8` through VPN interface.
- Route target is configurable, default `8.8.8.8`.
- Any physical Ethernet adapter counts.
- Virtual adapters never count and never alert.
- Wi-Fi is optional by checkbox.
- App starts watching automatically after configuration.
- App starts with Windows.
- Retry as fast as safely possible.
- One active reconnect attempt at a time.
- VPN is restored as soon as connected and route validation passes.
- Recovery alert should include detailed information.
- Verbose CSV log.
- Log retention 30 to 420 days in 30-day increments, default 30.
- Stop Watching cancels reconnect loop.
- Exit cancels cleanly.
- Correct spelling is `ConnectionPersistence`.
- First run requires Save.
- Include Connect and Test button.
- Alert on every valid non-virtual network status change.
- Reconnect popups on first attempt and every 10 attempts.
- Use a compact native utility UI style if feasible.
- Light/dark/system theme required.
- Avoid disrupting fullscreen/game sessions.
- Upper-right alert placement is desired because common game HUD areas often use other corners.
- Add sounds for connection up/down and VPN down/restored.
- Use Windows adapter statistics for bad-packet/counter logging, not packet capture.

## Source References

Primary technical references:

- `tray-icon`: https://docs.rs/tray-icon/latest/src/tray_icon/lib.rs.html
- egui viewport commands: https://docs.rs/egui/latest/egui/viewport/enum.ViewportCommand.html
- Rust for Windows: https://learn.microsoft.com/en-us/windows/dev-environment/rust/rust-for-windows
- RAS API header: https://learn.microsoft.com/en-us/windows/win32/api/ras/
- `RasDial`: https://learn.microsoft.com/en-us/windows/win32/api/ras/nf-ras-rasdialw
- `RasEnumEntries`: https://learn.microsoft.com/en-us/windows/win32/api/ras/nf-ras-rasenumentriesa
- `RasEnumConnections`: https://learn.microsoft.com/en-us/windows/win32/api/ras/nf-ras-rasenumconnectionsa
- `RasGetProjectionInfoEx`: https://learn.microsoft.com/en-us/windows/win32/api/ras/nf-ras-rasgetprojectioninfoex
- RAS overview: https://learn.microsoft.com/en-us/windows/win32/rras/remote-access-start-page
- RAS phone books: https://learn.microsoft.com/en-us/windows/win32/rras/ras-phone-books
- `GetAdaptersAddresses`: https://learn.microsoft.com/en-us/windows/win32/api/iphlpapi/nf-iphlpapi-getadaptersaddresses
- `GetIfTable2`: https://learn.microsoft.com/en-us/windows/win32/api/netioapi/nf-netioapi-getiftable2
- `MIB_IF_ROW2`: https://learn.microsoft.com/en-us/windows/win32/api/netioapi/ns-netioapi-mib_if_row2
- `NotifyIpInterfaceChange`: https://learn.microsoft.com/en-us/windows/win32/api/netioapi/nf-netioapi-notifyipinterfacechange
- `GetBestRoute2`: https://learn.microsoft.com/en-us/windows/win32/api/netioapi/nf-netioapi-getbestroute2
- `SHQueryUserNotificationState`: https://learn.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-shqueryusernotificationstate
- `PlaySound`: https://learn.microsoft.com/en-us/previous-versions/dd743680(v=vs.85)
- Run and RunOnce registry keys: https://learn.microsoft.com/en-us/windows/win32/setupapi/run-and-runonce-registry-keys
