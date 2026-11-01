# Part 11 — Real-World SOC Detection Patterns

> **Goal**: Production-ready detection logic mapped to MITRE ATT&CK. Modular, performant, explainable.

---

## 11.1 Detection as Code: Modular SPL

```spl
# macros.conf - Reusable detection components
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

---

## 11.2 Brute Force: Failed Logons (T1110.001)

### Basic
```spl
`auth_failed` earliest=-1h
| stats count AS failures BY user, src_ip
| where failures >= 10
| eval severity=case(failures>=50,"critical",failures>=20,"high",1=1,"medium")
```

### Advanced: Sequence-Aware (Failures → Success)
```spl
(`auth_failed` OR `auth_success`) earliest=-4h
| sort 0 _time
| streamstats count(eval(EventCode=4625)) AS failures
    reset_after="(EventCode=4624)"
    BY user, src_ip
| where EventCode=4624 AND failures >= 5
| eval technique="T1110.001"
| table _time user src_ip failures technique
```

### With Enrichment
```spl
`auth_failed` earliest=-1h
| stats count AS failures, values(src_ip) AS src_ips BY user
| `lookup_asset_user` user OUTPUT department, manager
| `lookup_threat_intel` src_ips OUTPUT reputation, country
| where failures >= 10 AND (reputation="malicious" OR country IN ("CN","RU","KP","IR"))
```

---

## 11.3 Password Spray: Distributed Failed Logons (T1110.003)

```spl
`auth_failed` earliest=-1h
| stats dc(user) AS unique_users, values(user) AS users BY src_ip
| where unique_users >= 10
| eval technique="T1110.003"
| eval severity=case(unique_users>=50,"critical",unique_users>=20,"high",1=1,"medium")
| table _time src_ip unique_users users severity technique
```

---

## 11.4 PowerShell: Encoded Commands (T1059.001)

### Basic
```spl
`process_creation` earliest=-1h
| where match(CommandLine, "(?i)-enc(odedcommand)?\s")
| eval technique="T1059.001"
| table _time host user process_name CommandLine technique
```

### Advanced: Entropy + Encoded
```spl
`process_creation` earliest=-1h
| where match(CommandLine, "(?i)powershell.*-enc")
| eval encoded_part=replace(CommandLine, ".*-enc(odedcommand)?\s*", "")
| eval entropy=...  # Shannon entropy calculation
| where entropy > 4.5
| eval technique="T1059.001"
| table _time host user CommandLine entropy technique
```

---

## 11.5 Lateral Movement: SMB/PSExec (T1021.002, T1021.004)

### SMB Session + Service Creation
```spl
(`auth_success` LogonType=3) OR (`process_creation` EventCode=4697)
| sort 0 _time
| streamstats current=f last(_time) AS prev_time, last(EventCode) AS prev_event BY src_ip, dest_ip
| eval time_diff=_time-prev_time
| where EventCode=4697 AND prev_event=4624 AND time_diff < 60
| eval technique="T1021.002"
| table _time src_ip dest_ip user time_diff technique
```

### PSExec (Service Creation + Named Pipe)
```spl
`process_creation` EventCode=4697
| where match(ServiceName, "(?i)psexec|paexec|remcom")
| eval technique="T1021.002"
| table _time host user ServiceName CommandLine technique
```

---

## 11.6 Beaconing: Regular Intervals (T1071.001)

```spl
`network_conn` earliest=-24h
| sort 0 _time
| streamstats current=f delta(_time) AS interval BY src_ip, dest_ip, dest_port
| where interval > 270 AND interval < 330    # 5 min ± 30s
| streamstats count AS beacon_count BY src_ip, dest_ip, dest_port
| where beacon_count >= 12
| stats max(beacon_count) AS count, values(interval) AS intervals BY src_ip, dest_ip, dest_port
| eval technique="T1071.001"
| table src_ip dest_ip dest_port count intervals technique
```

---

## 11.7 Data Exfiltration: Volume + Entropy (T1041)

```spl
`network_conn` src_zone=internal dest_zone=external earliest=-24h
| eval entropy=...  # Calculate on payload if available
| streamstats sum(bytes_out) AS cumulative_bytes BY src_ip
| where cumulative_bytes > 100000000    # 100 MB
| eval technique="T1041"
| table _time src_ip dest_ip cumulative_bytes entropy technique
```

---

## 11.8 Privilege Escalation: Token Manipulation (T1134)

```spl
`process_creation` (EventCode=4672 OR EventCode=4673 OR EventCode=4688)
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

