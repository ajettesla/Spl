# Splunk SPL + SPL2

# Chapter 6 — Edge/Ingest Processor Pipeline Design + Custom Commands

> **Goal**: Build ingestion pipelines that filter, mask, route, and enrich data BEFORE it hits the index. Extend SPL with custom logic when built-in commands aren't enough.

---

## 6.1 Two-Layer Configuration: Partition + Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    EDGE/INGEST PROCESSOR                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER 1: PARTITION (Builder UI)                               │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                                │
│  "Which subset of incoming data enters THIS pipeline?"         │
│                                                                 │
│  Example:                                                        │
│    sourcetype = cisco:asa                                       │
│    host = firewall-*                                            │
│                                                                 │
│  Configured in UI, NOT in SPL2.                                │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER 2: PIPELINE (SPL2)                                      │
│  ━━━━━━━━━━━━━━━━━━━━━━                                        │
│  "What happens to each event inside this pipeline?"            │
│                                                                 │
│  import route from /splunk/ingest/commands                      │
│                                                                 │
│  $pipeline = | from $source                                     │
│      | where profile != "debug"                                 │
│      | eval _raw=replace(_raw, /pwd=\w+/, "pwd=****")           │
│      | route profile="linux" [ | into $linux_dest ]             │
│      | into $default_dest;                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Critical**: Partition ≠ `where`. Partition defines pipeline scope. `where` filters inside pipeline.

---

## 6.2 Pipeline Syntax

```spl2
import route from /splunk/ingest/commands

$pipeline = | from $source
    | <processing commands>
    | into $destination;
```

| Component | Purpose |
|-----------|---------|
| `import` | Load commands (route, thru, branch, etc.) |
| `$pipeline` | Pipeline variable (required) |
| `from $source` | Input: events matching partition |
| Processing | Filter, transform, enrich, route |
| `into $destination` | Output: where events go |

---

## 6.3 Event Flow Model: 8 Stages

```
SOURCE (Partition)
    │
    ▼
EARLY FILTER (where on _raw)
    │  └─ Drop noise BEFORE parsing (save CPU)
    ▼
EXTRACTION (rex, spath)
    │  └─ Parse fields needed for routing/masking
    ▼
REDUCTION (dedup, stats)
    │  └─ Reduce volume if needed
    ▼
TRANSFORMATION (eval, replace)
    │  └─ Mask, normalize, enrich
    ▼
ENRICHMENT (lookup)
    │  └─ Add context (asset, geo, threat intel)
    ▼
ROUTING/COPYING (route, thru, branch)
    │  └─ Send to correct destination(s)
    ▼
DESTINATION (into)
```

**Not every pipeline needs every stage.**

---

## 6.4 Command Classification for Pipelines

| Category | Commands | Pipeline Use |
|----------|----------|--------------|
| **Input/Output** | `from`, `into` | Required |
| **Filter/Reduce** | `where`, `dedup`, `stats` | Drop/reduce events |
| **Parse/Extract** | `rex`, `spath` | Create fields |
| **Transform** | `eval`, `replace`, `rename`, `fields`, `if` | Modify events |
| **Enrich** | `lookup` | Add context |
| **Route/Copy** | `route`, `thru`, `branch` | Control flow |
| **Structure** | `expand`, `flatten`, `mvexpand` | Handle structured data |
| **Specialized** | `decrypt`, `ocsf` | Crypto, normalization |

---

## 6.5 `where`: Filtering in Pipelines

```spl2
$pipeline = | from $source
    # Early filter on raw (cheap)
    | where NOT match(_raw, /"action"\s*:\s*"debug"/i)
    | where NOT match(_raw, /"profile"\s*:\s*"test"/i)
    
    # Later filter on extracted fields
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    | where profile IN ("linux","dns","firewall")
    | into $dest;
```

### Pipeline `where` vs Search `where`

| Aspect | Search `where` | Pipeline `where` |
|--------|----------------|------------------|
| Dropped events | Hidden from results | **Never indexed** (save license) |
| Performance | Search-time | Ingestion-time |
| `_raw` access | Yes | Yes (before extraction) |

---

## 6.6 `rex`/`spath`: Extraction

```spl2
# Regex extraction (unstructured)
| rex field=_raw /"profile":"(?P<profile>[^"]+)"/
| rex field=_raw /"user":"(?P<user>[^"]+)"/

# Structured extraction (JSON/XML)
| spath input=_raw path=user.name
| spath input=_raw output=geo country_code
```

