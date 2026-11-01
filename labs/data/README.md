# Synthetic Dataset: `auth_events.csv`

## Description
35 synthetic Windows authentication and process events for hands-on labs. **No real user, host, IP, or credential data.**

## Schema

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Event time (2026-01-15 08:00-08:30) |
| `_raw` | string | Original event text |
| `host` | string | DC01, DC02, WEB01, FW01 |
| `source` | string | WinEventLog:Security, WinEventLog:System, firewall |
| `sourcetype` | string | WinEventLog:Security, WinEventLog:System, firewall |
| `index` | string | windows, firewall |
| `user` | string | alice, bob, charlie, admin |
| `EventCode` | int | 4624 (success), 4625 (failure), 4688 (process), 1102 (log clear) |
| `LogonType` | int | 3 (network) |
| `src_ip` | string | 10.0.0.5, 10.0.0.10, 192.168.1.100, 8.8.8.8 |
| `process_name` | string | powershell.exe, cmd.exe, wevtutil.exe |
| `CommandLine` | string | Process command line |
| `bytes_in` | int | Network bytes in |
| `bytes_out` | int | Network bytes out |
| `status` | string | allow (firewall) |
| `action` | string | success, failure, allow |

## Event Summary

| EventCode | Count | Description |
|-----------|-------|-------------|
| 4625 | 21 | Failed logons (alice:4, bob:2, charlie:15) |
| 4624 | 4 | Successful logons (alice:1, bob:1, charlie:1, admin:1) |
| 4688 | 5 | Process creation (PowerShell, cmd, wevtutil) |
| 1102 | 1 | Security log cleared |

## Attack Patterns Encoded

| Pattern | User | Events |
|---------|------|--------|
| Brute force (4 failures → success) | alice | 4×4625 + 1×4624 |
| Brute force (2 failures → success) | bob | 2×4625 + 1×4624 |
| Password spray (15 failures → success) | charlie | 15×4625 + 1×4624 |
| Encoded PowerShell | alice | 4688 with `-enc` |
| Process chain (winword → cmd → powershell) | alice | 4688 parent/child |
| Log clearing | admin | 1102 + 4688 wevtutil |
| Beaconing DNS | 10.0.0.5 | 11× firewall allow to 8.8.8.8:53 |

## Import Instructions

```bash
# In Splunk Web:
# Settings → Add Data → Upload → auth_events.csv
# Source type: _csv
# Index: lab_auth
# Timestamp: _time field
```

## Validation Queries

```spl
# Count by EventCode
index=lab_auth | stats count BY EventCode

# Failed logons by user
index=lab_auth EventCode=4625 | stats count BY user

# Process chain
index=lab_auth EventCode=4688 | sort 0 _time | autoregress process_name AS parent

# Brute force detection
index=lab_auth (EventCode=4624 OR EventCode=4625)
| sort 0 _time
| streamstats count(eval(EventCode=4625)) AS failures reset_after="(EventCode=4624)" BY user, src_ip
| where EventCode=4624 AND failures >= 3
```

## License
CC0 — Public domain. Use freely for learning and testing.