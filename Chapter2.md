# Splunk SPL + SPL2

# Chapter 2 — Command Classification Taxonomy: How Splunk Executes Searches

> **Goal**: Every SPL command has a runtime behavior. Knowing the classification tells you: where it runs, memory impact, parallelization, pipeline ordering, and SPL2 profile compatibility.

---

## 2.1 Why Classification Matters

| Question | Answer Requires Classification |
|----------|-------------------------------|
| Can this command run on indexers? | Distributed streaming vs centralized |
| Will this command kill my search memory? | Non-streaming (reporting) vs streaming |
| Can I put this before `stats`? | Streaming vs reporting |
| Does this work in Edge Processor pipeline? | Pipeline-compatible vs search-only |
| Why is my search slow? | Cardinality, streaming vs non-streaming boundary |

**The classification is not academic — it's operational.**

---

## 2.2 The Complete Taxonomy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SPL COMMAND CLASSIFICATION                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  GENERATING (Dataset Processing)                                            │
│  ├─ Search Generating: search, from, metadata, typeahead                    │
│  ├─ Lookup Generating: inputlookup, inputcsv                                │
│  ├─ REST Generating: rest, savedsearch                                      │
│  └─ Synthetic: makeresults, gentimes, range                                 │
│                                                                              │
│  STREAMING (Event → Event, distributable to indexers)                       │
│  ├─ Filtering: where, search, regex, match, CASE, TERM                     │
│  ├─ Transform: eval, rex, replace, rename, fields, fillnull, filldown      │
│  ├─ Enrichment: lookup (streaming mode)                                    │
│  ├─ Stats (windowed): streamstats, accum, autoregress, delta               │
│  ├─ Structure: mvexpand, expand, flatten, mvcombine, mvfilter              │
│  ├─ Anonymize: anonymize                                                    │
│  └─ Conversion: convert, strcat                                             │
│                                                                              │
│  STREAMING (Centralized - runs on search head only)                         │
│  ├─ Ordering: sort, head, tail, reverse                                    │
│  ├─ Deduplication: dedup                                                   │
│  ├─ Event Stats: eventstats                                                │
│  ├─ Global streamstats: streamstats (window=0, global=t)                   │
│  ├─ Sampling: sample, sampleratio                                          │
│  └─ Transaction: transaction (centralized)                                 │
│                                                                              │
│  REPORTING / NON-STREAMING (Events → Summary, centralized)                 │
│  ├─ Aggregation: stats, chart, timechart, sichart, sirare, sitop           │
│  ├─ Top/rare: top, rare                                                    │
│  ├─ Geospatial: geom, geostats, cluster                                     │
│  ├─ ML: fit, apply, kmeans, anomalydetection                               │
│  └─ Summary: sistats, sichart, sitimechart                                 │
│                                                                              │
│  ORCHESTRATING (Pipeline Control / Multi-dataset)                          │
│  ├─ Subsearch: join, selfjoin, append, appendpipe, appendcols, map         │
│  ├─ Lookup Subsearch: lookup (with subsearch)                              │
│  ├─ Control Flow: if, foreach, foreach                                     │
│  ├─ Pipeline (EP/IP): route, thru, branch                                  │
│  └─ Dataset: union, diff, intersect (SPL2)                                 │
│                                                                              │
│  CUSTOM COMMAND TYPES                                                       │
│  ├─ Generating (no input, produces events)                                 │
│  ├─ Streaming (event in → event out, distributable)                        │
│  ├─ Reporting (events in → summary out, centralized)                       │
│  └─ Distributable vs Centralized (configured in commands.conf)             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Streaming Commands (Distributable) 🟢

**Definition**: Process one event at a time, output zero or more events. Can run on indexers (map-reduce).

### Characteristics
- ✅ Runs on indexers (parallel)
- ✅ Memory: O(1) per event
- ✅ Can precede `stats`/`chart`/`timechart`
- ✅ Works in Edge/Ingest Processor pipelines

### Commands

