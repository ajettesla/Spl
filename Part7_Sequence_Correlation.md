# Part 7 — Sequence & Correlation Analysis

> **Goal**: Detect patterns across ordered events — brute force, beaconing, lateral movement, exfiltration.

---

## 7.1 Why Sequence Matters

```
Aggregation answers:     "How many?"
Sequence answers:        "In what order?"

Example: 6 events for user "admin"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Aggregation (stats):
  user=admin, failures=4, successes=2

Sequence (streamstats):
  failure → failure → failure → SUCCESS → failure → failure
              ↑                    ↑
         3 failures          success after
         before success      only 2 failures
```

**The sequence IS the signal.**

---

## 7.2 The Golden Rule: Order Must Be Defined

```spl
# WRONG: No guaranteed order
index=auth | streamstats count BY user

# RIGHT: Explicit chronological order
index=auth | sort 0 _time | streamstats count BY user
```

**Always `sort 0 _time` before sequence analysis.** The `0` means no limit — sort all results.

---

## 7.3 Failure → Success Patterns (Brute Force)

### Pattern: N Failures Followed by Success

```spl
index=windows EventCode IN (4624, 4625)
| sort 0 _time
| streamstats count(eval(EventCode=4625)) AS failures
    reset_after="(EventCode=4624)"
    BY user
| where EventCode=4624 AND failures >= 5
| table _time user failures src_ip
```

**Logic**:
1. Sort chronologically
2. Count failures per user
3. Reset counter **after** each success (`reset_after`)
4. Keep success events that had ≥5 preceding failures

### Why `reset_after` not `reset_before`?

| Reset Type | Success Event Carries | Use When |
|------------|----------------------|----------|
| `reset_before` | `failures=0` | Success starts new attack |
| `reset_after` | `failures=N` | Success **ends** attack (captures count) |

---

## 7.4 Consecutive Event Detection

### Pattern: N Consecutive Failures (No Success Between)

```spl
index=windows EventCode=4625
| sort 0 _time
| streamstats count AS consecutive_failures
    reset_before="(EventCode!=4625)"
    BY user
| where consecutive_failures >= 5
```

**How it works**: Counter increments on each 4625. Any non-4625 event resets to 0 before processing.

### Detecting "5 Failures in 5 Minutes" (Time-Window)

```spl
index=windows EventCode=4625
| sort 0 _time
| streamstats time_window=5m count AS failures_5min BY user
| where failures_5min >= 5
| dedup user
| table _time user failures_5min src_ip
```

---

## 7.5 Process Tree Analysis (Parent-Child)

### Pattern: Suspicious Process Chain

```spl
index=windows EventCode=4688
| sort 0 _time
| streamstats current=f last(process_name) AS parent_process BY user, host
| where process_name IN ("cmd.exe","powershell.exe","pwsh.exe")
  AND parent_process IN ("winword.exe","excel.exe","outlook.exe","acrord32.exe")
| table _time host user parent_process process_name CommandLine
```

**Logic**: Office app → shell = macro/script execution.

### Multi-Step Chain

```spl
index=windows EventCode=4688
| sort 0 _time
| autoregress process_name AS parent_process p=1
| autoregress process_name AS grandparent_process p=2
| where process_name="powershell.exe"
  AND parent_process="cmd.exe"
  AND grandparent_process="winword.exe"
| table _time host user grandparent_process parent_process process_name CommandLine
```

---

## 7.6 Beaconing Detection (Regular Intervals)

### Pattern: Fixed Interval Communication

```spl
index=proxy dest_ip=1.2.3.4
| sort 0 _time
| streamstats current=f delta(_time) AS interval BY src_ip,dest_ip
| eval interval_rounded=round(interval/10)*10
| stats count AS connections, values(interval_rounded) AS intervals BY src_ip,dest_ip
| where mvcount(intervals)=1 AND connections > 20
```

**Logic**: All intervals round to same value → regular beacon.

