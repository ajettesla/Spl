# Part 8 — SPL2 Edge/Ingest Processor Pipeline Design

> **Goal**: Build ingestion pipelines that filter, mask, route, and enrich data BEFORE it hits the index.

---

## 8.1 Two-Layer Configuration: Partition + Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    EDGE/INGEST PROCESSOR                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER 1: PARTITION (Builder UI)                               │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                                │
│  "Which subset of incoming data enters THIS pipeline?"         │
│                                                                 │
│  Example:                                                        │
│    sourcetype = cisco:asa                                       │
│    host = firewall-*                                            │
│                                                                 │
│  Configured in UI, NOT in SPL2.                                │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER 2: PIPELINE (SPL2)                                      │
│  ━━━━━━━━━━━━━━━━━━━━━━                                        │
│  "What happens to each event inside this pipeline?"            │
│                                                                 │
│  import route from /splunk/ingest/commands                      │
│                                                                 │
│  $pipeline = | from $source                                     │
│      | where profile != "debug"                                 │
│      | eval _raw=replace(_raw, /pwd=\w+/, "pwd=****")           │
│      | route profile="linux" [ | into $linux_dest ]             │
│      | into $default_dest;                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Critical**: Partition ≠ `where`. Partition defines pipeline scope. `where` filters inside pipeline.

---

## 8.2 Pipeline Syntax

```spl2
import route from /splunk/ingest/commands

$pipeline = | from $source
    | <processing commands>
    | into $destination;
```

| Component | Purpose |
|-----------|---------|
| `import` | Load commands (route, thru, branch, etc.) |
| `$pipeline` | Pipeline variable (required) |
| `from $source` | Input: events matching partition |
| Processing | Filter, transform, enrich, route |
| `into $destination` | Output: where events go |

---

## 8.3 Event Flow Model: 8 Stages

```
SOURCE (Partition)
    │
    ▼
EARLY FILTER (where on _raw)
    │  └─ Drop noise BEFORE parsing (save CPU)
    ▼
EXTRACTION (rex, spath)
    │  └─ Parse fields needed for routing/masking
    ▼
REDUCTION (dedup, stats)
    │  └─ Reduce volume if needed
    ▼
TRANSFORMATION (eval, replace)
    │  └─ Mask, normalize, enrich
    ▼
ENRICHMENT (lookup)
    │  └─ Add context (asset, geo, threat intel)
    ▼
ROUTING/COPYING (route, thru, branch)
    │  └─ Send to correct destination(s)
    ▼
DESTINATION (into)
```

**Not every pipeline needs every stage.**

---

## 8.4 Command Classification for Pipelines

| Category | Commands | Pipeline Use |
|----------|----------|--------------|
| **Input/Output** | `from`, `into` | Required |
| **Filter/Reduce** | `where`, `dedup`, `stats` | Drop/reduce events |
| **Parse/Extract** | `rex`, `spath` | Create fields |
| **Transform** | `eval`, `replace`, `rename`, `fields`, `if` | Modify events |
| **Enrich** | `lookup` | Add context |
| **Route/Copy** | `route`, `thru`, `branch` | Control flow |
| **Structure** | `expand`, `flatten`, `mvexpand` | Handle structured data |
| **Specialized** | `decrypt`, `ocsf` | Crypto, normalization |

---

## 8.5 `where`: Filtering in Pipelines

```spl2
$pipeline = | from $source
    # Early filter on raw (cheap)
    | where NOT match(_raw, /"action"\s*:\s*"debug"/i)
    | where NOT match(_raw, /"profile"\s*:\s*"test"/i)
    
    # Later filter on extracted fields
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    | where profile IN ("linux","dns","firewall")
    | into $dest;
```

### Pipeline `where` vs Search `where`

| Aspect | Search `where` | Pipeline `where` |
|--------|----------------|------------------|
| Dropped events | Hidden from results | **Never indexed** (save license) |
| Performance | Search-time | Ingestion-time |
| `_raw` access | Yes | Yes (before extraction) |

---

## 8.6 `rex`/`spath`: Extraction

```spl2
# Regex extraction (unstructured)
| rex field=_raw /"profile":"(?P<profile>[^"]+)"/
| rex field=_raw /"user":"(?P<user>[^"]+)"/

# Structured extraction (JSON/XML)
| spath input=_raw path=user.name
| spath input=_raw output=geo country_code
```

**Rule**: Use `spath` for JSON/XML. Use `rex` for unstructured or partial parsing.

