# SPL2 Edge Processor — Complete Data-Flow, Filtering, Dropping, Routing, Copying, Deduplication, and Optimization Guide

> **Purpose:** A systematic reference for designing SPL2 Edge Processor pipelines correctly, with concrete examples, event-flow diagrams, destination strategy, early filtering, masking, routing, `where`, `dedup`, `stats`, `if`, `route`, `thru`, and `branch`.
>
> **Documentation basis:** Current Splunk documentation checked in September 2026. The current Edge Processor pipeline syntax supports the commands and functions described in this guide; regular expressions in current pipelines use PCRE2.

---

## 1. The most important idea: think about the event flow

Do not learn Edge Processor commands as isolated syntax.

Instead, ask:

> **What do I want to happen to this event?**

There are several fundamentally different answers:

```text
I want to receive it
    -> from

I want to send it somewhere
    -> into

I want to change it
    -> eval / replace / rename / fields / rex / spath / ocsf / decrypt

I want to remove the event
    -> where

I want to remove duplicate events
    -> dedup

I want to summarize many events
    -> stats

I want to send a subset somewhere else
    -> route

I want an additional copy but want the original to continue
    -> thru

I want multiple complete processing paths
    -> branch

I want different processing depending on a condition
    -> if

I want to enrich it with external data
    -> lookup

I want to expand structured or multivalue data
    -> expand / flatten / mvexpand
```

This is the foundation for everything else.

---

# 2. The complete Edge Processor model

A practical pipeline can be thought of as:

```text
SOURCE / FORWARDER / DEVICE
          |
          v
   EDGE PROCESSOR
          |
          v
     PARTITION
          |
          v
   EARLY FILTERING
       where
          |
          v
   FIELD EXTRACTION
    rex / spath
          |
          v
   REDUCTION
 dedup / stats
          |
          v
  TRANSFORMATION
 eval / replace / rename / fields
          |
          v
   ENRICHMENT
       lookup
          |
          v
      ROUTING
       route
      /     \
     /       \
    v         v
dest-A     remaining
             |
             v
          dest-B
```

This is a **logical design model**, not a requirement that every pipeline contain every stage.

The order should be driven by what the event must do and by how much processing you can avoid.

---

# 3. Very important: an Edge Processor does NOT drop a network packet

This is where your eBPF comparison needs one correction.

### eBPF/network filtering

Conceptually:

```text
packet arrives
      |
      v
early kernel filter
      |
   unwanted?
    /    \
  YES     NO
   |       |
 DROP    continue
```

The packet can be discarded before later network processing.

### Edge Processor filtering

The event must first reach the Edge Processor:

```text
source / forwarder
       |
       | network traffic
       v
Edge Processor
       |
       v
   where filter
       |
    /       \
 unwanted   wanted
    |          |
   DROP      continue
```

`where` can prevent the event from being sent to the pipeline destination, but it cannot prevent the source or forwarder from having already transmitted the event to the Edge Processor.

Therefore:

```text
Want to reduce bytes sent TO the Edge Processor?
    -> filter earlier, at the source/agent/network layer.

Want to reduce processing and downstream data AFTER the Edge Processor receives it?
    -> use Edge Processor filtering and transformation.
```

Splunk describes Edge Processor as processing data after it is received and then sending the resulting data to destinations. It also states that Edge Processor filtering can reduce the amount of data sent downstream. See the official documentation listed in the Sources section.

---

# 4. Partition vs `where`: these are NOT the same

This is one of the most important concepts.

## 4.1 Partition

A pipeline partition defines the subset of incoming data that a pipeline is intended to process.

For example, you may configure a partition using:

```text
host
source
sourcetype
```

Conceptually:

```text
All incoming data
       |
       v
    partition
       |
   +---+---+
   |       |
selected  not selected
   |          |
   v          v
pipeline   unprocessed
```

The partition is a scope for the pipeline.

Splunk documents that data not selected by a pipeline partition is considered unprocessed. If the Edge Processor has a default destination, unprocessed data goes there; if no default destination is configured, unprocessed data is dropped.

## 4.2 `where`

`where` is an actual processing/filtering step inside the pipeline:

```spl
| where <condition>
```

Data that does not satisfy the predicate is removed from the pipeline and is not sent to the destination.

Example:

```spl
$pipeline = | from $source
    | where sourcetype == "firewall"
    | into $destination;
```

Conceptually:

```text
pipeline input
      |
      v
where sourcetype == "firewall"
      |
   +--+--+
   |     |
firewall other
   |      |
   v      v
keep    DROP
```

### Important current behavior

Current Edge Processor behavior treats `where` clauses in the pipeline body as filters. An older behavior treated certain `where` clauses immediately after `from $source` as partition conditions; Splunk changed that behavior in January 2024.

Therefore, when the goal is:

> **"Drop these events."**

Use the pipeline's filtering logic (`where`), and configure the partition separately for the broad dataset scope.

---

# 5. Best mental model for filtering

Use two levels:

```text
PARTITION
    "Which broad class of data should this pipeline process?"

WHERE
    "Which events inside that class should survive?"
```

Example:

### Requirement

Only process firewall logs from server01, but drop debug events.

Partition:

```text
host = server01
sourcetype = firewall
```

Pipeline:

```spl
$pipeline = | from $source
    | where NOT match(_raw, /debug/i)
    | into $destination;
```

Result:

```text
server01 + firewall + normal  -> keep
server01 + firewall + debug   -> DROP

other hosts / other sourcetypes
    -> outside this partition
    -> treated as unprocessed data
```

This is much cleaner than attempting to make one mechanism do both jobs.

---

# 6. `where`: the real DROP mechanism

There is no separate `drop` command required for normal Edge Processor filtering.

Use:

```spl
| where <predicate>
```

`where` returns only events for which the predicate evaluates to TRUE.

## Example 1 — drop Windows

```spl
$pipeline = | from $source
    | rex field=_raw /"profile"\s*:\s*"(?P<profile>[^"]+)"/
    | where profile != "windows"
    | into $destination;
```

Input:

```text
profile=linux
profile=dns
profile=windows
profile=firewall
```

Output:

```text
profile=linux
profile=dns
profile=firewall
```

`profile=windows` is gone.

## Example 2 — keep only security events

```spl
| where action IN ("login", "logout", "password_change")
```

Conceptually:

```text
login             -> KEEP
logout            -> KEEP
password_change   -> KEEP
debug             -> DROP
heartbeat         -> DROP
```

## Example 3 — drop debug logs

If `action` has already been extracted:

```spl
| where action != "debug"
```

If it has not been extracted and the raw data is JSON:

```spl
| where NOT match(_raw, /"action"\s*:\s*"debug"/i)
```

## Example 4 — allow-list filtering

Suppose you only want Linux, DNS, and firewall events:

```spl
| where profile IN ("linux", "dns", "firewall")
```

Everything else is dropped.

This is often much easier to reason about than writing several negative conditions.

---

# 7. When should you filter before extracting?

Suppose your event is:

```json
{"profile":"linux","action":"login","user":"teja","password":"Secret123"}
```

At the beginning you may only have:

```text
_raw
```

If your drop rule can be determined cheaply from `_raw`, you can filter without creating a temporary field.

Example:

```spl
| where NOT match(_raw, /"action"\s*:\s*"debug"/i)
```

Then extract only what survives:

```spl
| rex field=_raw /"profile"\s*:\s*"(?P<profile>[^"]+)"/
```

That is useful when:

```text
30% debug logs
70% useful logs
```

because the extraction and later transformations are not run on the dropped 30%.

But do not turn every filter into a giant regex just for performance. If a field is already available, use the field directly. If a structured field is needed later anyway, extracting it once can be clearer and cheaper than repeatedly searching `_raw`.

---

# 8. `rex`: extract fields from raw data

Suppose:

```text
_raw = {"profile":"linux","action":"login","user":"teja"}
```

Use:

```spl
| rex field=_raw /"profile":"(?P<profile>[^"]+)"/
```

The result is conceptually:

```text
_raw      = {"profile":"linux","action":"login","user":"teja"}
profile   = linux
```

Now you can write:

```spl
| where profile == "linux"
```

or:

```spl
| route profile == "linux", [
    | into $linux_destination
]
```

A key principle:

```text
RAW DATA
   |
   v
rex / spath
   |
   v
FIELD
   |
   +--> where
   +--> route
   +--> if
   +--> dedup
   +--> lookup
```

---

# 9. `spath`: prefer structured parsing when appropriate

If the event is genuinely JSON or XML, `spath` can be more appropriate than manually maintaining many regular expressions.

For example:

```json
{
  "profile": "linux",
  "action": "login",
  "user": {
    "name": "teja",
    "role": "admin"
  }
}
```

You may extract structured values using `spath`.

The important decision is:

```text
Plain/unstructured text
    -> rex is often useful

JSON/XML structured data
    -> spath is often preferable
```

Do not use regular expressions for structured parsing merely because regex can technically do it.

---

# 10. Masking: where should it happen?

Your requirement is:

> Mask sensitive information globally before routing.

That is a sound design.

For example:

```spl
| eval _raw=replace(
    _raw,
    /"password":"[^"]*"/,
    "\"password\":\"xxxxxxxx\""
)
```

And:

```spl
| eval _raw=replace(
    _raw,
    /CreditCard=[0-9]+/,
    "CreditCard=XXXXXXXXXXX"
)
```

The basic design should be:

```text
receive
   |
   v
early DROP
   |
   v
required extraction
   |
   v
GLOBAL MASKING
   |
   v
routing / copying
   |
   v
destinations
```

### Why mask before `branch` or `thru`?

Suppose you have:

```text
password=Secret123
```

and then:

```spl
| branch
    [
        | into $splunk_destination
    ],
    [
        | into $archive_destination
    ];
```

Both paths get the sensitive value.

If you mask first:

```text
password=xxxxxxxx
```

then every copy inherits the sanitized event.

This is especially important when one destination has a different trust boundary [security boundary or access level] from another.

---

# 11. Your masking regex: use the data format to guide the regex

For JSON:

```spl
| eval _raw=replace(
    _raw,
    /"password":"[^"]*"/,
    "\"password\":\"xxxxxxxx\""
)
```

This is usually more general than:

```regex
[A-Za-z0-9_@!-]+
```

because a real password may contain characters such as:

```text
$
%
#
&
*
(
)
[
]
{
}
=
+
?
/
```

If your requirement is:

> replace everything between the JSON quotes,

then:

```regex
[^"]*
```

directly expresses that requirement.

Likewise, your original rule:

```spl
| eval _raw=replace(
    _raw,
    /"credit_card":"[0-9]+"/,
    "card_number=XXXXXXXXXXX"
)
```

changes the field name as well as the value.

If the goal is only masking, preserve the field name:

```spl
| eval _raw=replace(
    _raw,
    /"credit_card":"[0-9]+"/,
    "\"credit_card\":\"XXXXXXXXXXX\""
)
```

---

# 12. `route`: move a subset to a different path

Use:

```spl
| route <condition>, [
    | into $destination2
]
| into $destination
```

Example:

```spl
| route profile == "linux", [
    | into $linux_destination
]
| into $default_destination;
```

Conceptually:

```text
                 events
                   |
                 route
                /     \
             linux    other
               |         |
               v         v
            linux      default
          destination destination
```

The Linux events are diverted to the additional path.

The remaining events continue down the main pipeline.

---

# 13. Multiple `route` commands

This is exactly the pattern you are using.

```spl
| route profile == "linux", [
    | into $linux_destination
]

| route profile == "dns", [
    | into $dns_destination
]

| route profile == "firewall", [
    | into $firewall_destination
]

| into $default_destination;
```

Data flow:

```text
ALL
 |
 +--> linux --> linux_destination
 |
 +--> remaining
        |
        +--> dns --> dns_destination
        |
        +--> remaining
               |
               +--> firewall --> firewall_destination
               |
               +--> remaining --> default_destination
```

The important property is that matching data is diverted and does not then continue through the later main-path routes.

---

# 14. `route` versus `where`

This distinction should be memorized.

## `where`

```spl
| where profile == "linux"
```

Means:

```text
Linux  -> KEEP
Other  -> DROP
```

## `route`

```spl
| route profile == "linux", [
    | into $linux_destination
]
| into $destination;
```

Means:

```text
Linux  -> linux_destination
Other  -> destination
```

Therefore:

```text
where = decide which events survive

route = decide where a subset goes
```

---

# 15. `thru`: copy AND continue

Use `thru` when you want an additional copy of the data while the original path continues.

Example:

```spl
| thru [
    | into $archive_destination
]
| into $splunk_destination;
```

Data flow:

```text
                 event
                   |
                  thru
               /       \
            COPY      ORIGINAL
              |           |
              v           v
          archive      continue
                           |
                           v
                       splunk
```

So:

```text
thru = "make one extra copy and keep going"
```

---

# 16. `thru` placement matters

