# Splunk SPL + SPL2

# Chapter 7 — Performance, Reference & Cheat Sheets

> **Goal**: Make searches fast. Quick lookup for decisions, syntax differences, and anti-patterns.

---

## 7.1 How Splunk Stores Data: tsidx Files

```
Index Structure:
━━━━━━━━━━━━━━━━━
Index (e.g., windows)
    ├── hot/          (actively writing)
    ├── warm/         (read-only, compressed)
    ├── cold/         (older, compressed)
    └── frozen/       (archived/deleted)

Each Bucket:
━━━━━━━━━━━━━━━━━
bucket_123/
    ├── journal.gz      # Raw events (_raw)
    ├── tsidx/          # Inverted index (time-series index)
    │   ├── _time.tsidx
    │   ├── host.tsidx
    │   ├── source.tsidx
    │   ├── sourcetype.tsidx
    │   └── <indexed_field>.tsidx
    └── metadata/       # Bucket info
```

### tsidx = Time Series Index
- **Inverted index**: term → list of event IDs
- One tsidx per **indexed field**
- Enables O(1) term lookup → event retrieval

---

## 7.2 Indexed Fields vs Search-Time Fields: Performance Impact

| Aspect | Indexed Field | Search-Time Field |
|--------|---------------|-------------------|
| **Storage** | In tsidx | Not in tsidx |
| **tstats** | ✅ Supported | ❌ Not supported |
| **walklex** | ✅ Visible | ❌ Not visible |
| **Filter speed** | Sub-second (tsidx lookup) | Slower (scan _raw) |
| **Index size** | Increases | No impact |
| **Schema change** | Re-index required | Config reload |
| **Use case** | High-cardinality, frequent filter | Everything else |

### How to Check If Field Is Indexed
```spl
# If this works → indexed
| tstats count WHERE index=windows BY src_ip

# If error → search-time only
| tstats count WHERE index=windows BY CommandLine
```

---

## 7.3 `tstats`: TSIDX-Based Statistics

**Classification**: 🔴 REPORTING (but uses tsidx, not raw events)

### Syntax
```spl
# Basic count by indexed field
| tstats count WHERE index=windows BY host

# Multiple aggregations
| tstats count, sum(bytes) AS total_bytes WHERE index=network BY src_ip, dest_ip

# Time-series (SPL2 exclusive: span in tstats)
| tstats count WHERE index=windows BY _time span=1h

# With WHERE clause (indexed fields only!)
| tstats count WHERE index=windows host=DC01 BY user
```

### tstats vs stats

| Aspect | `tstats` | `stats` |
|--------|----------|---------|
| Data source | tsidx (indexed fields only) | Raw events + search-time fields |
| Speed | 10-1000x faster | Baseline |
| Field support | Indexed only | All fields |
| `WHERE` clause | Indexed fields only | Any field |
| `BY` clause | Indexed fields only | Any field |
| Time range | `WHERE earliest=...` | `earliest=...` |

### When to Use `tstats`
- High-volume indexes (billions of events)
- Indexed fields only (`host`, `source`, `sourcetype`, custom indexed)
- Count/sum/avg/dc on indexed fields
- Time-series on `_time`

### When NOT to Use `tstats`
- Need search-time fields (`user`, `CommandLine`, `EventCode`)
- Need `rex`/`eval`/`lookup` before aggregation
- Low-volume ad-hoc searches

---

## 7.4 `walklex`: Exploring the Lexicon

```spl
# List all terms in a field's tsidx
| walklex index=windows type=field host

# Count unique terms
| walklex index=windows type=field host | stats count

# Find terms matching pattern
| walklex index=windows type=field host prefix="DC"
```

**Output**:
```
term    count
DC01    1250000
DC02    980000
WEB01   450000
```

**Use cases**:
- Discover indexed field values without scanning events
- Cardinality estimation
- Validate index-time extraction

---

## 7.5 Search Modes: Fast, Smart, Verbose

| Mode | Behavior | Use For |
|------|----------|---------|
| **Fast** | No field extraction, no lookups | `tstats`, `stats`, counting |
| **Smart** | Extract fields only when referenced | Default for most searches |
| **Verbose** | Extract ALL fields, all lookups | Debugging, field discovery |