| Command | Category | Description |
|---------|----------|-------------|
| `where` | Filtering | Boolean expression per event |
| `search` | Filtering | Keyword/field search syntax |
| `regex` | Filtering | PCRE filter (SPL only) |
| `eval` | Transform | Calculate fields, `replace()`, `if()` |
| `rex` | Transform | Regex field extraction |
| `replace` | Transform | String replacement (via `eval`) |
| `rename` | Transform | Rename fields |
| `fields` | Transform | Keep/remove fields |
| `fillnull` | Transform | Fill missing with default |
| `filldown` | Transform | Fill missing with previous value |
| `lookup` | Enrichment | Streaming mode (small lookup) |
| `streamstats` | Stats | Running calculations (windowed) |
| `accum` | Stats | Running sum |
| `autoregress` | Stats | Previous event values |
| `delta` | Stats | Difference from previous |
| `mvexpand` | Structure | Multivalue → multiple events |
| `expand` | Structure | Array/object → events |
| `flatten` | Structure | Object → fields |
| `mvcombine` | Structure | Combine multivalue fields |
| `mvfilter` | Structure | Filter multivalue with eval |
| `anonymize` | Anonymize | Hash/mask fields |
| `convert` | Conversion | Field type conversion |
| `strcat` | Conversion | Concatenate fields |

---

## 2.4 Streaming Commands (Centralized) 🟡

**Definition**: Stream events but require global view or search-head resources. Cannot run on indexers.

### Characteristics
- ❌ Runs only on search head
- ⚠️ Memory: O(n) where n = events seen so far
- ⚠️ Breaks map-reduce (forces all events to search head)
- ✅ Still event→event (not summarizing)

### Commands

| Command | Why Centralized |
|---------|-----------------|
| `sort` | Needs global ordering |
| `head` / `tail` | Needs global position |
| `reverse` | Needs full result set |
| `dedup` | Needs global seen-set |
| `eventstats` | Adds aggregate to every event |
| `streamstats (window=0, global=t)` | Global running calculation |
| `sample` / `sampleratio` | Global sampling decision |
| `transaction` | Groups events globally |

---

## 2.5 Non-Streaming / Reporting Commands 🔴

**Definition**: Consume all events, produce summary results (fewer rows than input). Always centralized.

### Characteristics
- ❌ Runs only on search head
- ❌ Memory: O(groups) — can explode with high cardinality
- ❌ Output is **summary rows**, not events
- ❌ Cannot be followed by streaming commands (without `eventstats`/`streamstats`)

### Commands

| Command | Output |
|---------|--------|
| `stats` | One row per `BY` group |
| `chart` | Pivot table (rows × columns) |
| `timechart` | Time-series (`_time` + series) |
| `top` / `rare` | Ranked frequency table |
| `sistats` / `sichart` / `sitimechart` | Summary index format |
| `geom` / `geostats` | Geospatial aggregation |
| `cluster` | Event clustering |
| `kmeans` / `anomalydetection` | ML model output |

### The Streaming → Reporting Boundary

```
STREAMING COMMANDS          REPORTING COMMANDS
(event → event)             (events → summary)
        │                           │
        ▼                           ▼
where, eval, rex, ...        stats, chart, timechart
        │                           │
        │        ┌──────────────────┘
        │        ▼
        │  BOUNDARY: After reporting command,
        │  you have summary rows, not events.
        │  Streaming commands won't work as expected.
        ▼
eventstats, streamstats   ← Special: can cross boundary
```

---

## 2.6 Generating Commands (Dataset Processing) 🟣

**Definition**: Create the initial event stream. No input events.

### Search-Generating
| Command | Purpose |
|---------|---------|
| `search` (implicit/explicit) | Query index |
| `from` (SPL2) | Query dataset (index, lookup, datamodel) |
| `metadata` | Index metadata (hosts, sources, sourcetypes) |
| `typeahead` | Search-time typeahead suggestions |

### Lookup-Generating
| Command | Purpose |
|---------|---------|
| `inputlookup` | Load lookup table as events |
| `inputcsv` | Load CSV file as events |

### REST-Generating
| Command | Purpose |
|---------|---------|
| `rest` | Query Splunk REST API |
| `savedsearch` | Run saved search |

### Synthetic
| Command | Purpose |
|---------|---------|
| `makeresults` | Generate test events |
| `gentimes` | Generate time series |
| `range` | Generate number sequence |

---

## 2.7 Orchestrating Commands (Pipeline Control) 🟠

**Definition**: Control flow, combine datasets, or manage pipeline branching.

### Subsearch / Dataset Combination
| Command | Behavior |
|---------|----------|
| `join` | Left join with subsearch (centralized) |
| `selfjoin` | Join events within same search |
| `append` | Concatenate subsearch results |
| `appendpipe` | Append subsearch on current results |
| `appendcols` | Append columns from subsearch |
| `map` | Run subsearch per input event |
| `lookup` (with subsearch) | Dynamic lookup |

### Control Flow (SPL2)
| Command | Purpose |
|---------|---------|
| `if` | Conditional pipeline branches |
| `foreach` | Apply subpipeline per field/value |

