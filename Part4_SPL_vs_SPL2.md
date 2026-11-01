# Part 4 — SPL vs SPL2: Deep Comparison

> **Goal**: SPL2 is not "new SPL" — it's a dual-syntax language. Understand semantic differences, not just syntax.

---

## 4.1 Philosophy: Evolution, Not Replacement

```
SPL (Traditional)          SPL2
━━━━━━━━━━━━━━             ━━━━━━
Pipeline-only              Pipeline + Structured clauses
Implicit search            Explicit generating commands
Keyword search syntax      Expression language (WHERE)
Space-separated args       Comma-separated args
No parentheses for funcs   Functions require ()
```

**Both run on the same engine.** SPL2 adds SQL-style structured syntax (`FROM ... WHERE ... GROUP BY ... SELECT ...`) while retaining pipeline syntax (`|`).

---

## 4.2 Generating Commands: `search` vs `FROM`

### Traditional SPL
```spl
index=windows EventCode=4625 earliest=-1h
# Implicit: search index=windows EventCode=4625 earliest=-1h
```

### SPL2 Pipeline Syntax
```spl2
search index=windows EventCode=4625 earliest=-1h
# Explicit search command required
```

### SPL2 Structured Syntax (`FROM`)
```spl2
FROM windows
WHERE EventCode=4625 AND earliest=-1h
```

### Semantic Differences

| Aspect | `search` | `FROM` |
|--------|----------|--------|
| **Data model** | Keyword filter on `_raw` | Dataset with schema |
| **Field existence** | Not required | Validated against dataset |
| **Time modifiers** | `earliest`/`latest` args | `WHERE earliest=...` |
| **Multiple datasets** | Subsearch / `join` | `JOIN` clause |
| **SQL familiarity** | Low | High |

---

## 4.3 Filtering: `search` vs `where` vs `WHERE` Clause

### Traditional SPL
```spl
# Implicit search (generating)
index=windows EventCode=4625

# Explicit search (filtering in pipeline)
index=windows | search EventCode=4625

# where command (expression)
index=windows | where src_ip=dest_ip
```

### SPL2 Pipeline
```spl2
# search command (generating)
search index=windows EventCode=4625

# search command (filtering)
FROM windows | search EventCode=4625

# where command (expression)
FROM windows | where src_ip=dest_ip

# WHERE clause (structured)
FROM windows WHERE src_ip=dest_ip
```

### The Critical Difference: `search` vs `where`/`WHERE`

| Capability | `search` / implicit | `where` / `WHERE` |
|------------|---------------------|-------------------|
| Field = value | ✅ `field=value` | ✅ `field=value` |
| Field ≠ value | ✅ `field!=value` | ✅ `field!=value` |
| Field A = Field B | ❌ | ✅ `fieldA=fieldB` |
| Arithmetic | ❌ | ✅ `bytes > 1000` |
| Function calls | ❌ | ✅ `match(field, "regex")` |
| `IN` list | ✅ `field IN (...)` | ✅ `field IN (...)` |
| Keyword in `_raw` | ✅ `"keyword"` | ❌ (use `match(_raw, "keyword")`) |
| Wildcard | `*` | `LIKE "%"` / `match()` |

### `NOT field=value` vs `field!=value` — Same in Both Languages

| Expression | Matches |
|------------|---------|
| `NOT user="admin"` | No `user` field **OR** `user ≠ "admin"` |
| `user!="admin"` | Has `user` field **AND** `user ≠ "admin"` |

---

## 4.4 Pipeline Syntax: `|` vs Structured Clauses

### Traditional SPL / SPL2 Pipeline
```spl
index=windows EventCode=4625
| eval normalized=lower(user)
| stats count BY normalized
| sort -count
| head 10
```

### SPL2 Structured (Clauses in Fixed Order)
```spl2
FROM windows
WHERE EventCode=4625
EVAL normalized=lower(user)
GROUP BY normalized
SELECT normalized, count() AS count
ORDER BY count DESC
LIMIT 10
```

### Clause Order (Fixed in SPL2 Structured)
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

## 4.5 Field Selection: `fields`/`table` vs `SELECT`

### Traditional SPL
```spl
# Keep specific fields (order not guaranteed in output)
| fields _time host user EventCode

# Table: specific order, tabular output
| table _time host user EventCode

# Remove fields
| fields - _raw _indextime
```

### SPL2 Pipeline
```spl2
| fields _time, host, user, EventCode    # Comma required!
| table _time, host, user, EventCode
| fields - _raw, _indextime
```

### SPL2 Structured (`SELECT`)
```spl2
FROM windows
WHERE EventCode=4625
SELECT _time, host, user, EventCode
```

### Key Difference: `SELECT` vs `fields`

