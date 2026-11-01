# Splunk SPL + SPL2 — Complete Reference

## Chapter 1 — Foundations: Events, Fields, Data Model & Search Fundamentals

> **Research baseline**: 1 November 2026 | **Products**: Splunk Enterprise 9.x / Cloud, Edge Processor, Ingest Processor

---

## 1.1 Why This Chapter Exists

Before writing **SPL** or **SPL2** searches, you must understand **what you are searching**. Most beginners start with syntax:

```spl
# Traditional SPL
index=windows EventCode=4625
| stats count by user

# SPL2
FROM windows
WHERE EventCode=4625
GROUP BY user
SELECT count(), user
```

They learn syntax but don't understand where `index`, `EventCode`, `user`, `_time`, `_raw` come from. That becomes a problem when working with:
- Field extraction (index-time vs search-time)
- Filtering, routing, masking
- `props.conf`, `transforms.conf`, `SEDCMD`, `INGEST_EVAL`
- Heavy Forwarders, Ingest Processor, Edge Processor
- SPL2 pipelines (different behavior from search)

**This chapter gives you the vocabulary for everything that follows.**

---

## 1.2 What Is an Event?

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

## 1.3 Where Do Events Come From?

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

## 1.4 What Is an Index?

An **index** is a physical storage bucket for events — like a database partition.

```spl
# Traditional SPL
index=windows

# SPL2 search syntax
search index=windows

# SPL2 FROM syntax
FROM windows
```

Indexes control: data retention, access permissions, performance.

---

## 1.5 SPL vs SPL2: Same Data, Different Syntax

| Task | Traditional SPL | SPL2 (search) | SPL2 (FROM) |
|------|-----------------|---------------|-------------|
| Select index | `index=windows` | `search index=windows` | `FROM windows` |
| Filter | `EventCode=4625` | `search EventCode=4625` | `WHERE EventCode=4625` |
| Pipe | `\| stats count` | `\| stats count()` | `\| stats count()` |

**The data doesn't change.** Only the language does.

---

## 1.6 Internal Fields (`_*` prefix)

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

## 1.7 Default Fields (Indexed, Automatic)

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

## 1.8 Extracted Fields: Search-Time vs Index-Time

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

## 1.9 Indexed Fields vs Search-Time Fields

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

## 1.10 Field Categories Summary

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

## 1.11 Ingestion-Time vs Search-Time Processing

**Most important architectural distinction in Splunk.**

### Search-Time Processing (What you do in SPL)
```spl
index=windows EventCode=4625
| stats count BY user
```
- Runs on **already-indexed** data
- Does **not** modify stored events
- Results exist only for this search

### Ingestion-Time Processing (Before indexing)
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

## 1.12 Filtering, Routing, Masking: Three Different Problems

| Problem | Question | Search-Time | Ingestion-Time (EP/IP) |
|---------|----------|-------------|------------------------|
| **Filtering** | Should this event exist in results? | `search EventCode=4625` | `where EventCode=4625` → drops event |
| **Routing** | Where should this event go? | N/A (results only) | `route profile="linux" [into $dest]` |
| **Masking** | Should sensitive data be hidden? | `eval _raw=replace(...)` (results only) | `eval _raw=replace(...)` → stored masked |

**Why this matters**: Beginners confuse these.
- "Drop debug logs" → Ingestion-time `where` (saves license, storage)
- "Hide passwords in dashboards" → Search-time `eval` or field-level masking
- "Windows logs in one index, Linux in another" → Ingestion-time routing (partition)

---

## 1.13 Key Takeaways

1. **Events = `_raw` + metadata fields**. `_raw` is immutable at search time.
2. **Index-time = write path**. Search-time = read path. Different tools.
3. **Default fields** (`index`, `host`, `source`, `sourcetype`) are indexed; use for fast filtering.
4. **Custom fields** are search-time by default. Only index them if `tstats` needs them.
5. **SPL and SPL2 query the same data**. Syntax differs; semantics don't.
6. **Filtering ≠ Routing ≠ Masking**. Know which problem you're solving.

---

## 1.14 Search Fundamentals: Pipeline, `search` vs `where`, SPL2 Syntax

### 1.13.1 The Pipeline Model

