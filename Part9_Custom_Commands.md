# Part 9 — Custom Commands & Python SDK

> **Goal**: Extend SPL with custom logic when built-in commands aren't enough.

---

## 9.1 Custom Command Architecture

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

## 9.2 Generating Commands (No Input)

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

## 9.3 Streaming Commands (Event → Event)

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

## 9.4 Reporting Commands (Events → Summary)

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

## 9.5 Distributable vs Centralized

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

## 9.6 Python SDK: `splunk-sdk` vs `splunk-internal`

| SDK | Purpose | Install |
|-----|---------|---------|
| `splunk-sdk` | External apps, REST API, search jobs | `pip install splunk-sdk` |
| `splunklib.searchcommands` | Custom search commands (internal) | Built into Splunk |

**For custom commands**: Use `splunklib.searchcommands` (bundled with Splunk).

---

## 9.7 Intersplunk vs Streaming Command Protocol

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

## 9.8 Checkpointing & State

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

## 9.9 Configuration: `commands.conf` Reference

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

## 9.10 Testing & Debugging Custom Commands

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

## 9.11 What's Next

- [Part 10: Performance](Part10_Performance.md) — `tstats`, `tsidx`, optimization
- [Part 11: SOC Patterns](Part11_SOC_Patterns.md) — Detection engineering
- [Part 12: Reference](Part12_Reference.md) — Cheat sheets