# Config And Paths

## App Name

```text
ConnectionPersistence
```

Use this exact spelling everywhere.

## Config Path

```text
%APPDATA%\ConnectionPersistence\config.json
```

Config should be JSON, not CSV. CSV is for logs.

## Log Paths

Main monthly event logs:

```text
%LOCALAPPDATA%\ConnectionPersistence\logs\events-YYYY-MM.csv
```

Per-adapter counter logs:

```text
%LOCALAPPDATA%\ConnectionPersistence\logs\ethernet\<sanitized-adapter-name>-YYYY-MM.csv
```

Startup error log if needed:

```text
%LOCALAPPDATA%\ConnectionPersistence\startup_error.log
```

## Config Model

Expected top-level settings:

```json
{
  "theme": "system",
  "watching_enabled": true,
  "persistence_enabled": true,
  "primary_vpn_profile": "",
  "backup_vpn_profile": "",
  "primary_min_retries_before_failover": 10,
  "backup_max_retries_before_primary_fallback": 10,
  "primary_probe_source": "auto",
  "primary_probe_host": "",
  "primary_probe_protocol": "auto",
  "primary_probe_port": 443,
  "primary_probe_interval_seconds": 60,
  "route_test_target": "8.8.8.8",
  "include_wifi_as_uplink": false,
  "managed_vpn_support": false,
  "startup_with_windows": true,
  "log_retention_days": 30,
  "sounds": {
    "enabled": true,
    "connection_up": null,
    "connection_down": null,
    "vpn_down": null,
    "vpn_restored": null
  },
  "alerts": {
    "visual_popups_enabled": true,
    "suppress_visuals_during_fullscreen": true
  }
}
```

The final schema may evolve, but these are the required concepts.

`primary_probe_source` controls whether endpoint probe values are derived from the Windows VPN profile or manually overridden.

Expected values:

```text
auto
manual
```

When set to `auto`, the app should read:

```text
RASENTRYW.szLocalPhoneNumber -> probe host
RASENTRYW.dwVpnStrategy      -> protocol/port inference
```

When set to `manual`, the configured host, protocol, and port take precedence.
