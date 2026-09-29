# SPL2 Edge Processor — Data Flow, Filtering, Dropping, Transformation, Routing, Copying, Deduplication, Aggregation, and Pipeline Design

## Purpose

This note explains how to design SPL2 pipelines for Splunk Edge Processor in a systematic way.

The focus is not on memorizing commands. The focus is understanding:

1. What happens to an event at each stage.
2. Which commands keep an event, remove it, change it, copy it, or redirect it.
3. Where filtering should happen.
4. How to choose between `where`, `route`, `thru`, `branch`, `dedup`, `stats`, and `if`.
5. How partitioning in the Edge Processor builder relates to `from $source`.
6. How to design a pipeline that is correct, understandable, and avoids unnecessary processing.

Splunk's current documentation describes an Edge Processor pipeline as an SPL2 module containing a `$pipeline` statement. A pipeline uses `from $source`, optional processing commands, and `into $destination`. The subset processed by `from $source` is determined by the **partition configured in the pipeline builder**. The builder also configures destinations used by `into`. [Official documentation](https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/working-with-pipelines/edge-processor-pipeline-syntax)

---

# 1. The first thing to understand: a pipeline has two configuration layers

An Edge Processor pipeline has configuration outside the SPL2 body and processing inside the SPL2 body.

## Layer 1 — Pipeline configuration in the builder

The Edge Processor UI asks you to configure things such as:

- Partition
- Sample data
- Destination

The **partition** defines the subset of incoming data that this particular pipeline receives.

The **destination** defines where `into $destination` sends the processed data.

For example:

```text
Incoming data
      |
      v
+------------------+
| Pipeline         |
| partition        |
| host = server01  |
| sourcetype=firewall
+------------------+
      |
      v
Only selected data enters this pipeline
```

Current Splunk documentation says that when creating an Edge Processor pipeline, you must specify the subset of received data to process by defining a partition in the pipeline builder. [Official documentation](https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/process-data-using-pipelines/filter-and-mask-data-using-an-edge-processor)

## Layer 2 — SPL2 pipeline body

The SPL2 then looks like:

```spl
import route from /splunk/ingest/commands

$pipeline = | from $source
    | <processing commands>
    | into $destination;
```

The partition is not normally written into the `$pipeline` statement itself. `$source` represents the subset selected by the partition configured in the pipeline builder. [Official documentation](https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-enterprise/10.4/working-with-pipelines/edge-processor-pipeline-syntax)

---

# 2. Why your pipeline says "configure the pipeline to specify subset of data to process"

Suppose you enter:

```spl
import route from /splunk/ingest/commands

$pipeline = | from $source
    | eval _raw=replace(_raw, /CreditCard=[0-9]+/, "CreditCard=XXXXXXXXXXX")
    | into $destination;
```

and the Edge Processor builder says something like:

> Configure the pipeline to specify a subset of data to process.

That message is referring to the **pipeline partition**, not to the `route` statement.

You need to configure the partition in the UI.

For example:

```text
Pipeline builder
    |
    +-- Partition
    |      |
    |      +-- Field: sourcetype
    |      +-- Action: Keep
    |      +-- Operator: =
    |      +-- Value: syslog
    |
    +-- Destination
           |
           +-- Splunk platform
```

Then the SPL2 can use:

```spl
$pipeline = | from $source
    | eval ...
    | into $destination;
```

The `from $source` command is the reference to the partition-selected subset. Splunk's current syntax documentation explicitly states that the subset for `from $source` is determined by the pipeline partition configured in the builder. [Official documentation](https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-enterprise/10.4/working-with-pipelines/edge-processor-pipeline-syntax)

---

# 3. Important correction to your comment

You currently have:

```spl
/*
A valid SPL2 statement for a pipeline must start with "$pipeline", and include "from $source"
and "into $destination".
*/
```

That comment is too strict.

The **pipeline statement** contains:

```spl
$pipeline = ...
```

but the complete SPL2 module can have an `import` statement before it.

Your own route pipeline correctly starts with:

```spl
import route from /splunk/ingest/commands
```

and then:

```spl
$pipeline = ...
```

Splunk's current route documentation uses exactly this pattern:

```spl
import route from /splunk/ingest/commands
$pipeline = | from $source
...
```

So a better comment is:

```spl
/*
An Edge Processor pipeline contains a $pipeline statement
with from $source and into $destination.
Additional import statements may appear before the $pipeline statement.
*/
```

---

# 4. The complete event-flow model

Think about the pipeline as a series of decisions.

```text
SOURCE
  |
  v
PARTITION
  |
  v
EARLY FILTER
  |
  v
EXTRACTION
  |
  v
REDUCTION
  |
  v
TRANSFORMATION
  |
  v
ENRICHMENT
  |
  v
ROUTING / COPYING
  |
  v
DESTINATION
```

Not every pipeline needs every stage.

The correct question is:

> What does this event need to become before it is sent to the destination?

---

# 5. The most important command classification

## Input and output

```text
from
into
```

## Filter or reduce event count

```text
where
dedup
stats
```

## Parse or extract

```text
rex
spath
```

## Transform an event

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

## Split or copy data flow

```text
route
thru
branch
```

## Expand structured data

```text
expand
flatten
mvexpand
```

This classification is more useful than memorizing syntax without understanding the event flow.

---

# 6. `where` — remove events from the pipeline

## Concept

Use `where` when your question is:

> Should this event continue?

The expression must evaluate to TRUE for the event to continue.

Example:

```spl
| where profile == "linux"
```

means:

```text
profile=linux      -> KEEP
profile=dns        -> DROP
profile=firewall   -> DROP
```

Splunk's current SPL2 documentation states that `where` removes results that do not satisfy the predicate. In Edge Processor pipelines, data that does not match the predicate is not sent to the `into` destination; depending on the partition/default-destination behavior, the data may be dropped or sent to the default destination. [Official documentation](https://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/where-command/where-command-overview-syntax-and-usage)

## Example

```spl
$pipeline = | from $source
    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/
    | where profile == "linux"
    | into $destination;
```

Input:

```text
{"profile":"linux","action":"login"}
{"profile":"dns","action":"query"}
{"profile":"firewall","action":"allow"}
```

Output:

```text
{"profile":"linux","action":"login"}
```

The other two events do not continue.

---

# 7. `where` for a deny rule

If you want:

> Keep everything except debug.

Use:

```spl
| where action != "debug"
```

Example:

```text
login    -> KEEP
logout   -> KEEP
query    -> KEEP
debug    -> DROP
```

This is usually clearer than routing the debug events to a destination that you do not actually need.

---

# 8. `where` for an allow list

Suppose only these event types are useful:

```text
linux
dns
firewall
```

Then:

```spl
| where profile IN ("linux", "dns", "firewall")
```

Data flow:

```text
linux       -> KEEP
dns         -> KEEP
firewall    -> KEEP
windows     -> DROP
test        -> DROP
unknown     -> DROP
```

This is a good pattern when you know exactly what you want to retain.

---

# 9. `where` before expensive processing

Consider this:

```spl
| rex ...
| rex ...
| spath ...
| eval ...
| eval ...
| where action != "debug"
```

A debug event receives all that processing before being removed.

If the debug condition can be identified reliably from the raw event, a better design can be:

```spl
| where NOT match(_raw, /"action"\s*:\s*"debug"/i)
| rex ...
| spath ...
| eval ...
```

Now the unwanted event is removed before the later processing.

The principle is:

```text
cheap high-value filtering
        |
        v
more expensive processing
```

But do not force every filter into a complicated regex. If a field is already available, use the field.

---

# 10. Partition versus `where`

These are different.

## Partition

Configured in the Edge Processor builder.

Question:

> What subset should this pipeline process?

Example:

```text
host = server01
sourcetype = firewall
```

This defines the pipeline's input scope.

## `where`

Written in the pipeline.

Question:

> Which events inside this pipeline should continue?

Example:

```spl
| where action != "debug"
```

So:

```text
PARTITION
"Which data enters this pipeline?"

WHERE
"Which events continue inside this pipeline?"
```

Current Splunk documentation explicitly separates partition configuration from pipeline filtering. [Official documentation](https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/process-data-using-pipelines/filter-and-mask-data-using-an-edge-processor)

---

# 11. Important current behavior of `where`

Splunk changed how `where` clauses immediately following `from $source` are interpreted.

Before the January 22, 2024 update, some such clauses were treated as partition conditions.

Current Edge Processor behavior treats these as filters in the pipeline body, so excluded data is dropped rather than being automatically sent to the default destination. Splunk recommends moving intended partition conditions into the partition configuration. [Official troubleshooting documentation](https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/troubleshooting/troubleshoot-the-edge-processor-solution)

Therefore, do not rely on:

```spl
$pipeline = | from $source
    | where host="server01"
```

as a substitute for configuring the pipeline partition.

Configure:

```text
Partition:
host = server01
```

in the builder.

Then use pipeline-level `where` for actual event filtering.

---

# 12. `rex` — extract values from `_raw`

Suppose:

```text
_raw = {"profile":"linux","action":"login"}
```

Use:

```spl
| rex field=_raw /"profile":"(?P<profile>[^"]+)"/
```

Now conceptually:

```text
_raw
profile = linux
```

The field can be used later:

```spl
| where profile == "linux"
```

or:

```spl
| route profile == "linux", [
    | into $linux_destination
]
```

Splunk's Edge Processor documentation uses this workflow: extract the value into a field and then use the extracted field for filtering/routing. [Official documentation](https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/process-data-using-pipelines/filter-and-mask-data-using-an-edge-processor)

---

# 13. `spath` — structured extraction

If the event is structured JSON/XML, consider `spath`.

Example:

```json
{
  "profile": "linux",
  "user": {
    "name": "teja",
    "role": "admin"
  }
}
```

The purpose of `spath` is to work with structured data rather than manually writing regular expressions for every field.

Use:

```text
rex
    for regex-oriented extraction

spath
    for structured JSON/XML extraction
```

Do not use regex for structured parsing just because regex can technically parse it.

---

# 14. `eval` — create or change fields

Examples:

```spl
| eval severity="high"
```

```spl
| eval total_bytes = bytes_in + bytes_out
```

```spl
| eval category = if(status >= 500, "server_error", "normal")
```

Your masking pattern is also `eval`:

```spl
| eval _raw=replace(
    _raw,
    /CreditCard=[0-9]+/,
    "CreditCard=XXXXXXXXXXX"
)
```

---

# 15. `replace()` — mask or rewrite text

The `replace()` function is used inside `eval`.

Example:

```spl
| eval _raw=replace(
    _raw,
    /CreditCard=[0-9]+/,
    "CreditCard=XXXXXXXXXXX"
)
```