### Setting Mode
```spl
# In search bar: dropdown
# In savedsearches.conf:
dispatch.search_mode = fast

# In REST API:
POST /services/search/jobs { search_mode: "fast" }
```

### Performance Impact
```
Verbose > Smart > Fast
    │        │        │
    ▼        ▼        ▼
Most      Selective Least
extraction extraction extraction
```

---

## 7.6 Filtering Early: The Cardinality Principle

### Golden Rule
```
Filter as early as possible with the most selective criteria.
```

### Cardinality Hierarchy (Most → Least Selective)
```
1. _time (earliest/latest)           → Bucket elimination
2. index                              → Index elimination
3. sourcetype                         → Bucket elimination (often)
4. source                             → Bucket elimination
5. host                               → Bucket elimination
6. Custom indexed fields              → tsidx lookup
7. Search-time fields (user, EventCode) → _raw scan
8. Keywords in _raw                   → Full _raw scan
```

### Example: Good vs Bad

```spl
# BAD: Filter late, broad first
index=* 
| search user=admin
| search EventCode=4625

# GOOD: Filter early, selective first
index=windows earliest=-1h sourcetype=WinEventLog:Security
| search EventCode=4625 user=admin
```

---

## 7.7 `stats` vs `eventstats` vs `streamstats` Memory

| Command | Memory Pattern | Risk |
|---------|----------------|------|
| `stats` | O(groups) | High if high cardinality `BY` |
| `eventstats` | O(groups) + O(events) | Very high |
| `streamstats` | O(window) or O(events) | Low (windowed) / Medium (global) |

### High-Cardinality Danger
```spl
# DANGEROUS: CommandLine has millions of unique values
| stats values(CommandLine) BY user

# SAFER: Limit cardinality
| stats dc(CommandLine) AS unique_cmds BY user
| where unique_cmds < 1000
```

---

## 7.8 High-Cardinality Fields: `values`, `list`, `dc`

| Function | Memory | Use When |
|----------|--------|----------|
| `count()` | O(1) | Always safe |
| `sum()` / `avg()` | O(1) | Always safe |
| `dc()` | O(unique) | Moderate cardinality |
| `estdc()` | O(1) | High cardinality (approximate) |
| `values()` | O(unique) | Low cardinality only |
| `list()` | O(events) | Avoid in production |

### Safe Patterns
```spl
# Instead of values(CommandLine)
| stats dc(CommandLine) AS unique_cmds BY user
| where unique_cmds > 100

# Sample instead of list
| stats count, first(CommandLine) AS sample_cmd BY user
```

---

## 7.9 Summary Indexing & `sistats`

```spl
# Hourly summary
| tstats count WHERE index=windows BY host, user _time span=1h
| sistats count BY host, user
| collect index=summary addtime=t marker="report=hourly_auth"

# Query summary (fast!)
| stats count BY host, user
```

**Benefit**: Pre-aggregated data → instant dashboards.

---

## 7.10 Data Model Acceleration

```spl
# Accelerate datamodel
| datamodel Authentication Authentication search
| where earliest=-24h
| stats count BY user, action

# Requires: Datamodel acceleration enabled in UI
# Stores tsidx for datamodel fields
```

---

## 7.11 Search Job Inspector: Reading the Metrics

```spl
# In search bar: Job ▸ Inspect Job
# Or: | rest /services/search/jobs | fields sid, scanCount, eventCount, resultCount, runDuration
```

### Key Metrics

| Metric | Meaning | Target |
|--------|---------|--------|
| `scanCount` | Events read from disk | Minimize |
| `eventCount` | Events after filtering | Minimize |
| `resultCount` | Final output rows | As needed |
| `runDuration` | Total search time | < 30s interactive |
| `command.search.kv` | KV extraction time | Low |
| `command.search.typer` | Event typing time | Low |

### Optimization Checklist from Inspector
- [ ] `scanCount` ≈ `eventCount` (not 1000x)
- [ ] No `command.search.rawdata` dominating time
- [ ] `dispatch.stream.local` not excessive (centralized commands)
- [ ] `resultCount` reasonable for use case

