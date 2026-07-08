# Logging

Logging should be verbose and human-readable.

## Main Event Log

Path:

```text
%LOCALAPPDATA%\ConnectionPersistence\logs\events-YYYY-MM.csv
```

Candidate columns:

```csv
timestamp,event_type,severity,message,vpn_name,uplink_type,uplink_name,adapter_id,route_target,down_time,up_time,duration_seconds,attempt,status,last_error,details
```

Additional VPN failover fields may be added as separate columns or encoded in `details`:

```csv
active_vpn_role,primary_vpn_name,backup_vpn_name,failure_classification,ras_error_code,probe_host,probe_port,probe_result,failover_reason,switchback_reason
```

## Retention

Retention is configurable:

- minimum 30 days,
- maximum 420 days,
- increments of 30 days,
- default 30 days.

Retention cleanup should run at startup and periodically while watching.

## Reconnect Logging

Reconnect attempts are logged individually.

Failure classification should be logged whenever available:

- timeout,
- unreachable endpoint,
- authentication failure,
- DNS/name resolution failure,
- route validation failure,
- credential prompt required,
- unknown RAS error.

Failover and switchback events should be explicit:

- `VPN_PRIMARY_RETRY_THRESHOLD_REACHED`
- `VPN_FAILOVER_STARTED`
- `VPN_FAILOVER_SUCCEEDED`
- `VPN_FAILOVER_FAILED`
- `VPN_PRIMARY_PROBE_STARTED`
- `VPN_PRIMARY_PROBE_SUCCEEDED`
- `VPN_PRIMARY_PROBE_FAILED`
- `VPN_SWITCHBACK_STARTED`
- `VPN_SWITCHBACK_SUCCEEDED`
- `VPN_SWITCHBACK_FAILED`

Popup cadence is intentionally less noisy than logging:

- popup on attempt 1,
- popup every 10 attempts,
- popup on final recovery.

## Adapter Counter Logs

Path:

```text
%LOCALAPPDATA%\ConnectionPersistence\logs\ethernet\<sanitized-adapter-name>-YYYY-MM.csv
```

Candidate columns:

```csv
timestamp,adapter_name,adapter_id,counter,previous_value,current_value,delta
```

Watched counters:

- `InErrors`
- `OutErrors`
- `InDiscards`
- `OutDiscards`
- `InUnknownProtos`

If a counter decreases, treat it as a reset and establish a new baseline.
