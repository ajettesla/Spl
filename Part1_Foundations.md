# Part 1 — Foundations: Events, Fields & Data Model

> **Goal**: Understand what Splunk actually stores and searches. Everything else builds on this.

---

## 1.1 What Is an Event?

An **event** is a single record representing something that happened at a point in time.

```
┌─────────────────────────────────────────────────────────────┐
│  One Splunk Event                                           │
├─────────────────────────────────────────────────────────────┤
│  _time      = 2026-09-23 10:15:22                          │
│  _raw       = Sep 23 10:15:22 host=DC01 user=admin         │
│              EventCode=4625 LogonType=3                    │
│  host       = DC01                                          │
│  source     = WinEventLog:Security                         │
│  sourcetype = WinEventLog:Security                         │
│  index      = windows                                       │
│  user       = administrator                                 │
│  EventCode  = 4625                                          │
│  LogonType  = 3                                             │
└─────────────────────────────────────────────────────────────┘
```

**Key insight**: An event is not a "row" in a database. It's a semi-structured blob (`_raw`) with metadata fields attached.

---

## 1.2 Where Do Events Come From?

```
Data Source          →  Forwarder  →  Indexer  →  Index
━━━━━━━━━━━━━━━━━━━━━   ━━━━━━━━━━   ━━━━━━━━━   ━━━━━━
Windows Event Logs      UF/HF          │       windows
Linux /var/log          UF/HF          │       linux
Firewall syslog         HF             │       firewall
Cloud APIs              HEC            │       cloud
Custom scripts          Scripted Input │       custom
```

**Critical path**: Data → Parsing (index-time) → Indexing → Search (search-time)

---

## 1.3 What Is an Index?

An **index** is a physical storage bucket for events. Think of it as a database partition.

```spl
# Traditional SPL
index=windows

# SPL2 search syntax
search index=windows

# SPL2 FROM syntax
FROM windows
```

Indexes control:
- Data retention (frozen, cold, warm, hot buckets)
- Access permissions (roles → indexes)
- Performance (smaller indexes = faster searches)

---

## 1.4 SPL vs SPL2: Same Data, Different Syntax

| Task | Traditional SPL | SPL2 (search) | SPL2 (FROM) |
|------|-----------------|---------------|-------------|
| Select index | `index=windows` | `search index=windows` | `FROM windows` |
| Filter | `EventCode=4625` | `search EventCode=4625` | `WHERE EventCode=4625` |
| Pipe | `\| stats count` | `\| stats count()` | `\| stats count()` |

**The data doesn't change.** Only the language does.

---

## 1.5 Internal Fields

Fields prefixed with `_` are Splunk-internal. They exist on every event.

| Field | Meaning | Use Case |
|-------|---------|----------|
| `_raw` | Original unparsed event text | Regex extraction, masking, debugging |
| `_time` | Event timestamp (UNIX epoch) | Time-range filtering, `timechart`, `bin` |
| `_indextime` | When Splunk indexed the event | Ingestion latency: `eval lag=_indextime-_time` |
| `_cd` | Clustered index: bucket ID + offset | `tstats`, `walklex`, troubleshooting |
| `_bkt` | Bucket ID | `tstats`, bucket-level analysis |

> **`_raw` is the source of truth.** All search-time extractions derive from it.

---

## 1.6 Default Fields

Splunk automatically adds these to every event at index time.

| Field | Description | Example |
|-------|-------------|---------|
| `index` | Which index stores this event | `windows`, `firewall` |
| `host` | Originating host | `DC01`, `web-01` |
| `source` | Input source (file, port, script) | `/var/log/secure`, `WinEventLog:Security` |
| `sourcetype` | Data format identifier | `linux_secure`, `cisco:asa` |

**Search usage:**
```spl
# SPL
index=windows host=DC01 sourcetype=WinEventLog:Security

# SPL2 FROM
FROM windows
WHERE host="DC01" AND sourcetype="WinEventLog:Security"
```

---

## 1.7 Extracted Fields: Search-Time vs Index-Time

### Index-Time Extraction (rare, expensive)
- Configured in `props.conf` / `transforms.conf` on Heavy Forwarder / Indexer
- Fields written to `tsidx` (inverted index)
- **Use case**: High-cardinality fields needed for `tstats`, sub-second filtering
- **Cost**: Larger index, slower ingestion, no schema flexibility

### Search-Time Extraction (default, recommended)
- Configured in `props.conf` on Search Head / Heavy Forwarder
- Fields extracted at search time from `_raw`
- **Use case**: Almost everything else
- **Benefit**: Flexible, no index bloat, schema-on-read

```
Index-Time:    Raw Event → [INDEXER: props/transforms] → tsidx + _raw
                                                          ↓
Search-Time:                                                    [SEARCH HEAD: props] → fields
```

---

## 1.8 Indexed Fields vs Search-Time Fields

