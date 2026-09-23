# Splunk SPL + SPL2

## Chapter 1 – Understanding Splunk Events, Fields, and Data

---

## Introduction

Before learning how to write **SPL** or **SPL2** searches, you need to understand **what you are searching**.

Many beginners immediately start writing searches such as:

### Traditional SPL

```spl
index=windows EventCode=4625
| stats count by user
```

### SPL2

```spl2
FROM windows
WHERE EventCode=4625
```

They learn the syntax, but they do not understand where `index`, `EventCode`, `user`, `_time`, or `_raw` came from.

That becomes a problem later when you start working with:

- field extraction
- index-time processing
- search-time processing
- filtering
- routing
- masking
- `props.conf`
- `transforms.conf`
- Heavy Forwarders
- Ingest Processor
- Edge Processor
- SPL pipelines
- SPL2 pipelines

So before learning advanced search commands, we need to understand the **data itself**.

Think of Splunk as a huge security-log system.

Instead of books, Splunk stores **events**.

Instead of columns in a spreadsheet, events contain **fields**.

Instead of searching storage blindly, you normally start with a **dataset** [dataset: a collection of data that can be searched], such as an index.

Both SPL and SPL2 ultimately work with the same underlying event data; what changes is the language and the context in which you use it. Splunk's current documentation also notes that SPL2 supports SPL-style syntax as well as SQL-style syntax. citeturn891251search4turn891251search3

Once you understand these ideas, learning both SPL and SPL2 becomes much easier.

---

## What Is an Event?

An **event** is a record representing something that happened.

For example:

- A user logs in.
- A process starts.
- A firewall blocks a connection.
- A DNS request occurs.
- A Windows service starts.
- An SSH login fails.

Each of these activities can become an event in Splunk.

For example, imagine Windows generates the following security log:

```text
Sep 23 10:15:22
Computer=DC01
User=administrator
SourceIP=10.10.10.50
EventCode=4625
LogonType=3
Status=0xC000006D
```

Splunk stores that information as an event.

Conceptually, you can think of it as:

```text
+----------------------------------------------------+
| One Splunk Event                                   |
+----------------------------------------------------+
| Time        = Sep 23 10:15:22                     |
| Host        = DC01                                 |
| User        = administrator                        |
| SourceIP    = 10.10.10.50                          |
| EventCode   = 4625                                 |
| LogonType   = 3                                    |
| Status      = 0xC000006D                           |
+----------------------------------------------------+
```

One event can contain many fields.

A single host can generate thousands or millions of events.

For example:

```text
DC01
 ├── Login event
 ├── Process event
 ├── Network event
 ├── PowerShell event
 ├── File event
 ├── Authentication event
 └── ... thousands more
```

Splunk's search system retrieves these events and processes their fields.

---

## Where Do Splunk Events Come From?

Splunk does not normally create the original security activity.

Instead, it receives data from external sources and indexes that data.

Examples include:

- Windows Event Logs
- Linux logs
- Firewall logs
- DNS logs
- Web server logs
- Sysmon
- Zeek
- Network appliances
- Applications
- Cloud services
- Security products
- APIs
- HTTP Event Collector

For example:

```text
Windows
   │
   ▼
Windows Event Log
   │
   ▼
Universal Forwarder
   │
   ▼
Heavy Forwarder / Indexer
   │
   ▼
Splunk Index
```

The exact architecture [architecture: the design of how system components are connected] can be different, but the important idea is:

```text
Data source
     ↓
Splunk data ingestion
     ↓
Parsing / processing
     ↓
Indexing
     ↓
Search
```

Splunk performs different kinds of processing during **index time** and **search time** [index time: processing performed while data is being prepared for indexing; search time: processing performed when a search runs].

This distinction will become very important when we study `props.conf`, `transforms.conf`, field extraction, routing, filtering, and masking.

---

## What Is an Index?

One of the first concepts you need to understand in Splunk is the **index**.

An index is where Splunk stores indexed event data.

For example:

```text
windows
linux
firewall
zeek
web
security
main
```

could be indexes in your environment.

Think of them like separate collections of events:

```text
                    Splunk
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     windows         linux         firewall
        │              │              │
     events          events         events
```

### Index in traditional SPL

```spl
index=windows
```

The `search` command is often implied when a traditional SPL search begins with an index expression in the Search bar.