Every SPL search is a **pipeline** — commands connected by `|`.

```
INPUT EVENTS → [COMMAND 1] → [COMMAND 2] → [COMMAND 3] → OUTPUT RESULTS
                    │              │              │
                    ▼              ▼              ▼
               filtering      transforming    reporting
```

```spl
index=windows EventCode=4625
| stats count BY user
| sort -count
| head 10
```

Flow:
1. `index=windows EventCode=4625` → Generate events (implicit `search`)
2. `\| stats count BY user` → Transform to summary (1 row per user)
3. `\| sort -count` → Reorder rows
4. `\| head 10` → Limit to 10 rows

---

### 1.13.2 Traditional SPL: Implicit `search`

In traditional SPL, a search starting with field-value pairs **implicitly** runs the `search` command.

```spl
# These are IDENTICAL:
index=windows EventCode=4625
search index=windows EventCode=4625
```

The implicit `search` is a **generating command** — it creates the initial event stream from the index.

**Later in pipeline**, `search` becomes a **filtering command**:
```spl
index=windows
| search EventCode=4625    # Filters the already-retrieved events
```

---

### 1.13.3 SPL2: Explicit `search` and `FROM`

SPL2 requires explicit generating commands.

#### `search` command (SPL-compatible)
```spl2
search index=windows EventCode=4625 earliest=-1h
```

#### `FROM` command (SQL-style, dataset-oriented)
```spl2
FROM windows
WHERE EventCode=4625 AND earliest=-1h
```

**Critical difference**: `FROM` treats the index as a **dataset** with schema awareness. `search` treats it as a keyword filter on `_raw`.

---

### 1.13.4 Three Ways to Write the Same Search

| Style | Syntax | When to Use |
|-------|--------|-------------|
| **Traditional SPL** | `index=windows EventCode=4625 \| stats count BY user` | Quick ad-hoc, muscle memory |
| **SPL2 Pipeline** | `search index=windows EventCode=4625 \| stats count() BY user` | Migrating SPL → SPL2 incrementally |
| **SPL2 Structured** | `FROM windows WHERE EventCode=4625 GROUP BY user SELECT count(), user` | New development, SQL backgrounds, complex queries |

All three produce **identical results** on the same data.

---

## 1.14 `search` Command vs `where` Command: The Critical Difference

**This is the #1 source of confusion. They are not interchangeable.**

### `search` command
- **Language**: Keyword/field search syntax (like Google)
- **Matches**: `_raw` by default, or specific fields
- **Wildcard**: `*` (matches any chars)
- **Use for**: Simple field-value filtering, keyword search

```spl
# Keyword search on _raw
index=windows "failed logon"

# Field-value search
index=windows EventCode=4625 user=admin

# Wildcard
index=windows CommandLine="*powershell*"
```

### `where` command (pipeline) / `WHERE` clause (SPL2 FROM)
- **Language**: Expression evaluation (like programming)
- **Evaluates**: Boolean expression per event
- **Wildcard**: `LIKE` with `%` and `_`, or `match()` for regex
- **Use for**: Field-to-field comparison, math, complex logic

```spl
# Field-to-field comparison (IMPOSSIBLE with search)
index=windows
| where src_ip=dest_ip

# Math
index=network
| where bytes_in > bytes_out * 10

# Complex boolean
index=auth
| where (action="failure" AND attempts>5) OR (user="admin" AND src_ip!="10.0.0.0/8")
```

### Decision Table: `search` vs `where`

| Need | Use |
|------|-----|
| Simple field=value | `search field=value` (or implicit) |
| Keyword in `_raw` | `search "keyword"` |
| Wildcard in field | `search field="prefix*"` |
| Field A = Field B | `where fieldA=fieldB` |
| Arithmetic/comparison | `where field > 100` |
| Complex boolean | `where (A AND B) OR C` |
| Regex match | `where match(field, "regex")` |
| SQL-style pattern | `WHERE field LIKE "prefix%"` |

### `NOT field=value` vs `field!=value` — Critical Distinction

| Expression | Matches Events Where |
|------------|---------------------|
| `NOT user="admin"` | `user` is not "admin" **OR `user` field does not exist** |
| `user!="admin"` | `user` exists **AND** is not "admin" |