### Advanced: Jitter-Tolerant Beaconing

```spl
index=proxy
| sort 0 _time
| streamstats current=f delta(_time) AS interval BY src_ip,dest_ip
| where interval > 270 AND interval < 330    # 5 min ± 30s
| streamstats count AS beacon_count BY src_ip,dest_ip
| where beacon_count > 10
| stats max(beacon_count) AS max_beacons BY src_ip,dest_ip
```

---

## 7.7 Lateral Movement Patterns

### Pattern: SMB/PSExec (Port 445 + Service Creation)

```spl
# SMB authentication + service creation within 1 minute
(index=windows EventCode=4624 LogonType=3) OR (index=windows EventCode=4697)
| sort 0 _time
| streamstats current=f last(_time) AS prev_time, last(EventCode) AS prev_event BY src_ip,dest_ip
| eval time_diff=_time-prev_time
| where EventCode=4697 AND prev_event=4624 AND time_diff < 60
| table _time src_ip dest_ip user time_diff
```

### Pattern: Pass-the-Hash (Ntlm Auth + Lateral)

```spl
index=windows EventCode=4624 LogonType=3 AuthenticationPackageName=NTLM
| sort 0 _time
| streamstats current=f last(user) AS prev_user BY src_ip
| where user!=prev_user AND prev_user!=""
| stats dc(user) AS unique_users BY src_ip
| where unique_users > 3
```

---

## 7.8 Data Exfiltration: Cumulative Volume

### Pattern: Cumulative Bytes Exceed Threshold

```spl
index=network src_ip=10.0.0.0/8 dest_ip!=10.0.0.0/8
| sort 0 _time
| streamstats sum(bytes_out) AS cumulative_bytes BY src_ip
| where cumulative_bytes > 100000000    # 100 MB
| head 1
| table _time src_ip dest_ip cumulative_bytes
```

### Pattern: Entropy + Volume (Compressed/Encrypted Exfil)

```spl
index=network
| eval entropy=...  # Calculate Shannon entropy of payload
| where entropy > 7.5
| streamstats sum(bytes_out) AS total_encrypted BY src_ip
| where total_encrypted > 50000000
```

---

## 7.9 Privilege Escalation: Token/Handle Patterns

### Pattern: Token Manipulation Sequence

```spl
index=windows (EventCode=4672 OR EventCode=4673 OR EventCode=4688)
| sort 0 _time
| streamstats current=f last(EventCode) AS prev_event BY user,host
| eval esc_sequence=case(
    EventCode=4672 AND prev_event=4688, "privilege_assigned_after_process",
    EventCode=4673 AND prev_event=4672, "token_used_after_assign",
    1=1, null()
)
| where isnotnull(esc_sequence)
| table _time host user EventCode prev_event esc_sequence CommandLine
```

---

## 7.10 Defense Evasion: Log Clearing + Activity

### Pattern: Clear Logs Then Act

```spl
index=windows (EventCode=1102 OR EventCode=4688)
| sort 0 _time
| streamstats current=f last(EventCode) AS prev_event BY host
| where EventCode=4688 AND prev_event=1102
| table _time host user process_name CommandLine
```

---

## 7.11 Credential Access: LSASS Dump Sequence

### Pattern: Process Access → Dump

```spl
index=windows (EventCode=4656 ObjectName="*lsass*" OR EventCode=4688 Image="*procdump*" OR Image="*comsvcs*")
| sort 0 _time
| streamstats current=f last(EventCode) AS prev_event, last(Image) AS prev_image BY host,pid
| where (EventCode=4688 AND prev_event=4656) OR (EventCode=4656 AND prev_event=4688)
| table _time host user Image TargetImage prev_event prev_image
```

---

## 7.12 `transaction` Command: When to Use

**Classification**: 🟡 CENT-STREAM (Memory intensive)

```spl
| transaction session_id maxspan=1h maxpause=10m
| transaction user maxspan=4h startswith=eval(action="login") endswith=eval(action="logout")
```

