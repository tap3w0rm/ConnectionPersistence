# Windows APIs

This project should use direct Windows APIs through Rust's `windows` or `windows-sys` crate.

## VPN / RAS

Primary APIs:

- `RasEnumEntries`
- `RasGetEntryProperties`
- `RasDial`
- `RasEnumConnections`
- `RasGetConnectStatus`
- `RasGetProjectionInfoEx`
- `RasHangUp`
- `RasGetErrorString`

Purpose:

- list Windows built-in VPN profiles,
- read `RASENTRYW.szLocalPhoneNumber` for the VPN server DNS name or IP address,
- read `RASENTRYW.dwType` to filter VPN entries,
- read `RASENTRYW.dwVpnStrategy` to infer VPN strategy/type,
- read `RASENTRYW.guidId` for stable profile identification,
- read `RASENTRYW.dwRedialCount` for diagnostic context,
- connect primary and backup profiles,
- inspect active RAS connections,
- retrieve connection/projection state,
- reset only the active managed VPN when route validation fails,
- collect RAS error codes for failure classification.

## Primary Endpoint Probe

Use normal Windows socket APIs through Rust networking or WinSock-compatible calls for the primary endpoint host/port probe.

The probe is a lightweight endpoint reachability check to determine whether the primary VPN endpoint appears reachable while the backup VPN is active. It is not a VPN authentication attempt.

Default probe metadata should be derived from RAS profile data:

```text
RASENTRYW.szLocalPhoneNumber -> probe host
RASENTRYW.dwVpnStrategy      -> protocol and default port inference
```

Recommended inference:

```text
SSTP   -> TCP 443, strong signal
PPTP   -> TCP 1723, partial signal because GRE protocol 47 is also required
IKEv2  -> UDP 500/4500, weak signal because UDP probing is not definitive
L2TP   -> UDP 500/4500/1701, weak signal because IPsec/ESP is not proven
Auto   -> infer when possible or require override
```

Manual override fields must exist for probe host, protocol, port, and interval.

## Adapters And Interface State

Primary APIs:

- `GetIfTable2`
- `GetIfEntry2`
- `GetAdaptersAddresses`
- `NotifyIpInterfaceChange`

Important `MIB_IF_ROW2` fields:

- `InterfaceAndOperStatusFlags.ConnectorPresent`
- `Type`
- `MediaConnectState`
- `OperStatus`
- `InErrors`
- `OutErrors`
- `InDiscards`
- `OutDiscards`
- `InUnknownProtos`

## Route Validation

Use:

- `GetBestRoute2`

The app must prove the configured route target, default `8.8.8.8`, routes through the active VPN interface.

Ping alone is not enough.

## Fullscreen / Notification Suitability

Use:

- `SHQueryUserNotificationState`
- foreground window inspection
- monitor/window bounds heuristics

Do not trust one API alone for game/fullscreen safety.

## Sounds

Use:

- `PlaySound`

Support system sound aliases and user-selected `.wav` file paths.

## Startup

Use the current-user Run key:

```text
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
```

This avoids requiring admin elevation.
