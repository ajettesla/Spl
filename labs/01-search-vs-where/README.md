# Lab: `search` vs `where` — When to Use Each

## Objective
Understand the functional difference between `search` and `where` by running both against the same dataset and observing which queries work and which fail.

## Dataset
Import `labs/data/auth_events.csv` into index `lab_auth`.

## Exercises

### 1. Simple Field Filter (Both Work)
```spl
# SPL
index=lab_auth EventCode=4625

# SPL2 pipeline
search index=lab_auth EventCode=4625

# SPL2 structured
FROM lab_auth WHERE EventCode=4625
```
**Expected**: All return 21 events (failed logons)

### 2. Field-to-Field Comparison (`where` Only)
```spl
# This FAILS in SPL - search cannot compare fields
index=lab_auth | search bytes_out > bytes_in

# This WORKS - where supports field-to-field
index=lab_auth | where bytes_out > bytes_in

# SPL2 structured
FROM lab_auth WHERE bytes_out > bytes_in
```
**Expected**: `search` version returns 0 or error; `where` returns events where outbound > inbound

### 3. Calculated Field Filter (`where` Only)
```spl
# Calculate then filter - MUST use where
index=lab_auth
| eval total=bytes_in + bytes_out
| where total > 100

# SPL2
FROM lab_auth
EVAL total=bytes_in + bytes_out
WHERE total > 100
```
**Expected**: Only events with combined bytes > 100

### 4. Function in Filter (`where` Only)
```spl
# CIDR match - where only
index=lab_auth | where cidrmatch("10.0.0.0/8", src_ip)

# LIKE pattern - where only
index=lab_auth | where like(process_name, "%powershell%")
```
**Expected**: Events matching CIDR/pattern

### 5. Raw Text Search (`search` Only)
```spl
# Keyword in _raw - search excels here
index=lab_auth "powershell -enc"

# where equivalent (verbose)
index=lab_auth | where like(_raw, "%powershell -enc%")
```
**Expected**: Both return encoded PowerShell events

### 6. Boolean Logic
```spl
# SPL - implicit AND
index=lab_auth EventCode=4625 user=alice

# SPL2 where - explicit AND
index=lab_auth | where EventCode=4625 AND user="alice"

# Mixed OR/AND - where requires parentheses
index=lab_auth | where (EventCode=4625 OR EventCode=4624) AND user="alice"
```

## Validation Checklist
- [ ] Exercise 1: Both return 21 events
- [ ] Exercise 2: `search` fails/returns 0, `where` returns results
- [ ] Exercise 3: Only `where` works with calculated field
- [ ] Exercise 4: Functions only work in `where`
- [ ] Exercise 5: `search` simpler for raw text
- [ ] Exercise 6: Boolean logic differs

## Job Inspector Evidence
For Exercise 2, open Job Inspector and check:
- `command.search` vs `command.where` execution time
- Both should be distributable streaming (run on indexers)

## Key Takeaway
> Use `search` for: simple field=value, keywords in `_raw`, wildcards
> Use `where` for: field-to-field, arithmetic, functions, complex boolean