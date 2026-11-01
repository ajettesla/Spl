# Part 6 — Time-Based Analysis & Streaming Commands

> **Goal**: Master time as a first-class dimension. Understand `streamstats`, `eventstats`, `autoregress`, `delta` for sequence detection.

---

## 6.1 Time as a First-Class Dimension

Time isn't just a filter — it's an analytical dimension.

```
Three time operations:
━━━━━━━━━━━━━━━━━━━━━━━
1. LIMIT time range        → earliest/latest, WHERE earliest=...
2. BUCKET time             → bin/span, timechart
3. COMPUTE over time       → streamstats, eventstats, autoregress, delta
```

---

## 6.2 Search Time Range: `earliest`/`latest` vs `WHERE`

### Traditional SPL
```spl
index=windows earliest=-1h latest=now
index=windows earliest=-24h@h latest=@h
```

### SPL2 Pipeline
```spl2
search index=windows earliest=-1h latest=now
```

### SPL2 Structured
```spl2
FROM windows WHERE earliest=-1h AND latest=now
```

### Relative Time Syntax
| Specifier | Meaning |
|-----------|---------|
| `-15m` | Last 15 minutes |
| `-1h` | Last hour |
| `-24h` | Last 24 hours |
| `-7d` | Last 7 days |
| `-1h@h` | Last hour, snapped to hour boundary |
| `-1d@d` | Last day, snapped to day boundary |
| `now` | Search start time |

---

## 6.3 Time Bucketing: `bin`/`bucket` vs `span`

### Traditional SPL
```spl
index=windows
| bin _time span=10m
| stats count BY _time

# bucket is alias
| bucket _time span=1h
```

### SPL2 Pipeline
```spl2
search index=windows
| bin _time span=10m
| stats count() BY _time

# SPL2 exclusive: span in stats
search index=windows
| stats count() BY _time span=10m
```

### SPL2 Structured
```spl2
FROM windows
GROUP BY span(_time, 10m)
SELECT _time, count() AS count
```

### Span Options
| Span | Use Case |
|------|----------|
| `1m` / `5m` / `10m` | High-resolution alerting |
| `1h` | Hourly trending |
| `1d` | Daily reporting |
| `1w` | Weekly patterns |

---

## 6.4 `timechart` Deep Dive

### Traditional SPL
```spl
index=windows EventCode=4625
| timechart span=1h count BY user

# With options
| timechart span=1h limit=10 useother=f count BY user
| timechart span=1h avg(bytes) BY host
| timechart span=5m dc(src_ip) AS uniq_ips
```

### SPL2 Pipeline
```spl2
search index=windows EventCode=4625
| timechart span=1h count() BY user
| timechart span=1h limit=10 useother=false count() BY user
```

### Key Options
| Option | Description |
|--------|-------------|
| `span` | Bucket size (required or auto) |
| `limit` | Top N series (default 10) |
| `useother` | Aggregate rest into `OTHER` (default true) |
| `usenull` | Include null series (default true) |

### `timechart` vs `stats` + `span`

| Aspect | `timechart` | `stats` + `span` |
|--------|-------------|------------------|
| Output | Time-series (guaranteed `_time` column) | Generic table |
| Visualization | Native chart support | Requires manual chart config |
| Fill gaps | ✅ Fills empty buckets with 0/null | ❌ Only buckets with data |
| Split-by | One field | Multiple fields |

---

## 6.5 `stats` with Time Span (SPL2 Exclusive)

```spl2
# No bin needed!
search index=windows
| stats count() BY _time span=10m

# Multiple aggregations
search index=network
| stats sum(bytes) AS total, dc(src_ip) AS uniq_ips BY _time span=1h, host
```

This is cleaner than `bin` + `stats` and avoids the extra pipeline step.

---

## 6.6 `streamstats`: Running Calculations

**Classification**: 🟢 DIST-STREAM (windowed) | 🟡 CENT-STREAM (global)

### Core Concept
```
stats:        Events → [AGGREGATE] → Summary (fewer rows)
streamstats:  Events → [RUNNING]   → Events + running state (same rows)
```

