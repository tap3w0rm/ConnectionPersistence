# Architecture

ConnectionPersistence should be built as a single Windows desktop process with clear internal boundaries.

## Process Shape

```text
connectionpersistence.exe
  tray controller
  config window
  watcher engine
  VPN persistence engine
  alert and sound manager
  CSV logging
```

The app starts with Windows, runs primarily in the tray, and shows the configuration window only when needed.

Closing the config window hides it. It does not stop the process. Only `Exit` from the tray exits the app.

## Module Boundaries

The core logic must not depend on egui. UI can change without rewriting VPN, adapter, route, sound, or logging code.

Suggested modules:

```text
app/
alerts/
config/
logging/
network/
tray/
ui/
vpn/
win32/
```

## Event Flow

```text
Windows network event or poll tick
  -> adapter monitor updates state
  -> VPN persistence engine evaluates state
  -> reconnect engine acts if needed
  -> logger records event
  -> alert manager decides popup/sound/tray behavior
```

## Fallback Strategy

Preferred:

```text
egui/eframe config UI + tray-icon + egui alert viewports
```

Fallback order:

```text
1. egui config + raw Win32 alert windows
2. custom winit + egui shell
3. Tauri only if native lifecycle blocks the project
```