### When `transaction` is Appropriate
- Session reconstruction (web, VPN, auth)
- Correlating start/end events
- Need `duration` and `eventcount` fields

### When to Prefer `streamstats`
- Simple sequence detection (failure→success)
- High-volume data (transaction OOMs easily)
- Need per-event state (not grouped)

### `streamstats` Alternative to `transaction`

```spl
# Session-like with streamstats
index=web
| sort 0 _time
| streamstats current=f last(_time) AS prev_time BY session_id
| eval gap=_time-prev_time
| eval new_session=if(gap>1800 OR isnull(prev_time), 1, 0)
| streamstats sum(new_session) AS session_id BY session_id
| stats min(_time) AS start, max(_time) AS end, count AS events BY session_id
```

---

## 7.13 Tool Decision Guide

| Detection Need | Primary Tool | Why |
|----------------|--------------|-----|
| N failures then success | `streamstats` + `reset_after` | Captures count on success event |
| N consecutive failures | `streamstats` + `reset_before` | Resets on any non-failure |
| N events in time window | `streamstats time_window` | Time-based, not count-based |
| Previous event field | `autoregress` | Simple, no window needed |
| Time between events | `delta` | Numeric difference |
| Event to group comparison | `eventstats` | Attach aggregate to each event |
| Session reconstruction | `transaction` OR `streamstats` | transaction=simple, streamstats=scalable |
| Parent-child process | `autoregress` + `streamstats` | Multi-step chains |

---

## 7.14 Complete Detection Examples

### Example 1: Brute Force Alert
```spl
index=windows (EventCode=4624 OR EventCode=4625)
| sort 0 _time
| streamstats count(eval(EventCode=4625)) AS failures
    reset_after="(EventCode=4624)"
    BY user, src_ip
| where EventCode=4624 AND failures >= 5
| eval severity=case(failures>=20,"critical",failures>=10,"high",1=1,"medium")
| table _time user src_ip failures severity
| sendalert email, subject="Brute Force: $user$ from $src_ip$"
```

### Example 2: Beaconing Alert
```spl
index=proxy
| sort 0 _time
| streamstats current=f delta(_time) AS interval BY src_ip,dest_ip
| where interval > 270 AND interval < 330
| streamstats count AS beacon_count BY src_ip,dest_ip
| where beacon_count >= 10
| stats max(beacon_count) AS count, values(interval) AS intervals BY src_ip,dest_ip
| eval severity=case(count>=50,"critical",count>=20,"high",1=1,"medium")
| table src_ip,dest_ip count intervals severity
```

### Example 3: Lateral Movement Alert
```spl
(index=windows EventCode=4624 LogonType=3) OR (index=windows EventCode=4697)
| sort 0 _time
| streamstats current=f last(_time) AS prev_time, last(EventCode) AS prev_event BY src_ip,dest_ip
| eval time_diff=_time-prev_time
| where EventCode=4697 AND prev_event=4624 AND time_diff < 60
| eval severity="high"
| table _time src_ip dest_ip user time_diff severity
```

---

## 7.15 Performance Considerations

| Technique | Memory Impact | Optimization |
|-----------|---------------|--------------|
| `sort 0 _time` | High (all events) | Filter first: `index=auth EventCode=4625 \| sort 0 _time` |
| `streamstats` global | Medium | Use `BY` to partition |
| `streamstats time_window` | Medium | Requires sorted `_time` |
| `transaction` | Very High | Avoid >100k events; use `streamstats` alternative |
| `autoregress` | Low | Preferred for simple prev-event |

---

## 7.16 What's Next

- [Part 8: Edge/Ingest Processor](Part8_Edge_Ingest_Processor.md) — Pipeline sequence processing
- [Part 11: SOC Patterns](Part11_SOC_Patterns.md) — More detection patterns
- [Part 12: Reference](Part12_Reference.md) — Streaming command options cheat sheet