### Basic Syntax
```spl
# Running count per user
| streamstats count AS seq BY user

# Running sum
| streamstats sum(bytes) AS running_total BY host

# Multiple aggregations
| streamstats count AS events, sum(bytes) AS total, avg(bytes) AS avg BY user
```

### Window Options

| Option | Description | Example |
|--------|-------------|---------|
| `window=N` | Last N events | `window=10` |
| `current=t/f` | Include current event | `current=false` |
| `time_window=X` | Time-based window | `time_window=5m` |
| `global=t/f` | Shared vs per-BY window | `global=false` |
| `reset_before` | Reset before matching event | `reset_before="(action=\"success\")"` |
| `reset_after` | Reset after matching event | `reset_after="(action=\"success\")"` |
| `reset_on_change` | Reset when BY field changes | `reset_on_change=t` |

### Window vs Time Window Decision

| Need | Use |
|------|-----|
| "Last 5 events" | `window=5` |
| "Last 5 minutes" | `time_window=5m` |
| "Last 5 events within 5 minutes" | `window=5 time_window=5m` |

### SPL2 Syntax Differences

```spl2
# SPL: reset_before="(action=\"success\")"
# SPL2: reset before (action="success")

| streamstats count() AS failures
    reset after (action="success")
    window=10
    current=false
    BY user
```

### Classic SPL Options Not in SPL2
- `allnum` — numeric safety
- `global` — merged into `reset` logic
- `time_window` — not yet in SPL2 `streamstats`

---

## 6.7 `eventstats`: Aggregate Without Losing Events

**Classification**: 🟡 CENT-STREAM

### Purpose
Add aggregate columns to **every event** without collapsing.

```spl
# Per-host average bytes
| eventstats avg(bytes) AS host_avg BY host
| where bytes > host_avg          # Events above host average

# Per-user unique IP count
| eventstats dc(src_ip) AS user_uniq_ips BY user
| where user_uniq_ips > 50        # Users with many source IPs
```

### `stats` vs `eventstats` vs `streamstats`

| Command | Output Rows | Use For |
|---------|-------------|---------|
| `stats` | 1 per group | Final summaries |
| `eventstats` | Same as input | Compare event to group aggregate |
| `streamstats` | Same as input | Running/sequential calculations |

---

## 6.8 `autoregress`: Previous Event Values

**Classification**: 🟢 DIST-STREAM

### Purpose
Copy field value from previous event(s) into current event.

```spl
# Previous event's src_ip
| autoregress src_ip AS prev_ip

# 3 events ago
| autoregress bytes p=3 AS bytes_3_ago

# Range: 1 to 3 events ago
| autoregress user p=1-3 AS user_1 user_2 user_3
```

### Common Pattern: Detect Changes
```spl
| sort 0 _time
| autoregress src_ip AS prev_ip
| eval ip_changed=if(src_ip!=prev_ip, 1, 0)
| where ip_changed=1
```

---

## 6.9 `delta`: Change Between Events

**Classification**: 🟢 DIST-STREAM

### Purpose
Numeric difference between current and previous event.

```spl
# Time difference between events
| sort 0 _time
| delta _time AS time_gap

# Bytes change
| delta bytes AS byte_delta

# Compare with event 2 back
| delta _time p=2 AS gap_2_events
```

### Sign Convention
```
delta = current_value - previous_value (in search order)
```
- Ascending `_time`: positive = increasing
- Descending `_time`: positive = decreasing

---

## 6.10 `accum`: Simple Running Sum

**Classification**: 🟢 DIST-STREAM

```spl
| accum bytes AS total_bytes
| accum count AS event_number
```
Simpler than `streamstats sum()` for single-field running totals.

---

## 6.11 Time Windows: `time_window` vs `window`

### Event Count Window (`window`)
```spl
| streamstats window=10 count AS events_last_10
```
**Use when**: "N events" is the rule (e.g., "5 failed logons")