| Behavior | `fields` / `table` | `SELECT` |
|----------|-------------------|----------|
| Internal fields (`_time`, `_raw`) | Kept by default | **Excluded** unless listed |
| Field order | `table` yes, `fields` no | Yes |
| Expressions | No | Yes (`SELECT count(), upper(user)`) |
| Renaming | `rename` / `AS` in pipeline | `SELECT field AS alias` |

---

## 4.6 Aggregation: `stats` vs `GROUP BY`/`SELECT`

### Traditional SPL
```spl
index=windows EventCode=4625
| stats count BY user, host
| stats sum(bytes) AS total, avg(bytes) AS avg BY host
```

### SPL2 Pipeline
```spl2
search index=windows EventCode=4625
| stats count() BY user, host
| stats sum(bytes) AS total, avg(bytes) AS avg BY host
```

### SPL2 Structured
```spl2
FROM windows
WHERE EventCode=4625
GROUP BY user, host
SELECT user, host, count() AS count

FROM network
GROUP BY host
SELECT host, sum(bytes) AS total, avg(bytes) AS avg
```

### Differences

| Aspect | SPL `stats` | SPL2 `stats` | SPL2 `GROUP BY` |
|--------|-------------|--------------|-----------------|
| Function parens | Optional: `count` | Required: `count()` | Required: `count()` |
| Multiple aggs | Space-separated | Comma-separated | Comma-separated in `SELECT` |
| `BY` fields | Space-separated | Comma-separated | Comma-separated |
| `AS` alias | `count AS c` | `count() AS c` | `count() AS c` |

---

## 4.7 Sorting: `sort` vs `ORDER BY`

### Traditional SPL / SPL2 Pipeline
```spl
| sort -count           # Descending
| sort +_time           # Ascending
| sort 0 -_time         # 0 = no limit (sort all)
| sort -count, +host    # Multi-field
```

### SPL2 Structured
```spl2
ORDER BY count DESC
ORDER BY _time ASC
ORDER BY count DESC, host ASC
```

### IP-Aware Sorting (SPL2 only)
```spl2
| sort ip(src_ip)       # Numeric IP sort, not lexicographic
ORDER BY ip(src_ip) ASC
```

---

## 4.8 Limiting: `head` vs `LIMIT`/`OFFSET`

### Traditional SPL / SPL2 Pipeline
```spl
| head 10               # First 10
| tail 5                # Last 5
```

### SPL2 Structured
```spl2
LIMIT 10
OFFSET 20 LIMIT 10      # Pagination
```

---

## 4.9 Wildcards: `*` vs `LIKE`/`like()`

### `search` Command (Both SPL/SPL2)
```spl
search CommandLine="*powershell*"
search host="DC*"
```

### `WHERE` Clause / `where` Command
```spl2
WHERE CommandLine LIKE "%powershell%"
WHERE host LIKE "DC%"
WHERE like(CommandLine, "%powershell%")   # Function form
```

### Pattern Mapping

| `search` pattern | `LIKE` pattern | Meaning |
|------------------|----------------|---------|
| `*` | `%` | Zero or more chars |
| (no single-char) | `_` | Exactly one char |
| `prefix*` | `prefix%` | Starts with |
| `*suffix` | `%suffix` | Ends with |
| `*middle*` | `%middle%` | Contains |

---

## 4.10 Regex: `regex` Command vs `match()` Function

### Traditional SPL
```spl
# regex command (filtering)
| regex CommandLine="(?i)powershell.*-enc"

# where + match() (expression)
| where match(CommandLine, "(?i)powershell.*-enc")
```

### SPL2
```spl2
# where command + match()
FROM windows | where match(CommandLine, "(?i)powershell.*-enc")

# WHERE clause + match()
FROM windows WHERE match(CommandLine, "(?i)powershell.*-enc")

# NO regex command in SPL2
```

**SPL2 does not have a `regex` command.** Use `match()` in `where`/`WHERE`.

---

## 4.11 Time Grouping: `bin`/`bucket` vs `span` in `stats`/`GROUP BY`

### Traditional SPL
```spl
index=windows
| bin _time span=10m
| stats count BY _time
```

### SPL2 Pipeline
```spl2
search index=windows
| bin _time span=10m
| stats count() BY _time

# OR: span directly in stats (SPL2 only!)
search index=windows
| stats count() BY _time span=10m
```

### SPL2 Structured
```spl2
FROM windows
GROUP BY span(_time, 10m)
SELECT _time, count() AS count
```

**SPL2 `stats` with `span`** is a major ergonomic improvement — no `bin` needed.

---

## 4.12 Time Series: `timechart` in Both

### Traditional SPL
```spl
index=windows EventCode=4625
| timechart span=1h count BY user
```

### SPL2 Pipeline
```spl2
search index=windows EventCode=4625
| timechart span=1h count() BY user
```