```spl
# Find events WITHOUT a user field (e.g., system events)
index=windows NOT user=*

# Find events where user exists but isn't admin
index=windows user!="admin"
```

---

## 1.15 SPL2 Structured Syntax: Clauses in Fixed Order

SPL2 Structured (`FROM ... WHERE ... GROUP BY ... SELECT ...`) has fixed clause order:

```
FROM
  ↓
JOIN
  ↓
WHERE
  ↓
EVAL
  ↓
GROUP BY
  ↓
SELECT
  ↓
HAVING
  ↓
ORDER BY
  ↓
LIMIT
  ↓
OFFSET
```

**You cannot reorder clauses.** Pipeline `|` allows any order.

---

## 1.16 Wildcards: `*` in `search` vs `%`/`_` in `LIKE`

### `search` command (and traditional SPL)
```spl
# * = zero or more characters
CommandLine="*powershell*"
host="DC*"
user="*admin*"
```

### `WHERE` clause / `where` command (expression language)
```spl2
# % = zero or more characters (LIKE)
WHERE CommandLine LIKE "%powershell%"
WHERE host LIKE "DC%"

# _ = exactly one character (LIKE)
WHERE host LIKE "DC0_"

# match() for full regex
WHERE match(CommandLine, "(?i)powershell")
```

**Mnemonic**: `search` uses `*` (glob-style). `WHERE` uses `%`/`_` (SQL-style).

---

## 1.17 Regular Expressions: `regex` vs `match()` vs `rex`

| Command/Function | Purpose | Syntax |
|------------------|---------|--------|
| `regex` (SPL command) | **Filter** events by regex | `\| regex field="pattern"` |
| `match()` (SPL/SPL2 function) | **Evaluate** boolean per event | `where match(field, "pattern")` |
| `rex` (SPL command) | **Extract** fields from text | `\| rex field=_raw "pattern"` |

### Examples

```spl
# FILTER: Keep only events matching pattern
index=windows
| regex CommandLine="(?i)powershell.*-enc"

# EVALUATE: Boolean result for where/if
index=windows
| where match(CommandLine, "(?i)powershell.*-enc")

# EXTRACT: Create new fields
index=windows
| rex field=CommandLine "(?<tool>powershell|pwsh|cscript)"
```

### SPL2: `regex` command not available in `FROM` pipelines
Use `match()` in `WHERE` or `where` command:
```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell.*-enc")

# Or pipeline style
FROM windows
| where match(CommandLine, "(?i)powershell.*-enc")
```

---

## 1.18 Building a Search Step by Step

**Investigation question**: "Which users had failed logons in the last hour?"

```
STEP 1: What data?          → index=windows (or FROM windows)
STEP 2: What events?        → EventCode=4625 (failed logon)
STEP 3: What time range?    → earliest=-1h
STEP 4: What to compute?    → count by user
STEP 5: How to present?     → sort by count desc, top 10
```

```spl
# Traditional SPL
index=windows EventCode=4625 earliest=-1h
| stats count BY user
| sort -count
| head 10

# SPL2 Structured
FROM windows
WHERE EventCode=4625 AND earliest=-1h
GROUP BY user
SELECT user, count() AS count
ORDER BY count DESC
LIMIT 10
```

**Think in questions, not syntax.**

---

## 1.19 Command Classification Primer (Preview)

Every SPL command has a runtime behavior. Knowing the classification tells you: where it runs, memory impact, parallelization, and pipeline ordering.

| Category | Examples | Runs Where | Memory |
|----------|----------|------------|--------|
| **Generating** | `search`, `from`, `inputlookup`, `makeresults` | Indexers/Search Head | — |
| **Distributable Streaming** | `where`, `eval`, `rex`, `fields`, `lookup` (small) | **Indexers** (parallel) | O(1)/event |
| **Centralized Streaming** | `sort`, `head`, `dedup`, `eventstats`, `streamstats` (global) | **Search Head** only | O(n) |
| **Reporting** | `stats`, `chart`, `timechart`, `top`, `rare` | Search Head | O(groups) |
| **Orchestrating** | `join`, `append`, `route`, `thru`, `branch`, `if` | Search Head / Pipeline | Varies |

