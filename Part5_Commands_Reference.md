# Part 5 — Search Commands Reference (Classified)

> **Goal**: Look up any command — see its classification, syntax, and key behaviors instantly.

---

## Classification Badges

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

## 5.1 Generating Commands 🟣

### `search` / Implicit Search
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

### `from` (SPL2)
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

### `inputlookup`
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

### `inputcsv`
```
🟣 GENERATING | 🟡 CENT-STREAM (file on search head)
```
**Purpose**: Load CSV file as events
```spl
| inputcsv mydata.csv
```

---

### `makeresults`
```
🟣 GENERATING | 🟡 CENT-STREAM
```
**Purpose**: Generate synthetic events for testing
```spl
| makeresults count=5
| makeresults count=1 data="test"
```

---

### `metadata` / `typeahead` / `rest` / `savedsearch`
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

## 5.2 Filtering Commands 🟢

### `where` 🟢
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

### `search` (filtering mode) 🟢
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

### `regex` (SPL only) 🟢
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

### `CASE()` / `TERM()` 🟢
```
🟢 DIST-STREAM (search modifiers)
```
**Purpose**: Case-sensitive / indexed-term matching
```spl
search CASE("ERROR")           # Case-sensitive
search TERM(127.0.0.1)         # Match as single indexed term
```

---

## 5.3 Streaming Transform Commands 🟢

### `eval` 🟢
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

### `rex` 🟢
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

### `replace` (via `eval`) 🟢
```
🟢 DIST-STREAM
```
```spl
| eval _raw=replace(_raw, /CreditCard=\d+/, "CreditCard=XXXX")
| eval message=replace(message, "error", "ERROR")
```

---

### `rename` 🟢
```
🟢 DIST-STREAM
```
```spl
| rename user AS Account, host AS Endpoint
| rename "field with spaces" AS field_no_spaces
```

---

### `fields` 🟢
```
🟢 DIST-STREAM
```
```spl
| fields _time host user EventCode      # Keep (comma in SPL2)
| fields - _raw _indextime              # Remove
```

---

### `fillnull` / `filldown` 🟢
```
🟢 DIST-STREAM
```
```spl
| fillnull value=0 bytes
| fillnull value="unknown" user host
| filldown user host                    # Fill with previous value
```

---

### `lookup` 🟢*
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

## 5.4 Streaming Stats Commands 🟢

### `streamstats` 🟢
```
🟢 DIST-STREAM (windowed) | 🟡 CENT-STREAM (window=0 global=t)
```
**Purpose**: Running calculations per event
```spl
# Running count
| streamstats count AS seq BY user

# Moving average (5-event window)
| streamstats window=5 current=f avg(bytes) AS moving_avg

# Reset on condition
| streamstats count(eval(action="failure")) AS failures
    reset_after="(action=\"success\")" BY user

# SPL2 syntax differences
| streamstats current=false reset before (action="success") count() BY user
```

---

### `eventstats` 🟡
```
🟡 CENT-STREAM
```
**Purpose**: Aggregate + attach to every event
```spl
| eventstats dc(src_ip) AS uniq_ips BY host
| eventstats avg(bytes) AS host_avg BY host
| where bytes > host_avg
```

---

### `autoregress` 🟢
```
🟢 DIST-STREAM
```
**Purpose**: Copy previous event's field value
```spl
| autoregress src_ip AS prev_ip
| autoregress bytes p=3 AS bytes_3_ago
| autoregress user p=1-3 AS user_1 user_2 user_3
```

---

### `delta` 🟢
```
🟢 DIST-STREAM
```
**Purpose**: Difference from previous event
```spl
| delta _time AS time_diff
| delta bytes AS byte_change p=2
```

---

### `accum` 🟢
```
🟢 DIST-STREAM
```
**Purpose**: Running sum
```spl
| accum bytes AS total_bytes
| accum count AS event_count
```

---

## 5.5 Centralized Streaming Commands 🟡

### `sort` 🟡
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

### `head` / `tail` 🟡
```
🟡 CENT-STREAM
```
```spl
| head 10
| tail 5
```
**SPL2**: `LIMIT 10`, `OFFSET 20 LIMIT 10`

---

### `reverse` 🟡
```
🟡 CENT-STREAM
```
```spl
| reverse
```

---

### `dedup` 🟡
```
🟡 CENT-STREAM
```
```spl
| dedup host
| dedup host, user
| dedup 2 host           # Keep 2 per host
```

---

### `sample` / `sampleratio` 🟡
```
🟡 CENT-STREAM
```
```spl
| sampleratio 0.1        # 10% random sample
| sample 100             # Exact count
```

---

### `transaction` 🟡
```
🟡 CENT-STREAM
```
**Purpose**: Group events into transactions
```spl
| transaction user maxspan=1h maxpause=5m startswith=eval(action="login") endswith=eval(action="logout")
```
**Warning**: Memory intensive. Prefer `streamstats` for sequence detection.

---

## 5.6 Reporting / Non-Streaming Commands 🔴

### `stats` 🔴
```
🔴 REPORTING
```
```spl
| stats count BY user, host
| stats sum(bytes) AS total, avg(bytes) AS avg, dc(src_ip) AS uniq_ips BY host
| stats values(CommandLine) AS cmds BY user   # High cardinality risk!
```

