# Splunk SPL + SPL2

# Chapter 4 — Search Commands Reference (Classified) & Time-Based Analysis

> **Goal**: Look up any command — see its classification, syntax, and key behaviors instantly. Master time as a first-class dimension with streaming commands.

---

## 4.1 Classification Badges

| Badge | Meaning |
|-------|---------|
| 🟢 **DIST-STREAM** | Distributed streaming (runs on indexers) |
| 🟡 **CENT-STREAM** | Centralized streaming (search head only) |
| 🔴 **REPORTING** | Non-streaming / reporting (events → summary) |
| 🟣 **GENERATING** | Generating command (creates event stream) |
| 🟠 **ORCHESTRATING** | Pipeline control / multi-dataset |
| 🔵 **PIPELINE-ONLY** | Edge/Ingest Processor pipeline only |
| ⚪ **CUSTOM** | Custom command type |

---

## 4.2 Search Commands Reference (Classified)

### 4.2.1 Generating Commands 🟣

#### `search` / Implicit Search
```
🟣 GENERATING | 🟢 DIST-STREAM (generating phase)
```
**Purpose**: Retrieve events from index
```spl
# Traditional SPL (implicit)
index=windows EventCode=4625

# Explicit
search index=windows EventCode=4625 earliest=-1h
```
**Key**: Implicit at start of traditional SPL pipeline. Keyword syntax on `_raw`.

---

#### `from` (SPL2)
```
🟣 GENERATING | 🟢 DIST-STREAM
```
**Purpose**: Retrieve from dataset (index, lookup, datamodel)
```spl2
# Pipeline
search index=windows WHERE EventCode=4625

# Structured
FROM windows WHERE EventCode=4625
FROM main WHERE sourcetype="cisco:asa"
FROM lookup:my_lookup WHERE key="value"
```

---

#### `inputlookup`
```
🟣 GENERATING | 🟢 DIST-STREAM
```
**Purpose**: Load lookup table as events
```spl
| inputlookup my_lookup
| inputlookup my_lookup WHERE status="active"
| inputlookup my_lookup MAXRESULTS=10000
```

---

#### `inputcsv`
```
🟣 GENERATING | 🟡 CENT-STREAM (file on search head)
```
**Purpose**: Load CSV file as events
```spl
| inputcsv mydata.csv
```

---

#### `makeresults`
```
🟣 GENERATING | 🟡 CENT-STREAM
```
**Purpose**: Generate synthetic events for testing
```spl
| makeresults count=5
| makeresults count=1 data="test"
```

---

#### `metadata` / `typeahead` / `rest` / `savedsearch`
```
🟣 GENERATING | Various
```
| Command | Purpose |
|---------|---------|
| `metadata` | Index metadata (hosts, sources, sourcetypes) |
| `typeahead` | Search-time typeahead |
| `rest` | Splunk REST API query |
| `savedsearch` | Execute saved search |

---

### 4.2.2 Filtering Commands 🟢

#### `where` 🟢
```
🟢 DIST-STREAM
```
**Purpose**: Boolean expression filter per event
```spl
# SPL
| where src_ip=dest_ip
| where bytes > 1000000
| where match(CommandLine, "(?i)powershell")
| where user IN ("admin","root","administrator")

# SPL2
| where src_ip=dest_ip
WHERE src_ip=dest_ip
```

---

#### `search` (filtering mode) 🟢
```
🟢 DIST-STREAM
```
**Purpose**: Keyword/field filter in pipeline
```spl
index=windows | search EventCode=4625
index=windows | search "failed logon"
index=windows | search CommandLine="*powershell*"
```

---

#### `regex` (SPL only) 🟢
```
🟢 DIST-STREAM
```
**Purpose**: PCRE filter
```spl
| regex CommandLine="(?i)powershell.*-enc"
| regex _raw="error|fail"
```
**Note**: Not available in SPL2. Use `match()` in `where`/`WHERE`.

---