### SPL2 Structured (No Direct Equivalent)
```spl2
# Use stats with span + ORDER BY
FROM windows
WHERE EventCode=4625
GROUP BY span(_time, 1h), user
SELECT _time, user, count() AS count
ORDER BY _time, user
```

---

## 4.13 Streaming Commands: `streamstats`, `eventstats`, `autoregress`, `delta`

### Traditional SPL
```spl
| streamstats count AS seq BY user
| streamstats window=5 current=f avg(bytes) AS moving_avg
| eventstats dc(src_ip) AS uniq_ips BY host
| autoregress src_ip AS prev_ip
| delta _time AS time_diff p=1
```

### SPL2 Pipeline
```spl2
| streamstats count() AS seq BY user
| streamstats window=5 current=false avg(bytes) AS moving_avg
| eventstats dc(src_ip) AS uniq_ips BY host
| autoregress src_ip AS prev_ip
| delta _time AS time_diff p=1
```

### SPL2 Structured
```spl2
# No direct structured equivalent for streaming commands
# Use pipeline syntax after FROM/WHERE
FROM windows
| streamstats count() AS seq BY user
```

**Streaming commands remain pipeline-only in SPL2.**

---

## 4.14 Pipeline-Only Commands: `route`, `thru`, `branch`, `if`

These exist **only in SPL2 Edge/Ingest Processor pipelines**, not in search.

```spl2
$pipeline = | from $source
    | where profile != "debug"
    | route profile="linux" [ | into $linux_dest ]
    | thru [ | into $archive_dest ]
    | branch
        [ | eval env="prod" | into $prod_dest ]
        [ | eval env="dev" | into $dev_dest ]
    ;
```

**No SPL equivalent.** These control data flow in ingestion pipelines.

---

## 4.15 SPL2 in Edge/Ingest Processor vs Search

| Feature | SPL2 Search | SPL2 Pipeline (EP/IP) |
|---------|-------------|----------------------|
| `FROM` dataset | Index, lookup, datamodel | `$source` (partition) |
| `INTO` | Not used | Required: `into $destination` |
| `route`/`thru`/`branch` | ❌ | ✅ |
| `where` drops events | Results only | Drops from pipeline (saves license) |
| `eval _raw=...` | Search results | Modifies stored event |
| `lookup` | Search-time | Ingestion-time enrichment |
| Partition config | Search-time `WHERE` | Builder UI (separate) |

---

## 4.16 Migration Decision Table

| You Have | You Want | Approach |
|----------|----------|----------|
| Working SPL searches | Keep working | Don't migrate |
| New detection development | SPL2 structured | Use `FROM/WHERE/GROUP BY/SELECT` |
| SQL-fluent team | SPL2 structured | Lower cognitive load |
| Edge/Ingest Processor | Pipeline SPL2 | Required |
| Complex subsearches (`join`, `map`) | SPL2 | Use `JOIN` clause or pipeline `join` |
| `regex` command | SPL2 | Replace with `match()` in `where`/`WHERE` |
| `sort` + `head` for top-N | SPL2 structured | `ORDER BY ... LIMIT` |

---

## 4.17 SPL2 Compatibility Profiles: Why Syntax Compiles in One Product but Fails in Another

### 4.17.1 The Core Rule

SPL2 is a shared language, but **each Splunk product implements a subset** of its commands and functions. A **compatibility profile** is the list of SPL2 features supported by a particular product or feature.

| Product / Feature | Profile |
|-------------------|---------|
| Splunk Search app | `splunkd` |
| Splunk Enterprise/Cloud application development | `splunkd` |
| Edge Processor Search tab | `splunkd` |
| **Edge Processor pipelines** | `edgeProcessor` |
| **Ingest Processor pipelines** | `ingestProcessor` |
| Federated search | `federatedSearch` |

**Official reference**: [SPL2 Compatibility Profiles](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/spl2-compatibility-profiles-and-quick-references/spl2-compatibility-profiles)

### 4.17.2 Why This Matters

A command valid in SPL2 Search (`splunkd` profile) may be **unsupported in Edge/Ingest Processor pipelines**.

```spl2
# Valid in Search (splunkd)
FROM main
WHERE status=500
| append [ search index=errors ]
| table _time host error_message

# INVALID in Edge Processor (edgeProcessor profile)
# append, join, table are not in the pipeline command set
```

### 4.17.3 Edge Processor Profile (`edgeProcessor`)

Current Edge Processor pipeline command list:

```text
branch, decrypt, dedup, eval, expand, fields, flatten,
from, if, into, lookup, mvexpand, ocsf, rename, replace,
rex, route, spath, stats, thru, where
```

**Pipeline skeleton:**
```spl2
$pipeline = | from $source
    | <processing commands>
    | into $destination;
```

- `from $source` subset is selected by the **pipeline partition** configured in the Edge Processor builder
- Destination variable is configured in the builder
- **Regex uses PCRE2** (not PCRE)