### Time Window (`time_window`) — Classic SPL Only
```spl
| streamstats time_window=5m count AS events_last_5min
```
**Use when**: "N minutes" is the rule (e.g., "5 failures in 5 minutes")

### Combined
```spl
| streamstats window=100 time_window=5m count AS events
# At most 100 events, within 5 minutes
```

### Ordering Requirement for `time_window`
```spl
# MUST sort by _time first!
| sort 0 _time
| streamstats time_window=5m count AS recent_failures
```

---

## 6.12 Reset Logic: `reset_before`, `reset_after`, `reset_on_change`

### `reset_before` — Reset BEFORE Current Event
```spl
# Count failures, reset when success occurs
# Success event starts fresh count
| streamstats count(eval(action="failure")) AS failures
    reset_before="(action=\"success\")"
    BY user
```
Result: Success event has `failures=0`.

### `reset_after` — Reset AFTER Current Event
```spl
# Count failures, reset after success
# Success event carries the failure count
| streamstats count(eval(action="failure")) AS failures
    reset_after="(action=\"success\")"
    BY user
```
Result: Success event has `failures=N` (the count that led to success).

### `reset_on_change` — Reset When BY Field Changes
```spl
# Separate counter per shift, reset when shift changes
| streamstats count BY shift reset_on_change=true
```

---

## 6.13 Rolling Statistics After `timechart`

Powerful pattern: aggregate to time buckets, then run rolling stats.

```spl
index=windows EventCode=4625
| timechart span=10m count AS failures
| streamstats window=6 current=f avg(failures) AS baseline
| eval anomaly_score=failures/baseline
| where anomaly_score > 3
```

Flow:
1. `timechart` → 1 row per 10-min bucket
2. `streamstats` → 6-bucket moving average (excludes current)
3. Compare current bucket to baseline

---

## 6.14 Time Zone Handling

### Key Principles
1. `_time` is **always UTC** (UNIX epoch) internally
2. Display timezone = UI/user setting
3. Search `earliest`/`latest` = UTC unless specified

### Explicit Timezone in Search
```spl
# Search in specific timezone
index=windows earliest=-1d@d latest=@d timezone=UTC
index=windows earliest=-1d@d latest=@d timezone=America/New_York
```

### Timezone in `eval`
```spl
| eval local_time=strftime(_time, "%Y-%m-%d %H:%M:%S", "America/New_York")
| eval utc_time=strftime(_time, "%Y-%m-%d %H:%M:%S", "UTC")
```

### Common Mistake
```spl
# WRONG: Adding hours to _time
| eval wrong_time=_time+3600

# RIGHT: Use timezone-aware formatting
| eval correct_display=strftime(_time, "%H:%M", "America/New_York")
```

---

## 6.15 Complete Time Analysis Patterns

### Pattern 1: Spike Detection
```spl
index=auth action=failure
| timechart span=5m count AS failures
| streamstats window=12 current=f avg(failures) AS baseline, stdev(failures) AS sigma
| eval z_score=(failures-baseline)/sigma
| where z_score > 3
```

### Pattern 2: Beaconing (Regular Intervals)
```spl
index=proxy
| sort 0 _time
| streamstats window=2 current=f delta(_time) AS interval BY src_ip,dest_ip
| where interval>290 AND interval<310    # ~5 min beacon
| stats count AS beacon_count BY src_ip,dest_ip,interval
| where beacon_count > 10
```

### Pattern 3: Session Duration
```spl
index=web
| transaction session_id maxpause=30m
| eval duration=duration
| stats avg(duration) AS avg_session, max(duration) AS max_session BY user
```

### Pattern 4: First/Last Activity
```spl
index=auth
| stats earliest(_time) AS first_seen, latest(_time) AS last_seen BY user
| eval tenure=last_seen-first_seen
```

---

## 6.16 What's Next

- [Part 7: Sequence & Correlation](Part7_Sequence_Correlation.md) — Detection patterns using streaming commands
- [Part 8: Edge/Ingest Processor](Part8_Edge_Ingest_Processor.md) — Pipeline time processing
- [Part 12: Reference](Part12_Reference.md) — Time window cheat sheet