---

### `chart` 🔴
```
🔴 REPORTING
```
```spl
| chart count BY user, host      # Pivot table
| chart sum(bytes) BY host, _time span=1h
```

---

### `timechart` 🔴
```
🔴 REPORTING
```
```spl
| timechart span=1h count BY user
| timechart span=5m avg(bytes) BY host
| timechart span=1h limit=10 useother=f count BY action
```

---

### `top` / `rare` 🔴
```
🔴 REPORTING
```
```spl
| top limit=10 user
| rare limit=5 host
```

---

### `sistats` / `sichart` / `sitimechart` 🔴
```
🔴 REPORTING (Summary Index format)
```
```spl
| sistats count BY user, host
| collect index=summary
```

---

### `geom` / `geostats` / `cluster` 🔴
```
🔴 REPORTING
```
```spl
| geostats latfield=lat longfield=lon count BY city
| cluster t=0.9
```

---

## 5.7 Orchestrating Commands 🟠

### `join` / `selfjoin` 🟠
```
🟠 ORCHESTRATING | 🟡 CENT-STREAM
```
```spl
| join user [ search index=auth | stats count BY user ]
| selfjoin user
```

---

### `append` / `appendpipe` / `appendcols` 🟠
```
🟠 ORCHESTRATING | 🟡 CENT-STREAM
```
```spl
| append [ search index=windows EventCode=4624 ]
| appendpipe [ stats count ]
| appendcols [ search index=perf | stats avg(cpu) ]
```

---

### `map` 🟠
```
🟠 ORCHESTRATING | 🟡 CENT-STREAM
```
```spl
| map search="search index=windows user=$user$ | head 1"
```

---

### `foreach` 🟠
```
🟠 ORCHESTRATING | SPL2
```
```spl2
| foreach * [ eval <<FIELD>>=upper(<<FIELD>>) ]
```

---

## 5.8 Dataset Processing Commands (SPL2)

### `union` / `diff` / `intersect`
```
🟠 ORCHESTRATING | SPL2
```
```spl2
| union [ FROM windows ], [ FROM linux ]
| diff [ FROM windows ], [ FROM linux ]
| intersect [ FROM windows ], [ FROM linux ]
```

---

## 5.9 Structure Commands 🟢

### `mvexpand` 🟢
```
🟢 DIST-STREAM
```
```spl
| mvexpand user
| mvexpand mvfield limit=100
```

---

### `expand` 🟢
```
🟢 DIST-STREAM
```
```spl
| expand array_field
```

---

### `flatten` 🟢
```
🟢 DIST-STREAM
```
```spl
| flatten object_field
```

---

### `mvcombine` / `mvfilter` 🟢
```
🟢 DIST-STREAM
```
```spl
| mvcombine delim="," user
| mvfilter match(user, "admin.*")
```

---

## 5.10 Lookup & Enrichment Commands

### `lookup` (see 5.3) 🟢/🟡

### `outputlookup` 🟡
```
🟡 CENT-STREAM
```
```spl
| outputlookup my_lookup
| outputlookup my_lookup append=true
```

---

### `lookup_editor` / `inputlookup` (see 5.1)

---

## 5.11 Geospatial Commands 🔴

### `iplocation` 🟢
```
🟢 DIST-STREAM
```
```spl
| iplocation src_ip
| iplocation dest_ip prefix=dest_
```

---

### `geom` / `geostats` 🔴
```
🔴 REPORTING
```
```spl
| geom geo_countries featureIdField=country
| geostats latfield=lat longfield=lon count BY city
```

---

## 5.12 Machine Learning Commands 🔴

### `fit` / `apply` 🔴
```
🔴 REPORTING
```
```spl
| fit LogisticRegression target from feature1 feature2 INTO model_v1
| apply model_v1
```

---

### `kmeans` / `anomalydetection` 🔴
```
🔴 REPORTING
```
```spl
| kmeans k=3 feature1 feature2
| anomalydetection _time, bytes
```

---

## 5.13 Pipeline-Only Commands (Edge/Ingest Processor) 🔵

| Command | Purpose | Syntax |
|---------|---------|--------|
| `route` | Divert subset to another path | `route predicate [ \| into $dest ]` |
| `thru` | Copy + continue main path | `thru [ \| into $dest ]` |
| `branch` | Split into independent paths | `branch [path1], [path2], ...` |
| `if` | Conditional processing | `if (cond) [ ... ] elseif [ ... ] else [ ... ]` |

```spl2
$pipeline = | from $source
    | where profile != "debug"
    | route profile="linux" [ | eval index="linux" | into $linux_dest ]
    | thru [ | into $archive_dest ]
    | branch
        [ | into $prod_dest ]
        [ | into $dev_dest ]
    ;
```

---

## 5.14 Custom Command Types ⚪

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

## 5.15 Quick Lookup Table

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

## 5.16 What's Next

- [Part 6: Time & Streaming Commands](Part6_Time_Analysis.md) — Deep dive on time-based analysis
- [Part 7: Sequence & Correlation](Part7_Sequence_Correlation.md) — Detection patterns
- [Part 12: Reference](Part12_Reference.md) — Decision tables and cheat sheets