Explicitly, you can also write:

```spl
search index=windows
```

### Index in SPL2

SPL2 can use the `search` command:

```spl2
search index=windows
```

Or the `FROM` clause:

```spl2
FROM windows
```

Splunk documents `FROM` as a generating command that retrieves data from a dataset, and an index is a kind of dataset. SPL2 also lets you use `SELECT * FROM windows`. citeturn753596search0turn891251search3

So the important idea is:

```text
index
  ↓
A place where indexed event data is stored
```

And in SPL2:

```text
FROM windows
      ↓
Retrieve data from the windows dataset
```

---

## SPL and SPL2: What Changes and What Does Not?

Before going further, keep this distinction clear.

### The data does not suddenly become different

The same Splunk event can be searched using traditional SPL or SPL2.

For example, suppose an event contains:

```text
EventCode=4625
user=administrator
host=DC01
```

The event itself does not change merely because you use a different search language.

### The search syntax can change

For the same basic filtering idea:

**SPL**

```spl
index=windows EventCode=4625
```

**SPL2 with the search command**

```spl2
search index=windows EventCode=4625
```

**SPL2 with FROM/WHERE**

```spl2
FROM windows
WHERE EventCode=4625
```

Splunk's current documentation explicitly shows that SPL2 has both the `search` command and the `FROM` clause, and that `FROM` can retrieve data from indexes and other datasets. citeturn891251search0turn753596search0

A useful learning rule is:

```text
SPL
 ↓
Learn the traditional search commands and pipeline

SPL2
 ↓
Learn the SPL-compatible form and the newer FROM / SELECT style

Both
 ↓
Understand what happens to the event and its fields
```

We will compare them throughout this course instead of treating them as two unrelated subjects.

---

## What Does an Event Look Like?

Suppose Splunk receives this raw log:

```text
2026-09-23 10:15:22 host=DC01 user=administrator
src_ip=10.10.10.50 EventCode=4625 LogonType=3
```

The event can expose fields such as:

| Field | Value |
|---|---|
| `_time` | 2026-09-23 10:15:22 |
| `host` | DC01 |
| `user` | administrator |
| `src_ip` | 10.10.10.50 |
| `EventCode` | 4625 |
| `LogonType` | 3 |
| `_raw` | Original event text |

This is an important point:

**A Splunk event is not simply a row containing only user-created fields.**

Splunk also has fields that describe the event itself.

That is where the field categories become important.

---

# Understanding Splunk Fields

Splunk does not use exactly the same field-prefix system as CrowdStrike LogScale.

You will commonly encounter:

- Internal fields
- Default fields
- Other extracted fields
- Indexed fields
- Search-time fields

These categories can overlap.

For example, `host` is a default field and is also indexed.

A field such as `user` may be extracted from `_raw` at search time, depending on the data and configuration.

So do not think:

```text
"Every field belongs to exactly one category."
```

Instead, think about two different questions:

```text
Question 1:
What kind of field is this?

Question 2:
How and when does Splunk make this field available?
```

That distinction will save you a lot of confusion later.

---

# 1. The `_raw` Field

One of the most important Splunk fields is:

```text
_raw
```

`_raw` contains the original raw data of the event.

For example:

```text
2026-09-23 10:15:22 host=DC01 user=administrator
src_ip=10.10.10.50 EventCode=4625
```

Conceptually:

```text
_raw
 │
 └── Original event data
```

Many fields can be extracted from this data.

For example:

```text
_raw
 │
 ├── host=DC01
 ├── user=administrator
 ├── src_ip=10.10.10.50
 └── EventCode=4625
```

Splunk's official documentation defines `_raw` as the original raw data of an event and notes that the `search` command uses `_raw` when performing searches and data extraction. citeturn891251search1

You should therefore remember:

```text
_raw = the raw event data
```

This field becomes especially important later when we learn **masking**.

For example, suppose the raw event contains:

```text
username=john password=Secret123
```

A processing rule can modify the raw content before the data reaches its destination, depending on where the rule is applied.

---

# 2. The `_time` Field

Another extremely important field is:

```text
_time
```

`_time` represents the event timestamp.

For example:

```text
_time = 2026-09-23 10:15:22
```

Splunk uses `_time` for the event timeline and for time-based searching. The field is stored internally in UTC and displayed in a human-readable form in search results. citeturn891251search1