Compare these two designs.

## Design A

```spl
| mask
| thru [
    | into $archive
]
| route ...
```

The archive receives all events that survived the mask stage.

```text
mask
 |
 +--> archive
 |
 +--> route
```

## Design B

```spl
| mask
| route profile == "linux", [
    | into $linux
]
| thru [
    | into $archive
]
| into $default;
```

Now the archive only receives the events that remain after the Linux route.

The Linux events were already diverted.

Therefore:

> **The location of `thru` determines which population it copies.**

---

# 17. `branch`: multiple complete copies

`branch` is for multiple independent paths.

Example:

```spl
$pipeline = | from $source
    | branch
        [
            | eval environment="production"
            | into $destination1
        ],
        [
            | eval environment="archive"
            | into $destination2
        ];
```

The incoming data is copied into both paths.

Conceptually:

```text
                   ALL EVENTS
                       |
                    branch
                  /        \
                 /          \
                v            v
          production      archive
             path           path
```

Every branch receives a complete copy of the incoming data.

---

# 18. `branch` versus `thru`

This is a common source of confusion.

## `thru`

```text
original path remains
       +
one additional copy
```

Example:

```spl
| thru [
    | into $archive
]
| into $main;
```

## `branch`

```text
multiple complete independent paths
```

Example:

```spl
| branch
    [
        | into $destination1
    ],
    [
        | into $destination2
    ];
```

A useful mental shortcut:

```text
route  = divert a subset

thru   = copy one path and continue original

branch = duplicate the entire input into multiple paths
```

Splunk documents that `route`, `thru`, and `branch` can be combined and used multiple times in the same pipeline.

---

# 19. A powerful real-world pattern: branch + route

Suppose you want:

1. All data masked.
2. One complete sanitized copy archived.
3. Another sanitized copy routed to different Splunk destinations.

A good structure is:

```spl
$pipeline = | from $source

    /* Global masking */
    | eval _raw=replace(
        _raw,
        /"password":"[^"]*"/,
        "\"password\":\"xxxxxxxx\""
    )

    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/

    | branch
        [
            /* Main processing copy */
            | route profile == "linux", [
                | into $linux_destination
            ]
            | route profile == "dns", [
                | into $dns_destination
            ]
            | into $default_destination
        ],
        [
            /* Archive copy */
            | into $archive_destination
        ];
```

Data flow:

```text
                 masked events
                      |
                   branch
                  /      \
                 /        \
                v          v
            MAIN COPY    ARCHIVE COPY
                |
              route
            /   |   \
         linux dns  other
           |    |     |
           v    v     v
         dest  dest  default
```

This is a strong pattern when the requirement really is "copy the full sanitized stream and independently process another copy."

---

# 20. `dedup`: remove duplicate events

`dedup` is NOT a routing command.

It reduces the event set by removing events that have the same combination of values for the fields you specify.

Example:

```spl
| dedup host
```

Input:

```text
host=server1 action=login
host=server1 action=logout
host=server1 action=login
host=server2 action=query
host=server2 action=query
```

Result:

```text
host=server1 action=login
host=server2 action=query
```

Only one event per host remains.

## Multiple fields

```spl
| dedup host, action
```

Now uniqueness is based on:

```text
host + action
```

Example:

```text
server1 + login
server1 + logout
server1 + login
server2 + query
server2 + query
```

becomes:

```text
server1 + login
server1 + logout
server2 + query
```

### Important current SPL2 syntax

In SPL2 the fields are comma-delimited:

```spl
| dedup host, action
```

not:

```spl
| dedup host action
```

Options, when used, come before the field list.

Splunk's current documentation also warns against deduplicating large volumes of `_raw`, because keeping the complete event text in memory can affect performance.

---

# 21. When should you use `dedup`?

Use it when you have a meaningful identifier or combination that actually represents a duplicate.

Good candidates:

```text
event_id
request_id
transaction_id
message_id
```

For example:

```spl
| dedup request_id
```

Potentially useful:

```spl
| dedup host, action
```

but only if your definition of duplicate really is:

> same host AND same action.

Do NOT blindly write:

```spl
| dedup _raw
```

for high-volume data.

Also be careful with:

```spl
| dedup host
```

because that literally means:

> keep only one event per host.

That would destroy legitimate events from the same host.

---

# 22. Deduplication order

Suppose:

```text
Event A:
password=Secret123
request_id=1001

Event B:
password=Secret999
request_id=1001
```

If the actual duplicate identity is `request_id`, then:

```spl
| dedup request_id
```

is sensible.

But if you mask first:

```text
Event A:
password=xxxxxxxx
request_id=1001

Event B:
password=xxxxxxxx
request_id=1001
```

the two events become even more similar.

This is one more reason to deduplicate using a real event identifier instead of `_raw`.

---

# 23. `stats`: aggregation rather than ordinary filtering

`stats` does something very different.

Suppose:

```text
host=server1 bytes=100
host=server1 bytes=200
host=server1 bytes=300
host=server2 bytes=500
```

Run:

```spl
| stats sum(bytes) AS total_bytes BY host
```

Result:

```text
server1   600
server2   500
```

You started with four events and emitted two aggregated results.

That means:

```text
dedup
    removes duplicate events

stats
    combines many events into summarized results
```

Current Edge Processor aggregation support includes:

```text
count
max
min
sum
```

and `span` for grouping by time. Edge Processor aggregation also supports state-window controls such as `@maxdelay` and `@maxdisk`.

Example:

```spl
$pipeline = | from $source
    | @maxdelay("10m")
    | @maxdisk("1GB")
    | stats sum(bytes_out) BY server_name
    | into $destination;
```

The precise aggregation behavior is different from search-time `stats`; Edge Processor aggregates continuously streaming data inside a state window and emits aggregation results.

Also note that `avg` is not directly supported as an Edge Processor statistical function. A documented approach is to aggregate `sum` and `count`, then calculate the average later.

---

# 24. When should `stats` be used?

Use `stats` when your downstream use case does not require every original event.

Example:

You receive:

```text
1,000,000 flow records
```

but the downstream requirement is:

```text
bytes by host per time period
```

Then sending every raw flow record may be unnecessary.

An aggregation can reduce the number of events sent downstream.

But do NOT use `stats` if your SOC investigation requires individual events.

For example:

```text
authentication events
process creation events
network connection events
file execution events
```

often need their original detail for investigation and detection.

So:

```text
Need original events?
    -> do not aggregate away the evidence.

Need only a metric/summary?
    -> stats can reduce volume.
```