---

## 7.12 Optimization Checklist

### Query Design
- [ ] Use `earliest`/`latest` (not `where _time>`)
- [ ] Specify `index=` (not `index=*`)
- [ ] Filter on indexed fields first (`host`, `sourcetype`, `source`)
- [ ] Use `tstats` for indexed-field aggregations
- [ ] Avoid `join`/`append` (use `stats` + `eval`)
- [ ] Limit `values()`/`list()`/`dc()` on high-cardinality fields

### Pipeline Order
- [ ] Generating → Distributed Streaming → Centralized Streaming → Reporting
- [ ] `sort`/`dedup`/`head` as late as possible
- [ ] `eval`/`rex` after filtering (not before)

### Search Mode
- [ ] Fast mode for `tstats`/`stats`/`chart`
- [ ] Smart mode for field-heavy searches
- [ ] Verbose only for debugging

### Infrastructure
- [ ] Index-time extraction for high-value filter fields
- [ ] Data model acceleration for common dashboards
- [ ] Summary indexing for recurring reports
- [ ] Partition indexes by data source/retention

---

## 7.13 Reference: Decision Tables & Cheat Sheets

### 7.13.1 `search` vs `where` vs `WHERE` Decision Table

| Question | Answer | Use |
|----------|--------|-----|
| Simple field=value? | Yes | `search field=value` (or implicit) |
| Keyword in `_raw`? | Yes | `search "keyword"` |
| Wildcard in field? | Yes | `search field="prefix*"` |
| Field A = Field B? | Yes | `where fieldA=fieldB` |
| Arithmetic/comparison? | Yes | `where bytes > 1000000` |
| Complex boolean? | Yes | `where (A AND B) OR C` |
| Regex match? | Yes | `where match(field, "regex")` |
| SQL-style pattern? | Yes | `WHERE field LIKE "prefix%"` |
| Need `IN` list? | Yes | Both support `field IN (...)` |
| Field might not exist? | `NOT field=value` | Matches missing field |
| Field must exist? | `field!=value` | Only matches existing field |

---

### 7.13.2 Wildcard & Pattern Matching Cheat Sheet

#### `search` Command (Glob-style)
| Pattern | Matches | Example |
|---------|---------|---------|
| `*` | Any chars | `host="DC*"` |
| `?` | Single char | `host="DC0?"` |
| `[abc]` | Char class | `host="DC[012]"` |

#### `WHERE` / `where` / `LIKE` (SQL-style)
| Pattern | Matches | Example |
|---------|---------|---------|
| `%` | Any chars | `WHERE host LIKE "DC%"` |
| `_` | Single char | `WHERE host LIKE "DC0_"` |
| `[abc]` | Char class | `WHERE host LIKE "DC[012]"` |
| `ESCAPE '\'` | Literal `%`/`_` | `WHERE field LIKE "100\%" ESCAPE '\'` |

#### `match()` Function (Regex)
| Pattern | Meaning |
|---------|---------|
| `(?i)` | Case-insensitive |
| `^` | Start of string |
| `$` | End of string |
| `\b` | Word boundary |
| `\d` | Digit |
| `\w` | Word char |
| `\s` | Whitespace |

---

### 7.13.3 Regex Syntax: SPL vs SPL2

| Feature | SPL `regex` | SPL2 `match()` |
|---------|-------------|----------------|
| Command | `\| regex field="pattern"` | `where match(field, "pattern")` |
| Delimiter | Quotes | Quotes |
| Case-insensitive | `(?i)` in pattern | `(?i)` in pattern |
| Anchored | No (unanchored) | No (unanchored) |
| Anchored explicitly | `^pattern$` | `^pattern$` |
| Default field | `_raw` | Must specify field |

**Migration**:
```spl
# SPL
| regex CommandLine="(?i)powershell"

# SPL2
| where match(CommandLine, "(?i)powershell")
# OR
WHERE match(CommandLine, "(?i)powershell")
```

---

### 7.13.4 Aggregation Function Quick Reference

