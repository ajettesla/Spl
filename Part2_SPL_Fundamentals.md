# Part 2 — SPL Fundamentals: Search, Pipeline & Filtering

> **Goal**: Master the mental model of how searches execute. Understand `search` vs `where` — the #1 confusion point.

---

## 2.1 The Pipeline Model: `|` Operator

Every SPL search is a **pipeline** — a sequence of commands connected by `|`.

```
INPUT EVENTS → [COMMAND 1] → [COMMAND 2] → [COMMAND 3] → OUTPUT RESULTS
                    │              │              │
                    ▼              ▼              ▼
               filtering      transforming    reporting
```

**Key principle**: Each command receives events/results from the previous command, processes them, and passes output forward.

```spl
index=windows EventCode=4625
| stats count BY user
| sort -count
| head 10
```

Flow:
1. `index=windows EventCode=4625` → Generate events (implicit `search`)
2. `| stats count BY user` → Transform to summary (1 row per user)
3. `| sort -count` → Reorder rows
4. `| head 10` → Limit to 10 rows

---

## 2.2 Traditional SPL: Implicit `search` Command

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

## 2.3 SPL2: Explicit `search` and `FROM`

SPL2 requires explicit generating commands.

### `search` command (SPL-compatible)
```spl2
search index=windows EventCode=4625
```

### `FROM` command (SQL-style, dataset-oriented)
```spl2
FROM windows
WHERE EventCode=4625
```

**Critical difference**: `FROM` treats the index as a **dataset** with schema awareness. `search` treats it as a keyword filter on `_raw`.

---

## 2.4 Three Ways to Write the Same Search

| Style | Syntax | When to Use |
|-------|--------|-------------|
| **Traditional SPL** | `index=windows EventCode=4625 \| stats count BY user` | Quick ad-hoc, muscle memory |
| **SPL2 Pipeline** | `search index=windows EventCode=4625 \| stats count() BY user` | Migrating SPL → SPL2 incrementally |
| **SPL2 Structured** | `FROM windows WHERE EventCode=4625 GROUP BY user SELECT count(), user` | New development, SQL backgrounds, complex queries |

All three produce **identical results** on the same data.

---

## 2.5 Building a Search Step by Step

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

## 2.6 Field-Value Filtering: `=`, `!=`, `>`, `<`, `>=`, `<=`

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | Equals | `EventCode=4625` |
| `!=` | Not equals | `action!="success"` |
| `>` | Greater than | `bytes>1000000` |
| `<` | Less than | `duration<30` |
| `>=` | Greater or equal | `EventCode>=4624` |
| `<=` | Less or equal | `LogonType<=3` |

**SPL2 note**: In `WHERE` clause, strings **require double quotes**: `WHERE user="admin"`. In `search` command, quotes optional for simple values.

---

## 2.7 `search` Command vs `where` Command: The Critical Difference

This is the #1 source of confusion. **They are not interchangeable.**

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

---

## 2.8 Boolean Logic: `AND`, `OR`, `NOT`

### Traditional SPL / SPL2 `search`
```spl
# Implicit AND
index=windows EventCode=4625 user=admin

# Explicit OR
index=windows (EventCode=4625 OR EventCode=4624)

# NOT
index=windows NOT user=admin
index=windows user!=admin   # Different! See below
```

### SPL2 `WHERE` clause
```spl2
FROM windows
WHERE EventCode=4625 AND user="admin"
WHERE EventCode=4625 OR EventCode=4624
WHERE NOT user="admin"
WHERE user!="admin"   # Different from NOT!
```

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

## 2.9 Wildcards: `*` in `search` vs `%`/`_` in `LIKE`

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

## 2.10 Regular Expressions: `regex` vs `match()` vs `rex`

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

## 2.11 Aggregation Preview: `stats`, `chart`, `timechart`

| Command | Output Shape | Use Case |
|---------|-------------|----------|
| `stats` | Table: one row per group | General aggregation |
| `chart` | Table: rows × columns (pivot) | Two-dimensional grouping |
| `timechart` | Time-series: `_time` + series | Trending over time |

```spl
# stats: one row per group
index=windows EventCode=4625
| stats count BY user, host

# chart: matrix (user × host)
index=windows EventCode=4625
| chart count BY user, host

# timechart: time buckets + split-by
index=windows EventCode=4625
| timechart span=1h count BY user
```

---

## 2.12 Result Shaping: `table`, `fields`, `SELECT`, `rename`, `AS`

### Traditional SPL
```spl
# Table: specific columns, specific order
| table _time host user EventCode

# fields: keep/remove fields (not reorder)
| fields _time host user EventCode
| fields - _raw   # remove _raw

# Rename
| rename user AS Account, host AS Endpoint
```

### SPL2 Structured (`SELECT`)
```spl2
FROM windows
WHERE EventCode=4625
SELECT _time, host AS Endpoint, user AS Account, EventCode
```

**Key difference**: `SELECT` in SPL2 can **exclude internal fields** (`_time`, `_raw`) unless explicitly included. `fields` keeps them by default.

---

## 2.13 Sorting & Limiting: `sort`, `ORDER BY`, `head`, `LIMIT`, `OFFSET`

### Traditional SPL
```spl
| sort -count           # Descending (newest/highest first)
| sort +_time           # Ascending (oldest first)
| sort 0 -_time         # 0 = no limit (sort all)
| head 10               # First 10 results
| tail 5                # Last 5 results
```

### SPL2 Structured
```spl2
ORDER BY count DESC
ORDER BY _time ASC
LIMIT 10
OFFSET 20 LIMIT 10   # Pagination: skip 20, take 10
```

### Pipeline vs Structured Order Matters!
```spl
# WRONG: head then sort = sort only 10 events
| head 10 | sort -_time

# RIGHT: sort then head = top 10 of all
| sort -_time | head 10
```

---

## 2.14 Search-Time Filtering vs Ingestion-Time Filtering

### Search-Time (SPL/SPL2 search)
```spl
index=windows EventCode=4625
```
- Runs **after** data is indexed
- Does **not** reduce license usage or storage
- Only affects **this search's results**

### Ingestion-Time (Edge/Ingest Processor SPL2 Pipeline)
```spl2
$pipeline = | from $source
    | where EventCode=4625
    | into $destination;
```
- Runs **before** data hits the index
- **Drops** unwanted events → saves license, storage, search time
- Affects **all downstream consumers**

### When to Use Which?

| Scenario | Approach |
|----------|----------|
| Ad-hoc investigation | Search-time |
| Recurring high-volume noise (debug, health checks) | Ingestion-time `where` |
| Compliance: never store PII | Ingestion-time masking |
| Route firewall logs to security team | Ingestion-time `route` |
| Dashboard / scheduled report | Search-time |

---

## 2.15 Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|----------------|-----|
| `where field=*` | `where` doesn't support `*` | Use `search field="*"` or `WHERE field LIKE "%"` |
| `search field1=field2` | `search` can't compare fields | Use `where field1=field2` |
| `NOT field=value` expecting only non-matching | Also matches events **without** the field | Use `field!=value` if field must exist |
| `head 10 | sort -count` | Sorts only the 10 sampled events | `sort -count | head 10` |
| `stats count BY field` then `where field=value` | `field` not in stats output | Filter **before** `stats` |

---

## 2.16 What's Next

- [Part 3: Command Classification](Part3_Command_Classification.md) — Every command categorized by behavior
- [Part 4: SPL vs SPL2 Deep Comparison](Part4_SPL_vs_SPL2.md) — Complete syntax/semantic mapping
- [Part 5: Commands Reference](Part5_Commands_Reference.md) — Lookup any command with classification