---

# 25. `if`: conditional processing

The current SPL2 `if` command provides if/elseif/else processing.

Example:

```spl
| if (profile == "linux") [
    | eval index="linux"
]
elseif (profile == "dns") [
    | eval index="dns"
]
elseif (profile == "firewall") [
    | eval index="firewall"
]
else [
    | eval index="other"
]
| into $destination;
```

Here the events are not necessarily going to different destinations.

Instead, you are changing the event:

```text
linux     -> index=linux
dns       -> index=dns
firewall  -> index=firewall
other     -> index=other
```

and then all events go to:

```text
$destination
```

This is different from:

```spl
| route ...
```

which creates a separate path and destination.

Think:

```text
if
    "How should I process this event?"

route
    "Where should this subset go?"
```

---

# 26. `eval`: modify the event

Examples:

```spl
| eval severity="high"
```

```spl
| eval total_bytes=bytes_in + bytes_out
```

```spl
| eval category=if(status >= 500, "server_error", "normal")
```

You are using `eval` for masking:

```spl
| eval _raw=replace(...)
```

That is valid because `replace()` is an evaluation function used inside `eval`.

---

# 27. `replace()` versus `replace`

These are different.

## `replace()` function

Used inside `eval`:

```spl
| eval _raw=replace(
    _raw,
    /CreditCard=[0-9]+/,
    "CreditCard=XXXXXXXXXXX"
)
```

This modifies text inside the field.

## `replace` command

The SPL2 `replace` command is a different command for replacing field values.

For your raw-data masking use case, you are correctly using the `replace()` function inside `eval`.

---

# 28. `fields`: remove fields, NOT events

Example:

```spl
| fields - profile, action
```

The event remains.

Only the fields are removed.

Before:

```text
_raw
profile
action
host
```

After:

```text
_raw
host
```

Therefore:

```text
where
    removes EVENTS

fields
    removes FIELDS
```

This is a very important distinction.

---

# 29. Why remove temporary fields?

Suppose you do:

```spl
| rex field=_raw /"profile":"(?P<profile>[^"]+)"/
| route profile == "linux", [
    | into $linux
]
```

If `profile` was only a temporary field used for routing and does not need to be stored downstream, you can remove it on the appropriate path.

Conceptually:

```text
extract
   |
route
   |
fields -
   |
destination
```

This keeps the outgoing event cleaner.

---

# 30. `rename`: change field names

Example:

```spl
| rename profileTest AS profile
```

Multiple SPL2 renames are comma-separated:

```spl
| rename profileTest AS profile, action2 AS action
```

Use this when the incoming field name is not the field name you want downstream.

---

# 31. `lookup`: enrichment

Suppose you have:

```text
src_ip=10.10.10.20
```

and a lookup dataset:

```text
ip            asset_type
10.10.10.20   server
10.10.10.30   firewall
```

A lookup can enrich:

```text
src_ip=10.10.10.20
asset_type=server
```

Then you can route based on the result:

```text
src_ip
  |
lookup
  |
asset_type
  |
route
```

This is useful when the raw event itself does not contain the business/security classification you need.

The lookup dataset must be imported into the Edge Processor pipeline before using the `lookup` command.

---

# 32. `decrypt`

Use this only when data is already encrypted and the pipeline needs to decrypt a field.

Conceptually:

```text
encrypted field
       |
       v
     decrypt
       |
       v
decrypted value
       |
       v
mask / transform / route
```

Splunk documents private-key handling through the lookup mechanism for this command.

This is specialized processing, not ordinary parsing.

---

# 33. `ocsf`

`ocsf` converts supported event data into the Open Cybersecurity Schema Framework format.

Conceptually:

```text
vendor event
     |
     v
  ocsf
     |
     v
normalized security event
```

This is useful when different products represent similar security activity using different field names.

For example:

```text
vendor field A
vendor field B
vendor field C
```

can be transformed toward a common security schema.

Use it when your downstream architecture benefits from standardized event structure; do not apply it automatically to everything simply because it exists.

---

# 34. `expand`, `flatten`, and `mvexpand`

These are easy to confuse.

## `expand`

For arrays of objects.

Example concept:

```json
[
  {"name":"A","ip":"10.0.0.1"},
  {"name":"B","ip":"10.0.0.2"}
]
```

`expand` operates on that array structure.

## `flatten`

For an object:

```text
{
  "name":"server01",
  "ip":"10.0.0.1"
}
```

and promotes first-level key/value pairs into fields:

```text
name = server01
ip   = 10.0.0.1
```

## `mvexpand`

For a multivalue field.

Example:

```text
iplist =
10.0.0.1
10.0.0.2
10.0.0.3
```

```spl
| mvexpand iplist
```

can produce separate results for the values.

Mental model:

```text
expand
    array of objects -> expanded results

flatten
    object -> fields

mvexpand
    multivalue field -> multiple events
```

---

# 35. Destination: do not confuse DESTINATION with INDEX

This matters a lot in your lab.

A pipeline destination is the **endpoint** to which the Edge Processor sends the processed data.

An index is where the data is stored in the Splunk platform deployment.

You might have:

```text
one Splunk destination
    |
    +--> linux index
    +--> dns index
    +--> firewall index
```

or:

```text
destination A -> Splunk deployment A
destination B -> Splunk deployment B
destination C -> S3
```

These are different levels.

---

# 36. Best destination strategy

## Case 1 — same Splunk deployment, different indexes

Often you do NOT need a separate destination just because you need a separate index.

For example:

```spl
| route profile == "linux", [
    | eval index="linux"
    | into $splunk_destination
]
| route profile == "dns", [
    | eval index="dns"
    | into $splunk_destination
]
| into $splunk_destination;
```

This is conceptually:

```text
             ONE SPLUNK DESTINATION
                    |
          +---------+---------+
          |         |         |
        linux      dns      default
        index      index
```

Whether the resulting index is selected from the event metadata, pipeline `eval`, or destination configuration depends on the documented index precedence for the protocol and setup you are using.

## Case 2 — different systems

Use separate destinations when data truly needs to go to different endpoints.

Example:

```text
Linux security events -> Splunk
Archive copy          -> S3
Application events    -> another Splunk deployment
```

That is a genuine destination difference.

---

# 37. Index precedence matters

For Splunk platform S2S and HEC destinations, Splunk documents an index precedence order.

For S2S, configurations can include:

1. Splunk platform routing configuration.
2. The pipeline's `eval index="..."`.
3. Index metadata already carried in the event.
4. The deployment's default index.