Input:

```text
user=teja CreditCard=4111111111111111
```

Output:

```text
user=teja CreditCard=XXXXXXXXXXX
```

For JSON:

```spl
| eval _raw=replace(
    _raw,
    /"password":"[^"]*"/,
    "\"password\":\"xxxxxxxx\""
)
```

Input:

```json
{"user":"teja","password":"Secret123"}
```

Output:

```json
{"user":"teja","password":"xxxxxxxx"}
```

Keep the field name unchanged when the goal is masking only.

---

# 16. Your current masking rule has a correction

You currently have:

```spl
| eval _raw=replace(
    _raw,
    /"credit_card":"[0-9]+"/,
    "card_number=XXXXXXXXXXX"
)
```

This changes:

```text
"credit_card":"123456789"
```

into:

```text
card_number=XXXXXXXXXXX
```

That changes the structure as well as the value.

If you only want masking:

```spl
| eval _raw=replace(
    _raw,
    /"credit_card":"[0-9]+"/,
    "\"credit_card\":\"XXXXXXXXXXX\""
)
```

Now:

```text
"credit_card":"123456789"
```

becomes:

```text
"credit_card":"XXXXXXXXXXX"
```

---

# 17. `route` — send a subset down another path

Use `route` when your question is:

> Does this subset need a different path or destination?

Syntax:

```spl
| route <predicate>, [
    | into $destination2
]
```

Splunk's current documentation gives the syntax:

```spl
| route <predicate>, [ | into $destination2 ]
```

and explains that `route` creates an additional path, selects a subset, and diverts that subset to the new path. [Official documentation](https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/route-data-using-pipelines/process-a-subset-of-data-using-an-edge-processor)

---

# 18. Example: Linux to another destination

```spl
| route profile == "linux", [
    | into $linux_destination
]
| into $default_destination;
```

Data flow:

```text
              events
                |
              route
             /     \
         linux     other
           |         |
           v         v
        linux      default
      destination destination
```

The Linux subset is diverted.

The other events continue through the main path.

---

# 19. Multiple `route` commands

For your use case:

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

Flow:

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

Splunk explicitly documents that `route`, `thru`, and `branch` can be used multiple times and can be nested or sequential. [Official documentation](https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/route-data-using-pipelines/routing-data-in-the-same-edge-processor-pipeline-to-different-actions-and-destinations)

---

# 20. Important correction in your current pipeline: `index=linux`

You wrote:

```spl
| eval index=linux
```

Use:

```spl
| eval index="linux"
```

because `linux` here is intended to be a literal string value.

For example:

```spl
| route profileTest == "linux", [
    | eval index="linux"
    | into $destination2
]
```

This is the clearer and correct form for assigning a literal index name.

---

# 21. `thru` — make an additional copy and continue

Use `thru` when you want:

> An extra copy goes somewhere, but the original event continues through the main path.

Example:

```spl
| thru [
    | into $archive_destination
]
| into $main_destination;
```

Flow:

```text
                 event
                   |
                  thru
                /     \
             COPY     ORIGINAL
               |          |
               v          v
            archive     continue
                           |
                           v
                         main
```

Splunk describes `thru` as creating an additional path, sending a complete copy there, and allowing the original data to continue downstream. [Official documentation](https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-enterprise/10.4/route-data-using-pipelines/process-a-copy-of-data-using-an-edge-processor)

---

# 22. The position of `thru` is important

Consider:

```spl
| mask
| thru [
    | into $archive
]
| route profile == "linux", [...]
```

The archive receives the events after masking and before routing.

Conceptually:

```text
mask
 |
 +--> archive
 |
 +--> routes
```

Now compare:

```spl
| mask
| route profile == "linux", [...]
| thru [
    | into $archive
]
| into $default;
```

Now Linux events have already been diverted.

The `thru` only sees the remaining main-path data.

Therefore:

> A `thru` copies whatever reaches the point where `thru` is placed.

---

# 23. `branch` — create multiple complete paths

Use `branch` when you want:

> Every event entering the branch should be copied into multiple independent paths.

Example:

```spl
$pipeline = | from $source
    | branch
        [
            | into $destination1
        ],
        [
            | into $destination2
        ];
```

Flow:

```text
              ALL EVENTS
                   |
                branch
               /      \
              /        \
             v          v
         destination1 destination2
```

