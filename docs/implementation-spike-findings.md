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
- A local-only tray integration build completed successfully with `tray-icon` and `eframe` in the same process.
- The local-only tray-enabled executable launched and stayed alive as a background process.

Open item:

- Tray menu command behavior still needs manual UI verification from the Windows notification area.
- Close-to-hide behavior still needs manual UI verification.

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

PowerShell diagnostics showed that some adapter fields are serialized as numeric enum values rather than strings. For example, `Get-NetAdapter -Physical` can return `MediaConnectionState` as `1` or `2`. Diagnostic parsers and production adapters should normalize both numeric and string representations before presenting state or writing logs.

Observed diagnostic mapping:

- `MediaConnectionState = 1`: connected.
- `MediaConnectionState = 2`: disconnected.

The local-only implementation now verifies that adapter discovery can be performed directly through `GetIfTable2` without PowerShell. The API returns a `MIB_IF_TABLE2` containing `MIB_IF_ROW2` rows and the allocated table must be released with `FreeMibTable`.

Verified `MIB_IF_ROW2` fields for this project:

- `Alias`
- `Description`
- `InterfaceIndex`
- `Type`
- `PhysicalAddressLength`
- `OperStatus`
- `MediaConnectState`
- `TransmitLinkSpeed`
- `ReceiveLinkSpeed`
- `InErrors`
- `OutErrors`
- `InDiscards`
- `OutDiscards`
- `InUnknownProtos`

The local-only filter currently uses `Type == IF_TYPE_ETHERNET_CSMACD`, nonzero `PhysicalAddressLength`, and the existing virtual-name exclusion list. Further filtering should evaluate `InterfaceAndOperStatusFlags.ConnectorPresent` and related flags before finalizing the production physical-adapter rule.

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
