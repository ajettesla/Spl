# Part 12 — Reference: Decision Tables & Cheat Sheets

> **Goal**: Quick lookup for common decisions, syntax differences, and anti-patterns.

---

## 12.1 `search` vs `where` vs `WHERE` Decision Table

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

## 12.2 Wildcard & Pattern Matching Cheat Sheet

### `search` Command (Glob-style)
| Pattern | Matches | Example |
|---------|---------|---------|
| `*` | Any chars | `host="DC*"` |
| `?` | Single char | `host="DC0?"` |
| `[abc]` | Char class | `host="DC[012]"` |

### `WHERE` / `where` / `LIKE` (SQL-style)
| Pattern | Matches | Example |
|---------|---------|---------|
| `%` | Any chars | `WHERE host LIKE "DC%"` |
| `_` | Single char | `WHERE host LIKE "DC0_"` |
| `[abc]` | Char class | `WHERE host LIKE "DC[012]"` |
| `ESCAPE '\'` | Literal `%`/`_` | `WHERE field LIKE "100\%" ESCAPE '\'` |

### `match()` Function (Regex)
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

## 12.3 Regex Syntax: SPL vs SPL2

| Feature | SPL `regex` | SPL2 `match()` |
|---------|-------------|----------------|
| Command | `| regex field="pattern"` | `where match(field, "pattern")` |
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

## 12.4 Aggregation Function Quick Reference

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

## 12.5 Streaming Command Options Quick Reference

### `streamstats`
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

### `eventstats`
```spl
| eventstats <stats-func> [BY field-list]
```

### `autoregress`
```spl
| autoregress <field> [AS <newfield>] [p=1|2|3...|1-3]
```

### `delta`
```spl
| delta <field> [AS <newfield>] [p=1]
```

---

## 12.6 Time Window Cheat Sheet

| Need | SPL | SPL2 |
|------|-----|------|
| Search time range | `earliest=-1h latest=now` | `WHERE earliest=-1h AND latest=now` |
| Bucket by time | `bin _time span=10m` | `span=10m` in `stats`/`GROUP BY` |
| Time series | `timechart span=1h count` | `timechart span=1h count()` |
| Running window (events) | `streamstats window=10` | `streamstats window=10` |
| Running window (time) | `streamstats time_window=5m` | Not yet in SPL2 |
| Rolling avg after timechart | `streamstats window=6 current=f avg()` | Same |

### Relative Time Specifiers
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

## 12.7 Command Classification Quick Lookup

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

## 12.8 SPL → SPL2 Migration Cheat Sheet

| SPL | SPL2 Pipeline | SPL2 Structured |
|-----|---------------|-----------------|
| `index=foo` | `search index=foo` | `FROM foo` |
| `field=value` | `search field=value` | `WHERE field=value` |
| `field!=value` | `search field!=value` | `WHERE field!=value` |
| `field=*` | `search field="*"` | `WHERE field LIKE "%"` |
| `field="prefix*"` | `search field="prefix*"` | `WHERE field LIKE "prefix%"` |
| `regex field="..."` | `| where match(field, "...")` | `WHERE match(field, "...")` |
| `\| stats count BY f` | `\| stats count() BY f` | `GROUP BY f SELECT count()` |
| `\| stats sum(x) AS s` | `\| stats sum(x) AS s` | `SELECT sum(x) AS s` |
| `\| sort -count` | `\| sort -count` | `ORDER BY count DESC` |
| `\| head 10` | `\| head 10` | `LIMIT 10` |
| `\| fields a b` | `\| fields a, b` | `SELECT a, b` |
| `\| rename a AS b` | `\| rename a AS b` | `SELECT a AS b` |
| `\| bin _time span=1h` | `\| bin _time span=1h` | `GROUP BY span(_time, 1h)` |
| `\| timechart span=1h c` | `\| timechart span=1h c()` | `GROUP BY span(_time, 1h)` |

---

## 12.9 Pipeline vs Search Context Behavior

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

## 12.10 Performance Decision Tree

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

## 12.11 Common Mistakes & Anti-Patterns

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

## 12.12 Glossary

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

## 12.13 File Inventory

```
Spl/
├── README.md                      # Overview & navigation
├── INDEX.md                       # Complete table of contents
├── Part1_Foundations.md           # Events, fields, data model
├── Part2_SPL_Fundamentals.md      # Pipeline, search, filtering
├── Part3_Command_Classification.md # Streaming, reporting, etc.
├── Part4_SPL_vs_SPL2.md           # Deep syntax/semantic comparison
├── Part5_Commands_Reference.md    # Every command with classification
├── Part6_Time_Analysis.md         # Time, streamstats, autoregress
├── Part7_Sequence_Correlation.md  # Detection patterns
├── Part8_Edge_Ingest_Processor.md # Pipeline design
├── Part9_Custom_Commands.md       # Python SDK
├── Part10_Performance.md          # tstats, tsidx, optimization
├── Part11_SOC_Patterns.md         # MITRE-mapped detections
└── Part12_Reference.md            # This file
```

---

## 12.14 Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.0 | 2026 | Complete restructure with classification taxonomy, SPL2, EP/IP, SOC patterns |
| 1.0 | 2024 | Original chapter-based format |

---

## 12.15 Contributing

Found an error? Missing command? Want to add a detection pattern?

1. Fork the repo
2. Edit the relevant `Part*.md` file
3. Submit PR with description of change

**Style guide**:
- Use classification badges (🟢🟡🔴🟣🟠🔵⚪)
- Include both SPL and SPL2 syntax
- Explain WHY, not just syntax
- Map to MITRE ATT&CK for detections
- Performance implications noted