Splunk documents that `branch` creates two or more paths and sends a complete copy of the data to each path. [Official documentation](https://help.splunk.com/en/splunk-enterprise/process-data-at-the-edge/use-edge-processors-for-splunk-enterprise/10.2/route-data-using-pipelines/process-multiple-copies-of-data-using-an-edge-processor)

---

# 24. `route` vs `thru` vs `branch`

| Command | Event flow | Use it when |
|---|---|---|
| `route` | Diverts matching subset | A subset belongs on another path |
| `thru` | Copies the current stream and original continues | You need an extra copy |
| `branch` | Copies the whole input into multiple paths | You need multiple independent full paths |

Remember:

```text
route
    subset -> another path

thru
    copy + original continues

branch
    every event -> multiple paths
```

---

# 25. Example: route + thru

Suppose:

> Linux events go to the Linux destination. Other events stay on the main path. Also archive the remaining main-path data.

```spl
| route profile == "linux", [
    | into $linux_destination
]

| thru [
    | into $archive_destination
]

| into $default_destination;
```

Flow:

```text
ALL
 |
route linux
 |\
 | \ 
 |  +---- remaining
 |
 +---- Linux -> linux_destination

remaining
    |
   thru
  /    \
copy   original
 |         |
 v         v
archive   default
```

This example is useful because it shows that `thru` acts on the data that reaches it.

---

# 26. `branch` + `route`

A common advanced pattern is:

> Create a sanitized copy for the archive, while another copy is independently routed.

Example:

```spl
import route from /splunk/ingest/commands

$pipeline = | from $source

    | eval _raw=replace(
        _raw,
        /"password":"[^"]*"/,
        "\"password\":\"xxxxxxxx\""
    )

    | rex field=_raw /"profile":"(?P<profile>[^"]+)"/

    | branch
        [
            | route profile == "linux", [
                | into $linux_destination
            ]
            | route profile == "dns", [
                | into $dns_destination
            ]
            | into $default_destination
        ],
        [
            | into $archive_destination
        ];
```

Flow:

```text
                 MASKED DATA
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
```

Because masking happens before the branch, both copies contain the masked version.

---

# 27. `dedup` — remove duplicate events

`dedup` answers a different question:

> Have I already seen an event with this same field-value combination?

Example:

```spl
| dedup event_id
```

Input:

```text
event_id=1001
event_id=1001
event_id=1002
event_id=1003
event_id=1003
```

Result:

```text
event_id=1001
event_id=1002
event_id=1003
```

The current SPL2 `dedup` documentation defines duplicate identity from the specified fields.

---

# 28. Multiple-field `dedup`

```spl
| dedup host, action
```

Uniqueness is based on:

```text
host + action
```

Example:

```text
server1 + login
server1 + login
server1 + logout
server2 + login
server2 + login
```

becomes:

```text
server1 + login
server1 + logout
server2 + login
```

---

# 29. Choose the dedup key carefully

Good candidates include identifiers such as:

```text
event_id
request_id
transaction_id
message_id
```

Bad generic choice:

```spl
| dedup host
```

unless your actual requirement is:

> Keep only one event for every host.

Otherwise you could discard legitimate events.

Also be cautious with:

```spl
| dedup _raw
```

on large streams. The event text itself may need to be retained for duplicate detection, which can increase memory use.

---

# 30. Do not dedup after masking if `_raw` defines uniqueness

Suppose:

```text
request_id=1001 password=ABC
request_id=1001 password=XYZ
```

After masking:

```text
request_id=1001 password=XXXX
request_id=1001 password=XXXX
```

The events become even more similar.

Therefore, if you have a real identifier:

```spl
| dedup request_id
```

is normally better than:

```spl
| dedup _raw
```

---

# 31. `stats` — summarize events

`stats` is not ordinary filtering.

It combines many events into aggregate results.

Example:

```spl
| stats count() BY profile
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

Result:

```text
profile      count
linux          3
dns            2
firewall       1
```

You no longer have the original six events.

You have summary results.

Current Edge Processor documentation specifically describes aggregation as a way to reduce event volume. [Official documentation](https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/process-data-using-pipelines/aggregate-event-data-using-edge-processor)

---

# 32. `dedup` vs `stats`

```text
dedup
    remove duplicate events

stats
    combine many events into summaries
```

Example:

```spl
| dedup request_id
```

keeps a representative event for each duplicate key.

Example:

```spl
| stats count() BY host
```

creates a new summary result for each host.

---

# 33. When should you use `stats`?

Use it when the downstream system needs a summary rather than every raw event.

Example:

```text
Need:
"bytes sent by host"

Do:
stats sum(bytes) BY host
```

Do not use it when the SOC needs individual evidence such as:

```text
authentication event
process creation event
network connection event
file creation event
```

unless you intentionally want only aggregated information.

---

# 34. `if` — conditional processing

`if` is useful when you want different processing but do not necessarily need separate destinations.

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

Conceptually:

```text
               all events
                   |
                   v
                  if
              /    |    \
           linux  dns  firewall
             |     |      |
             +-----+------+
                    |
                    v
              same destination
```

The event is being processed differently, not necessarily sent through a different destination.

Splunk documents the current SPL2 `if` command as conditional processing with `if`, `elseif`, and `else` paths. [Official documentation](https://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/if-command/if-command-overview-syntax-and-usage)

---

# 35. `if` vs `route`

Use `if` when you want:

```text
"How should this event be changed?"
```

Use `route` when you want:

```text
"Where should this subset go?"
```

Example:

```spl
if profile == "linux"
    -> eval index="linux"
```

versus:

```spl
route profile == "linux"
    -> into $linux_destination
```

---

# 36. `fields` — remove fields, not events

Example:

```spl
| fields - profile, action
```

This does not remove the event.

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

So:

```text
where
    removes unwanted EVENTS from the continuing pipeline

fields
    removes unwanted FIELDS from an event
```

---

# 37. Temporary field cleanup

You might extract a field only for routing:

```spl
| rex field=_raw /"profile":"(?P<profile>[^"]+)"/
```

and then use:

```spl
| route profile == "linux", [
    | ...
]
```

If the field is not useful downstream, you can remove it on the appropriate path with `fields`.

Do not remove it if the destination needs it.

---

# 38. `rename` — change field names

Example:

```spl
| rename profileTest AS profile
```

Multiple renames:

```spl
| rename profileTest AS profile, action2 AS action
```

Use this when the field name should change without changing the underlying event meaning.

---

# 39. `lookup` — enrich data

Suppose:

```text
src_ip=10.10.10.20
```

A lookup dataset contains:

```text
ip            asset_type
10.10.10.20   server
10.10.10.30   firewall
```

A lookup can enrich the event:

```text
src_ip=10.10.10.20
asset_type=server
```

Then:

```text
src_ip
  |
