# VPN Persistence

VPN persistence centers on a primary Windows built-in VPN profile and can optionally fail over to one backup Windows built-in VPN profile.

## Profiles

Configuration includes:

- primary VPN profile,
- backup VPN profile,
- minimum primary retry count before backup failover,
- maximum backup retry count before attempting fallback behavior,
- primary endpoint host,
- primary endpoint port,
- primary endpoint probe protocol,
- primary endpoint probe interval.

The primary profile remains preferred. The backup profile is a continuity path when the primary repeatedly fails.

## VPN Profile Metadata

The app should read Windows built-in VPN profile metadata from RAS instead of requiring duplicate manual entry.

Native metadata source:

```text
RasEnumEntriesW
  -> profile names

RasGetEntryPropertiesW
  -> RASENTRYW
```

Relevant `RASENTRYW` fields:

```text
szLocalPhoneNumber
dwType
dwVpnStrategy
guidId
dwRedialCount
```

Usage:

- `dwType` identifies whether an entry is a VPN entry.
- `szLocalPhoneNumber` is the VPN server DNS name or IP address for VPN profiles and should be used as the default primary probe host.
- `dwVpnStrategy` describes the VPN strategy/type and should be used to derive a default probe method.
- `guidId` provides a stable profile identifier for logs/config correlation.
- `dwRedialCount` is useful diagnostic context but does not replace app-managed retry policy.

PowerShell `Get-VpnConnection` may be used for diagnostics because it exposes similar high-level fields such as `ServerAddress` and `TunnelType`, but the application should use RAS APIs directly.

## Derived Probe Defaults

Endpoint probe defaults should be derived from `RASENTRYW.dwVpnStrategy` when possible.

```text
VS_SstpOnly / VS_SstpFirst:
  host = szLocalPhoneNumber
  protocol = tcp
  port = 443
  confidence = strong

VS_PptpOnly / VS_PptpFirst:
  host = szLocalPhoneNumber
  protocol = tcp
  port = 1723
  confidence = partial
  limitation = GRE protocol 47 is also required and is not proven by TCP 1723

VS_Ikev2Only / VS_Ikev2First:
  host = szLocalPhoneNumber
  protocol = udp
  ports = 500, 4500
  confidence = weak
  limitation = UDP probing is not definitive

VS_L2tpOnly / VS_L2tpFirst:
  host = szLocalPhoneNumber
  protocol = udp
  ports = 500, 4500, 1701
  confidence = weak
  limitation = IPsec/ESP behavior is not proven by UDP socket behavior

VS_Default / Automatic:
  host = szLocalPhoneNumber
  protocol = inferred when possible
  confidence = variable
  limitation = Windows may try multiple protocols or alter strategy after success/failure
```

Manual probe override must remain available because Windows profiles do not always expose a clean custom probe port, and protocol-derived UDP probes are only reachability signals.

## Health Definition

The active VPN is healthy only when both are true:

1. RAS/Windows reports the active VPN connected.
2. `GetBestRoute2` shows the configured route target uses the active VPN interface.

Default route target:

```text
8.8.8.8
```

## Reconnect Rules

```text
one active connection attempt at a time
no overlapping attempts
no retry limit
retry immediately after each failed attempt returns
pause when physical Ethernet link is absent
resume immediately when Ethernet link returns
```

Manual VPN disconnect is overridden when:

- watching is enabled,
- persistence is enabled,
- a primary or active backup VPN is selected,
- physical Ethernet link is present.

## State Machine

```text
Watching disabled:
  no persistence action

No physical Ethernet link:
  pause reconnect attempts

Ethernet link present + primary VPN healthy:
  normal primary state

Ethernet link present + primary VPN unhealthy:
  retry primary until minimum failover retry count is reached

Primary still failing after minimum retry count:
  attempt backup VPN

Backup VPN healthy:
  remain on backup
  periodically probe primary endpoint host and port

Primary endpoint probe succeeds:
  attempt controlled switchback to primary

Primary restored and route validation passes:
  return to normal primary state

Active VPN connected but route invalid:
  treat as unhealthy
  reset/retry active profile according to primary/backup state
```

## Failure Classification

Connection failures must retain as much diagnostic detail as Windows provides.

At minimum, classify:

- timeout,
- unreachable endpoint,
- authentication failure,
- DNS/name resolution failure,
- route validation failure,
- user/credential prompt required,
- unknown RAS error.

Timeout and unreachable failures are important for failover. If primary attempts fail by timeout or unreachable status but the backup connects, the app should continue probing the primary endpoint instead of assuming primary recovery.

## Primary Endpoint Probe

When backup is active, the app should periodically test whether the primary VPN endpoint is connectable.

This is a lightweight host/port check, not a VPN login attempt.

Example:

```text
primary_vpn_host = vpn.example.com
primary_vpn_port = 443
primary_vpn_probe_protocol = tcp
probe_interval_seconds = 60
```

Probe behavior:

```text
backup VPN active
  -> wait configured interval
  -> run the configured or derived primary endpoint probe
  -> if connectable, attempt controlled switchback to primary
  -> if primary switchback fails, keep backup active and continue probing
```

The probe must not disconnect a healthy backup VPN until primary switchback is actually being attempted.

Probe result states should include:

- reachable,
- timeout,
- unreachable,
- DNS failure,
- unsupported probe type,
- inconclusive.

For UDP-based VPN types, `reachable` should be treated as a weak signal unless a separate TCP health-check endpoint is configured.

## Recovery Summary

When restored, alert and log:

- active VPN name,
- whether restoration used primary or backup,
- whether failover or switchback occurred,
- down time,
- restored time,
- outage duration,
- attempt count,
- route target,
- last error if relevant.
