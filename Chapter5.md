# Splunk SPL + SPL2

# Chapter 5 — Sequence & Correlation Analysis + SOC Detection Patterns

> **Goal**: Detect patterns across ordered events — brute force, beaconing, lateral movement, exfiltration. Production-ready detection logic mapped to MITRE ATT&CK.

---

## 5.1 Why Sequence Matters

```
Aggregation answers:     "How many?"
Sequence answers:        "In what order?"

Example: 6 events for user "admin"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
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

## 5.2 The Golden Rule: Order Must Be Defined

```spl
# WRONG: No guaranteed order
index=auth | streamstats count BY user

# RIGHT: Explicit chronological order
index=auth | sort 0 _time | streamstats count BY user
```

**Always `sort 0 _time` before sequence analysis.** The `0` means no limit — sort all results.

---

## 5.3 Failure → Success Patterns (Brute Force)

### Pattern: N Failures Followed by Success

```spl
index=windows EventCode IN (4624, 4625)
| sort 0 _time
| streamstats count(eval(EventCode=4625)) AS failures
    reset_after="(EventCode=4624)"
    BY user, src_ip
| where EventCode=4624 AND failures >= 5
| eval technique="T1110.001"
| table _time user src_ip failures technique
```

**Logic**:
1. Sort chronologically
2. Count failures per user/src_ip
3. Reset counter **after** each success (`reset_after`)
4. Keep success events that had ≥5 preceding failures

### Why `reset_after` not `reset_before`?

| Reset Type | Success Event Carries | Use When |
|------------|----------------------|----------|
| `reset_before` | `failures=0` | Success starts new attack |
| `reset_after` | `failures=N` | Success **ends** attack (captures count) |

### With Enrichment
```spl
index=windows EventCode=4625
| stats count AS failures, values(src_ip) AS src_ips BY user
| lookup assets user OUTPUT department, manager
| lookup threat_intel src_ips OUTPUT reputation, country
| where failures >= 10 AND (reputation="malicious" OR country IN ("CN","RU","KP","IR"))
```

---

## 5.4 Password Spray: Distributed Failed Logons

```spl
index=windows EventCode=4625 earliest=-1h
| stats dc(user) AS unique_users, values(user) AS users BY src_ip
| where unique_users >= 10
| eval technique="T1110.003"
| eval severity=case(unique_users>=50,"critical",unique_users>=20,"high",1=1,"medium")
| table _time src_ip unique_users users severity technique
```

---

## 5.5 Consecutive Event Detection

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

## 5.6 Process Tree Analysis (Parent-Child)

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

## 5.7 Beaconing Detection (Regular Intervals)

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
| stats max(beacon_count) AS count, values(interval) AS intervals BY src_ip,dest_ip
```

---

## 5.8 Lateral Movement Patterns

### Pattern: SMB/PSExec (Port 445 + Service Creation)

```spl
# SMB authentication + service creation within 1 minute
(index=windows EventCode=4624 LogonType=3) OR (index=windows EventCode=4697)
| sort 0 _time
| streamstats current=f last(_time) AS prev_time, last(EventCode) AS prev_event BY src_ip,dest_ip
| eval time_diff=_time-prev_time
| where EventCode=4697 AND prev_event=4624 AND time_diff < 60
| eval technique="T1021.002"
| table _time src_ip dest_ip user time_diff technique
```

### Pattern: PSExec (Service Creation + Named Pipe)

```spl
index=windows EventCode=4697
| where match(ServiceName, "(?i)psexec|paexec|remcom")
| eval technique="T1021.002"
| table _time host user ServiceName CommandLine technique
```

---

## 5.9 Data Exfiltration: Cumulative Volume

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

## 5.10 Privilege Escalation: Token Manipulation

```spl
index=windows (EventCode=4672 OR EventCode=4673 OR EventCode=4688)
| sort 0 _time
| streamstats current=f last(EventCode) AS prev_event BY host, pid
| eval esc_sequence=case(
    EventCode=4672 AND prev_event=4688, "privilege_assigned_after_process",
    EventCode=4673 AND prev_event=4672, "token_used_after_assign",
    1=1, null()
)
| where isnotnull(esc_sequence)
| eval technique="T1134"
| table _time host user EventCode prev_event esc_sequence CommandLine technique
```

---

## 5.11 Persistence: Scheduled Tasks

```spl
index=windows EventCode=4698
| where match(TaskName, "(?i)update|sync|backup|microsoft|windows")
| eval technique="T1053.005"
| table _time host user TaskName CommandLine technique
```

---

## 5.12 Defense Evasion: Clear Logs

```spl
index=windows (EventCode=1102 OR EventCode=4688)
| sort 0 _time
| streamstats current=f last(EventCode) AS prev_event BY host
| where EventCode=4688 AND prev_event=1102
| eval technique="T1070.001"
| table _time host user process_name CommandLine technique
```

---

## 5.13 Credential Access: LSASS Dump

```spl
index=windows
| where match(Image, "(?i)procdump|comsvcs|rundll32.*comsvcs|sqldumper|werfault")
   OR match(CommandLine, "(?i)lsass.*dump|minidump|full.*dump")
| eval technique="T1003.001"
| table _time host user Image CommandLine technique
```

---

## 5.14 Discovery: System Info