| Aspect | Indexed Field | Search-Time Field |
|--------|---------------|-------------------|
| Storage | In `tsidx` (inverted index) | Not in `tsidx` |
| `tstats` support | ✅ Yes | ❌ No |
| `walklex` visible | ✅ Yes | ❌ No |
| Schema change | Requires re-index | Instant (config reload) |
| Index size impact | Increases | None |
| Ingestion latency | Adds processing | No impact |
| Typical use | `host`, `source`, `sourcetype`, custom high-value fields | `user`, `src_ip`, `CommandLine`, `EventCode` |

**How to check if a field is indexed:**
```spl
| tstats count WHERE index=windows BY <field>
# If it works → indexed
# If error → search-time only
```

---

## 1.9 Field Categories Summary Table

```
┌────────────────────────────────────────────────────────────────────┐
│                    FIELD ORIGIN TAXONOMY                           │
├──────────────────┬────────────────────┬───────────────────────────┤
│ Category         │ Examples           │ When Available            │
├──────────────────┼────────────────────┼───────────────────────────┤
│ Internal         │ _raw, _time,       │ Always                    │
│                  │ _indextime, _cd    │                           │
├──────────────────┼────────────────────┼───────────────────────────┤
│ Default (indexed)│ index, host,       │ At index time             │
│                  │ source, sourcetype │ (auto)                    │
├──────────────────┼────────────────────┼───────────────────────────┤
│ Custom Indexed   │ CustomField        │ At index time             │
│                  │ (via props.conf)   │ (configured)              │
├──────────────────┼────────────────────┼───────────────────────────┤
│ Search-Time      │ user, src_ip,      │ At search time            │
│ (auto KV)        │ EventCode          │ (auto key=value)          │
├──────────────────┼────────────────────┼───────────────────────────┤
│ Search-Time      │ CustomField        │ At search time            │
│ (regex/REX)      │ (via props/REX)    │ (configured)              │
├──────────────────┼────────────────────┼───────────────────────────┤
│ Calculated       │ lag, is_admin,     │ At search time            │
│ (eval)           │ bytes_per_sec      │ (inline)                  │
└──────────────────┴────────────────────┴───────────────────────────┘
```

---

## 1.10 Ingestion-Time vs Search-Time Processing

This is the **most important architectural distinction** in Splunk.

### Search-Time Processing (What you do in SPL)
```spl
index=windows EventCode=4625
| stats count BY user
```
- Runs on **already-indexed** data
- Does **not** modify stored events
- Results exist only for this search

### Ingestion-Time Processing (What happens before indexing)
```
Raw Data → [HF/Indexer: props.conf, transforms.conf, SEDCMD, INGEST_EVAL]
                    ↓
           Modified _raw / routed / dropped
                    ↓
              Written to Index
```
- Modifies **what gets stored**
- Affects **all future searches**
- Configured on Heavy Forwarder or Indexer

### Modern: Edge Processor / Ingest Processor (SPL2 Pipelines)
```
Raw Data → [EP/IP: SPL2 Pipeline] → Destination
```
- SPL2 pipelines (`from $source | ... | into $destination`)
- Filter, mask, route, enrich **before** indexing
- Replaces legacy `props.conf`/`transforms.conf` for new deployments

---

## 1.11 Filtering, Routing, Masking: Three Different Problems

| Problem | Question | Search-Time | Ingestion-Time (EP/IP) |
|---------|----------|-------------|------------------------|
| **Filtering** | Should this event exist in results? | `search EventCode=4625` | `where EventCode=4625` → drops event |
| **Routing** | Where should this event go? | N/A (results only) | `route profile="linux" [into $dest]` |
| **Masking** | Should sensitive data be hidden? | `eval _raw=replace(...)` (results only) | `eval _raw=replace(...)` → stored masked |

**Why this matters**: Beginners confuse these.
- "I want to drop debug logs" → Ingestion-time `where` (saves license, storage)
- "I want to hide passwords in dashboards" → Search-time `eval` or field-level masking
- "I want Windows logs in one index, Linux in another" → Ingestion-time routing (partition)

---

## 1.12 Key Takeaways

1. **Events = `_raw` + metadata fields**. `_raw` is immutable at search time.
2. **Index-time = write path**. Search-time = read path. They have different tools.
3. **Default fields** (`index`, `host`, `source`, `sourcetype`) are indexed; use them for fast filtering.
4. **Custom fields** are search-time by default. Only index them if `tstats` needs them.
5. **SPL and SPL2 query the same data**. Syntax differs; semantics don't.
6. **Filtering ≠ Routing ≠ Masking**. Know which problem you're solving.

---

## 1.13 What's Next

- [Part 2: SPL Fundamentals](Part2_SPL_Fundamentals.md) — Pipeline, filtering, `search` vs `where`
- [Part 3: Command Classification](Part3_Command_Classification.md) — Streaming, reporting, generating, orchestrating
- [Part 4: SPL vs SPL2 Deep Comparison](Part4_SPL_vs_SPL2.md) — Side-by-side syntax and semantics