#### `CASE()` / `TERM()` 🟢
```
🟢 DIST-STREAM (search modifiers)
```
**Purpose**: Case-sensitive / indexed-term matching
```spl
search CASE("ERROR")           # Case-sensitive
search TERM(127.0.0.1)         # Match as single indexed term
```

---

### 4.2.3 Streaming Transform Commands 🟢

#### `eval` 🟢
```
🟢 DIST-STREAM
```
**Purpose**: Calculate/transform fields
```spl
| eval normalized=lower(user)
| eval is_admin=if(user="admin",1,0)
| eval _raw=replace(_raw, /password=\w+/, "password=****")
| eval total=bytes_in+bytes_out
```

---

#### `rex` 🟢
```
🟢 DIST-STREAM
```
**Purpose**: Regex field extraction
```spl
| rex field=_raw "(?<ip>\d+\.\d+\.\d+\.\d+)"
| rex field=CommandLine "(?<tool>powershell|pwsh|cscript)"
| rex mode=sed "s/password=\w+/password=****/g"
```

---

#### `replace` (via `eval`) 🟢
```
🟢 DIST-STREAM
```
```spl
| eval _raw=replace(_raw, /CreditCard=\d+/, "CreditCard=XXXX")
| eval message=replace(message, "error", "ERROR")
```

---

#### `rename` 🟢
```
🟢 DIST-STREAM
```
```spl
| rename user AS Account, host AS Endpoint
| rename "field with spaces" AS field_no_spaces
```

---

#### `fields` 🟢
```
🟢 DIST-STREAM
```
```spl
| fields _time host user EventCode      # Keep (comma in SPL2)
| fields - _raw _indextime              # Remove
```

---

#### `fillnull` / `filldown` 🟢
```
🟢 DIST-STREAM
```
```spl
| fillnull value=0 bytes
| fillnull value="unknown" user host
| filldown user host                    # Fill with previous value
```

---

#### `lookup` 🟢*
```
🟢 DIST-STREAM* (small lookups) | 🟡 CENT-STREAM (large lookups)
```
**Purpose**: Enrich events from lookup table
```spl
| lookup ip_reputation ip AS src_ip OUTPUT reputation, category
| lookup user_assets user OUTPUT department, manager
```
*Large lookups force centralized execution.

---

### 4.2.4 Streaming Stats Commands 🟢

#### `streamstats` 🟢
```
🟢 DIST-STREAM (windowed) | 🟡 CENT-STREAM (window=0 global=t)
```
**Purpose**: Running calculations per event
```spl
# Running count per user
| streamstats count AS seq BY user

# Moving average (5-event window)
| streamstats window=5 current=f avg(bytes) AS moving_avg

# Reset on condition
| streamstats count(eval(action="failure")) AS failures
    reset_after="(action=\"success\")"
    BY user

# SPL2 syntax differences
| streamstats current=false reset before (action="success") count() BY user
```

---

#### `eventstats` 🟡
```
🟡 CENT-STREAM
```
**Purpose**: Aggregate + attach to every event
```spl
# Per-host average bytes
| eventstats avg(bytes) AS host_avg BY host
| where bytes > host_avg          # Events above host average

# Per-user unique IP count
| eventstats dc(src_ip) AS user_uniq_ips BY user
| where user_uniq_ips > 50        # Users with many source IPs
```

---

#### `autoregress` 🟢
```
🟢 DIST-STREAM
```
**Purpose**: Copy previous event's field value
```spl
# Previous event's src_ip
| autoregress src_ip AS prev_ip

# 3 events ago
| autoregress bytes p=3 AS bytes_3_ago

# Range: 1 to 3 events ago
| autoregress user p=1-3 AS user_1 user_2 user_3
```

---

#### `delta` 🟢
```
🟢 DIST-STREAM
```
**Purpose**: Numeric difference from previous event
```spl
# Time difference between events
| sort 0 _time
| delta _time AS time_gap

# Bytes change
| delta bytes AS byte_delta

# Compare with event 2 back
| delta _time p=2 AS gap_2_events
```

---