### Pipeline Control (Edge/Ingest Processor) 🔵
| Command | Purpose |
|---------|---------|
| `route` | Divert subset to another path |
| `thru` | Copy + continue main path |
| `branch` | Split into multiple independent paths |

### Dataset Operations (SPL2)
| Command | Purpose |
|---------|---------|
| `union` | Combine datasets (UNION ALL) |
| `diff` | Dataset difference |
| `intersect` | Dataset intersection |

---

## 2.8 Custom Command Types ⚪

When building custom commands (Python SDK), you choose a type:

| Type | Input | Output | Runs Where | Use Case |
|------|-------|--------|------------|----------|
| **Generating** | None | Events | Search head | Data source connectors |
| **Streaming** | Events | Events | Indexers (if distributable) | Enrichment, parsing, filtering |
| **Reporting** | Events | Summary | Search head | Custom aggregation, ML |

### Distributable vs Centralized (in `commands.conf`)

```ini
[my_streaming_command]
type = streaming
distributed = true        # Runs on indexers
streaming = true
local = false             # Not search-head only

[my_reporting_command]
type = reporting
distributed = false       # Centralized
local = true
```

**Rule**: If your command maintains global state (dedup, sort, sessionization), it **must** be centralized.

---

## 2.9 Command Classification Reference Table

| Command | Class | Distributed? | Memory | Pipeline OK? | EP/IP Profile |
|---------|-------|--------------|--------|--------------|---------------|
| `search` | Generating | ✅ | — | ✅ | `splunkd` |
| `from` | Generating | ✅ | — | ✅ (EP/IP) | `splunkd` |
| `where` | Streaming | ✅ | O(1) | ✅ | All |
| `eval` | Streaming | ✅ | O(1) | ✅ | All |
| `rex` | Streaming | ✅ | O(1) | ✅ | All |
| `fields` | Streaming | ✅ | O(1) | ✅ | All |
| `rename` | Streaming | ✅ | O(1) | ✅ | All |
| `lookup` | Streaming* | ✅* | O(lookup) | ✅ | `splunkd`, `edgeProcessor`, `ingestProcessor` |
| `streamstats` | Streaming | ✅ | O(window) | ✅ | `splunkd` |
| `eventstats` | Streaming (centralized) | ❌ | O(groups) | ✅ | `splunkd` |
| `sort` | Streaming (centralized) | ❌ | O(n) | ⚠️ | `splunkd` |
| `dedup` | Streaming (centralized) | ❌ | O(unique) | ⚠️ | `splunkd`, `edgeProcessor` |
| `head` | Streaming (centralized) | ❌ | O(n) | ⚠️ | `splunkd` |
| `stats` | Reporting | ❌ | O(groups) | ❌ | All |
| `chart` | Reporting | ❌ | O(groups) | ❌ | `splunkd` |
| `timechart` | Reporting | ❌ | O(groups) | ❌ | `splunkd` |
| `top` | Reporting | ❌ | O(groups) | ❌ | `splunkd` |
| `join` | Orchestrating | ❌ | O(subsearch) | ❌ | `splunkd` |
| `append` | Orchestrating | ❌ | O(subsearch) | ❌ | `splunkd` |
| `route` | Orchestrating | N/A | — | ✅ (EP/IP only) | `edgeProcessor`, `ingestProcessor` |
| `thru` | Orchestrating | N/A | — | ✅ (EP/IP only) | `edgeProcessor`, `ingestProcessor` |
| `branch` | Orchestrating | N/A | — | ✅ (EP/IP only) | `edgeProcessor`, `ingestProcessor` |

* `lookup` is distributed only for small lookups; large lookups force centralized.

---

## 2.10 How Classification Affects Performance

### Distributed Streaming = Fast
```spl
index=windows
| where EventCode=4625          # Runs on indexers
| eval normalized_user=lower(user)  # Runs on indexers
| stats count BY normalized_user   # Runs on indexers (map), then search head (reduce)
```

### Centralized Streaming = Bottleneck
```spl
index=windows
| where EventCode=4625
| sort -_time           # FORCES all events to search head
| head 100
| stats count BY user   # Now runs only on search head
```

### Reporting = Memory Risk
```spl
index=windows
| stats values(CommandLine) BY user   # HIGH CARDINALITY = OOM RISK
```

---

## 2.11 Distributed Streaming vs Centralized Streaming: Decision Guide

```
Need to process every event?
    │
    ├─ YES → Can it work on one event at a time?
    │           │
    │           ├─ YES → Does it need global knowledge (sort, dedup, seen-set)?
    │           │           │
    │           │           ├─ YES → CENTRALIZED STREAMING (sort, dedup, head, eventstats)
    │           │           │
    │           │           └─ NO  → DISTRIBUTED STREAMING (where, eval, rex, streamstats windowed)
    │           │
    │           └─ NO (needs all events) → REPORTING (stats, chart, timechart)
    │
    └─ NO (creates events) → GENERATING (search, from, inputlookup)
```