lookup
  |
asset_type
  |
route
```

This is useful when the classification needed for routing is not present in the original event.

The lookup dataset has to be imported/configured for the pipeline. [Official documentation](https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/working-with-pipelines/edge-processor-pipeline-syntax)

---

# 40. `decrypt`

Conceptually:

```text
encrypted field
      |
      v
   decrypt
      |
      v
decrypted field
      |
      v
mask / transform / route
```

This is a specialized command for data that is already encrypted.

It is not a general-purpose encryption command.

---

# 41. `ocsf`

`ocsf` is for converting supported event data into the Open Cybersecurity Schema Framework structure.

Conceptually:

```text
vendor-specific event
        |
        v
      ocsf
        |
        v
normalized security event
```

Use it when you have a real need for schema normalization [using a common field structure].

---

# 42. `expand`, `flatten`, `mvexpand`

These are structural commands and should not be confused.

## `expand`

Used for arrays of objects.

Concept:

```json
[
  {"name":"A","ip":"10.0.0.1"},
  {"name":"B","ip":"10.0.0.2"}
]
```

## `flatten`

Used to promote first-level object values into fields.

Concept:

```text
{
  "name":"server01",
  "ip":"10.0.0.1"
}
```

becomes conceptually:

```text
name=server01
ip=10.0.0.1
```

## `mvexpand`

Works on multivalue fields.

Concept:

```text
iplist = 10.0.0.1, 10.0.0.2, 10.0.0.3
```

can become separate results.

Mental model:

```text
expand
    array/object structure -> expanded results

flatten
    object -> fields

mvexpand
    multivalue field -> multiple results
```

---

# 43. A complete pipeline design for your current use case

Requirement:

> 1. Select the correct broad data set for the pipeline.
> 2. Drop unwanted events early.
> 3. Extract routing fields.
> 4. Deduplicate only when a real duplicate key exists.
> 5. Mask sensitive values.
> 6. Route Linux, DNS, and Firewall separately.
> 7. Send everything else to a default destination.

### Step 1 — configure the partition in the builder

Example:

```text
Partition:
sourcetype = my_security_logs
```

That is configured in the Edge Processor UI.

### Step 2 — pipeline

```spl
import route from /splunk/ingest/commands

$pipeline = | from $source

    /* 1. Early filtering */
    | where NOT match(_raw, /"action"\s*:\s*"debug"/i)
    | where NOT match(_raw, /"profile"\s*:\s*"test"/i)

    /* 2. Extract only what is needed */
    | rex field=_raw /"profile"\s*:\s*"(?P<profile>[^"]+)"/
    | rex field=_raw /"action"\s*:\s*"(?P<action>[^"]+)"/

    /* 3. Dedup only if the event has a real duplicate key */
    /* | dedup request_id */

    /* 4. Global masking */
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

    /* 5. Linux */
    | route profile == "linux", [
        | eval index="linux"
        | into $destination2
    ]

    /* 6. DNS */
    | route profile == "dns", [
        | eval index="dns"
        | into $destination3
    ]

    /* 7. Firewall */
    | route profile == "firewall", [
        | eval index="firewall"
        | into $destination4
    ]

    /* 8. Remaining events */
    | into $destination;
```

---

# 44. What happens to one event?

Suppose the input is:

```json
{
  "profile":"linux",
  "action":"login",
  "password":"Secret123",
  "credit_card":"4111111111111111"
}
```

### Stage 1 — partition

The event must belong to the pipeline's configured partition.

### Stage 2 — early filtering

It is not:

```text
action=debug
```

so it survives.

### Stage 3 — extraction

The pipeline creates:

```text
profile=linux
action=login
```

### Stage 4 — masking

The event becomes:

```json
{
  "profile":"linux",
  "action":"login",
  "password":"xxxxxxxx",
  "credit_card":"XXXXXXXXXXX"
}
```

### Stage 5 — routing

```text
profile == linux
```

is TRUE.

Therefore the event is diverted to:

```text
$destination2
```

and does not continue down the later main-path routes.

---

# 45. What happens to a DNS event?

Input:

```json
{
  "profile":"dns",
  "action":"query",
  "password":"Secret123"
}
```

After extraction:

```text
profile=dns
```

After masking:

```text
password=xxxxxxxx
```

First route:

```text
profile == linux
```

FALSE, so it remains on the main path.

Second route:

```text
profile == dns
```

TRUE.

Therefore:

```text
destination3
```

It is diverted there and does not reach the remaining main-path destination.

---

# 46. What happens to an unknown event?

Input:

```json
{
  "profile":"application",
  "action":"login"
}
```

Linux route:

```text
FALSE
```

DNS route:

```text
FALSE
```

Firewall route:

```text
FALSE
```

It reaches:

```spl
| into $destination
```

Therefore:

```text
unknown -> default destination
```

This is why you should normally have a clear default path.

---

# 47. Dropping a selected event class

Suppose you want to drop:

```text
profile=test
```

and you can identify it from raw data.

Use:

```spl
| where NOT match(_raw, /"profile"\s*:\s*"test"/i)
```

Then:

```text
test     -> DROP
linux    -> continue
dns      -> continue
firewall -> continue
```

This is usually better than sending `test` to a destination you do not need.

---

# 48. Keeping only one class

If you want:

```text
Linux -> keep
Everything else -> drop
```

use:

```spl
| rex field=_raw /"profile"\s*:\s*"(?P<profile>[^"]+)"/
| where profile == "linux"
| into $destination;
```

Do not use `route` unless you actually need another path.

---

# 49. Sending one class somewhere else

If you want:

```text
Linux -> destination2
Everything else -> destination
```

use:

```spl
| route profile == "linux", [
    | into $destination2
]
| into $destination;
```

---

# 50. Dropping one class but keeping another class on a separate destination

Example:

```text
debug   -> DROP
linux   -> destination2
other   -> destination
```

A good design is:

```spl
| where NOT match(_raw, /"action"\s*:\s*"debug"/i)

