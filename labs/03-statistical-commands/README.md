# Lab: `stats` vs `eventstats` vs `streamstats`

## Objective
Understand how each statistical command transforms the event stream differently.

## Dataset
Import `labs/data/auth_events.csv` into index `lab_auth`.

## Exercises

### 1. `stats` — Collapses Events to Summary
```spl
index=lab_auth EventCode=4625
| stats count BY user, src_ip
```
**Input**: 21 events
**Output**: 4 rows (alice/10.0.0.5, bob/10.0.0.10, charlie/192.168.1.100)
**Key**: Original events GONE. Only group fields + aggregates remain.

### 2. `eventstats` — Keeps Events, Adds Aggregate
```spl
index=lab_auth EventCode=4625
| eventstats count AS user_failures BY user
```
**Input**: 21 events
**Output**: 21 events (same count), each with new `user_failures` field
- alice events: `user_failures=4`
- bob events: `user_failures=2`
- charlie events: `user_failures=15`

**Use case**: Compare each event to group aggregate
```spl
index=lab_auth
| eventstats avg(bytes_out) AS avg_out BY src_ip
| where bytes_out > avg_out
```

### 3. `streamstats` — Running Calculation Per Event
```spl
index=lab_auth EventCode=4625
| sort 0 _time
| streamstats count AS seq BY user, src_ip
```
**Input**: 21 events (sorted)
**Output**: 21 events with `seq` field
- alice/10.0.0.5: seq=1,2,3,4
- bob/10.0.0.10: seq=1,2
- charlie/192.168.1.100: seq=1..15

**With window**:
```spl
index=lab_auth
| sort 0 _time
| streamstats window=3 current=f avg(bytes_out) AS moving_avg BY src_ip
```

**With reset**:
```spl
index=lab_auth
| sort 0 _time
| streamstats count(eval(EventCode=4625)) AS failures
    reset_after="(EventCode=4624)"
    BY user, src_ip
| where EventCode=4624 AND failures >= 3
```

### 4. Side-by-Side Comparison
```spl
index=lab_auth EventCode=4625
| sort 0 _time
| streamstats count AS stream_count BY user
| eventstats count AS event_count BY user
| stats count AS stats_count BY user
| table user stream_count event_count stats_count
```
**Result**: Each row shows the three different "count" meanings for that user.

### 5. Brute Force Detection Pattern
```spl
index=lab_auth (EventCode=4624 OR EventCode=4625)
| sort 0 _time
| streamstats count(eval(EventCode=4625)) AS failures
    reset_after="(EventCode=4624)"
    BY user, src_ip
| where EventCode=4624 AND failures >= 3
| table _time user src_ip failures
```
**Expected**: 
- alice/10.0.0.5: 4 failures then success → **MATCH** (failures=4)
- bob/10.0.0.10: 2 failures then success → no match
- charlie/192.168.1.100: 15 failures then success → **MATCH** (failures=15)

## Memory Comparison

| Command | Memory Pattern | Risk |
|---------|----------------|------|
| `stats` | O(groups) | Medium |
| `eventstats` | O(groups) + O(events) | High |
| `streamstats` | O(window) or O(events) | Low (windowed) / Medium (global) |

**Test**: Run each with `index=lab_auth` and check Job Inspector `command.stats.memory` / `command.streamstats.memory`

## Validation Checklist
- [ ] `stats` returns fewer rows than input
- [ ] `eventstats` returns same row count, adds field
- [ ] `streamstats` returns same row count, running value
- [ ] Brute force pattern matches alice (4) and charlie (15), not bob (2)
- [ ] Job Inspector shows memory differences

## Key Takeaway
> `stats` = **collapse** to summary
> `eventstats` = **annotate** each event with group aggregate
> `streamstats` = **running** calculation with optional window/reset