---

## 8.7 `eval`/`replace`: Transformation & Masking

### Masking Patterns
```spl2
# Credit card
| eval _raw=replace(_raw, /CreditCard=\d+/, "CreditCard=XXXXXXXX")

# JSON password
| eval _raw=replace(_raw, /"password":"[^"]*"/, "\"password\":\"xxxx\"")

# Multiple patterns
| eval _raw=replace(_raw, /password=\w+/, "password=****")
| eval _raw=replace(_raw, /api_key=\w+/, "api_key=****")
| eval _raw=replace(_raw, /token=\w+/, "token=****")
```

### Field Creation
```spl2
| eval severity=case(
    match(_raw, /ERROR/), "critical",
    match(_raw, /WARN/), "high",
    1=1, "info"
)
| eval index=lower(profile)
```

---

## 8.8 `route`: Divert Subset to Another Destination

```spl2
import route from /splunk/ingest/commands

$pipeline = | from $source
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    
    # Linux → dedicated index
    | route profile="linux" [
        | eval index="linux"
        | into $linux_destination
    ]
    
    # DNS → dedicated index
    | route profile="dns" [
        | eval index="dns"
        | into $dns_destination
    ]
    
    # Remaining → default
    | into $default_destination;
```

### Flow Diagram
```
ALL EVENTS
    │
    ├─ route profile="linux" ───→ Linux events ───→ $linux_destination
    │
    ├─ route profile="dns" ─────→ DNS events ─────→ $dns_destination
    │
    └─ (remaining) ─────────────→ Other events ───→ $default_destination
```

---

## 8.9 `thru`: Copy + Continue Main Path

```spl2
$pipeline = | from $source
    | eval _raw=replace(_raw, /password=\w+/, "password=****")
    
    # Archive masked copy
    | thru [
        | into $archive_destination
    ]
    
    # Continue routing on main path
    | route profile="linux" [ | into $linux_dest ]
    | into $default_destination;
```

### Flow Diagram
```
EVENT
    │
    ▼
MASK
    │
    ├─ thru ───→ COPY ───→ $archive_destination
    │
    └─ (original) ──→ route ───→ $linux_dest / $default_destination
```

---

## 8.10 `branch`: Multiple Independent Paths

```spl2
$pipeline = | from $source
    | eval _raw=replace(_raw, /password=\w+/, "password=****")
    
    | branch
        # Path 1: Full routing logic
        [
            | route profile="linux" [ | into $linux_dest ]
            | route profile="dns" [ | into $dns_dest ]
            | into $default_dest
        ],
        # Path 2: Raw archive (before masking? after? your choice)
        [
            | into $raw_archive_dest
        ];
```

### Flow Diagram
```
MASKED EVENT
    │
    ▼
BRANCH
   /     \
  ▼       ▼
PATH 1   PATH 2
  │         │
  ▼         ▼
route...  archive
```

---

## 8.11 `route` vs `thru` vs `branch` vs `if`: Decision Guide

| Need | Command | Example |
|------|---------|---------|
| "Send subset elsewhere, rest continues" | `route` | Linux logs → security index |
| "Keep copy for archive, continue processing" | `thru` | Audit trail + normal routing |
| "Same event → multiple independent pipelines" | `branch` | Prod + Dev + Archive simultaneously |
| "Transform differently per condition, same dest" | `if` | Set index per profile, single destination |

### `if` Example (Conditional Transform)
```spl2
$pipeline = | from $source
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    | if (profile="linux") [
        | eval index="linux", sourcetype="linux_syslog"
    ]
    elseif (profile="dns") [
        | eval index="dns", sourcetype="dns_logs"
    ]
    else [
        | eval index="default", sourcetype="generic"
    ]
    | into $destination;
```

---

## 8.12 `dedup`: Deduplication

```spl2
# Dedupe on event ID (good)
| dedup event_id

# Dedupe on host (probably bad - loses events)
| dedup host

# Dedupe on composite key
| dedup host, user, EventCode
```

**Pipeline dedupe**: Removes events from pipeline → saves downstream processing.

---

## 8.13 `stats`: Aggregation in Pipelines

```spl2
# Reduce volume: per-host byte summary every 5 min
| stats sum(bytes) AS total_bytes BY host span=5m
| into $metrics_destination;
```

**Use case**: Send metrics instead of raw events to reduce license.

---

## 8.14 `lookup`: Enrichment