For HEC, the precedence chain differs and includes HEC destination/token/default-index settings.

Therefore:

> Do not assume that `eval index="linux"` is always the only thing controlling the final index.

Check the actual destination protocol and precedence rules in your environment.

---

# 38. Internal logs: destination choice matters

Splunk documents that when routing internal logs to a Splunk platform deployment using S2S, the event metadata can preserve the intended index.

That is useful because internal logs often already carry index metadata such as:

```text
_internal
_audit
_introspection
```

For these cases, a Splunk platform S2S destination can be preferable when your intention is to preserve the original index metadata.

---

# 39. The best high-volume design principle

Your eBPF idea can be converted into this Edge Processor rule:

> **Remove data as early as you safely can, and do expensive transformations only on data you still need.**

But "early" must respect the available fields.

A practical order is:

```text
1. PARTITION
   define the broad population

2. EARLY WHERE
   drop clearly unwanted events

3. MINIMAL EXTRACTION
   create only the fields needed for decisions

4. DEDUP, if there is a legitimate duplicate key

5. MASK
   sanitize data before any copy/output

6. TRANSFORM / ENRICH
   eval / rename / lookup / ocsf

7. AGGREGATE, if detailed events are not required
   stats

8. ROUTE
   route subsets

9. COPY
   thru / branch when required

10. INTO
   send the final data
```

This is a design principle, not a mandatory command order.

---

# 40. Why masking is sometimes BEFORE filtering

There is a security trade-off.

Suppose your filter condition requires a sensitive field:

```text
password
```

If you only need to decide whether the field exists:

```spl
| where match(_raw, /"password"\s*:/)
```

then you may not need to expose its value to a field extraction.

If the condition itself depends on a sensitive value, think carefully before extracting that value into a new field.

A good principle is:

```text
Use the minimum sensitive data necessary to make the decision.
```

---

# 41. A complete pipeline matching your lab

Below is a cleaner version of the pipeline you have been building.

```spl
import route from /splunk/ingest/commands

$pipeline = | from $source

    /* ==========================================
       1. EARLY FILTERING
       Drop data that should never be processed.
       ========================================== */

    | where NOT match(_raw, /"action"\s*:\s*"debug"/i)
    | where NOT match(_raw, /"profile"\s*:\s*"test"/i)


    /* ==========================================
       2. MINIMAL FIELD EXTRACTION
       Extract fields needed for routing/logic.
       ========================================== */

    | rex field=_raw /"profile"\s*:\s*"(?P<profile>[^"]+)"/
    | rex field=_raw /"action"\s*:\s*"(?P<action>[^"]+)"/


    /* ==========================================
       3. OPTIONAL DEDUP
       Use only a real duplicate identifier.
       ========================================== */

    /* | dedup request_id */


    /* ==========================================
       4. GLOBAL MASKING
       Sanitize before routing/copying.
       ========================================== */

    | eval _raw=replace(
        _raw,
        /CreditCard=[0-9]+/,
        "CreditCard=XXXXXXXXXXX"
    )

    | eval _raw=replace(
        _raw,
        /credit_card=[0-9]+/,
        "credit_card=XXXXXXXXXXX"
    )

    | eval _raw=replace(
        _raw,
        /card_number=[0-9]+/,
        "card_number=XXXXXXXXXXX"
    )

    | eval _raw=replace(
        _raw,
        /"credit_card":"[0-9]+"/,
        "\"credit_card\":\"XXXXXXXXXXX\""
    )

    | eval _raw=replace(
        _raw,
        /"password":"[^"]*"/,
        "\"password\":\"xxxxxxxx\""
    )

    | eval _raw=replace(
        _raw,
        /Password=[A-Za-z0-9_@!-]+/,
        "Password=XXXXXXX"
    )

    | eval _raw=replace(
        _raw,
        /password=[A-Za-z0-9_@!-]+/,
        "password=XXXXXXX"
    )


    /* ==========================================
       5. ROUTE LINUX
       ========================================== */

    | route profile == "linux", [
        | eval index="linux"
        | into $destination2
    ]


    /* ==========================================
       6. ROUTE DNS
       ========================================== */

    | route profile == "dns", [
        | eval index="dns"
        | into $destination3
    ]


    /* ==========================================
       7. ROUTE FIREWALL
       ========================================== */

    | route profile == "firewall", [
        | eval index="firewall"
        | into $destination4
    ]


    /* ==========================================
       8. DEFAULT REMAINING DATA
       ========================================== */

    | into $destination;
```

---

# 42. If your real requirement is "drop everything except Linux"

Do NOT overcomplicate this with routing.

Use:

```spl
$pipeline = | from $source
    | rex field=_raw /"profile"\s*:\s*"(?P<profile>[^"]+)"/
    | where profile == "linux"
    | eval _raw=replace(
        _raw,
        /"password":"[^"]*"/,
        "\"password\":\"xxxxxxxx\""
    )
    | into $destination;
```

The data flow is:

```text
ALL EVENTS
    |
    v
extract profile
    |
    v
where profile == linux
    |
 +--+--+
 |     |
linux other
 |      |
 v      v
mask   DROP
 |
 v
destination
```

This is clearer than:

```spl
route linux -> destination
then somehow drop everything else
```

When you simply want:

> keep one subset and discard everything else,

`where` is normally the clearest statement.

---

# 43. If you want Linux somewhere and DROP everything else

Use:

```spl
| where profile == "linux"
| into $linux_destination;
```

No route is necessary.

Because:

```text
Linux -> destination
Other -> DROP
```

---

# 44. If you want Linux somewhere and all other data somewhere else

Use:

```spl
| route profile == "linux", [
    | into $linux_destination
]
| into $other_destination;
```

Because:

```text
Linux -> linux_destination
Other -> other_destination
```

---

# 45. If you want Linux copied to another destination AND continue

Use:

```spl
| route profile == "linux", [
    | thru [
        | into $extra_destination
    ]
    | into $linux_destination
]
| into $other_destination;
```

The exact nested structure can be adjusted to the required destinations, but the key concept is:

```text
route = select subset
thru  = copy that subset while its path continues
```

---

# 46. If you want every event copied

Use `branch` or a `thru` depending on the desired structure.

Simple two-copy case:

```spl
| thru [
    | into $archive
]
| into $main;
```

More independent multi-path case:

```spl
| branch
    [
        | into $destination1
    ],
    [
        | into $destination2
    ];
```

The documentation's routing examples show that these commands can also be nested and combined.

---

# 47. When `branch` is NOT appropriate