#### `accum` 🟢
```
🟢 DIST-STREAM
```
**Purpose**: Running sum
```spl
| accum bytes AS total_bytes
| accum count AS event_number
```

---

### 4.2.5 Centralized Streaming Commands 🟡

#### `sort` 🟡
```
🟡 CENT-STREAM
```
```spl
| sort -count
| sort +_time
| sort 0 -_time          # 0 = no limit
| sort -count, +host
```
**SPL2**: `ORDER BY count DESC, host ASC`

---

#### `head` / `tail` 🟡
```
🟡 CENT-STREAM
```
```spl
| head 10
| tail 5
```
**SPL2**: `LIMIT 10`, `OFFSET 20 LIMIT 10`

---

#### `reverse` 🟡
```
🟡 CENT-STREAM
```
```spl
| reverse
```

---

#### `dedup` 🟡
```
🟡 CENT-STREAM
```
```spl
| dedup host
| dedup host, user
| dedup 2 host           # Keep 2 per host
```

---

#### `sample` / `sampleratio` 🟡
```
🟡 CENT-STREAM
```
```spl
| sampleratio 0.1        # 10% random sample
| sample 100             # Exact count
```

---

#### `transaction` 🟡
```
🟡 CENT-STREAM
```
**Purpose**: Group events into transactions
```spl
| transaction user maxspan=1h maxpause=5m startswith=eval(action="login") endswith=eval(action="logout")
```
**Warning**: Memory intensive. Prefer `streamstats` for sequence detection.

---

### 4.2.6 Reporting / Non-Streaming Commands 🔴

#### `stats` 🔴
```
🔴 REPORTING
```
```spl
| stats count BY user, host
| stats sum(bytes) AS total, avg(bytes) AS avg, dc(src_ip) AS uniq_ips BY host
| stats values(CommandLine) AS cmds BY user   # High cardinality risk!
```

---

#### `chart` 🔴
```
🔴 REPORTING
```
```spl
| chart count BY user, host      # Pivot table
| chart sum(bytes) BY host, _time span=1h
```

---

#### `timechart` 🔴
```
🔴 REPORTING
```
```spl
| timechart span=1h count BY user
| timechart span=5m avg(bytes) BY host
| timechart span=1h limit=10 useother=f count BY action
```

---

#### `top` / `rare` 🔴
```
🔴 REPORTING
```
```spl
| top limit=10 user
| rare limit=5 host
```

---

#### `sistats` / `sichart` / `sitimechart` 🔴
```
🔴 REPORTING (Summary Index format)
```
```spl
| sistats count BY user, host
| collect index=summary
```

---

#### `geom` / `geostats` / `cluster` 🔴
```
🔴 REPORTING
```
```spl
| geostats latfield=lat longfield=lon count BY city
| cluster t=0.9
```

---

### 4.2.7 Orchestrating Commands 🟠

#### `join` / `selfjoin` 🟠
```
🟠 ORCHESTRATING | 🟡 CENT-STREAM
```
```spl
| join user [ search index=auth | stats count BY user ]
| selfjoin user
```

---

#### `append` / `appendpipe` / `appendcols` 🟠
```
🟠 ORCHESTRATING | 🟡 CENT-STREAM
```
```spl
| append [ search index=windows EventCode=4624 ]
| appendpipe [ stats count ]
| appendcols [ search index=perf | stats avg(cpu) ]
```

---

#### `map` 🟠
```
🟠 ORCHESTRATING | 🟡 CENT-STREAM
```
```spl
| map search="search index=windows user=$user$ | head 1"
```

---

#### `foreach` 🟠
```
🟠 ORCHESTRATING | SPL2
```
```spl2
| foreach * [ eval <<FIELD>>=upper(<<FIELD>>) ]
```

---

### 4.2.8 Dataset Processing Commands (SPL2)

#### `union` / `diff` / `intersect`
```
🟠 ORCHESTRATING | SPL2
```
```spl2
| union [ FROM windows ], [ FROM linux ]
| diff [ FROM windows ], [ FROM linux ]
| intersect [ FROM windows ], [ FROM linux ]
```