| rex field=_raw /"profile"\s*:\s*"(?P<profile>[^"]+)"/

| route profile == "linux", [
    | into $destination2
]

| into $destination;
```

Flow:

```text
                 ALL EVENTS
                     |
                     v
               drop debug
                /       \
             debug     remaining
              |            |
             DROP          route
                         /       \
                     linux       other
                       |           |
                       v           v
                    dest2       dest
```

This is a very clean example of **filter first, route second**.

---

# 51. Global masking and routing

Your original goal was:

> Mask the data globally, then route by profile.

A good design is:

```text
             partition
                 |
                 v
          early filtering
                 |
                 v
             extraction
                 |
                 v
            global mask
                 |
                 v
              routing
```

Why?

Because the same sanitized [sensitive values replaced or removed] version is then used by every later path.

---

# 52. If an archive copy is required

Use `thru`:

```spl
| eval _raw=replace(...)
| thru [
    | into $archive_destination
]
| route profile == "linux", [
    | into $linux_destination
]
| into $default_destination;
```

Now the archive receives the masked event before routing changes which path receives it.

---

# 53. If multiple complete copies are required

Use `branch`.

Example:

```spl
| eval _raw=replace(...)
| branch
    [
        | route profile == "linux", [
            | into $linux_destination
        ]
        | into $default_destination
    ],
    [
        | into $archive_destination
    ];
```

Now:

```text
sanitized event
      |
   branch
   /    \
  /      \
main    archive
 |
route
```

---

# 54. Choosing the best command

Use this decision table.

| Requirement | Use |
|---|---|
| "Should this event remain?" | `where` |
| "Remove duplicates?" | `dedup` |
| "Turn many events into a summary?" | `stats` |
| "Send this subset somewhere else?" | `route` |
| "Make an extra copy and continue?" | `thru` |
| "Create multiple complete paths?" | `branch` |
| "Change a field/value?" | `eval` |
| "Mask text in `_raw`?" | `eval` + `replace()` |
| "Extract from raw text?" | `rex` |
| "Parse structured JSON/XML?" | `spath` |
| "Remove fields?" | `fields` |
| "Rename fields?" | `rename` |
| "Change processing based on condition?" | `if` |
| "Add external information?" | `lookup` |
| "Normalize supported data?" | `ocsf` |
| "Decrypt supported data?" | `decrypt` |

---

# 55. Event-count thinking

Another useful mental model is asking:

> Does this command remove events, preserve them, or create copies?

## Usually reduces the number of events

```text
where
dedup
stats
```

## Can create additional output/copies

```text
branch
thru
expand
mvexpand
```

## Normally transforms the existing events

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

## Changes path rather than intentionally making a second copy

```text
route
```

This helps explain why command placement matters.

---

# 56. The "early reduction" design rule

When designing a high-volume pipeline, ask:

1. Can I discard something now?
2. Can I avoid extracting a field I do not need?
3. Can I avoid processing events that will later be discarded?
4. Do I really need duplicate copies?
5. Do I really need every raw event downstream?

A strong general pattern is:

```text
PARTITION
    ↓
EARLY FILTER
    ↓
MINIMAL EXTRACTION
    ↓
DEDUP (only when justified)
    ↓
MASK
    ↓
TRANSFORM / ENRICH
    ↓
AGGREGATE (only when raw detail is not required)
    ↓
ROUTE / COPY
    ↓
DESTINATION
```

This is a design guide, not a strict command order.

Security and correctness requirements can require an earlier transformation.

---

# 57. Why masking can belong before `branch` or `thru`

Suppose the original event contains:

```text
password=Secret123
```

If you branch first:

```spl
| branch
    [
        | eval _raw=replace(...)
        | into $destination1
    ],
    [
        | into $destination2
    ];
```

the second path receives the unmasked copy.

That is not what you want when both destinations must receive sanitized data.

Prefer:

```spl
| eval _raw=replace(...)
| branch
    [
        | into $destination1
    ],
    [
        | into $destination2
    ];
```

Now both paths receive the masked event.

---

# 58. Why `dedup` and `stats` should not be used automatically

Both can reduce downstream volume, but they can also remove information.

Use:

```spl
| dedup request_id
```

only when duplicate identity is clearly defined.

Use:

```spl
| stats count() BY host
```

only when a summary is sufficient.

Do not optimize by destroying data that the SOC or downstream application needs.

---

# 59. Destination design

A destination is the place where processed data is sent.

Do not automatically create one destination per index.

For example, if multiple streams are going to the same Splunk platform deployment, the design may be:

```text
                one Splunk destination
                     |
         +-----------+-----------+
         |           |           |
      linux        dns       firewall
       index       index        index