**Rule**: Use `spath` for JSON/XML. Use `rex` for unstructured or partial parsing.

---

## 6.7 `eval`/`replace`: Transformation & Masking

### Masking Patterns
```spl2
# Credit card
| eval _raw=replace(_raw, /CreditCard=\d+/, "CreditCard=XXXX")

# JSON password
| eval _raw=replace(_raw, /"password":"[^"]*"/, "\"password\":\"xxxx\"")

# Multiple patterns
| eval _raw=replace(_raw, /password=\w+/, "password=****")
| eval _raw=replace(_raw, /api_key=\w+/, "api_key=****")
| eval _raw=replace(_raw, /token=\w+/, "token=****")
```

### Field Creation
```spl2
| eval severity=case(
    match(_raw, /ERROR/), "critical",
    match(_raw, /WARN/), "high",
    1=1, "info"
)
| eval index=lower(profile)
```

---

## 6.8 `route`: Divert Subset to Another Destination

```spl2
import route from /splunk/ingest/commands

$pipeline = | from $source
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    
    # Linux → dedicated index
    | route profile="linux" [
        | eval index="linux"
        | into $linux_destination
    ]
    
    # DNS → dedicated index
    | route profile="dns" [
        | eval index="dns"
        | into $dns_destination
    ]
    
    # Remaining → default
    | into $default_destination;
```

### Flow Diagram
```
ALL EVENTS
    │
    ├─ route profile="linux" ───→ Linux events ───→ $linux_destination
    │
    ├─ route profile="dns" ─────→ DNS events ─────→ $dns_destination
    │
    └─ (remaining) ─────────────→ Other events ───→ $default_destination
```

---

## 6.9 `thru`: Copy + Continue Main Path

```spl2
$pipeline = | from $source
    | eval _raw=replace(_raw, /password=\w+/, "password=****")
    
    # Archive masked copy
    | thru [
        | into $archive_destination
    ]
    
    # Continue routing on main path
    | route profile="linux" [ | into $linux_dest ]
    | into $default_destination;
```

### Flow Diagram
```
EVENT
    │
    ▼
MASK
    │
    ├─ thru ───→ COPY ───→ $archive_destination
    │
    └─ (original) ──→ route ───→ $linux_dest / $default_destination
```

---

## 6.10 `branch`: Multiple Independent Paths

```spl2
$pipeline = | from $source
    | eval _raw=replace(_raw, /password=\w+/, "password=****")
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    
    | branch
        # Path 1: Full routing logic
        [
            | route profile="linux" [ | into $linux_dest ]
            | route profile="dns" [ | into $dns_dest ]
            | into $default_dest
        ],
        # Path 2: Raw archive (before masking? after? your choice)
        [
            | into $raw_archive_dest
        ];
```

### Flow Diagram
```
MASKED EVENT
    │
    ▼
BRANCH
   /     \
  ▼       ▼
PATH 1   PATH 2
  │         │
  ▼         ▼
route...  archive
```

---

## 6.11 `route` vs `thru` vs `branch` vs `if`: Decision Guide

| Need | Command | Example |
|------|---------|---------|
| "Send subset elsewhere, rest continues" | `route` | Linux logs → security index |
| "Keep copy for archive, continue processing" | `thru` | Audit trail + normal routing |
| "Same event → multiple independent pipelines" | `branch` | Prod + Dev + Archive simultaneously |
| "Transform differently per condition, same dest" | `if` | Set index per profile, single destination |

### `if` Example (Conditional Transform)
```spl2
$pipeline = | from $source
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    | if (profile="linux") [
        | eval index="linux", sourcetype="linux_syslog"
    ]
    elseif (profile="dns") [
        | eval index="dns", sourcetype="dns_logs"
    ]
    else [
        | eval index="default", sourcetype="generic"
    ]
    | into $destination;
```

---

## 6.12 `dedup`: Deduplication

```spl2
# Dedupe on event ID (good)
| dedup event_id

# Dedupe on host (probably bad - loses events)
| dedup host

# Dedupe on composite key
| dedup host, user, EventCode
```

**Pipeline dedupe**: Removes events from pipeline → saves downstream processing.

---

## 6.13 `stats`: Aggregation in Pipelines

```spl2
# Reduce volume: per-host byte summary every 5 min
| stats sum(bytes) AS total_bytes BY host span=5m
| into $metrics_destination;
```