| Function | Alias | Input | Output | Memory |
|----------|-------|-------|--------|--------|
| `count()` | `count` | Events | Number | O(1) |
| `count(X)` | — | Field | Non-null count | O(1) |
| `sum(X)` | — | Numeric | Number | O(1) |
| `avg(X)` | — | Numeric | Number | O(1) |
| `min(X)` | — | Any | Min value | O(1) |
| `max(X)` | — | Any | Max value | O(1) |
| `dc(X)` | `distinct_count(X)` | Field | Unique count | O(unique) |
| `estdc(X)` | — | Field | Est. unique | O(1) |
| `values(X)` | — | Field | Multivalue (distinct) | O(unique) |
| `list(X)` | — | Field | Multivalue (ordered) | O(events) |
| `first(X)` | — | Field | First value | O(1) |
| `last(X)` | — | Field | Last value | O(1) |
| `earliest(X)` | — | Time field | Earliest time | O(1) |
| `latest(X)` | — | Time field | Latest time | O(1) |
| `range(X)` | — | Numeric | Max-min | O(1) |
| `stdev(X)` | — | Numeric | Std dev | O(1) |
| `perc<N>(X)` | — | Numeric | Nth percentile | O(1) |

**SPL2**: All functions require `()` — e.g., `count()`, `sum(bytes)`.

---

### 7.13.5 Streaming Command Options Quick Reference

#### `streamstats`
```spl
| streamstats
    [current=true|false]           # Include current event
    [window=5]                     # Event count window
    [time_window=5m]               # Time window (SPL only)
    [global=true|false]            # Shared window across BY
    [reset_before="(cond)"]        # Reset before event
    [reset_after="(cond)"]         # Reset after event
    [reset_on_change=true|false]   # Reset on BY change
    [allnum=true|false]            # Numeric safety (SPL only)
    <stats-func> [BY field-list]
```

#### `eventstats`
```spl
| eventstats <stats-func> [BY field-list]
```

#### `autoregress`
```spl
| autoregress <field> [AS <newfield>] [p=1|2|3...|1-3]
```

#### `delta`
```spl
| delta <field> [AS <newfield>] [p=1]
```

---

### 7.13.6 Time Window Cheat Sheet

| Need | SPL | SPL2 |
|------|-----|------|
| Search time range | `earliest=-1h latest=now` | `WHERE earliest=-1h AND latest=now` |
| Bucket by time | `bin _time span=10m` | `span=10m` in `stats`/`GROUP BY` |
| Time series | `timechart span=1h count` | `timechart span=1h count()` |
| Running window (events) | `streamstats window=10` | `streamstats window=10` |
| Running window (time) | `streamstats time_window=5m` | Not yet in SPL2 |
| Rolling avg after timechart | `streamstats window=6 current=f avg()` | Same |

#### Relative Time Specifiers
| Spec | Meaning |
|------|---------|
| `-15m` | 15 minutes ago |
| `-1h` | 1 hour ago |
| `-24h` | 24 hours ago |
| `-7d` | 7 days ago |
| `-1h@h` | Hour boundary |
| `-1d@d` | Day boundary |
| `now` | Search start time |

---

### 7.13.7 Command Classification Quick Lookup

| Command | Class | Distributed? | Memory | Pipeline OK? |
|---------|-------|--------------|--------|--------------|
| `search` | 🟣 | ✅ | — | ✅ |
| `from` | 🟣 | ✅ | — | ✅ (EP/IP) |
| `where` | 🟢 | ✅ | O(1) | ✅ |
| `eval` | 🟢 | ✅ | O(1) | ✅ |
| `rex` | 🟢 | ✅ | O(1) | ✅ |
| `fields` | 🟢 | ✅ | O(1) | ✅ |
| `rename` | 🟢 | ✅ | O(1) | ✅ |
| `lookup` | 🟢/🟡 | ✅* | O(lookup) | ✅ |
| `streamstats` | 🟢/🟡 | ✅ | O(window) | ✅ |
| `eventstats` | 🟡 | ❌ | O(groups) | ✅ |
| `autoregress` | 🟢 | ✅ | O(1) | ✅ |
| `delta` | 🟢 | ✅ | O(1) | ✅ |
| `sort` | 🟡 | ❌ | O(n) | ⚠️ |
| `head` | 🟡 | ❌ | O(n) | ⚠️ |
| `dedup` | 🟡 | ❌ | O(unique) | ⚠️ |
| `stats` | 🔴 | ❌ | O(groups) | ❌ |
| `chart` | 🔴 | ❌ | O(groups) | ❌ |
| `timechart` | 🔴 | ❌ | O(groups) | ❌ |
| `join` | 🟠 | ❌ | O(subsearch) | ❌ |
| `route` | 🔵 | N/A | — | ✅ (EP/IP) |
| `thru` | 🔵 | N/A | — | ✅ (EP/IP) |
| `branch` | 🔵 | N/A | — | ✅ (EP/IP) |