## 11.9 Persistence: Scheduled Tasks (T1053.005)

```spl
`process_creation` EventCode=4698
| where match(TaskName, "(?i)update|sync|backup|microsoft|windows")
| eval technique="T1053.005"
| table _time host user TaskName CommandLine technique
```

---

## 11.10 Defense Evasion: Clear Logs (T1070.001)

```spl
`process_creation` (EventCode=1102 OR EventCode=4688)
| sort 0 _time
| streamstats current=f last(EventCode) AS prev_event BY host
| where EventCode=4688 AND prev_event=1102
| eval technique="T1070.001"
| table _time host user process_name CommandLine technique
```

---

## 11.11 Credential Access: LSASS Dump (T1003.001)

```spl
`process_creation` 
| where match(Image, "(?i)procdump|comsvcs|rundll32.*comsvcs|sqldumper|werfault")
   OR match(CommandLine, "(?i)lsass.*dump|minidump|full.*dump")
| eval technique="T1003.001"
| table _time host user Image CommandLine technique
```

---

## 11.12 Discovery: System Info (T1082, T1083)

```spl
`process_creation`
| where match(CommandLine, "(?i)systeminfo|whoami|net\s+(user|group|localgroup|view)|wmic\s+(computersystem|os|cpu|memory)")
| stats count, values(CommandLine) AS cmds BY host, user
| where count >= 3
| eval technique="T1082,T1083"
| table _time host user count cmds technique
```

---

## 11.13 Collection: Archive Files (T1560)

```spl
`process_creation`
| where match(Image, "(?i)7z|winrar|tar|gzip|zip|compress")
   AND match(CommandLine, "(?i)-r|-a|create|archive")
| eval technique="T1560"
| table _time host user Image CommandLine technique
```

---

## 11.14 Command & Control: DNS Tunneling (T1071.004)

```spl
index=dns query_type IN (TXT, NULL, CNAME)
| stats count, dc(query) AS unique_queries, sum(query_length) AS total_len BY src_ip, dest_ip
| where unique_queries > 50 AND total_len > 10000
| eval technique="T1071.004"
| table src_ip dest_ip count unique_queries total_len technique
```

---

## 11.15 Exfiltration Over Web Service (T1567)

```spl
`network_conn` dest_port IN (80,443,8080,8443)
| stats sum(bytes_out) AS total_out, dc(dest_ip) AS unique_dests BY src_ip, user
| where total_out > 50000000 AND unique_dests < 5
| eval technique="T1567"
| table src_ip user total_out unique_dests technique
```

---

## 11.16 MITRE ATT&CK Mapping Table

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

## 11.17 Detection Engineering Best Practices

### 1. Modular Macros
```spl
# macros.conf
[win_auth]
definition = index=windows sourcetype=WinEventLog:Security

[win_process]
definition = index=windows sourcetype=WinEventLog:Security EventCode=4688
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

## 11.18 Performance-Aware Detection

| Pattern | Optimization |
|---------|--------------|
| Brute force | `tstats` on `Authentication` datamodel |
| Beaconing | `tstats` on `Network_Traffic` + `streamstats` |
| Process chain | `tstats` on `Endpoint_Processes` + `autoregress` |
| Volume exfil | `tstats` on `Network_Traffic` + `streamstats sum` |

---

## 11.19 What's Next

- [Part 12: Reference](Part12_Reference.md) — Decision tables, cheat sheets, migration guide