You will use `_time` constantly.

For traditional SPL:

```spl
index=windows earliest=-1h
```

You can also explicitly work with `_time` in a command or expression.

For SPL2, time modifiers can be used in the search, for example:

```spl2
FROM windows
WHERE earliest=-1h
```

Or you can use `_time` in expressions where appropriate.

Think:

```text
_time
  │
  └── When did this event happen?
```

---

# 3. The `index` Field

The:

```text
index
```

field identifies the Splunk index associated with the event.

For example:

```text
index=windows
```

or:

```text
index=firewall
```

Conceptually:

```text
index
  │
  └── Which index contains this event?
```

For traditional SPL:

```spl
index=windows
```

For SPL2 using `search`:

```spl2
search index=windows
```

For SPL2 using `FROM`:

```spl2
FROM windows
```

The last form is a major syntax difference you need to recognize immediately.

---

# 4. The `host` Field

The:

```text
host
```

field identifies the host associated with the event.

Example:

```text
host=DC01
```

Another event might have:

```text
host=WEB01
```

Conceptually:

```text
host
  │
  └── Which host is associated with this event?
```

Traditional SPL:

```spl
index=windows host=DC01
```

SPL2 search syntax:

```spl2
search index=windows host=DC01
```

SPL2 FROM syntax:

```spl2
FROM windows
WHERE host="DC01"
```

Splunk lists `host` among its default fields. citeturn891251search1

---

# 5. The `source` Field

The:

```text
source
```

field tells you the source associated with the event.

For file-based data, this can commonly correspond to the path or file from which Splunk read the data.

For example:

```text
source=/var/log/auth.log
```

or:

```text
source=C:\Windows\System32\winevt\Logs\Security.evtx
```

Think:

```text
source
  │
  └── What source provided this event?
```

This is different from `host`.

For example:

```text
host=WEB01
source=/var/log/nginx/access.log
```

means:

```text
HOST
  ↓
WEB01

SOURCE
  ↓
/var/log/nginx/access.log
```

Traditional SPL:

```spl
index=web source="/var/log/nginx/access.log"
```

SPL2:

```spl2
FROM web
WHERE source="/var/log/nginx/access.log"
```

`source` is also one of Splunk's default fields. citeturn891251search1

---

# 6. The `sourcetype` Field

The:

```text
sourcetype
```

field identifies the type or format of the incoming data.

Examples:

```text
sourcetype=access_combined
sourcetype=linux_secure
sourcetype=WinEventLog:Security
```

Think:

```text
host
   ↓
Which host?

source
   ↓
Which source?

sourcetype
   ↓
What type of data is this?
```

For example:

```text
host=DC01
source=Security.evtx
sourcetype=WinEventLog:Security
```

This combination tells you much more than any single field.

Traditional SPL:

```spl
index=windows sourcetype=WinEventLog:Security
```

SPL2:

```spl2
FROM windows
WHERE sourcetype="WinEventLog:Security"
```

Splunk lists `sourcetype` as a default field and documents it as the field that specifies the format of the input data. citeturn891251search1

---

# The Four Fields You Must Not Confuse

These four fields are so important that you should become comfortable with them before moving further:

```text
index
host
source
sourcetype
```

Think of them like this:

```text
                         EVENT
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
       ▼                   ▼                   ▼
     index               host                source
       │                   │                   │
 Which index?         Which host?        Which source?
                           │
                           ▼
                      sourcetype
                           │
                      What data type?
```

For example:

```text
index=windows
host=DC01
source=Security.evtx
sourcetype=WinEventLog:Security
```

### The same idea in searches

Traditional SPL:

```spl
index=windows host=DC01 sourcetype=WinEventLog:Security
```

SPL2 using `search`:

```spl2
search index=windows host=DC01 sourcetype=WinEventLog:Security
```

SPL2 using `FROM` and `WHERE`:

```spl2
FROM windows
WHERE host="DC01"
  AND sourcetype="WinEventLog:Security"
```

These fields are useful for narrowing searches, and Splunk documents `index`, `source`, and `sourcetype` as default fields that can be used to scope searches. citeturn891251search1turn891251search3

---

# 7. Other Internal Fields

Splunk has several other internal fields.

Examples include:

```text
_raw
_time
_indextime
_cd
_bkt
```

