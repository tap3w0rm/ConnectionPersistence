# Adapter Monitoring

The app only monitors real physical network adapters for alerts and persistence decisions.

## Physical Ethernet Rule

A valid Ethernet uplink should satisfy:

```text
physical connector present
Ethernet-like interface type
media/link state connected
not virtual
not tunnel
not VPN
not loopback
not WAN miniport
```

USB Ethernet and docking-station Ethernet count.

## Virtual Adapter Exclusion

Virtual adapters never:

- count as uplinks,
- generate alerts,
- get per-adapter bad-counter logs.

Examples:

- VMware adapters,
- Hyper-V virtual switches,
- loopback,
- tunnel adapters,
- VPN adapters,
- WAN miniports,
- Bluetooth PAN.

## Data Sources

Use:

- `GetIfTable2`
- `GetIfEntry2`
- `GetAdaptersAddresses`
- `NotifyIpInterfaceChange`

## Link Present

The persistence gate is Ethernet link present, not internet reachability.

Do not require DNS, ping, or route availability before allowing VPN reconnect attempts.