**Golden Rule**: Filter early (distributed), sort/dedup late (centralized), aggregate last (reporting).

---

## 1.20 SPL2 Compatibility Profiles: Why Syntax Compiles in One Product but Fails in Another

SPL2 is a shared language, but **each Splunk product implements a subset**. A **compatibility profile** is the list of SPL2 features supported by a particular product.

| Product / Feature | Profile |
|-------------------|---------|
| Splunk Search app | `splunkd` |
| Splunk Enterprise/Cloud app development | `splunkd` |
| Edge Processor Search tab | `splunkd` |
| **Edge Processor pipelines** | `edgeProcessor` |
| **Ingest Processor pipelines** | `ingestProcessor` |
| Federated search | `federatedSearch` |

**Why this matters**: A command valid in SPL2 Search (`splunkd`) may be **unsupported in Edge/Ingest Processor pipelines**.

```spl2
# Valid in Search (splunkd)
FROM main
WHERE status=500
| append [ search index=errors ]
| table _time host error_message

# INVALID in Edge Processor (edgeProcessor profile)
# append, join, table are not in the pipeline command set
```

### Edge Processor Profile (`edgeProcessor`)
Current command list: `branch, decrypt, dedup, eval, expand, fields, flatten, from, if, into, lookup, mvexpand, ocsf, rename, replace, rex, route, spath, stats, thru, where`

### Ingest Processor Profile (`ingestProcessor`)
Current command list: `branch, decrypt, eval, expand, fields, flatten, from, if, into, logs_to_metrics, lookup, mvexpand, ocsf, rename, replace, rex, route, spath, stats, thru, where`
- **Key difference**: Includes `logs_to_metrics`; **does NOT include `dedup`** (EP has it)

**Compiler errors are evidence about the profile** — don't override them because general SPL2 Search reference shows similar syntax.

---

## 1.21 Evidence Standard (Required for All Examples)

Every substantial example in this reference follows this standard:

| Requirement | Description |
|-------------|-------------|
| **Product & Version** | Splunk Enterprise 9.x / Cloud, Edge Processor, Ingest Processor |
| **Syntax Family** | Classic SPL or SPL2 (explicitly stated) |
| **SPL2 Compatibility Profile** | `splunkd` (Search), `edgeProcessor`, `ingestProcessor`, `federatedSearch` |
| **Input Example** | Small synthetic event showing relevant fields |
| **Exact Query/Pipeline** | Copy-pasteable SPL/SPL2 |
| **Expected Output** | Table or description of result |
| **Why This Command** | Explanation of classification & intent |
| **Execution/Performance Note** | Where it runs, memory impact, boundary crossing |
| **Official Splunk Reference** | Link to current docs |

**Official documentation is the authority.** Community answers and articles are useful for learning patterns and troubleshooting, but must not override current product documentation.

---

## 1.22 Labs (Hands-On Practice)

This repository includes a synthetic dataset for hands-on validation:

```
labs/
├── data/
│   └── auth_events.csv          # Synthetic auth events (no real data)
├── 01-search-vs-where/          # Lab: when to use each
├── 02-command-execution/        # Lab: Job Inspector evidence
├── 03-statistical-commands/     # Lab: stats vs eventstats vs streamstats
└── 04-spl2-and-pipelines/       # Lab: SPL2 structured vs pipeline vs EP/IP
```

**To use labs:**
1. Import `labs/data/auth_events.csv` into a test Splunk index
2. Run examples after changing the index name
3. Compare result counts and fields
4. Inspect Job Inspector for execution placement
5. Adapt to your own lab only after validating expected output

---

## 1.23 What's Next

- **Chapter 2**: Command Classification Taxonomy — every command categorized by execution behavior
- **Chapter 3**: SPL vs SPL2 Deep Comparison — 40+ syntax mappings, compatibility profiles
- **Chapter 4**: Search Commands Reference — every command with classification badge
- **Chapter 5**: Time-Based Analysis & Streaming Commands — `streamstats`, `eventstats`, `autoregress`, `delta`
- **Chapter 6**: Sequence & Correlation Analysis — detection patterns
- **Chapter 7**: Edge/Ingest Processor Pipeline Design + Performance + Reference