**Use case**: Send metrics instead of raw events to reduce license.

---

## 6.14 `lookup`: Enrichment

```spl2
$pipeline = | from $source
    | rex field=_raw /"src_ip":"(?P<src_ip>[^"]+)"/
    | lookup ip_assets ip AS src_ip OUTPUT asset_type, owner, criticality
    | route criticality="high" [ | into $high_priority_dest ]
    | into $default_dest;
```

**Requirement**: Lookup must be configured in Edge Processor.

---

## 6.15 `decrypt` / `ocsf`

```spl2
# Decrypt encrypted field
| decrypt field=encrypted_payload algorithm=AES256 key=$key$

# Normalize to OCSF
| ocsf
```

---

## 6.16 `expand` / `flatten` / `mvexpand`

```spl2
# JSON array → multiple events
| expand events_array

# Nested object → flat fields
| flatten user_object

# Multivalue field → multiple events
| mvexpand tags
```

---

## 6.17 Complete Pipeline Design Patterns

### Pattern 1: Filter → Mask → Route
```spl2
import route from /splunk/ingest/commands

$pipeline = | from $source
    # 1. Drop noise early
    | where NOT match(_raw, /"action":"debug"/i)
    
    # 2. Extract routing field
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    
    # 3. Global masking
    | eval _raw=replace(_raw, /"password":"[^"]*"/, "\"password\":\"xxxx\"")
    | eval _raw=replace(_raw, /CreditCard=\d+/, "CreditCard=XXXX")
    
    # 4. Route by profile
    | route profile="linux" [ | eval index="linux" | into $linux_dest ]
    | route profile="dns" [ | eval index="dns" | into $dns_dest ]
    | route profile="firewall" [ | eval index="firewall" | into $fw_dest ]
    
    # 5. Default
    | into $default_dest;
```

### Pattern 2: Archive Raw → Process → Route
```spl2
import route from /splunk/ingest/commands

$pipeline = | from $source
    # Archive RAW (before masking)
    | thru [ | into $raw_archive_dest ]
    
    # Process
    | eval _raw=replace(_raw, /password=\w+/, "password=****")
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    
    # Route
    | route profile="linux" [ | into $linux_dest ]
    | into $default_dest;
```

### Pattern 3: Branch for Multiple Teams
```spl2
import route from /splunk/ingest/commands

$pipeline = | from $source
    | eval _raw=replace(_raw, /password=\w+/, "password=****")
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    
    | branch
        # Security team: full routing
        [
            | route profile="linux" [ | into $sec_linux ]
            | route profile="firewall" [ | into $sec_fw ]
            | into $sec_default
        ],
        # Infra team: just Linux/DNS
        [
            | route profile="linux" [ | into $infra_linux ]
            | route profile="dns" [ | into $infra_dns ]
            | into $infra_default
        ],
        # Compliance: raw archive
        [
            | into $compliance_archive
        ];
```

---

## 6.18 Partition vs `where`: Critical Distinction

```
┌────────────────────────────────────────────────────────────────┐
│ PARTITION (Builder UI)                                         │
│ ━━━━━━━━━━━━━━━━━━━━                                           │
│ Question: "What data enters THIS pipeline?"                    │
│                                                                 │
│ Example: sourcetype=cisco:asa                                   │
│                                                                 │
│ Effect: Data NOT matching partition NEVER reaches pipeline.    │
│         Other pipelines (or default) handle it.                │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│ WHERE (Pipeline SPL2)                                          │
│ ━━━━━━━━━━━━━━━━                                               │
│ Question: "Which events INSIDE THIS PIPELINE continue?"        │
│                                                                 │
│ Example: | where action != "debug"                             │
│                                                                 │
│ Effect: Events dropped here are GONE (not indexed, not routed) │
└────────────────────────────────────────────────────────────────┘
```

**Rule**: Put broad filters in Partition. Put fine-grained filters in `where`.

---

## 6.19 Pipeline Debugging & Testing

### Sample Data in Builder
1. Paste sample events in "Sample Data" tab
2. Run pipeline
3. Inspect output at each stage (click command)

### Common Debug Patterns
```spl2
# Debug: see what fields exist
| fields *

# Debug: count events at this stage
| stats count

# Debug: sample first 10
| head 10
```

### Testing Partition
- If events not appearing: check partition config
- If wrong events appearing: partition too broad

