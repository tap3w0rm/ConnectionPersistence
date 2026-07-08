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
- primary endpoint probe interval.

The primary profile remains preferred. The backup profile is a continuity path when the primary repeatedly fails.

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
probe_interval_seconds = 60
```

Probe behavior:

```text
backup VPN active
  -> wait configured interval
  -> attempt TCP connect to primary host:port
  -> if connectable, attempt controlled switchback to primary
  -> if primary switchback fails, keep backup active and continue probing
```

The probe must not disconnect a healthy backup VPN until primary switchback is actually being attempted.

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