---

## 2.12 Pipeline Ordering Rules

```
GENERATING
    │
    ▼
DISTRIBUTED STREAMING (filter, transform, enrich)
    │
    ▼
[OPTIONAL] CENTRALIZED STREAMING (sort, dedup, head) — MINIMIZE USE
    │
    ▼
REPORTING (stats, chart, timechart) — BOUNDARY
    │
    ▼
POST-REPORTING STREAMING (eventstats, streamstats on summary) — LIMITED
    │
    ▼
ORCHESTRATING (join, append, route, branch) — END OF PIPELINE
```

**Golden Rule**: Filter early (distributed), sort/dedup late (centralized), aggregate last (reporting).

---

## 2.13 Indexer-to-Search-Head Handoff: Visual Execution Model

### How Distributed Search Actually Works

```
Search head dispatches search
          |
          v
┌─────────────────────────────────────────────────────────────────────┐
│                    INDEXER LAYER (Map Phase)                        │
├─────────────┬─────────────┬─────────────┬───────────────────────────┤
│  Indexer A  │  Indexer B  │  Indexer C  │        ...                │
│     |       │     |       │     |       │                           │
│  search     │  search     │  search     │                           │
│  lookup     │  lookup     │  lookup     │  (distributable streaming)│
│  where      │  where      │  where      │                           │
│  eval       │  eval       │  eval       │                           │
│  fields     │  fields     │  fields     │                           │
│     \       │      \      │      \      │                           │
│      \______|_______\_____|_______\_____|                           │
│                  |                                               │
│          partial results return                                   │
└──────────────────|────────────────────────────────────────────────┘
                   |
                   v
┌─────────────────────────────────────────────────────────────────────┐
│                    SEARCH HEAD LAYER (Reduce Phase)                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Merge partial results from all indexers                        │
│                                                                      │
│  2. Execute centralized commands:                                  │
│     sort, head, dedup, eventstats, streamstats (global),           │
│     transaction, sample, join, append, map                         │
│                                                                      │
│  3. Execute remaining distributable commands (now on SH):          │
│     where, eval, rex, stats, chart, timechart, etc.               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Practical Example: The Boundary Crossing

```spl
# GOOD: Filter early, centralized late
index=web
| where status=500                    # Distributable - runs on indexers
| eval total=bytes_in+bytes_out       # Distributable - runs on indexers
| stats count BY uri                  # Distributable (map) on indexers
| sort -count                         # CENTRALIZED - forces to SH
| head 10                             # CENTRALIZED - on SH
| table uri count                     # On search head

# BAD: Centralized early wastes distribution
index=web
| sort -_time                         # CENTRALIZED - forces ALL to SH FIRST
| where status=500                    # Now runs on SH (not indexers!)
| head 100
| stats count BY uri
```

The second search brings ALL events to the search head before filtering, wasting network and indexer CPU.

### Job Inspector Evidence

In Job Inspector, look for:
- `dispatch.stream.remote` → work done on indexers
- `dispatch.stream.local` → work done on search head
- `command.search.kv` → key-value extraction time
- `command.search.typer` → event typing time

**Optimization goal**: Maximize `dispatch.stream.remote`, minimize `dispatch.stream.local`.

---

## 2.14 Mode-Dependent Command Classification (Critical)

**Splunk's six categories are NOT mutually exclusive.** A command's classification can depend on its **position, arguments, options, or mode**. Do not memorize one fixed label for every command.

### 2.14.1 `search` — Position Changes Everything

| Position | Classification | Why |
|----------|----------------|-----|
| First command | Generating | Creates event stream from index |
| After pipe (`| search ...`) | Distributable Streaming | Filters existing stream |

```spl
# Generating
search index=web status=500

# Distributable streaming
index=web | search status=500
```

### 2.14.2 `lookup` — `local` Option Changes Classification

| Option | Classification | Why |
|--------|----------------|-----|
| `local=false` (default) | Distributable Streaming | Runs on indexers |
| `local=true` | Orchestrating | Forces lookup to search head |

```spl
# Distributable streaming (default)
index=auth | lookup user_assets user OUTPUT department

