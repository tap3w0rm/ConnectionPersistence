# Technology Choices

## Primary Stack

- Rust
- `egui` / `eframe` for the configuration UI
- `tray-icon` for Windows tray icon and menu
- `windows` or `windows-sys` for direct Windows APIs
- `serde` and `serde_json` for config
- `csv` for user-readable logs
- `time` or `chrono` for timestamps and retention logic

## UI Direction

The config window should use a native compact utility style:

- compact panels,
- dense operational layout,
- explicit light/dark colors,
- small labels,
- simple bordered sections.

Theme options:

```text
System
Light
Dark
```

## Why Not Tauri First

Tauri is a valid fallback, but the project target is a native Rust utility. Starting with egui keeps the app smaller and avoids a webview.

## Why Direct Windows APIs

This app is Windows-only and relies on Windows networking behavior. Direct APIs provide better control and logging than shelling out to PowerShell or parsing command output.

`rasdial.exe` and PowerShell are diagnostic fallbacks, not primary mechanisms.
