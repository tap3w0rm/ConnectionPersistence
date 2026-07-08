# VPN Persistence

VPN persistence applies only to one selected Windows built-in VPN profile.

## Health Definition

The VPN is healthy only when both are true:

1. RAS/Windows reports the selected VPN connected.
2. `GetBestRoute2` shows the configured route target uses the selected VPN interface.

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
- a VPN is selected,
- physical Ethernet link is present.

## State Machine

```text
Watching disabled:
  no persistence action

No physical Ethernet link:
  pause reconnect attempts

Ethernet link present + VPN healthy:
  normal

Ethernet link present + VPN unhealthy:
  reconnect selected VPN until healthy

VPN connected but route invalid:
  treat as unhealthy
  reset/retry selected VPN
```

## Recovery Summary

When restored, alert and log:

- VPN name,
- down time,
- restored time,
- outage duration,
- attempt count,
- route target,
- last error if relevant.