---

### 7.13.8 SPL → SPL2 Migration Cheat Sheet

| SPL | SPL2 Pipeline | SPL2 Structured |
|-----|---------------|-----------------|
| `index=foo` | `search index=foo` | `FROM foo` |
| `field=value` | `search field=value` | `WHERE field=value` |
| `field!=value` | `search field!=value` | `WHERE field!=value` |
| `field=*` | `search field="*"` | `WHERE field LIKE "%"` |
| `field="prefix*"` | `search field="prefix*"` | `WHERE field LIKE "prefix%"` |
| `regex field="..."` | `\| where match(field, "...")` | `WHERE match(field, "...")` |
| `\| stats count BY f` | `\| stats count() BY f` | `GROUP BY f SELECT count()` |
| `\| stats sum(x) AS s` | `\| stats sum(x) AS s` | `SELECT sum(x) AS s` |
| `\| sort -count` | `\| sort -count` | `ORDER BY count DESC` |
| `\| head 10` | `\| head 10` | `LIMIT 10` |
| `\| fields a b` | `\| fields a, b` | `SELECT a, b` |
| `\| rename a AS b` | `\| rename a AS b` | `SELECT a AS b` |
| `\| bin _time span=1h` | `\| bin _time span=1h` | `GROUP BY span(_time, 1h)` |
| `\| timechart span=1h c` | `\| timechart span=1h c()` | `GROUP BY span(_time, 1h)` |

---

### 7.13.9 Pipeline vs Search Context Behavior

| Command | Search Context | Pipeline (EP/IP) Context |
|---------|----------------|--------------------------|
| `where` | Filters results | **Drops event** (not indexed) |
| `eval _raw=...` | Modifies result | **Modifies stored event** |
| `fields - x` | Hides field | **Removes field** from event |
| `dedup` | Removes from results | **Removes from pipeline** |
| `stats` | Summarizes results | **Summarizes** (can reduce volume) |
| `route` | ❌ Not available | Diverts subset to another path |
| `thru` | ❌ Not available | Copies + continues |
| `branch` | ❌ Not available | Splits into independent paths |

---

### 7.13.10 Performance Decision Tree

```
START: Need to search data?
    │
    ├─ YES → What fields to filter?
    │         │
    │         ├─ _time, index, host, sourcetype, source
    │         │     → Use in search/WHERE clause (INDEXED)
    │         │
    │         └─ Custom fields (user, EventCode, etc.)
    │               → Search-time extraction
    │               → Filter AFTER indexed fields
    │
    ├─ Need aggregation?
    │         │
    │         ├─ Indexed fields only?
    │         │     → Use tstats (FASTEST)
    │         │
    │         └─ Search-time fields needed?
    │               → stats (after filtering)
    │
    ├─ Need sequence detection?
    │         │
    │         ├─ Per-event state?
    │         │     → streamstats (windowed)
    │         │
    │         ├─ Compare to group aggregate?
    │         │     → eventstats
    │         │
    │         └─ Previous event value?
    │               → autoregress / delta
    │
    └─ Need ingestion processing?
              │
              ├─ Filter/drop? → where in pipeline
              ├─ Mask? → eval _raw=replace in pipeline
              ├─ Route? → route/thru/branch in pipeline
              └─ Enrich? → lookup in pipeline
```