---

## 6.20 Custom Commands & Python SDK

> **Goal**: Extend SPL with custom logic when built-in commands aren't enough.

---

## 6.21 Custom Command Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CUSTOM COMMAND TYPES                          │
├──────────────────┬────────────────┬───────────────┬─────────────┤
│                  │ GENERATING     │ STREAMING     │ REPORTING   │
├──────────────────┼────────────────┼───────────────┼─────────────┤
│ Input            │ None           │ Events        │ Events      │
│ Output           │ Events         │ Events        │ Summary     │
│ Runs on          │ Search Head    │ Indexers*     │ Search Head │
│ Memory model     │ O(1)           │ O(1)/event    │ O(groups)   │
│ Use cases        │ Data connectors│ Enrich, parse │ Custom agg, │
│                  │ Synthetic data │ Filter, mask  │ ML, stats   │
└──────────────────┴────────────────┴───────────────┴─────────────┘
* If distributed=true in commands.conf
```

---

## 6.22 Generating Commands (No Input)

**Purpose**: Create events from external sources.

```python
# generating_command.py
from splunklib.searchcommands import GeneratingCommand, Configuration, Option, validators

@Configuration(type='reporting')
class MyDataSourceCommand(GeneratingCommand):
    api_url = Option(require=True)
    limit = Option(default=100, validate=validators.Integer())
    
    def generate(self):
        import requests
        response = requests.get(self.api_url)
        for item in response.json()[:self.limit]:
            yield {
                '_time': time.time(),
                '_raw': json.dumps(item),
                **item
            }
```

**commands.conf**:
```ini
[my_datasource]
type = generating
python.version = python3
filename = generating_command.py
```

**Usage**:
```spl
| my_datasource api_url="https://api.example.com/alerts" limit=50
```

---

## 6.23 Streaming Commands (Event → Event)

**Purpose**: Transform each event independently. Can run on indexers.

```python
# streaming_command.py
from splunklib.searchcommands import StreamingCommand, Configuration, Option, validators

@Configuration(distributed=True, streaming=True)
class EnrichIPCommand(StreamingCommand):
    lookup_file = Option(require=True)
    
    def __init__(self):
        super().__init__()
        self.lookup = {}
    
    def prepare(self):
        # Load lookup once
        with open(self.lookup_file) as f:
            for line in f:
                ip, owner = line.strip().split(',')
                self.lookup[ip] = owner
    
    def stream(self, records):
        for record in records:
            ip = record.get('src_ip')
            if ip in self.lookup:
                record['ip_owner'] = self.lookup[ip]
            yield record
```

**commands.conf**:
```ini
[enrich_ip]
type = streaming
distributed = true
local = false
streaming = true
python.version = python3
filename = streaming_command.py
```

**Usage**:
```spl
index=firewall | enrich_ip lookup_file="/opt/splunk/etc/apps/myapp/lookups/ip_owners.csv"
```

---

## 6.24 Reporting Commands (Events → Summary)

**Purpose**: Aggregate events into custom summaries. Always on search head.

```python
# reporting_command.py
from splunklib.searchcommands import ReportingCommand, Configuration, Option, validators

@Configuration(type='reporting')
class SessionizeCommand(ReportingCommand):
    timeout = Option(default=1800, validate=validators.Integer())
    
    def map(self, records):
        # Process on indexers (if distributed)
        for record in records:
            yield record
    
    def reduce(self, records):
        # Aggregate on search head
        sessions = []
        current_session = []
        last_time = None
        
        for record in sorted(records, key=lambda r: r['_time']):
            if last_time and (record['_time'] - last_time) > self.timeout:
                sessions.append(current_session)
                current_session = []
            current_session.append(record)
            last_time = record['_time']
        
        if current_session:
            sessions.append(current_session)
        
        for i, session in enumerate(sessions):
            yield {
                'session_id': i,
                'start': session[0]['_time'],
                'end': session[-1]['_time'],
                'duration': session[-1]['_time'] - session[0]['_time'],
                'event_count': len(session),
                'user': session[0].get('user')
            }
```

---

## 6.25 Distributable vs Centralized

### Configuration (`commands.conf`)

```ini
# DISTRIBUTED STREAMING (runs on indexers)
[my_streaming_cmd]
type = streaming
distributed = true
local = false
streaming = true
supports_multivalues = true
required_fields = src_ip,dest_ip

