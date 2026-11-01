# Lab: SPL2 Structured vs Pipeline vs Edge/Ingest Processor

## Objective
Compare SPL2 syntax styles and understand which commands work in which context (Search vs EP/IP pipelines).

## Dataset
Import `labs/data/auth_events.csv` into index `lab_auth`.

## Part 1: SPL2 Search — Structured vs Pipeline

### 1A. Pipeline Syntax (SPL-compatible)
```spl2
search index=lab_auth EventCode=4625 earliest=-1h
| eval total=bytes_in + bytes_out
| where total > 100
| stats count BY user
| sort -count
| head 10
```

### 1B. Structured Syntax (SQL-style)
```spl2
FROM lab_auth
WHERE EventCode=4625 AND earliest=-1h
EVAL total=bytes_in + bytes_out
WHERE total > 100
GROUP BY user
SELECT user, count() AS count
ORDER BY count DESC
LIMIT 10
```

### 1C. Verify Identical Results
Run both and compare output — they should match exactly.

### 1D. SPL2-Only Features
```spl2
# span in stats (no bin needed)
search index=lab_auth
| stats count() BY _time span=1h

# IP-aware sort
search index=lab_auth
| sort ip(src_ip)
```

## Part 2: Edge Processor Pipeline (`edgeProcessor` profile)

### 2A. Basic Pipeline
```spl2
import route from /splunk/ingest/commands

$pipeline = | from $source
    | where profile != "debug"
    | eval _raw=replace(_raw, /password=\w+/, "password=****")
    | route profile="linux" [
        | eval index="linux"
        | into $linux_dest
    ]
    | into $default_dest;
```

### 2B. Test in EP Builder
1. Open Edge Processor UI
2. Create pipeline with partition: `sourcetype=WinEventLog:Security`
3. Paste pipeline code
4. Use sample data from `labs/data/auth_events.csv`
5. Run and inspect output at each stage

### 2C. Commands NOT Available in EP
```spl2
# These FAIL in Edge Processor
| append [ search index=lab_auth ]
| join user [ search index=lab_auth ]
| table _time user EventCode
| timechart span=1h count
```

### 2D. EP vs Search Comparison

| Feature | SPL2 Search (`splunkd`) | EP Pipeline (`edgeProcessor`) |
|---------|------------------------|------------------------------|
| `FROM` dataset | Index, lookup, DM | `$source` (partition) |
| `INTO` | N/A | Required |
| `route`/`thru`/`branch` | ❌ | ✅ |
| `where` drops | Results only | Event (not indexed) |
| `eval _raw=` | Result only | Modifies stored event |
| Regex engine | PCRE | **PCRE2** |

## Part 3: Ingest Processor Pipeline (`ingestProcessor` profile)

### 3A. Key Differences from EP
```spl2
# Ingest Processor has logs_to_metrics
$pipeline = | from $source
    | stats count() BY _time span=1m, host
    | logs_to_metrics
    | into $metrics_dest;

# NO dedup in Ingest Processor (unlike EP)
```

## Part 4: Migration Exercise

### Convert This SPL to SPL2 Structured
```spl
index=lab_auth EventCode=4625
| stats count BY user, src_ip
| sort -count
| head 5
```

**SPL2 Structured Answer:**
```spl2
FROM lab_auth
WHERE EventCode=4625
GROUP BY user, src_ip
SELECT user, src_ip, count() AS count
ORDER BY count DESC
LIMIT 5
```

### Convert This EP Pipeline to Search SPL2
```spl2
$pipeline = | from $source
    | where EventCode=4625
    | eval severity="high"
    | route src_ip="10.0.0.5" [ | into $alert_dest ]
    | into $default_dest;
```

**Search SPL2 (results only, no routing):**
```spl2
FROM lab_auth
WHERE EventCode=4625
EVAL severity="high"
SELECT _time, user, src_ip, severity
```

## Validation Checklist
- [ ] Both pipeline/structured SPL2 return identical results
- [ ] EP pipeline syntax validates in builder
- [ ] EP commands not in profile correctly error
- [ ] Migration exercise produces equivalent output

## Key Takeaway
> SPL2 is one language with **multiple profiles**. 
> - Search (`splunkd`): Full command set, structured + pipeline syntax
> - Edge Processor (`edgeProcessor`): Subset, pipeline-only, routing commands
> - Ingest Processor (`ingestProcessor`): Subset, pipeline-only, metrics commands
> 
> **Always check the compatibility profile** before using a command.