```spl2
$pipeline = | from $source
    | rex field=_raw /"src_ip":"(?P<src_ip>[^"]+)"/
    | lookup ip_assets ip AS src_ip OUTPUT asset_type, owner, criticality
    | route criticality="high" [ | into $high_priority_dest ]
    | into $default_dest;
```

**Requirement**: Lookup must be configured in Edge Processor.

---

## 8.15 `decrypt` / `ocsf`

```spl2
# Decrypt encrypted field
| decrypt field=encrypted_payload algorithm=AES256 key=$key$

# Normalize to OCSF
| ocsf
```

---

## 8.16 `expand` / `flatten` / `mvexpand`

```spl2
# JSON array → multiple events
| expand events_array

# Nested object → flat fields
| flatten user_object

# Multivalue field → multiple events
| mvexpand tags
```

---

## 8.17 Complete Pipeline Design Patterns

### Pattern 1: Filter → Mask → Route
```spl2
import route from /splunk/ingest/commands

$pipeline = | from $source
    # 1. Drop noise early
    | where NOT match(_raw, /"action":"debug"/i)
    
    # 2. Extract routing field
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    
    # 3. Global masking
    | eval _raw=replace(_raw, /"password":"[^"]*"/, "\"password\":\"xxxx\"")
    | eval _raw=replace(_raw, /CreditCard=\d+/, "CreditCard=XXXX")
    
    # 4. Route by profile
    | route profile="linux" [ | eval index="linux" | into $linux_dest ]
    | route profile="dns" [ | eval index="dns" | into $dns_dest ]
    | route profile="firewall" [ | eval index="firewall" | into $fw_dest ]
    
    # 5. Default
    | into $default_dest;
```

### Pattern 2: Archive Raw → Process → Route
```spl2
import route from /splunk/ingest/commands

$pipeline = | from $source
    # Archive RAW (before masking)
    | thru [ | into $raw_archive_dest ]
    
    # Process
    | eval _raw=replace(_raw, /password=\w+/, "password=****")
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    
    # Route
    | route profile="linux" [ | into $linux_dest ]
    | into $default_dest;
```

### Pattern 3: Branch for Multiple Teams
```spl2
import route from /splunk/ingest/commands

$pipeline = | from $source
    | eval _raw=replace(_raw, /password=\w+/, "password=****")
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    
    | branch
        # Security team: full routing
        [
            | route profile="linux" [ | into $sec_linux ]
            | route profile="firewall" [ | into $sec_fw ]
            | into $sec_default
        ],
        # Infra team: just Linux/DNS
        [
            | route profile="linux" [ | into $infra_linux ]
            | route profile="dns" [ | into $infra_dns ]
            | into $infra_default
        ],
        # Compliance: raw archive
        [
            | into $compliance_archive
        ];
```

---

## 8.18 Partition vs `where`: Critical Distinction

```
┌────────────────────────────────────────────────────────────────┐
│ PARTITION (Builder UI)                                         │
│ ━━━━━━━━━━━━━━━━━━━━                                           │
│ Question: "What data enters THIS pipeline?"                    │
│                                                                 │
│ Example: sourcetype=cisco:asa                                   │
│                                                                 │
│ Effect: Data NOT matching partition NEVER reaches pipeline.    │
│         Other pipelines (or default) handle it.                │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│ WHERE (Pipeline SPL2)                                          │
│ ━━━━━━━━━━━━━━━━                                               │
│ Question: "Which events INSIDE THIS PIPELINE continue?"        │
│                                                                 │
│ Example: | where action != "debug"                             │
│                                                                 │
│ Effect: Events dropped here are GONE (not indexed, not routed) │
└────────────────────────────────────────────────────────────────┘
```

**Rule**: Put broad filters in Partition. Put fine-grained filters in `where`.

---

## 8.19 Pipeline Debugging & Testing

### Sample Data in Builder
1. Paste sample events in "Sample Data" tab
2. Run pipeline
3. Inspect output at each stage (click command)

### Common Debug Patterns
```spl2
# Debug: see what fields exist
| fields *

# Debug: count events at this stage
| stats count

# Debug: sample first 10
| head 10
```

### Testing Partition
- If events not appearing: check partition config
- If wrong events appearing: partition too broad

---

## 8.20 What's Next

- [Part 9: Custom Commands](Part9_Custom_Commands.md) — Extend SPL with Python
- [Part 10: Performance](Part10_Performance.md) — `tstats`, `tsidx`, optimization
- [Part 11: SOC Patterns](Part11_SOC_Patterns.md) — Detection engineering