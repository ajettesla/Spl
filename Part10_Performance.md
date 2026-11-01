# Part 10 — Performance: tstats, tsidx & Optimization

> **Goal**: Make searches fast. Understand how Splunk stores data and how to query it efficiently.

---

## 10.1 How Splunk Stores Data: tsidx Files

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

## 10.2 Indexed Fields vs Search-Time Fields: Performance Impact

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

## 10.3 `tstats`: TSIDX-Based Statistics

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

## 10.4 `walklex`: Exploring the Lexicon

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

## 10.5 Search Modes: Fast, Smart, Verbose

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

## 10.6 Filtering Early: The Cardinality Principle

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

## 10.7 `stats` vs `eventstats` vs `streamstats` Memory

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
| stats count, values(CommandLine) AS sample_cmds BY user
| where unique_cmds < 1000
```

---

## 10.8 High-Cardinality Fields: `values`, `list`, `dc`

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

## 10.9 Summary Indexing & `sistats`

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

## 10.10 Data Model Acceleration

```spl
# Accelerate datamodel
| datamodel Authentication Authentication search
| where earliest=-24h
| stats count BY user, action

# Requires: Datamodel acceleration enabled in UI
# Stores tsidx for datamodel fields
```

---

## 10.11 Search Job Inspector: Reading the Metrics

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

## 10.12 Optimization Checklist

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

## 10.13 What's Next

- [Part 11: SOC Patterns](Part11_SOC_Patterns.md) — Detection engineering with performance in mind
- [Part 12: Reference](Part12_Reference.md) — Performance decision tree