# Orchestrating - forces central processing
index=auth | lookup local=true user_assets user OUTPUT department
```

**SPL2 Note**: The `local` and `update` options are **not supported in SPL2**. Always check the command reference for your language/profile.

### 2.14.3 `dedup` — Hybrid, Mode-Dependent

| Mode / Options | Classification |
|----------------|----------------|
| Default (distributed pre-dedup + centralized merge) | Hybrid |
| `sortby` specified | Dataset Processing |
| `keepevents=true` | Dataset Processing |

### 2.14.4 `fillnull` — Field List Changes Behavior

| Options | Classification |
|---------|----------------|
| Field list specified | Distributable Streaming |
| No field list (fill all) | Dataset Processing |

### 2.14.5 `bin` — `span` Changes Behavior

| Options | Classification |
|---------|----------------|
| `span` specified | Streaming |
| No `span` (auto) | Dataset Processing |

### 2.14.6 `join` — Field List Changes Behavior

| Options | Classification |
|---------|----------------|
| Join fields defined | Centralized Streaming |
| No field list | Dataset Processing |

### 2.14.7 `xyseries` — `grouped` Option Changes Behavior

| Option | Classification |
|--------|----------------|
| `grouped=false` (default) | Streaming |
| `grouped=true` | Transforming (Reporting) |

### 2.14.8 `addtotals` — Row vs Column Changes Behavior

| Type | Classification |
|------|----------------|
| Row totals | Distributable Streaming |
| Column totals | Transforming (Reporting) |

### 2.14.9 Why This Matters for Pipeline Design

```
Indexer-side (distributable)          Search-head side (centralized)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━       ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
where (filtering)                     sort
eval (transform)                      head / tail
rex (extract)                         dedup
fields (project)                      eventstats
lookup (local=false)                  streamstats (global)
fillnull (with fields)                transaction
bin (with span)                       streamstats (window=0)
addtotals (row)                       join (no fields)
```

**Golden Rule**: Put distributable streaming commands BEFORE any centralized command. Once data crosses to the search head, it doesn't go back to indexers.

---

## 2.15 SPL2 Compatibility Profiles: Pipeline Command Availability

Not all commands exist in every SPL2 profile. This affects pipeline design for Edge/Ingest Processor.

| Command | `splunkd` (Search) | `edgeProcessor` | `ingestProcessor` |
|---------|-------------------|-----------------|-------------------|
| `from` (dataset) | ✅ | ❌ (uses `$source`) | ❌ (uses `$source`) |
| `from $source` | ❌ | ✅ | ✅ |
| `into` | ❌ | ✅ | ✅ |
| `route` | ❌ | ✅ | ✅ |
| `thru` | ❌ | ✅ | ✅ |
| `branch` | ❌ | ✅ | ✅ |
| `if` | ✅ | ✅ | ✅ |
| `append` | ✅ | ❌ | ❌ |
| `join` | ✅ | ❌ | ❌ |
| `table` | ✅ | ❌ | ❌ |
| `chart`/`timechart` | ✅ | ❌ | ❌ |
| `dedup` | ✅ | ✅ | ❌ |
| `logs_to_metrics` | ❌ | ❌ | ✅ |
| `stats` | ✅ | ✅ | ✅ |
| `streamstats` | ✅ | ❌ | ❌ |
| `eventstats` | ✅ | ❌ | ❌ |
| `transaction` | ✅ | ❌ | ❌ |
| `lookup` (with `local`) | ✅ | ❌ | ❌ |

**Pipeline-only commands** (EP/IP only): `route`, `thru`, `branch`, `if`, `into`

---

## 2.16 Decision Guide: Which Classification Do I Need?

```
Is this a custom command?
    │
    ├─ YES → Choose type in commands.conf:
    │         Generating / Streaming / Reporting
    │         Set distributed=true/false
    │
    ├─ NO (built-in) → What does it do?
    │
    ├─ Creates events from nothing? → Generating
    │
    ├─ Transforms each event independently?
    │       │
    │       ├─ Can run on indexers? → Distributable Streaming
    │       └─ Needs global view? → Centralized Streaming
    │
    ├─ Collapses events to summary? → Reporting
    │
    ├─ Controls flow / combines datasets? → Orchestrating
    │
    └─ SPL2 pipeline only? → Check profile: edgeProcessor / ingestProcessor
```

---

## 2.17 What's Next

- **Chapter 3**: SPL vs SPL2 Deep Comparison — 40+ syntax mappings, compatibility profiles
- **Chapter 4**: Search Commands Reference — every command with classification badge
- **Chapter 5**: Time-Based Analysis & Streaming Commands — `streamstats`, `eventstats`, `autoregress`, `delta`
- **Chapter 6**: Edge/Ingest Processor Pipeline Design + Sequence Detection
- **Chapter 7**: Performance, SOC Patterns & Reference