---

### 4.2.9 Structure Commands 🟢

#### `mvexpand` 🟢
```
🟢 DIST-STREAM
```
```spl
| mvexpand user
| mvexpand mvfield limit=100
```

---

#### `expand` 🟢
```
🟢 DIST-STREAM
```
```spl
| expand array_field
```

---

#### `flatten` 🟢
```
🟢 DIST-STREAM
```
```spl
| flatten object_field
```

---

#### `mvcombine` / `mvfilter` 🟢
```
🟢 DIST-STREAM
```
```spl
| mvcombine delim="," user
| mvfilter match(user, "admin.*")
```

---

### 4.2.10 Lookup & Enrichment Commands

#### `lookup` (see 4.2.3) 🟢/🟡

#### `outputlookup` 🟡
```
🟡 CENT-STREAM
```
```spl
| outputlookup my_lookup
| outputlookup my_lookup append=true
```

---

#### `lookup_editor` / `inputlookup` (see 4.2.1)

---

### 4.2.11 Geospatial Commands 🔴

#### `iplocation` 🟢
```
🟢 DIST-STREAM
```
```spl
| iplocation src_ip
| iplocation dest_ip prefix=dest_
```

---

#### `geom` / `geostats` 🔴
```
🔴 REPORTING
```
```spl
| geom geo_countries featureIdField=country
| geostats latfield=lat longfield=lon count BY city
```

---

### 4.2.12 Machine Learning Commands 🔴

#### `fit` / `apply` 🔴
```
🔴 REPORTING
```
```spl
| fit LogisticRegression target from feature1 feature2 INTO model_v1
| apply model_v1
```

---

#### `kmeans` / `anomalydetection` 🔴
```
🔴 REPORTING
```
```spl
| kmeans k=3 feature1 feature2
| anomalydetection _time, bytes
```

---

### 4.2.13 Pipeline-Only Commands (Edge/Ingest Processor) 🔵

| Command | Purpose | Syntax |
|---------|---------|--------|
| `route` | Divert subset to another path | `route predicate [ \| into $dest ]` |
| `thru` | Copy + continue main path | `thru [ \| into $dest ]` |
| `branch` | Split into independent paths | `branch [path1], [path2], ...` |
| `if` | Conditional processing | `if (cond) [ ... ] elseif [ ... ] else [ ... ]` |

```spl2
$pipeline = | from $source
    | where profile != "debug"
    | route profile="linux" [ | into $linux_dest ]
    | thru [ | into $archive_dest ]
    | branch
        [ | into $prod_dest ]
        [ | into $dev_dest ]
    ;
```

---

### 4.2.14 Custom Command Types ⚪

| Type | Generating | Streaming | Reporting |
|------|------------|-----------|-----------|
| **Input** | None | Events | Events |
| **Output** | Events | Events | Summary |
| **Runs on** | Search head | Indexers (if distributed) | Search head |
| **Memory** | O(1) | O(1) per event | O(groups) |
| **Use case** | Data connectors | Enrichment, parsing, filter | Custom agg, ML |

**Configuration** (`commands.conf`):
```ini
[my_streaming_cmd]
type = streaming
distributed = true
local = false
streaming = true
supports_multivalues = true

[my_reporting_cmd]
type = reporting
distributed = false
local = true
```

---

## 4.3 Quick Lookup Table

