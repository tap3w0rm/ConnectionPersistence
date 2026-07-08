# ConnectionPersistence

ConnectionPersistence is a Windows-only Rust tray application planned to keep a primary Windows built-in VPN alive whenever a real physical Ethernet link is present, with optional failover to a configured backup VPN profile.

The app will run primarily from the system tray, reconnect the configured primary VPN aggressively when it drops, fail over to a backup VPN after a configurable retry threshold, verify that a configurable route target uses the active VPN interface, avoid disrupting fullscreen games, and write verbose CSV logs.

The application must avoid operator disruption. It should not steal focus, should suppress visual alerts during risky fullscreen/game states, and should rely on sounds, tray state, and logs when popups are unsafe.

## Why This Exists

Windows built-in VPN auto-reconnect is not always persistent enough. In some failure modes Windows may retry, fail, and eventually stop trying. That leaves the machine off the expected VPN until manual reconnection occurs.

ConnectionPersistence exists to keep trying. If the primary VPN is down or connected but not routing the configured test target through the VPN interface, the app treats it as unhealthy and keeps reconnecting while a real Ethernet link is present. There is no retry limit, and every attempt is logged.

If the primary VPN continues failing after the configured minimum retry count, the app can fail over to a configured backup VPN profile. While the backup is active, the app periodically probes the primary VPN endpoint host and port. When the primary endpoint becomes reachable again, the app attempts a controlled switchback to the primary VPN.

Primary endpoint probe defaults should be derived from the Windows VPN profile when possible. The app should read the RAS profile server address and VPN strategy, use the server address as the default probe host, infer a protocol-aware default port, and allow manual override when the inferred probe is weak or incomplete.

Failure reasons should be recorded from Windows/RAS and app-level checks. Timeout and unreachable failures are especially important because a timeout against the primary followed by a successful backup connection indicates that the primary endpoint may be unavailable while general network/VPN capability still works.

## Adapter Error Monitoring

The app also monitors physical non-virtual Ethernet adapters for Windows-reported packet/error counters. It does not capture packet contents and does not require Npcap, WinPcap, or traffic inspection.

For each real Ethernet adapter, the app reads Windows interface statistics and writes a per-adapter CSV row when counters increase, including:

- inbound errors,
- outbound errors,
- inbound discards,
- outbound discards,
- unknown protocol packets.

Each adapter gets its own log file based on the sanitized adapter name, so the reported physical NIC errors and driver-level packet issues are easy to inspect.

## Documentation

- [Product Plan](plan.md): Complete product plan and decision record.
- [Architecture](docs/architecture.md): Process shape, modules, lifecycle, and boundaries.
- [Technology Choices](docs/technology.md): Rust crates, UI stack, fallback order, and storage choices.
- [Windows APIs](docs/windows-apis.md): Windows APIs selected for VPN, adapters, routing, sound, startup, and alerts.
- [Config And Paths](docs/config-and-paths.md): Config model, directories, and file locations.
- [Logging](docs/logging.md): CSV files, schemas, retention, and adapter counter logs.
- [VPN Persistence](docs/vpn-persistence.md): VPN reconnect state machine and route validation rules.
- [Adapter Monitoring](docs/adapter-monitoring.md): Physical adapter filtering and Ethernet link detection.
- [Alerts And Sounds](docs/alerts-and-sounds.md): Popup, fullscreen, and WAV sound behavior.
- [Prototype Spike Plan](docs/spike-plan.md): Technical proof plan before full implementation.
- [Implementation Spike Findings](docs/implementation-spike-findings.md): Non-code findings from early implementation exploration.
- [Build And Release](docs/build-and-release.md): Expected build, packaging, and release direction.

## First Engineering Step

Build a narrow spike before the full app:

1. Start hidden in the tray.
2. Open and hide an egui config window from tray actions.
3. Enumerate Windows built-in VPN profiles.
4. Read VPN server address and strategy from RAS profile properties.
5. Derive a default endpoint probe from the VPN strategy.
6. Enumerate physical Ethernet adapters and exclude virtual adapters.
7. Read adapter counters.
8. Validate route lookup for `8.8.8.8`.
9. Classify at least timeout and unreachable VPN connection failures.
10. Probe a configured or derived primary VPN endpoint host and port.
11. Show a safe upper-right test alert or fall back to raw Win32.
12. Write one CSV event row.

## Current Status

This repository currently contains planning and implementation documentation only. The first implementation step is a technical spike that proves tray lifecycle, VPN profile enumeration, physical adapter detection, route validation, and safe alert behavior.