Do not use:

```spl
| branch
    [
        | route ...
    ],
    [
        | route ...
    ];
```

just because there are multiple classifications.

If your requirement is simply:

```text
Linux -> A
DNS -> B
Firewall -> C
Other -> D
```

then sequential `route` commands are easier to understand:

```spl
| route profile == "linux", [...]
| route profile == "dns", [...]
| route profile == "firewall", [...]
| into $default;
```

Use `branch` when you actually want multiple copies of the input.

---

# 48. A critical anti-pattern: branch before masking

Bad design:

```spl
| branch
    [
        | eval ...mask...
        | into $destination1
    ],
    [
        | into $destination2
    ];
```

Why is it dangerous?

Because:

```text
destination2
```

gets the unmasked copy.

Better:

```spl
| eval ...mask...
| branch
    [
        | into $destination1
    ],
    [
        | into $destination2
    ];
```

Now both branches receive sanitized data.

---

# 49. A critical anti-pattern: expensive parsing before obvious dropping

Less efficient:

```spl
| rex big_complex_regex...
| rex another_big_regex...
| spath...
| eval...
| eval...
| where action != "debug"
```

Better when `debug` can be recognized directly:

```spl
| where NOT match(_raw, /"action"\s*:\s*"debug"/i)
| rex ...
| spath ...
| eval...
```

The principle is:

```text
cheap, high-volume reduction
        before
expensive processing
```

But do not sacrifice correctness simply to move a command earlier.

---

# 50. A critical anti-pattern: dedup on `_raw` without a reason

Avoid:

```spl
| dedup _raw
```

for large streams.

The current SPL2 dedup documentation warns that `_raw` requires retaining the event text and can affect performance.

Prefer:

```spl
| dedup event_id
```

or another meaningful identifier when the event format provides one.

---

# 51. A critical anti-pattern: `dedup host`

Be very careful with:

```spl
| dedup host
```

because it literally means:

> Keep one event for a host.

If a server generates:

```text
login
logout
process_start
network_connection
file_creation
```

you could throw away almost all of the useful information.

Deduplication must match the real definition of a duplicate.

---

# 52. A critical anti-pattern: using `stats` just to reduce volume

This:

```spl
| stats count() BY host
```

is not "compressing the events."

It is changing:

```text
many original events
```

into:

```text
summary events
```

If the SOC later needs the original authentication event, process creation event, or network connection event, that evidence has been removed from the downstream stream.

Use `stats` when the downstream use case genuinely needs an aggregate.

---

# 53. A critical anti-pattern: using many destinations when one destination is sufficient

Suppose:

```text
Linux -> index=linux
DNS   -> index=dns
FW    -> index=firewall
```

and all three indexes belong to the same Splunk platform deployment.

You may be able to use:

```text
one Splunk destination
+
different index metadata
```

rather than:

```text
three separate destinations
```

The correct choice depends on your destination/protocol and index-routing requirements.

The destination is about **where the data goes**.

The index is about **where it is stored within the Splunk deployment**.

---

# 54. Global masking + route: recommended pattern for your lab

Your requirement is:

> "First remove unwanted events, then mask everything that remains, then route by profile."

A clean design is:

```text
                     SOURCE
                       |
                       v
                   PARTITION
                       |
                       v
                 EARLY WHERE
                       |
                       | DROP unwanted
                       v
                MINIMAL EXTRACT
                       |
                       v
                    DEDUP
                 (if required)
                       |
                       v
                 GLOBAL MASK
                       |
                       v
                    ROUTE
                /      |      \
               /       |       \
           Linux       DNS     Firewall
             |           |        |
             v           v        v
           dest1       dest2    dest3
               \        |       /
                \       |      /
                 remaining
                     |
                     v
                  default
```

This is the pattern I recommend learning first.

---

# 55. But what if you want an archive too?

Then:

```text
                       SOURCE
                         |
                         v
                      FILTER
                         |
                         v
                       MASK
                         |
                         v
                       BRANCH
                    /         \
                   /           \
                  v             v
             MAIN COPY      ARCHIVE COPY
                 |
                ROUTE
              /   |    \
           Linux DNS Firewall
             |    |      |
             v    v      v
           dest1 dest2  dest3
```

This is where `branch` earns its place.

---

# 56. What if the archive is only an additional copy of the final stream?

Use `thru` instead:

```text
filter
  |
mask
  |
route
  |
thru ---> archive
  |
default destination
```

But remember that `thru` only copies the events that actually reach that point.

---

# 57. What if I want to calculate metrics and not send raw data?

Use `stats`.

Example:

```spl
$pipeline = | from $source
    | stats count() BY profile
    | into $destination;
```

Input:

```text
linux
linux
linux
dns
dns
firewall
```

Output:

```text
linux     3
dns       2
firewall  1
```

This is suitable only when the downstream destination needs the summary rather than the original event stream.

---

# 58. Current Edge Processor command map

The current Edge Processor pipeline documentation lists these processing commands:

```text
branch
decrypt
dedup
eval
expand
fields
flatten
from
if
into
lookup
mvexpand
ocsf
rename
replace
rex
route
spath
stats
thru
where
```

The current documentation also specifies that Edge Processor pipelines support a subset of SPL2 and that regular expressions use PCRE2.

Do not assume that every SPL2 search command is therefore valid in an Edge Processor pipeline.

---

# 59. Command map by job

## Input / output

```text
from
into
```

## Parse / extract

```text
rex
spath
```

## Modify

```text
eval
replace
rename
fields
if
ocsf
decrypt
```

## Enrich

```text
lookup
```

## Filter / reduce

```text
where
dedup
stats
```

## Copy / routing

```text
route
thru
branch
```

## Structured-data expansion

```text
expand
flatten
mvexpand
```

---

# 60. Event-count behavior

Another extremely useful way to classify commands is by what they can do to event count.

## Can remove events

```text
where
dedup
stats
```

## Can create additional copies/results

```text
branch
thru
expand
mvexpand
```

These do not all increase event count in the same way:

- `branch` creates multiple copies of the incoming stream.
- `thru` creates an additional copy while the original continues.
- `expand` and `mvexpand` expand structured or multivalue data into multiple results.

## Normally transform the existing event

```text
eval
replace
rename
fields
rex
spath
if
decrypt
ocsf
```

## Redirect subset

```text
route
```

`route` primarily changes the path taken by matching events rather than automatically making a second copy.

---

# 61. Performance design: how to think about "best"

There is no universal command order that is always fastest.

Instead, evaluate every step using four questions:

### Question 1 — Can I eliminate the event without parsing everything?

Example:

```text
debug event
```

If yes:

```spl
| where NOT match(_raw, /debug/i)
```

can be early.

### Question 2 — Do I need this field later?

If not, do not extract it.

### Question 3 — Can I reduce event count safely?

Use:

```text
dedup
stats
```

only when their semantics match the business requirement.

### Question 4 — Am I creating copies unnecessarily?

Every:

```text
thru
branch
```

can increase the downstream amount of data.

Use them because a second path is required, not simply because multiple destinations exist.

---

# 62. The "cheapest useful operation first" rule

A good design principle is:

```text
CHEAP FILTER
    |
    v
MINIMAL PARSE
    |
    v
REDUCE
    |
    v
EXPENSIVE TRANSFORM
    |
    v
ENRICH
    |
    v
ROUTE / COPY
```

But security requirements can change the order.

For example:

```text
If a copy will leave the trusted boundary:
    mask before creating that copy.
```

Therefore the actual goal is:

> **Do the earliest safe operation that removes the most unnecessary work.**

---

# 63. "Drop first" versus "mask first"

These are not always competing requirements.

Suppose 20% of your events should be dropped and 80% need masking.

A sensible structure is:

```text
EARLY DROP
    |
    v
80% remaining
    |
    v
MASK
```

instead of:

```text
100%
 |
MASK
 |
DROP 20%
```

because you otherwise spend masking work on data that will never be sent.

However, if the filter requires a field that only exists after parsing, you must perform enough extraction to make the decision.

---

# 64. "Extract first" versus "filter raw"

Use this decision:

```text
Can the unwanted event be identified reliably from _raw?
       |
      YES
       |
       v
cheap raw filtering can happen first

      NO
       |
       v
extract the minimum field required
       |
       v
where
```

Example:

```text
_raw contains:
"profile":"test"
```

A simple raw match can be enough:

```spl
| where NOT match(_raw, /"profile"\s*:\s*"test"/i)
```

But if the rule depends on:

```text
nested JSON
computed value
lookup result
```

then extraction/enrichment may be necessary before the filter.

---

# 65. Best practice for routing fields

Do not extract ten fields when you only need one for routing.

If routing requires:

```text
profile
```

extract:

```spl
| rex field=_raw /"profile":"(?P<profile>[^"]+)"/
```

Do not immediately parse:

```text
profile
action
username
department
browser
country
application
device
version
...
```

unless those fields are actually required.

---

# 66. Best practice for temporary fields

Temporary routing fields can be removed after the decision.

For example:

```spl
| rex field=_raw /"profile":"(?P<profile>[^"]+)"/
| route profile == "linux", [
    | fields - profile
    | into $linux_destination
]
```

Whether you should remove it depends on whether the destination needs the field.

Do not remove useful context merely for cleanliness.

---

# 67. Best practice for `where`

Use `where` when the question is:

> "Should this event remain in the pipeline?"

Examples:

```spl
| where status == 200
```

```spl
| where profile IN ("linux", "dns")
```

```spl
| where NOT match(_raw, /healthcheck/i)
```

Avoid using `route` merely to simulate a drop.

---

# 68. Best practice for `route`

Use `route` when the question is:

> "This event belongs somewhere different."

Example:

```spl
| route profile == "linux", [
    | eval index="linux"
    | into $destination
]
| into $destination;
```

Use multiple `route`s when classification is sequential:

```text
Linux
DNS
Firewall
Everything else
```

---

# 69. Best practice for `thru`

Use `thru` when:

```text
"I need an additional copy, and the original should keep going."
```

Example:

```spl
| thru [
    | into $archive
]
| into $main;
```

A classic use is:

```text
sanitized stream
   |
   +---- archive
   |
   +---- main Splunk stream
```

---

# 70. Best practice for `branch`

Use `branch` when:

```text
"I need multiple complete copies that can be processed independently."
```

Example:

```spl
| branch
    [
        | ...process A...
        | into $destinationA
    ],
    [
        | ...process B...
        | into $destinationB
    ];
```

The more branches you add, the more downstream data can be generated.

Therefore use it deliberately.

---

# 71. Best practice for `dedup`

Ask:

```text
"What field combination defines a duplicate?"
```

Only then write:

```spl
| dedup event_id
```

or:

```spl
| dedup host, request_id
```

Do not start with `_raw` simply because it is available.

Also remember that in current SPL2, dedup keeps the first event encountered for a duplicate combination unless you specify a different count.

---

# 72. Best practice for `stats`

Ask:

```text
"Does the downstream system need every original event?"
```

If YES:

```text
do not aggregate the raw event stream away
```

If NO:

```text
stats can reduce the output volume
```

For example:

```spl
| stats count() BY host
```

can replace thousands of individual events with a much smaller set of summary records.

For Edge Processor, remember that `stats` is a streaming aggregation with a state window, not the same operational model as search-time statistics.

---

# 73. Best practice for destinations

A useful hierarchy is:

```text
Different endpoint/system?
    -> different destination

Same Splunk deployment, different index?
    -> often one destination + index routing

Need archive copy?
    -> thru or branch + archive destination

Need only a filtered subset?
    -> where before into

Need classification only?
    -> if/eval may be enough
```

Always verify the actual index precedence for your destination protocol.

---

# 74. Default destination: important safety setting

Splunk recommends configuring a default destination to avoid unintended data loss for unprocessed data.

Without a default destination, unprocessed data can be dropped.

This is different from intentionally filtering data with `where`.

Therefore distinguish:

```text
INTENTIONAL DROP
    where ...

UNPROCESSED DATA
    partition mismatch / no applicable pipeline
```

Those are not the same event-flow condition.

---

# 75. Your complete architecture for a production-style security stream

A robust design can look like:

```text
                SOURCE
                  |
                  v
             EDGE PROCESSOR
                  |
                  v
            PARTITION SCOPE
                  |
                  v
          EARLY FILTER / DROP
                  |
          unwanted --> DROP
                  |
                  v
        MINIMAL EXTRACTION
                  |
                  v
              DEDUP?
                  |
                  v
         GLOBAL SANITIZATION
         password / card / PII
                  |
                  v
            ENRICHMENT?
             lookup / OCSF
                  |
                  v
            AGGREGATION?
                stats
                  |
                  v
              ROUTING
           /      |      \
          /       |       \
      Linux      DNS    Firewall
        |          |        |
        v          v        v
      dest1      dest2    dest3
          \        |       /
           \       |      /
            remaining
                |
                v
          default destination
```

