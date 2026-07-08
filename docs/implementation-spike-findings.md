# Implementation Spike Findings

This document records non-code findings from the initial implementation spike.

## Native UI Stack

The native Rust UI stack remains viable for the configuration window.

Confirmed direction:

- `egui` and `eframe` can support a compact native utility window.
- The current `eframe` app trait uses a root `ui` method rather than the older `update` method.
- A compact panel-based layout works for the planned configuration surface.
- A local-only release build of the initial native configuration app completed successfully.
- The local-only release executable launched successfully on Windows.

Open item:

- Tray-resident lifecycle still needs a dedicated spike before implementation is accepted.

## Configuration And Logging

The planned file locations remain appropriate:

- config under `%APPDATA%\ConnectionPersistence\config.json`,
- logs under `%LOCALAPPDATA%\ConnectionPersistence\logs`.

The configuration schema needs explicit probe metadata fields:

- `primary_probe_source`,
- `primary_probe_protocol`,
- `primary_probe_host`,
- `primary_probe_port`,
- `primary_probe_interval_seconds`.

The local-only build verified that the config/control surface can represent the primary/backup VPN settings, probe settings, route target, theme, alerts, sounds, and retention controls in one compact utility window.

## VPN Profile Discovery

PowerShell can expose high-level VPN profile fields for diagnostics, including profile name, server address, tunnel type, connection status, and GUID.

The production implementation should still use RAS APIs directly:

- `RasEnumEntriesW` for profile names,
- `RasGetEntryPropertiesW` for `RASENTRYW`,
- `RASENTRYW.szLocalPhoneNumber` for server address,
- `RASENTRYW.dwType` for VPN entry filtering,
- `RASENTRYW.dwVpnStrategy` for strategy/type inference,
- `RASENTRYW.guidId` for stable profile correlation.

## Probe Derivation

Protocol-aware endpoint probing should remain classified by confidence:

- SSTP: TCP 443, strong signal.
- PPTP: TCP 1723, partial signal because GRE protocol 47 is not proven.
- IKEv2: UDP 500/4500, weak signal because UDP probing is not definitive.
- L2TP/IPsec: UDP 500/4500/1701, weak signal because IPsec/ESP is not proven.
- Automatic: infer if possible, otherwise require manual override.

TCP endpoint probing is straightforward for SSTP-like cases. UDP probing should be recorded as inconclusive unless a stronger health-check endpoint is configured.

## Adapter Discovery

Physical adapter discovery should be implemented with Windows APIs, but PowerShell diagnostics can help validate filtering assumptions.

Production implementation should use:

- `GetIfTable2`,
- `GetIfEntry2`,
- `GetAdaptersAddresses`,
- `MIB_IF_ROW2`.

Virtual adapter filtering must remain explicit and conservative.

## Remaining Required Spikes

These items are not yet proven:

- tray icon lifecycle with hidden/showable config window,
- direct RAS reconnect engine,
- exact active VPN interface mapping,
- `GetBestRoute2` route validation,
- physical adapter error counter polling,
- non-activating upper-right alert windows,
- WAV playback behavior,
- fullscreen/game suppression heuristics.