| Command | Class | SPL | SPL2 Pipeline | SPL2 Structured | Pipeline (EP/IP) |
|---------|-------|-----|---------------|-----------------|------------------|
| `search` | 🟣 | ✅ | ✅ | ❌ | ✅ |
| `from` | 🟣 | ❌ | ✅ | ✅ | ✅ ($source) |
| `where` | 🟢 | ✅ | ✅ | ✅ (WHERE) | ✅ |
| `eval` | 🟢 | ✅ | ✅ | ✅ (EVAL) | ✅ |
| `rex` | 🟢 | ✅ | ✅ | ❌ | ✅ |
| `fields` | 🟢 | ✅ | ✅ (comma) | ✅ (SELECT) | ✅ |
| `rename` | 🟢 | ✅ | ✅ | ✅ (AS in SELECT) | ✅ |
| `lookup` | 🟢/🟡 | ✅ | ✅ | ❌ | ✅ |
| `streamstats` | 🟢/🟡 | ✅ | ✅ | ❌ | ✅ |
| `eventstats` | 🟡 | ✅ | ✅ | ❌ | ✅ |
| `autoregress` | 🟢 | ✅ | ✅ | ❌ | ✅ |
| `delta` | 🟢 | ✅ | ✅ | ❌ | ✅ |
| `sort` | 🟡 | ✅ | ✅ | ✅ (ORDER BY) | ❌ |
| `head` | 🟡 | ✅ | ✅ | ✅ (LIMIT) | ❌ |
| `dedup` | 🟡 | ✅ | ✅ | ❌ | ✅ |
| `stats` | 🔴 | ✅ | ✅ | ✅ (GROUP BY/SELECT) | ✅ |
| `chart` | 🔴 | ✅ | ✅ | ❌ | ❌ |
| `timechart` | 🔴 | ✅ | ✅ | ❌ | ❌ |
| `join` | 🟠 | ✅ | ✅ | ✅ (JOIN clause) | ❌ |
| `append` | 🟠 | ✅ | ✅ | ❌ | ❌ |
| `route` | 🔵 | ❌ | ❌ | ❌ | ✅ |
| `thru` | 🔵 | ❌ | ❌ | ❌ | ✅ |
| `branch` | 🔵 | ❌ | ❌ | ❌ | ✅ |
| `if` | 🔵 | ❌ | ✅ | ❌ | ✅ |

---

## 4.4 Time-Based Analysis & Streaming Commands

> **Goal**: Master time as a first-class dimension. Understand `streamstats`, `eventstats`, `autoregress`, `delta` for sequence detection.

---

### 4.5 Time as a First-Class Dimension

Time isn't just a filter — it's an analytical dimension.

```
Three time operations:
━━━━━━━━━━━━━━━━━━━━━━━
1. LIMIT time range        → earliest/latest, WHERE earliest=...
2. BUCKET time             → bin/span, timechart
3. COMPUTE over time       → streamstats, eventstats, autoregress, delta
```

---

### 4.6 Search Time Range: `earliest`/`latest` vs `WHERE`

#### Traditional SPL
```spl
index=windows earliest=-1h latest=now
index=windows earliest=-24h@h latest=@h
```

#### SPL2 Pipeline
```spl2
search index=windows earliest=-1h latest=now
```

#### SPL2 Structured
```spl2
FROM windows WHERE earliest=-1h AND latest=now
```

#### Relative Time Syntax
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

### 4.7 Time Bucketing: `bin`/`bucket` vs `span`

#### Traditional SPL
```spl
index=windows
| bin _time span=10m
| stats count BY _time

# bucket is alias
| bucket _time span=1h
```

#### SPL2 Pipeline
```spl2
search index=windows
| bin _time span=10m
| stats count() BY _time

# SPL2 exclusive: span in stats
search index=windows
| stats count() BY _time span=10m
```

#### SPL2 Structured
```spl2
FROM windows
GROUP BY span(_time, 10m)
SELECT _time, count() AS count
```

#### Span Options
| Span | Use Case |
|------|----------|
| `1m` / `5m` / `10m` | High-resolution alerting |
| `1h` | Hourly trending |
| `1d` | Daily reporting |
| `1w` | Weekly patterns |

---

### 4.8 `timechart` Deep Dive

#### Traditional SPL
```spl
index=windows EventCode=4625
| timechart span=1h count BY user

# With options
| timechart span=1h limit=10 useother=f count BY user
| timechart span=1h avg(bytes) BY host
| timechart span=5m dc(src_ip) AS uniq_ips
```

#### SPL2 Pipeline
```spl2
search index=windows EventCode=4625
| timechart span=1h count() BY user
| timechart span=1h limit=10 useother=false count() BY user
```