```

You may instead need multiple destinations when the data truly goes to different external endpoints.

For example:

```text
Splunk deployment A
Splunk deployment B
Amazon S3
```

These are genuine destination differences.

---

# 60. `index` is not the same as `destination`

This is very important.

```text
destination
    = where the pipeline sends data

index
    = where the data is stored within a Splunk platform destination
```

Therefore:

```spl
| eval index="linux"
```

does not mean:

```text
"send the event to the Linux server"
```

It means:

```text
"set the event's index value to linux"
```

The actual destination is still controlled by:

```spl
| into $destination
```

or another destination variable.

---

# 61. Current index-routing caution

For Splunk platform destinations, the final index can depend on the destination protocol and the index precedence [the order of rules used to decide the final index].

Therefore, always check the current destination-specific documentation before assuming:

```spl
| eval index="linux"
```

is the only index-setting mechanism.

---

# 62. Default destination and accidental data loss

Splunk recommends configuring a default destination to prevent unintended data loss.

This matters because there are two different ideas:

```text
Intentional filtering
    |
    v
where
```

versus:

```text
Data not processed by an applicable pipeline / no default destination
    |
    v
may be dropped
```

These should never be confused.

Current Splunk documentation explicitly warns to configure a default destination to avoid unintended data loss. [Official documentation](https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/process-data-using-pipelines/filter-and-mask-data-using-an-edge-processor)

---

# 63. A complete architecture for your lab

Use this as your default design pattern:

```text
                        INPUT
                          |
                          v
                    PIPELINE PARTITION
                          |
                          v
                    SELECTED INPUT
                          |
                          v
                 EARLY WHERE FILTERS
                    /           \
              unwanted        wanted
                 |               |
                DROP             v
                            MINIMAL EXTRACTION
                                  |
                                  v
                              DEDUP?
                                  |
                                  v
                             GLOBAL MASK
                                  |
                                  v
                          TRANSFORM / ENRICH
                                  |
                                  v
                              ROUTING
                       /         |          \
                      /          |           \
                   Linux        DNS       Firewall
                     |            |           |
                     v            v           v
                   dest1        dest2       dest3
                      \            |          /
                       \           |         /
                            remaining
                                |
                                v
                         default destination
```

If you also need an archive:

```text
                             MASKED DATA
                                  |
                               branch
                              /      \
                             /        \
                         MAIN        ARCHIVE
                          |
                        route
                   /       |       \
                 Linux     DNS   Firewall
                   |        |       |
                   v        v       v
                 dest1    dest2    dest3
```

---

# 64. Recommended learning order

Learn the pipeline behavior in this order:

## Stage 1 — Pipeline basics

```text
$pipeline
from $source
into $destination
partition
destination
```

## Stage 2 — Basic event transformation

```text
eval
replace()
rex
spath
fields
rename
```

## Stage 3 — Filtering and reduction

```text
where
dedup
stats
```

## Stage 4 — Data flow

```text
route
thru
branch
```

## Stage 5 — Conditional processing

```text
if
```

## Stage 6 — Enrichment and specialized processing

```text
lookup
ocsf
decrypt
```

## Stage 7 — Structured expansion

```text
expand
flatten
mvexpand
```

This order builds the concepts in the same order that the data moves through a real pipeline.

---

# 65. The final decision framework

When writing a new pipeline, ask the following questions in order.

### Question 1

**What data should this pipeline receive?**

Configure:

```text
Partition
```

### Question 2

**Which events do I never want to process?**

Use:

```spl
where
```

as early as reliable information allows.

### Question 3

**What information do I need to make later decisions?**

Use:

```text
rex
spath
```

and extract only what you need.

### Question 4

**Are there real duplicate events?**

Use:

```text
dedup
```

with a meaningful duplicate key.

### Question 5

**Do I need every original event, or only a summary?**

If summary is enough:

```text
stats
```

Otherwise retain the original events.

### Question 6

**Does the event need to be changed?**

Use:

```text
eval
replace
rename
fields
if
```

### Question 7

**Does the event need outside information?**

Use:

```text
lookup
```

### Question 8

**Does one subset need a different destination?**

Use:

```text
route
```

### Question 9

**Do I need an additional copy?**

Use:

```text
thru
```

### Question 10

**Do I need several complete independent copies?**

Use:

```text
branch
```

### Question 11

**Where should the remaining events go?**

Use:

```spl
| into $destination;
```

as the main/default path where appropriate.

---

# 66. Your exact pipeline, cleaned up

```spl
import route from /splunk/ingest/commands

/*
The Edge Processor pipeline receives the subset defined by
the pipeline partition configured in the Edge Processor builder.
*/