---

### 7.13.11 Common Mistakes & Anti-Patterns

| Mistake | Why Wrong | Fix |
|---------|-----------|-----|
| `index=* \| search foo` | Scans all indexes | `index=specific \| search foo` |
| `where field=*` | `where` doesn't support `*` | `search field="*"` or `WHERE field LIKE "%"` |
| `search field1=field2` | Can't compare fields | `where field1=field2` |
| `NOT field=value` for "not equal" | Also matches missing field | Use `field!=value` if field must exist |
| `head 10 \| sort -count` | Sorts only 10 events | `sort -count \| head 10` |
| `stats count BY field \| where field=value` | `field` not in stats output | Filter BEFORE stats |
| `values(high_card_field)` | OOM risk | Use `dc()` or `estdc()` |
| `join` for enrichment | Slow, subsearch limits | Use `lookup` |
| `transaction` on high volume | OOM, slow | Use `streamstats` alternative |
| `eval` before filter | Wasted computation | Filter FIRST, then eval |
| `sort` without `0` | Default limit 10000 | Use `sort 0 _time` for full sort |

---

### 7.13.12 Glossary

| Term | Definition |
|------|------------|
| **Event** | Single record with `_raw` + metadata fields |
| **Index** | Physical storage bucket for events |
| **tsidx** | Time-series index (inverted index per field) |
| **Indexed field** | Field in tsidx (fast filter, `tstats` compatible) |
| **Search-time field** | Extracted at search from `_raw` (flexible) |
| **Pipeline** | Commands connected by `\|` |
| **Generating command** | Creates event stream (`search`, `from`, `inputlookup`) |
| **Streaming command** | Event → event, O(1) memory (`eval`, `rex`, `where`) |
| **Centralized streaming** | Event → event, needs global view (`sort`, `dedup`) |
| **Reporting command** | Events → summary (`stats`, `chart`, `timechart`) |
| **Orchestrating command** | Controls flow (`join`, `route`, `branch`) |
| **Partition** | EP/IP: subset of data entering pipeline (UI config) |
| **Cardinality** | Number of unique values in a field |
| **High cardinality** | Many unique values (e.g., `CommandLine`, `session_id`) |
| **Low cardinality** | Few unique values (e.g., `host`, `action`, `LogonType`) |
| **SPL** | Search Processing Language (traditional) |
| **SPL2** | SPL version 2 (pipeline + structured syntax) |
| **EP** | Edge Processor (ingestion at edge) |
| **IP** | Ingest Processor (ingestion in cloud) |

---

### 7.13.13 File Inventory

```
Spl/
├── README.md                      # Overview & navigation
├── INDEX.md                       # Complete table of contents
├── Chapter1_Foundations.md        # Events, fields, data model
├── Chapter2_Command_Classification.md # Taxonomy + execution model
├── Chapter3_SPL_vs_SPL2.md        # Deep syntax/semantic comparison
├── Chapter4_Commands_Reference.md # Every command with classification
├── Chapter5_Sequence_Detection.md # Brute force, beaconing, etc.
├── Chapter6_EP_IP_Custom.md       # Pipeline design + custom commands
├── Chapter7_Performance_Reference.md # This file
├── labs/
│   ├── data/auth_events.csv
│   ├── 01-search-vs-where/
│   ├── 02-command-execution/
│   ├── 03-statistical-commands/
│   └── 04-spl2-and-pipelines/
└── (original SPL2_EP_IP.md preserved)
```

---

### 7.13.14 Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.0 | 2026 | Complete restructure with classification taxonomy, SPL2, EP/IP, SOC patterns |
| 1.0 | 2024 | Original chapter-based format |

---

### 7.13.15 Contributing

Found an error? Missing command? Want to add a detection pattern?

1. Fork the repo
2. Edit the relevant `Chapter*.md` file
3. Submit PR with description of change

**Style guide**:
- Use classification badges (🟢🟡🔴🟣🟠🔵⚪)
- Include both SPL and SPL2 syntax
- Explain WHY, not just syntax
- Map to MITRE ATT&CK for detections
- Performance implications noted