```spl
index=windows
| where match(CommandLine, "(?i)systeminfo|whoami|net\s+(user|group|localgroup|view)|wmic\s+(computersystem|os|cpu|memory)")
| stats count, values(CommandLine) AS cmds BY host, user
| where count >= 3
| eval technique="T1082,T1083"
| table _time host user count cmds technique
```

---

## 5.15 Collection: Archive Files

```spl
index=windows
| where match(Image, "(?i)7z|winrar|tar|gzip|zip|compress")
   AND match(CommandLine, "(?i)-r|-a|create|archive")
| eval technique="T1560"
| table _time host user Image CommandLine technique
```

---

## 5.16 Command & Control: DNS Tunneling

```spl
index=dns query_type IN (TXT, NULL, CNAME)
| stats count, dc(query) AS unique_queries, sum(query_length) AS total_len BY src_ip, dest_ip
| where unique_queries > 50 AND total_len > 10000
| eval technique="T1071.004"
| table src_ip dest_ip count unique_queries total_len technique
```

---

## 5.17 Exfiltration Over Web Service

```spl
index=network dest_port IN (80,443,8080,8443)
| stats sum(bytes_out) AS total_out, dc(dest_ip) AS unique_dests BY src_ip, user
| where total_out > 50000000 AND unique_dests < 5
| eval technique="T1567"
| table src_ip user total_out unique_dests technique
```

---

## 5.18 MITRE ATT&CK Mapping Table

| Detection | Technique | Tactic | Data Source |
|-----------|-----------|--------|-------------|
| Brute Force | T1110.001 | Credential Access | Windows Security |
| Password Spray | T1110.003 | Credential Access | Windows Security |
| PowerShell Encoded | T1059.001 | Execution | Windows Process |
| SMB/PSExec Lateral | T1021.002 | Lateral Movement | Windows Security |
| Beaconing | T1071.001 | C2 | Proxy/Firewall |
| Data Exfiltration | T1041 | Exfiltration | Network/Firewall |
| Token Manipulation | T1134 | Privilege Escalation | Windows Process |
| Scheduled Task | T1053.005 | Persistence | Windows Security |
| Clear Logs | T1070.001 | Defense Evasion | Windows Security |
| LSASS Dump | T1003.001 | Credential Access | Windows Process |
| System Discovery | T1082,T1083 | Discovery | Windows Process |
| Archive Collection | T1560 | Collection | Windows Process |
| DNS Tunneling | T1071.004 | C2 | DNS |
| Web Service Exfil | T1567 | Exfiltration | Proxy/Firewall |

---

## 5.19 Detection Engineering Best Practices

### 1. Modular Macros
```spl
# macros.conf
[auth_failed]
definition = index=windows EventCode=4625

[auth_success]
definition = index=windows EventCode=4624

[process_creation]
definition = index=windows EventCode=4688

[network_conn]
definition = index=firewall action=allow
```

**Usage**:
```spl
`auth_failed` earliest=-1h
| `enrich_user`
| `detect_bruteforce`
```

### 2. Enrichment Lookups
```spl
# assets.csv: ip,hostname,owner,department,criticality
# threat_intel.csv: ip,reputation,country,actor

| lookup assets ip AS src_ip OUTPUT hostname AS src_host, criticality
| lookup threat_intel ip AS dest_ip OUTPUT reputation, country
```

### 3. Alert Suppression
```spl
# Suppress known admin activity
| where NOT (user IN ("svc_backup","svc_monitoring") AND src_ip="10.0.0.5")

# Suppress already alerted (requires notable index)
| where NOT [ search index=notable rule="bruteforce" earliest=-24h | fields user, src_ip | format ]
```

### 4. Risk-Based Scoring
```spl
| eval risk_score=case(
    technique="T1003.001", 90,
    technique="T1134", 80,
    technique="T1059.001", 70,
    technique="T1110.001", 60,
    1=1, 30
)
| eval risk_object=user, risk_object_type="user"
```

### 5. Notable Event Creation
```spl
| sendalert notable, param.rule_name="Brute Force - $user$", param.severity=$severity$
```

---

## 5.20 Performance-Aware Detection

| Pattern | Optimization |
|---------|--------------|
| Brute force | `tstats` on `Authentication` datamodel |
| Beaconing | `tstats` on `Network_Traffic` + `streamstats` |
| Process chain | `tstats` on `Endpoint_Processes` + `autoregress` |
| Volume exfil | `tstats` on `Network_Traffic` + `streamstats sum` |

---

## 5.21 Tool Decision Guide

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

## 5.22 Complete Detection Examples

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
| table src_ip dest_ip count intervals severity
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

## 5.23 Performance Considerations

| Technique | Memory Impact | Optimization |
|-----------|---------------|--------------|
| `sort 0 _time` | High (all events) | Filter first: `index=auth EventCode=4625 \| sort 0 _time` |
| `streamstats` global | Medium | Use `BY` to partition |
| `streamstats time_window` | Medium | Requires sorted `_time` |
| `transaction` | Very High | Avoid >100k events; use `streamstats` alternative |
| `autoregress` | Low | Preferred for simple prev-event |

---

## 5.24 What's Next

- **Chapter 6**: Edge/Ingest Processor Pipeline Design + Custom Commands
- **Chapter 7**: Performance, Reference & Cheat Sheets