# CENTRALIZED STREAMING (search head only)
[my_centralized_cmd]
type = streaming
distributed = false
local = true
streaming = true

# REPORTING (always centralized)
[my_reporting_cmd]
type = reporting
distributed = false
local = true
```

### When to Use Which

| Scenario | Type |
|----------|------|
| Per-event transform, no global state | Distributed Streaming |
| Need global seen-set (dedup) | Centralized Streaming |
| Custom aggregation | Reporting |
| Data source connector | Generating |

---

## 6.26 Python SDK: `splunk-sdk` vs `splunk-internal`

| SDK | Purpose | Install |
|-----|---------|---------|
| `splunk-sdk` | External apps, REST API, search jobs | `pip install splunk-sdk` |
| `splunklib.searchcommands` | Custom search commands (internal) | Built into Splunk |

**For custom commands**: Use `splunklib.searchcommands` (bundled with Splunk).

---

## 6.27 Intersplunk vs Streaming Command Protocol

### Legacy: Intersplunk (Deprecated)
```python
import splunk.Intersplunk

results, dummy, settings = splunk.Intersplunk.getOrganizedResults()
for result in results:
    result['new_field'] = 'value'
splunk.Intersplunk.outputResults(results)
```

### Modern: Streaming Command Protocol (Use This)
```python
from splunklib.searchcommands import StreamingCommand, Configuration

@Configuration()
class MyCommand(StreamingCommand):
    def stream(self, records):
        for record in records:
            record['new_field'] = 'value'
            yield record
```

**Benefits**: Chunked processing, distributed execution, better memory, type hints.

---

## 6.28 Checkpointing & State

For commands that need persistent state across runs (e.g., incremental API pulls):

```python
import os
import json
from splunklib.searchcommands import GeneratingCommand, Configuration, Option

@Configuration(type='generating')
class IncrementalAPICommand(GeneratingCommand):
    checkpoint_dir = Option(default='/opt/splunk/var/lib/splunk/modinputs/checkpoints')
    
    def get_checkpoint(self, name):
        path = os.path.join(self.checkpoint_dir, f"{name}.json")
        if os.path.exists(path):
            with open(path) as f:
                return json.load(f)
        return {}
    
    def save_checkpoint(self, name, data):
        path = os.path.join(self.checkpoint_dir, f"{name}.json")
        with open(path, 'w') as f:
            json.dump(data, f)
    
    def generate(self):
        checkpoint = self.get_checkpoint('my_api')
        last_id = checkpoint.get('last_id', 0)
        
        # Fetch new data
        new_data = fetch_from_api(since_id=last_id)
        
        if new_data:
            max_id = max(item['id'] for item in new_data)
            self.save_checkpoint('my_api', {'last_id': max_id})
            
            for item in new_data:
                yield item
```

---

## 6.29 Configuration: `commands.conf` Reference

```ini
[my_command]
# Required
type = streaming | reporting | generating
python.version = python3
filename = my_command.py

# Streaming options
distributed = true | false
local = true | false
streaming = true | false
supports_multivalues = true | false
required_fields = field1,field2
maxinputs = 50000

# Reporting options
enableheader = true | false
retainsevents = true | false
generating = true | false

# Generating options
generating = true
```

---

## 6.30 Testing & Debugging Custom Commands

### Unit Test Pattern
```python
# test_my_command.py
import unittest
from my_command import MyCommand

class TestMyCommand(unittest.TestCase):
    def test_streaming(self):
        cmd = MyCommand()
        records = [{'src_ip': '1.2.3.4', 'bytes': 100}]
        results = list(cmd.stream(records))
        self.assertIn('new_field', results[0])
```

### Debug in Splunk
```spl
# See command output
| my_command | head 5

# Debug: see all fields
| my_command | fields *

# Check errors
index=_internal sourcetype=splunkd component=ScriptRunner
```

### Common Issues
| Issue | Cause | Fix |
|-------|-------|-----|
| Command not found | `commands.conf` not deployed | Deploy to all search heads/indexers |
| `distributed=true` but runs on SH | Missing `streaming=true` | Add `streaming=true` |
| OOM on reporting | High cardinality | Add `maxinputs` or pre-aggregate |
| Checkpoint not persisting | Wrong path | Use `$SPLUNK_HOME/var/lib/splunk/modinputs/checkpoints` |

---

## 6.31 What's Next

- **Chapter 7**: Performance, Reference & Cheat Sheets