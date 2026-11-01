# Lab: Command Execution Placement — Indexer vs Search Head

## Objective
Use Job Inspector to observe which commands run on indexers (distributed) vs search head (centralized), and how pipeline order affects execution placement.

## Dataset
Import `labs/data/auth_events.csv` into index `lab_auth`.

## Exercises

### 1. All Distributable — Stays on Indexers
```spl
index=lab_auth
| where EventCode=4625
| eval total_bytes=bytes_in + bytes_out
| fields _time user src_ip total_bytes
```
**Job Inspector**: Check `dispatch.stream.remote` — should show all commands. `dispatch.stream.local` should be minimal.

### 2. Centralized Command Forces to Search Head
```spl
index=lab_auth
| sort 0 _time
| where EventCode=4625
| eval total_bytes=bytes_in + bytes_out
```
**Job Inspector**: 
- `dispatch.stream.remote` — only the initial search
- `dispatch.stream.local` — `sort`, `where`, `eval` all on search head
- **Observation**: Once `sort` runs, everything after stays on search head

### 3. Optimal Order — Filter First, Sort Late
```spl
# GOOD: Filter on indexers, sort on SH
index=lab_auth
| where EventCode=4625
| sort 0 _time
| head 10

# BAD: Sort first, filter on SH
index=lab_auth
| sort 0 _time
| where EventCode=4625
| head 10
```
**Compare**: 
- Good: `dispatch.stream.remote` has `where`; `sort`/`head` in local
- Bad: `sort` in local first; `where` also in local (wasted distribution)

### 4. Reporting Command Boundary
```spl
index=lab_auth
| where EventCode=4625
| stats count BY user
| sort -count
| head 5
```
**Job Inspector**:
- `stats` (map) runs on indexers
- `stats` (reduce) + `sort` + `head` on search head

### 5. Streaming After Reporting (Limited)
```spl
index=lab_auth
| stats count BY user
| streamstats count AS rank
```
**Job Inspector**: `streamstats` runs on search head (post-reporting)

## Validation with Job Inspector

### Key Metrics to Check
| Metric | Meaning | Target |
|--------|---------|--------|
| `dispatch.stream.remote` | Work on indexers | Maximize |
| `dispatch.stream.local` | Work on search head | Minimize |
| `command.search.kv` | KV extraction time | Low |
| `command.sort` | Sort execution time | After filtering |

### How to Read
1. Run search
2. Job ▸ Inspect Job
3. Scroll to "Search job properties" → "Performance"
4. Look at command breakdown

## Expected Results

| Search | Remote Commands | Local Commands |
|--------|-----------------|----------------|
| 1. Filter only | where, eval, fields | (minimal) |
| 2. Sort first | (search only) | sort, where, eval |
| 3. Filter→Sort | where | sort, head |
| 4. Stats | stats (map) | stats (reduce), sort, head |
| 5. Post-stats | (none) | stats, streamstats |

## Key Takeaway
> **Pipeline order = execution placement**. Put distributable streaming (where, eval, rex) BEFORE centralized (sort, dedup, head). Once data crosses to search head, it never goes back to indexers.