**Official reference**: [Edge Processor Pipeline Syntax](https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/working-with-pipelines/edge-processor-pipeline-syntax)

### 4.17.4 Ingest Processor Profile (`ingestProcessor`)

Current Ingest Processor pipeline command list:

```text
branch, decrypt, eval, expand, fields, flatten,
from, if, into, logs_to_metrics, lookup, mvexpand,
ocsf, rename, replace, rex, route, spath, stats, thru, where
```

**Key differences from Edge Processor:**
- Includes `logs_to_metrics` (not in EP)
- **Does NOT include `dedup`** (EP has it)

**Official reference**: [Ingest Processor Pipeline Syntax](https://help.splunk.com/en/data-management/process-data-at-ingest-time/use-ingest-processor/working-with-pipelines/ingest-processor-pipeline-syntax)

### 4.17.5 `splunkd` Profile (Search & App Development)

Much broader command set including search-time commands:
- `append`, `join`, `transaction`, `makeresults`, `table`, `chart`, `timechart`, `top`, `rare`, `map`, `foreach`, `localop`, `noop`, `redistribute`, `require`, `xyseries`, `addtotals`, `fieldsummary`, `dbinspect`, `metadata`, `typeahead`, `rest`, `savedsearch`, `inputlookup`, `inputcsv`, `outputlookup`, `collect`, `mcollect`, `meventcollect`, `sendalert`, `runscript`, `script`, `python`, `r`, `apply`, `fit`, `anomalydetection`, `kmeans`, `cluster`, `geom`, `geostats`, `iplocation`, `lookup` (with `local`/`update`), `dedup` (with `sortby`/`keepevents`), `fillnull` (without field list), `bin` (without span), `sort`, `eventstats`, `streamstats`, `autoregress`, `delta`, `accum`, `head`, `tail`, `reverse`, `sample`, `sampleratio`, `where`, `search`, `regex`, `rex`, `spath`, `eval`, `fields`, `rename`, `replace`, `mvcombine`, `mvfilter`, `mvexpand`, `expand`, `flatten`, `anonymize`, `convert`, `strcat`, `makecontinuous`, `filldown`, `rangemap`, `erex`, `regsub`, `typer`, `kvtovc`, `vtojson`, `json`, `xml`, `csv`, `tscollect`, `tsidx`, `walklex`, `gauge`, `marker`, `addinfo`, `addcoltotals`, `eventcount`, `distribtune`, `map`, `foreach`, `appendpipe`, `appendcols`, `selfjoin`, `diff`, `intersect`, `union`, `set`, `searchtxn`, `transaction`, `contingency`, `correlate`, `predict`, `trendline`, `x11`, `series`, `smooth`, `kalman`, `arma`, `arima`, `ets`, `hw`, `decompose`, `seasonal_decompose`, `entropy`, `outlier`, `changepoint`, `detect`, `anomaly`, `alert`, `threshold`, `where`, `search`, `eval`, `fields`, `rename`, `replace`, `rex`, `spath`, `lookup`, `mvexpand`, `expand`, `flatten`, `mvcombine`, `mvfilter`, `anonymize`, `convert`, `strcat`, `makecontinuous`, `filldown`, `rangemap`, `erex`, `regsub`, `typer`, `kvtovc`, `vtojson`, `json`, `xml`, `csv`, `tscollect`, `tsidx`, `walklex`, `gauge`, `marker`, `addinfo`, `addcoltotals`, `eventcount`, `distribtune`

### 4.17.6 Example: Same Concept, Different Syntax Context

**Search-time SPL2 (`splunkd` profile):**
```spl2
FROM main
WHERE status = 500
SELECT host, status
```

**Edge Processor Pipeline (`edgeProcessor` profile):**
```spl2
$pipeline = | from $source
    | where status == 500
    | into $destination;
```

The first reads a named search dataset. The second processes a partition-selected ingestion stream.

### 4.17.7 Compiler Errors Are Evidence About the Profile

If a command compiles in Search but fails in Edge Processor:

1. Check the command is listed in the target profile
2. Check the argument type expected by that profile
3. Check whether search syntax is available in a pipeline
4. Check whether command form and evaluation-function form differ
5. Check the exact Edge/Ingest Processor release documentation

**Do not override compiler output** merely because a general SPL2 Search reference shows similar syntax.

### 4.17.8 Quick Compatibility Reference

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

---

## 4.18 What's Next

- [Part 5: Commands Reference](Part5_Commands_Reference.md) — Every command with classification
- [Part 6: Time & Streaming](Part6_Time_Analysis.md) — `streamstats`, `eventstats`, time windows
- [Part 8: Edge/Ingest Processor](Part8_Edge_Ingest_Processor.md) — Pipeline design patterns