This gives you a disciplined way to decide where every command belongs.

---

# 76. A complete decision table

| Requirement | Command | Result |
|---|---|---|
| Read incoming data | `from` | Starts pipeline input |
| Send data | `into` | Terminates a path and sends data |
| Drop unwanted events | `where` | Matching false events leave the pipeline |
| Remove duplicate combinations | `dedup` | Duplicate events are removed |
| Summarize many events | `stats` | Emits aggregate results |
| Extract from raw | `rex` | Creates fields |
| Parse JSON/XML | `spath` | Extracts structured values |
| Change field/value | `eval` | Adds/modifies fields |
| Mask text | `eval` + `replace()` | Changes content |
| Remove fields | `fields` | Event remains, selected fields removed |
| Rename fields | `rename` | Changes field names |
| Conditional processing | `if` | First matching branch processes event |
| Add lookup information | `lookup` | Enriches events |
| Normalize to OCSF | `ocsf` | Converts supported data to OCSF |
| Decrypt field | `decrypt` | Decrypts supported encrypted data |
| Expand object arrays | `expand` | Expands structured arrays |
| Flatten object | `flatten` | Promotes first-level keys to fields |
| Expand multivalue field | `mvexpand` | Produces separate results |
| Send a subset elsewhere | `route` | Diverts matching subset |
| Make a copy and continue | `thru` | Copy + original continues |
| Make multiple complete paths | `branch` | Every branch receives a copy |

---

# 77. A simple rulebook to memorize

```text
DROP?
    where

DUPLICATE?
    dedup

SUMMARY?
    stats

CHANGE?
    eval / replace / rename / fields

EXTRACT?
    rex / spath

DIFFERENT DESTINATION?
    route

EXTRA COPY?
    thru

MULTIPLE COMPLETE PATHS?
    branch

CONDITIONAL TRANSFORMATION?
    if

ENRICH?
    lookup

NORMALIZE?
    ocsf

DECRYPT?
    decrypt

EXPAND STRUCTURED DATA?
    expand / flatten / mvexpand
```

---

# 78. Your eBPF analogy, finalized

The most useful comparison is:

```text
eBPF / kernel world

packet
  |
  v
early filter
  |
  +--> DROP
  |
  v
expensive processing
```

Versus:

```text
Edge Processor

event arrives
  |
  v
partition scope
  |
  v
where
  |
  +--> DROP
  |
  v
parse
  |
  v
dedup / transform / enrich
  |
  v
route
  |
  v
destination
```

The architectural goal is similar:

> **Do not spend expensive downstream work on data that you already know you do not need.**

But the layer is different.

eBPF can affect packet processing before the event reaches the logging pipeline.

Edge Processor filters event data after it has arrived at the Edge Processor.

---

# 79. Final recommended design for your current lab

Given your exact use case:

> "Receive raw logs → remove unwanted traffic → extract profile/action → globally mask sensitive data → route Linux/DNS/Firewall → send the remainder normally."

Use:

```text
PARTITION
    ↓
EARLY WHERE
    ↓
REX / SPATH
    ↓
OPTIONAL DEDUP
    ↓
GLOBAL MASKING
    ↓
ROUTE LINUX
    ↓
ROUTE DNS
    ↓
ROUTE FIREWALL
    ↓
DEFAULT DESTINATION
```

And if you need an archive:

```text
PARTITION
    ↓
EARLY WHERE
    ↓
EXTRACT
    ↓
MASK
    ↓
BRANCH
   ├── main routing
   │      ├── Linux
   │      ├── DNS
   │      ├── Firewall
   │      └── default
   │
   └── archive
```

If you only need an additional copy of the current stream:

```text
...processing...
      |
     thru
    /    \
 archive  continue
            |
            v
         destination
```

---

# 80. Official documentation used

1. **Edge Processor pipeline syntax**  
   https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/working-with-pipelines/edge-processor-pipeline-syntax

2. **Filter and mask data using an Edge Processor**  
   https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/process-data-using-pipelines/filter-and-mask-data-using-an-edge-processor

3. **How data moves through the Edge Processor solution**  
   https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-enterprise/10.4/how-the-edge-processor-solution-works/how-data-moves-through-the-edge-processor-solution

4. **Process a subset of data using an Edge Processor (`route`)**  
   https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/route-data-using-pipelines/process-a-subset-of-data-using-an-edge-processor

5. **Routing data in the same Edge Processor pipeline**  
   https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/route-data-using-pipelines/routing-data-in-the-same-edge-processor-pipeline-to-different-actions-and-destinations

6. **SPL2 `dedup` command**  
   https://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/dedup-command/dedup-command-overview-syntax-and-usage

7. **SPL2 `where` command**  
   https://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/where-command/where-command-overview-syntax-and-usage

8. **SPL2 `stats` examples**  
   https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/stats-command/stats-command-examples

9. **Aggregate event data using Edge Processor**  
   https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/process-data-using-pipelines/aggregate-event-data-using-edge-processor

10. **Send data from Edge Processors to Splunk Cloud Platform**  
    https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/send-data-out-from-edge-processors/send-data-from-edge-processors-to-the-splunk-cloud-platform-deployment-connected-to-your-tenant

---

## The one-page mental model

```text
                  ┌──────────────────────────┐
                  │         SOURCE           │
                  └────────────┬─────────────┘
                               |
                               v
                         PARTITION
                    "scope of pipeline"
                               |
                               v
                         WHERE / DROP
                    "should event survive?"
                               |
                               v
                       REX / SPATH
                   "what fields do I need?"
                               |
                               v
                         DEDUP?
                   "is this a duplicate?"
                               |
                               v
                     MASK / TRANSFORM
                    "sanitize and modify"
                               |
                               v
                        LOOKUP / OCSF
                    "enrich / normalize"
                               |
                               v
                          STATS?
                     "do I need summaries?"
                               |
                               v
                           ROUTE
                    "where should subsets go?"
                         /     |      \
                        /      |       \
                       v       v        v
                    Linux     DNS    Firewall
                       \       |       /
                        \      |      /
                         v     v     v
                       DEFAULT / REMAINING
                               |
                               v
                             INTO
                               |
                               v
                         DESTINATION
```

**Design rule:** filter unwanted events as early as the available information allows, avoid unnecessary extraction and copying, sanitize before creating any less-trusted copy, use `route` for destination selection, `thru` for an extra copy that keeps the original path, `branch` for multiple complete copies, `dedup` only with a real duplicate definition, and `stats` only when the original event detail is not required downstream.