$pipeline = | from $source

    /* =========================================
       1. EARLY FILTERING
       Remove events that should not be processed.
       ========================================= */

    | where NOT match(_raw, /"action"\s*:\s*"debug"/i)
    | where NOT match(_raw, /"profile"\s*:\s*"test"/i)


    /* =========================================
       2. FIELD EXTRACTION
       Extract only fields needed later.
       ========================================= */

    | rex field=_raw /"profile"\s*:\s*"(?P<profile>[^"]+)"/
    | rex field=_raw /"action"\s*:\s*"(?P<action>[^"]+)"/


    /* =========================================
       3. OPTIONAL DEDUP
       Enable only when a real duplicate key exists.
       ========================================= */

    /* | dedup request_id */


    /* =========================================
       4. GLOBAL MASKING
       ========================================= */

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


    /* =========================================
       5. ROUTE LINUX
       ========================================= */

    | route profile == "linux", [
        | eval index="linux"
        | into $destination2
    ]


    /* =========================================
       6. ROUTE DNS
       ========================================= */

    | route profile == "dns", [
        | eval index="dns"
        | into $destination3
    ]


    /* =========================================
       7. ROUTE FIREWALL
       ========================================= */

    | route profile == "firewall", [
        | eval index="firewall"
        | into $destination4
    ]


    /* =========================================
       8. DEFAULT MAIN PATH
       Events not diverted by previous routes.
       ========================================= */

    | into $destination;
```

---

# 67. The complete mental model

```text
PARTITION
    |
    |  "What input belongs to this pipeline?"
    v
FROM $SOURCE
    |
    v
WHERE
    |
    |  "Should this event continue?"
    |
    +------ NO ------> DROP / not sent to this pipeline destination
    |
    v
REX / SPATH
    |
    |  "What fields do I need?"
    v
DEDUP
    |
    |  "Is this event a duplicate?"
    v
MASK / TRANSFORM
    |
    |  "How should I change it?"
    v
LOOKUP / OCSF / DECRYPT
    |
    |  "Do I need enrichment or special processing?"
    v
STATS
    |
    |  "Do I need a summary instead of every raw event?"
    v
ROUTE
    |
    |  "Where should subsets go?"
    |
    +---- Linux ------> destination2
    |
    +---- DNS --------> destination3
    |
    +---- Firewall ---> destination4
    |
    +---- Remaining --> main path
                          |
                          v
                        INTO
```

---

# 68. The five concepts that explain most Edge Processor pipelines

If you understand these five, most pipeline designs become straightforward:

```text
1. PARTITION
   Defines the pipeline's input scope.

2. WHERE
   Removes events that should not continue.

3. TRANSFORM
   Changes or extracts event information.

4. ROUTE
   Diverts a subset to another path.

5. INTO
   Sends each completed path to a destination.
```

Then add:

```text
dedup
    when duplicates must be removed

stats
    when many events should become summaries

thru
    when an extra copy is needed

branch
    when several complete copies are needed

if
    when different processing is needed without necessarily changing destination
```

---

# 69. Official Splunk documentation

Use these as the primary references for this chapter:

- Edge Processor pipeline syntax  
  https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-enterprise/10.4/working-with-pipelines/edge-processor-pipeline-syntax

- Filter and mask data using an Edge Processor  
  https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/process-data-using-pipelines/filter-and-mask-data-using-an-edge-processor

- Process a subset of data using an Edge Processor (`route`)  
  https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/route-data-using-pipelines/process-a-subset-of-data-using-an-edge-processor

- Routing data in the same Edge Processor pipeline  
  https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/route-data-using-pipelines/routing-data-in-the-same-edge-processor-pipeline-to-different-actions-and-destinations

- Process a copy of data using an Edge Processor (`thru`)  
  https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-enterprise/10.4/route-data-using-pipelines/process-a-copy-of-data-using-an-edge-processor

- Process multiple copies of data using an Edge Processor (`branch`)  
  https://help.splunk.com/en/splunk-enterprise/process-data-at-the-edge/use-edge-processors-for-splunk-enterprise/10.2/route-data-using-pipelines/process-multiple-copies-of-data-using-an-edge-processor

- SPL2 `where` command  
  https://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/where-command/where-command-overview-syntax-and-usage

- SPL2 `dedup` command  
  https://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/dedup-command/dedup-command-overview-syntax-and-usage

- SPL2 `if` command  
  https://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/if-command/if-command-overview-syntax-and-usage

- Aggregate event data using Edge Processor  
  https://help.splunk.com/en/data-management/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/process-data-using-pipelines/aggregate-event-data-using-edge-processor

- Troubleshoot Edge Processor partition/filter behavior  
  https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/troubleshooting/troubleshoot-the-edge-processor-solution

---

# 70. Final rules to remember

```text
PARTITION
    configured in the builder
    -> defines what the pipeline processes

WHERE
    -> removes events from the continuing pipeline

REX / SPATH
    -> obtain fields needed for decisions

DEDUP
    -> removes duplicate events based on specified fields

STATS
    -> turns many events into aggregate results

EVAL / REPLACE
    -> changes event data

ROUTE
    -> diverts a subset to another path

THRU
    -> makes an additional copy while the original continues

BRANCH
    -> makes multiple complete paths

IF
    -> performs conditional processing

INTO
    -> sends the path to its configured destination
```

The strongest default design for a high-volume transformation pipeline is:

```text
PARTITION
    ↓
EARLY WHERE
    ↓
MINIMAL EXTRACTION
    ↓
DEDUP (only if justified)
    ↓
MASK / TRANSFORM
    ↓
ENRICH / NORMALIZE if required
    ↓
ROUTE / COPY
    ↓
INTO
```

The exact order should always follow the actual data and the required result. The goal is not to use every command; the goal is to use the smallest number of correct operations in the right places.