#### Key Options
| Option | Description |
|--------|-------------|
| `span` | Bucket size (required or auto) |
| `limit` | Top N series (default 10) |
| `useother` | Aggregate rest into `OTHER` (default true) |
| `usenull` | Include null series (default true) |

#### `timechart` vs `stats` + `span`

| Aspect | `timechart` | `stats` + `span` |
|--------|-------------|------------------|
| Output | Time-series (guaranteed `_time` column) | Generic table |
| Visualization | Native chart support | Requires manual chart config |
| Fill gaps | ✅ Fills empty buckets with 0/null | ❌ Only buckets with data |
| Split-by | One field | Multiple fields |

---

### 4.9 `stats` with Time Span (SPL2 Exclusive)

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

### 4.10 `streamstats`: Running Calculations

**Classification**: 🟢 DIST-STREAM (windowed) | 🟡 CENT-STREAM (global)

#### Core Concept
```
stats:        Events → [AGGREGATE] → Summary (fewer rows)
streamstats:  Events → [RUNNING]   → Events + running state (same rows)
```

#### Basic Syntax
```spl
# Running count per user
| streamstats count AS seq BY user

# Running sum
| streamstats sum(bytes) AS running_total BY host

# Multiple aggregations
| streamstats count AS events, sum(bytes) AS total, avg(bytes) AS avg BY user
```

#### Window Options

| Option | Description | Example |
|--------|-------------|---------|
| `window=N` | Last N events | `window=10` |
| `current=t/f` | Include current event | `current=false` |
| `time_window=X` | Time-based window | `time_window=5m` |
| `global=t/f` | Shared vs per-BY window | `global=false` |
| `reset_before` | Reset before matching event | `reset_before="(action=\"success\")"` |
| `reset_after` | Reset after matching event | `reset_after="(action=\"success\")"` |
| `reset_on_change` | Reset when BY field changes | `reset_on_change=t` |

#### Window vs Time Window Decision

| Need | Use |
|------|-----|
| "Last 5 events" | `window=5` |
| "Last 5 minutes" | `time_window=5m` |
| "Last 5 events within 5 minutes" | `window=5 time_window=5m` |

#### SPL2 Syntax Differences

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

### 4.11 `eventstats`: Aggregate Without Losing Events

**Classification**: 🟡 CENT-STREAM

#### Purpose
Add aggregate columns to **every event** without collapsing.

```spl
# Per-host average bytes
| eventstats avg(bytes) AS host_avg BY host
| where bytes > host_avg          # Events above host average

# Per-user unique IP count
| eventstats dc(src_ip) AS user_uniq_ips BY user
| where user_uniq_ips > 50        # Users with many source IPs
```

#### `stats` vs `eventstats` vs `streamstats`

| Command | Output Rows | Use For |
|---------|-------------|---------|
| `stats` | 1 per group | Final summaries |
| `eventstats` | Same as input | Compare event to group aggregate |
| `streamstats` | Same as input | Running/sequential calculations |

---

### 4.12 `autoregress`: Previous Event Values

**Classification**: 🟢 DIST-STREAM

#### Purpose
Copy field value from previous event(s) into current event.

```spl
# Previous event's src_ip
| autoregress src_ip AS prev_ip

# 3 events ago
| autoregress bytes p=3 AS bytes_3_ago

# Range: 1 to 3 events ago
| autoregress user p=1-3 AS user_1 user_2 user_3
```

#### Common Pattern: Detect Changes
```spl
| sort 0 _time
| autoregress src_ip AS prev_ip
| eval ip_changed=if(src_ip!=prev_ip, 1, 0)
| where ip_changed=1
```

---

### 4.13 `delta`: Change Between Events

**Classification**: 🟢 DIST-STREAM

#### Purpose
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

#### Sign Convention
```
delta = current_value - previous_value (in search order)
```
- Ascending `_time`: positive = increasing
- Descending `_time`: positive = decreasing

---

### 4.14 `accum`: Simple Running Sum

**Classification**: 🟢 DIST-STREAM