Fields beginning with `_` are internal fields. Splunk's documentation lists `_raw`, `_time`, `_indextime`, `_cd`, and `_bkt` as internal fields. citeturn891251search1

You will not normally need all of them every day.

Two that are especially important are:

```text
_time
_raw
```

Another useful field is:

```text
_indextime
```

`_indextime` tells you when Splunk indexed the event.

That means you can compare:

```text
_time
```

with:

```text
_indextime
```

to investigate ingestion delay [ingestion delay: the time between an event being generated and Splunk indexing it].

Traditional SPL example:

```spl
| eval delay_sec=_indextime-_time
```

The underlying idea is:

```text
_event happened
      │
      │  _time
      ▼
  Splunk receives it
      │
      ▼
  Splunk indexes it
      │
      │  _indextime
      ▼
    Search
```

This is very useful when troubleshooting data latency.

---

# 8. Event Fields

Now we reach the fields you will probably use most during security investigations.

These fields are usually extracted from the event data.

Examples include:

```text
user
src
src_ip
dest
dest_ip
src_port
dest_port
action
status
EventCode
LogonType
process
parent_process
CommandLine
file_name
hash
```

For example:

```text
EventCode=4625
LogonType=3
user=administrator
src_ip=10.10.10.50
```

These fields describe what actually happened.

Unlike `host`, `source`, and `sourcetype`, their names depend heavily on the data source and on the extraction or normalization [normalization: converting different data into a consistent field naming and format] used in your environment.

A Windows event may contain:

```text
EventCode
LogonType
TargetUserName
IpAddress
```

A firewall event might contain:

```text
src_ip
dest_ip
src_port
dest_port
action
```

A Zeek event might contain:

```text
id.orig_h
id.resp_h
id.orig_p
id.resp_p
proto
service
```

So you should **never assume that every Splunk event has the same security fields**.

The field names depend on how the data was generated, onboarded, extracted, and possibly normalized.

---

# 9. Indexed Fields vs Search-Time Fields

This is one of the most important concepts for our future chapters.

Suppose the event contains:

```text
user=administrator
src_ip=10.10.10.50
EventCode=4625
```

There are different ways Splunk can know about those values.

## Index Time

Some fields are identified and indexed while the event is being indexed.

Splunk automatically indexes default fields such as:

```text
host
source
sourcetype
timestamp
```

Custom fields can also be configured for index-time extraction, although Splunk generally recommends search-time extraction unless there is a specific reason to index the field. citeturn891251search1

Conceptually:

```text
Incoming event
      │
      ▼
Index-time processing
      │
      ├── host
      ├── source
      ├── sourcetype
      └── selected indexed fields
      │
      ▼
     Index
```

## Search Time

Other fields are extracted when a search actually runs.

For example:

```text
_raw

"user=administrator src_ip=10.10.10.50 EventCode=4625"
```

Splunk can extract:

```text
user
src_ip
EventCode
```

at search time.

Conceptually:

```text
Indexed event
      │
      ▼
Search starts
      │
      ▼
Field extraction
      │
      ├── user
      ├── src_ip
      └── EventCode
```

Splunk documents search-time processing as including operations such as search-time field extraction, field aliases, lookups, event types, and tagging. citeturn891251search1

---

# Why Does This Matter?

Because later we will ask:

> Why can I search this field?

> Why is this field available only after extraction?

> Why would I create an indexed field?

> Why would I use `props.conf` and `transforms.conf`?

> Why would I use `SEDCMD` or `INGEST_EVAL`?

> Why would I use Ingest Processor or Edge Processor?

These questions make much more sense once you understand:

```text
             Raw event
                 │
                 ▼
          Index-time processing
                 │
                 ▼
               Index
                 │
                 ▼
          Search-time processing
                 │
                 ▼
             Search result
```

Splunk's documentation recommends search-time extraction in many cases because custom index-time extraction can add indexing work and increase index size. citeturn891251search1

---

# 10. The Difference Between a Field and an Indexed Field

This distinction is especially important because you have already been learning about `tsidx`.

Suppose you have:

```text
user=administrator
```

You might ask:

> Is `user` a field?

Yes.

Then:

> Is `user` indexed?

That is a separate question.

A field can exist as a search-time field without being a custom indexed field.

So do not think:

```text
field = indexed field
```