```spl
| accum bytes AS total_bytes
| accum count AS event_number
```
Simpler than `streamstats sum()` for single-field running totals.

---

### 4.15 Time Windows: `time_window` vs `window`

#### Event Count Window (`window`)
```spl
| streamstats window=10 count AS events_last_10
```
**Use when**: "N events" is the rule (e.g., "5 failed logons")

#### Time Window (`time_window`) — Classic SPL Only
```spl
| streamstats time_window=5m count AS events_last_5min
```
**Use when**: "N minutes" is the rule (e.g., "5 failures in 5 minutes")

#### Combined
```spl
| streamstats window=100 time_window=5m count AS events
# At most 100 events, within 5 minutes
```

#### Ordering Requirement for `time_window`
```spl
# MUST sort by _time first!
| sort 0 _time
| streamstats time_window=5m count AS recent_failures
```

---

### 4.16 Reset Logic: `reset_before`, `reset_after`, `reset_on_change`

#### `reset_before` — Reset BEFORE Current Event
```spl
# Count failures, reset when success occurs
# Success event starts fresh count
| streamstats count(eval(action="failure")) AS failures
    reset_before="(action=\"success\")"
    BY user
```
Result: Success event has `failures=0`.

#### `reset_after` — Reset AFTER Current Event
```spl
# Count failures, reset after success
# Success event carries the failure count
| streamstats count(eval(action="failure")) AS failures
    reset_after="(action=\"success\")"
    BY user
```
Result: Success event has `failures=N` (the count that led to success).

#### `reset_on_change` — Reset When BY Field Changes
```spl
# Separate counter per shift, reset when shift changes
| streamstats count BY shift reset_on_change=true
```

---

### 4.17 Rolling Statistics After `timechart`

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

### 4.18 Time Zone Handling

#### Key Principles
1. `_time` is **always UTC** (UNIX epoch) internally
2. Display timezone = UI/user setting
3. Search `earliest`/`latest` = UTC unless specified

#### Explicit Timezone in Search
```spl
# Search in specific timezone
index=windows earliest=-1d@d latest=@d timezone=UTC
index=windows earliest=-1d@d latest=@d timezone=America/New_York
```

#### Timezone in `eval`
```spl
| eval local_time=strftime(_time, "%Y-%m-%d %H:%M:%S", "America/New_York")
| eval utc_time=strftime(_time, "%Y-%m-%d %H:%M:%S", "UTC")
```

#### Common Mistake
```spl
# WRONG: Adding hours to _time
| eval wrong_time=_time+3600

# RIGHT: Use timezone-aware formatting
| eval correct_display=strftime(_time, "%H:%M", "America/New_York")
```

---

### 4.19 Complete Time Analysis Patterns

#### Pattern 1: Spike Detection
```spl
index=auth action=failure
| timechart span=5m count AS failures
| streamstats window=12 current=f avg(failures) AS baseline, stdev(failures) AS sigma
| eval z_score=(failures-baseline)/sigma
| where z_score > 3
```

#### Pattern 2: Beaconing (Regular Intervals)
```spl
index=proxy
| sort 0 _time
| streamstats current=f delta(_time) AS interval BY src_ip,dest_ip
| eval interval_rounded=round(interval/10)*10
| stats count AS connections, values(interval_rounded) AS intervals BY src_ip,dest_ip
| where mvcount(intervals)=1 AND connections > 20
```

#### Pattern 3: Session Duration
```spl
index=web
| transaction session_id maxpause=30m
| eval duration=duration
| stats avg(duration) AS avg_session, max(duration) AS max_session BY user
```

#### Pattern 4: First/Last Activity
```spl
index=auth
| stats earliest(_time) AS first_seen, latest(_time) AS last_seen BY user
| eval tenure=last_seen-first_seen
```

---

### 4.20 What's Next

- **Chapter 5**: Sequence & Correlation Analysis — detection patterns using streaming commands
- **Chapter 6**: Edge/Ingest Processor Pipeline Design + Custom Commands
- **Chapter 7**: Performance, SOC Patterns & Reference