Instead:

```text
Field
 │
 ├── may be indexed
 │
 └── may be extracted at search time
```

Splunk provides tools such as `tstats` and `walklex` that can help you investigate indexed fields and index structures.

Later we will connect this directly to your work with `.tsidx` files and `walklex`.

---

# 11. A Simple Event Model

At this point, think of one Splunk event like this:

```text
                         EVENT
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
       ▼                   ▼                   ▼
   Internal             Default              Other
    fields              fields              fields
       │                   │                   │
       │                   │                   │
      _raw                index                user
      _time               host                 src_ip
      _indextime           source               EventCode
                          sourcetype            action
```

This is a learning model, not a statement that these are three separate physical storage areas.

The important thing is to ask:

```text
What is the field?
How did Splunk get it?
When is it available?
Is it indexed?
How can I use it in a search?
```

---

# 12. The Fields You Should Learn First

Before moving into advanced SPL and SPL2, become comfortable with these:

| Field | Question it answers |
|---|---|
| `index` | Which index contains the event? |
| `host` | Which host is associated with it? |
| `source` | What source provided it? |
| `sourcetype` | What type of input data is it? |
| `_time` | When did the event happen? |
| `_indextime` | When did Splunk index it? |
| `_raw` | What is the raw event data? |

Then learn common security fields:

| Field | Typical meaning |
|---|---|
| `user` | Account or user |
| `src` / `src_ip` | Source system or source IP |
| `dest` / `dest_ip` | Destination system or destination IP |
| `src_port` | Source port |
| `dest_port` | Destination port |
| `action` | Action that occurred |
| `status` | Result or status |
| `process` | Process involved |
| `CommandLine` | Command that was executed |
| `EventCode` | Windows event identifier |
| `LogonType` | Windows logon category |

These are **examples**, not universal Splunk fields. The actual field names depend on the data source, add-on, extraction rules, and normalization used in your environment.

---

# 13. Searching These Fields: SPL and SPL2 Together

Now we can start connecting the field vocabulary to the two languages.

Suppose we want Windows events.

### Traditional SPL

```spl
index=windows
```

### SPL2 using `search`

```spl2
search index=windows
```

### SPL2 using `FROM`

```spl2
FROM windows
```

The SPL2 `from` command retrieves data from a dataset; an index is one kind of dataset. citeturn753596search0

---

## Filter by host

### SPL

```spl
index=windows host=DC01
```

### SPL2 search syntax

```spl2
search index=windows host=DC01
```

### SPL2 FROM syntax

```spl2
FROM windows
WHERE host="DC01"
```

---

## Filter by sourcetype

### SPL

```spl
index=windows sourcetype=WinEventLog:Security
```

### SPL2

```spl2
FROM windows
WHERE sourcetype="WinEventLog:Security"
```

---

## Filter by EventCode

### SPL

```spl
index=windows EventCode=4625
```

### SPL2

```spl2
FROM windows
WHERE EventCode=4625
```

---

## Filter by EventCode and LogonType

### SPL

```spl
index=windows EventCode=4625 LogonType=3
```

### SPL2

```spl2
FROM windows
WHERE EventCode=4625
  AND LogonType=3
```

Now the query is no longer just syntax.

You can read it conceptually:

```text
index=windows
      ↓
Which data?

EventCode=4625
      ↓
Which events?

LogonType=3
      ↓
What additional condition?
```

That way of thinking is more important than memorizing punctuation.

---

# 14. Search Syntax You Will Repeatedly Use

Before moving into advanced commands, learn the basic search vocabulary that appears again and again.

## Exact field-value search

SPL:

```spl
index=windows user=administrator
```

SPL2:

```spl2
FROM windows
WHERE user="administrator"
```

## Multiple conditions

SPL:

```spl
index=windows user=administrator EventCode=4625
```

SPL2:

```spl2
FROM windows
WHERE user="administrator"
  AND EventCode=4625
```

## OR

SPL:

```spl
index=windows EventCode=4625 OR EventCode=4624
```

SPL2 search style:

```spl2
search index=windows EventCode=4625 OR EventCode=4624
```

SPL2 `WHERE` style:

```spl2
FROM windows
WHERE EventCode=4625 OR EventCode=4624
```

## NOT

SPL:

```spl
index=windows NOT user=administrator
```

SPL2:

```spl2
FROM windows
WHERE NOT user="administrator"
```

Splunk documents `AND`, `OR`, and `NOT` as supported logical operators in the SPL2 `search` command. It also documents an important difference between `NOT field=value` and `field!=value`, so we will study that carefully rather than treating them as identical. citeturn891251search0turn891251search7

---

# 15. Why `_raw` Will Become Important Later

Suppose we receive:

```text
user=john password=Secret123 src_ip=10.10.10.5
```

The raw event is:

```text
_raw =
user=john password=Secret123 src_ip=10.10.10.5
```

From it, Splunk might expose:

```text
user=john
password=Secret123
src_ip=10.10.10.5
```

Later, we may want to:

```text
FILTER
```

the event.

Or:

```text
ROUTE
```

it to another destination.

Or:

```text
MASK
```

the password.

These are three different operations.

```text
FILTER
      ↓
Should the data continue?

ROUTE
      ↓
Where should the data go?

MASK
      ↓
Should sensitive content be changed?
```

This distinction is one of the major subjects of the chapters that follow.

---

# 16. Filtering: Search-Time vs Ingestion-Time

This is where we connect your SPL/SPL2 learning with your earlier work on Splunk data processing.

## Search-time filtering

Suppose the event is already indexed and you only want to see failed logons.

### SPL

```spl
index=windows EventCode=4625
```

### SPL2

```spl2
FROM windows
WHERE EventCode=4625
```

This does **not** remove those events from the index.

It only filters the events returned by that search.

Think:

```text
Indexed events
      │
      ▼
    Search
      │
      ▼
    FILTER
      │
      ▼
 Search results
```

That is very different from filtering data before it is indexed.

---

## Ingestion-time filtering

Now imagine your company sends a huge amount of noisy data into Splunk and decides that a certain class of events should never reach the destination.

That is an ingestion-time processing problem.

The event is handled before the final destination is written.

Conceptually:

```text
Incoming data
      │
      ▼
Processing
      │
   FILTER
      │
      ├── Keep ──► Destination
      │
      └── Drop
```

Traditional Splunk can use configuration-based mechanisms for routing and data modification.

SPL2 can also be used in data-processing products such as Ingest Processor and Edge Processor, where pipeline stages can filter and transform incoming data.

The specific configuration methods will be studied separately; do not mix **search-time filtering** with **ingestion-time filtering**.

---

# 17. Routing: A Different Problem From Filtering

Suppose you receive:

```text
Windows
Firewall
Linux
DNS
```

and want different data to go to different destinations.

That is routing.

Conceptually:

```text
                    Incoming data
                         │
                 ┌───────┼───────┐
                 ▼       ▼       ▼
              Windows Firewall   DNS
                 │       │       │
                 ▼       ▼       ▼
             Index A   Index B   Index C
```

The question is not:

> Should this event exist?

The question is:

> **Where should this event go?**

Later we will compare the major routing mechanisms, including:

```text
props.conf
transforms.conf
_TCP_ROUTING
Heavy Forwarder routing
Ingest Processor routing
Edge Processor routing
```

SPL and SPL2 searches help you identify the data, but ingestion routing is a separate processing problem.

---

# 18. Masking: A Third Different Problem

Suppose the incoming event is:

```text
user=john
password=Secret123
card=4111111111111111
src_ip=10.10.10.5
```

You may want to keep the event while hiding sensitive values.

For example:

```text
user=john
password=********
card=********
src_ip=10.10.10.5
```

That is masking.

The event remains useful, but the sensitive values are changed or removed.

Again:

```text
FILTER
  = remove / exclude data

ROUTE
  = send data somewhere specific

MASK
  = keep the data but hide sensitive content
```

Later we will compare traditional configuration methods such as `SEDCMD` and transformation rules with newer SPL2 pipeline methods.

---

# 19. Why This Leads Directly Into SPL2 Pipelines

SPL2 is not only used for searching indexed data.

Splunk also uses SPL2 in data-processing products such as **Ingest Processor** and **Edge Processor**.

A pipeline has a data-flow model.

Conceptually:

```text
Incoming Data
      │
      ▼
     FROM
      │
      ▼
  PROCESSING
      │
 ┌────┼────┐
 ▼    ▼    ▼
FILTER MASK ROUTE
      │
      ▼
 DESTINATION
```

For searches, SPL2 `FROM` retrieves data from a dataset.

For pipelines, `FROM $source` selects the incoming data from the pipeline's internal source. Splunk documents the syntax difference between searches and pipelines. citeturn753596search0turn753596search1

A pipeline can then transform or route the data before it reaches its destination.

Splunk documents filtering with `where`, masking with functions such as `replace()`/`rex`, and routing through pipeline routing commands in the relevant Ingest Processor documentation.

This is why the fields come first.

You cannot confidently write a filter until you know **which field identifies the data**.

You cannot confidently write a masking rule until you know **where the sensitive value exists**.

You cannot confidently route data until you know **what condition identifies the data to be routed**.

---

# 20. The Mental Model for the Entire Course

At this point, keep two parallel views in your head.

## Search view

```text
                         SPLUNK
                            │
                            ▼
                         EVENTS
                            │
                            ▼
                          FIELDS
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
      _raw/_time      index/host/source   Event fields
                      /sourcetype         user, src_ip,
                                         EventCode, etc.
                            │
                            ▼
                     INDEX / DATASET
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
                SPL                   SPL2
                 │                     │
                 ▼                     ▼
             Search pipeline       Search pipeline
                 │                     │
                 └──────────┬──────────┘
                            ▼
                         RESULTS
```

## Data-processing view

```text
Incoming data
      │
      ▼
 Parsing / Processing
      │
 ┌────┼─────────┐
 ▼    ▼         ▼
FILTER MASK     ROUTE
 │    │         │
 └────┴─────────┘
      │
      ▼
 DESTINATION
```

The first view is mainly about **searching and analyzing** data.

The second view is about **processing data before or while it moves toward a destination**.

Both are important in a real Splunk environment.

---

# Chapter Summary

By the end of this chapter, you should understand these core ideas:

- A **Splunk event** is a record representing something that happened.
- Events come from external data sources such as Windows, Linux, firewalls, applications, and security products.
- An **index** stores indexed event data.
- In SPL2, an index is a type of **dataset**.
- `_raw` contains the raw event data.
- `_time` represents the event timestamp.
- `_indextime` represents when Splunk indexed the event.
- `index`, `host`, `source`, and `sourcetype` are important default fields that Splunk automatically associates with events.
- Other fields such as `user`, `src_ip`, `EventCode`, and `action` depend on the data source and extraction rules.
- Some fields are available through index-time processing.
- Other fields are extracted at search time.
- An **indexed field** and a **field** are not the same concept.
- Traditional SPL and SPL2 can search the same underlying event data.
- SPL2 provides the `search` command as well as `FROM`/`SELECT` syntax.
- `FROM` in SPL2 retrieves data from a dataset, while `FROM $source` is used differently inside Edge Processor/Ingest Processor pipelines.
- Search-time filtering does not remove an event from the index.
- Ingestion-time filtering happens before the event reaches its final destination.
- Routing decides where data goes.
- Masking changes sensitive content while retaining the event.
- `props.conf`, `transforms.conf`, `SEDCMD`, `_TCP_ROUTING`, Heavy Forwarder routing, Ingest Processor, and Edge Processor will be studied as different ways of solving different data-processing problems.

The core mental model to remember is:

```text
                 SPLUNK
                    │
                    ▼
                  EVENTS
                    │
                    ▼
                  FIELDS
                    │
        ┌───────────┼────────────┐
        │           │            │
        ▼           ▼            ▼
      _raw       index/host    Event fields
      _time      source/       user, src_ip,
      _indextime sourcetype    EventCode, etc.
        │           │            │
        └───────────┼────────────┘
                    ▼
             INDEX / DATASET
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
         SPL                 SPL2
          │                   │
          └─────────┬─────────┘
                    ▼
                  SEARCH
                    │
                    ▼
                 RESULTS
```

And for ingestion/data processing:

```text
Incoming Data
      │
      ▼
PROCESS
      │
 ┌────┼─────────┐
 ▼    ▼         ▼
FILTER MASK     ROUTE
      │
      ▼
DESTINATION
```

This chapter gives us the vocabulary needed to study **both SPL and SPL2**, and it sets up the next chapters on searching, filtering, field extraction, routing, masking, `props.conf`, `transforms.conf`, Heavy Forwarders, Ingest Processor, and Edge Processor.
