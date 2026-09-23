# Splunk SPL + SPL2

# Chapter 5 — Grouping and Aggregation

In Chapters 1–4, we learned how to identify events, search their fields, work with text patterns, and shape the results.

So far, most searches have answered questions such as:

> "Show me the PowerShell events."

Real SOC questions usually go one level higher:

> How many PowerShell executions happened on each host?  
> Which user generated the most failed logons?  
> How many unique source IPs contacted each server?  
> What was the total amount of data transferred by each host?  
> Which hosts produced the highest number of authentication failures?

These questions are not mainly about individual events.

They are about **summarizing many events into a smaller result set**.

That is what aggregation [aggregation: combining many events to calculate a smaller set of summary values] is for.

In LogScale, the equivalent idea is commonly expressed with `groupBy()`.

In Splunk, the central command is:

```text
stats
```

This chapter therefore teaches the same investigation concept using **traditional SPL and SPL2**, rather than copying the LogScale `groupBy()` syntax.

---

# 5.1 Events vs Summaries

Suppose the events look like this:

```text
_time       host    FileName
--------------------------------
10:15:22    PC-001  powershell.exe
10:15:31    PC-001  powershell.exe
10:16:47    PC-009  powershell.exe
10:18:02    PC-001  powershell.exe
```

These are four events.

Now suppose the investigation question is:

> How many PowerShell executions happened on each host?

We do not need four separate event rows.

We want:

```text
host      count
----------------
PC-001    3
PC-009    1
```

The events have been transformed into a summary.

Conceptually:

```text
EVENTS
  many
    ↓
 AGGREGATION
    ↓
SUMMARY
  fewer
```

This is the key idea for this chapter.

---

# 5.2 The Splunk Equivalent of LogScale `groupBy()`

The original LogScale concept is:

```text
groupBy(ComputerName)
```

The Splunk equivalent is:

### Traditional SPL

```spl
index=windows
| stats count BY host
```

### SPL2 pipeline syntax

```spl2
FROM windows
| stats count() BY host
```

### SPL2 structured `FROM` syntax

```spl2
FROM windows
GROUP BY host
SELECT count(), host
```

All three mean:

> Group the incoming events by `host` and count them.

This is one of the most important SPL/SPL2 mappings in the course.

Do not memorize:

```text
LogScale groupBy()
→ Splunk stats
```

as a syntax rule only.

Understand the underlying operation:

```text
Many events
    ↓
Group by a key
    ↓
Perform a calculation per group
    ↓
Return summary rows
```

The current SPL2 documentation explicitly describes `stats` as an aggregation command and documents the `GROUP BY ... SELECT ...` form as an alternative to the pipeline form.  
Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/stats-command/stats-command-overview-syntax-and-usage

---

# 5.3 What Does `stats` Actually Do?

The most important Splunk aggregation command is:

```spl
stats
```

It calculates aggregate statistics over the incoming results.

For example:

```spl
index=windows
| stats count
```

This produces one row.

Why?

Because no `BY` clause was supplied.

Conceptually:

```text
ALL MATCHING EVENTS
        ↓
      count
        ↓
ONE SUMMARY ROW
```

Now add:

```spl
| stats count BY host
```

The result becomes one row for each distinct host.

```text
host      count
----------------
DC01      1200
DC02      850
WEB01     420
```

Splunk documents this behavior directly: without `BY`, `stats` returns one row for the entire incoming result set; with `BY`, it returns one row for each distinct grouping value.  
Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/stats-command/stats-command-overview-syntax-and-usage

---

# 5.4 SPL vs SPL2 `count`

There is an important syntax difference.

### Traditional SPL

```spl
| stats count BY host
```

In traditional SPL, the `count` aggregate can be written without parentheses.

### SPL2

```spl2
| stats count() BY host
```

In SPL2, the parentheses are required.

So remember:

```text
SPL
count

SPL2
count()
```

This is a real language difference documented by Splunk.

The same distinction matters when you use other functions because SPL2 uses the function-call form explicitly.

---

# 5.5 Grouping by Multiple Fields

Suppose the question is:

> How many PowerShell events happened for each host and user combination?

### Traditional SPL

```spl
index=windows
| stats count BY host user
```

### SPL2 pipeline

```spl2
FROM windows
| stats count() BY host, user
```

### SPL2 structured form

```spl2
FROM windows
GROUP BY host, user
SELECT count(), host, user
```

The result might be:

```text
host    user          count
---------------------------
PC-001  John          20
PC-001  Alice         4
PC-009  Alice         9
```

The grouping key is not merely `host`.

It is the combination:

```text
host + user
```

This means:

```text
PC-001 / John
PC-001 / Alice
PC-009 / Alice
```

are separate groups.

In traditional SPL, fields in the `BY` clause are space-separated.

In SPL2, the `BY` field list is comma-delimited.

This is another syntax difference documented in the current SPL2 `stats` reference.

---

# 5.6 `count` vs `count(field)`

These answer different questions.

### Count all incoming events

Traditional SPL:

```spl
| stats count BY host
```

SPL2:

```spl2
| stats count() BY host
```

Question:

> How many events are in each host group?

Now:

```spl
| stats count(CommandLine) BY host
```

or:

```spl2
| stats count(CommandLine) BY host
```

Question:

> How many events in each host group contain a value for `CommandLine`?

This difference matters.

Suppose:

```text
host    CommandLine
----------------------------
PC-001  powershell.exe
PC-001  cmd.exe
PC-001  [missing]
```

Then:

```text
count
```

counts all three events.

But:

```text
count(CommandLine)
```

counts the events where `CommandLine` has a value.

---

# 5.7 Distinct Count — How Many Different Values?

A very common SOC question is:

> How many different IP addresses contacted each host?

Counting events is not enough.

Suppose:

```text
host=WEB01
src_ip=10.0.0.5
```

appears 10,000 times.

That is:

```text
10,000 events
```

but only:

```text
1 unique IP
```

Traditional SPL:

```spl
index=network
| stats dc(src_ip) AS UniqueIPs BY host
```

You can also write:

```spl
| stats distinct_count(src_ip) AS UniqueIPs BY host
```

SPL2:

```spl2
FROM network
| stats dc(src_ip) AS UniqueIPs BY host
```

or:

```spl2
FROM network
| stats distinct_count(src_ip) AS UniqueIPs BY host
```

The current Splunk documentation defines `dc()` as the abbreviation for `distinct_count()`.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/aggregate-functions

So:

```text
count(src_ip)
    ↓
How many matching field values/events?

dc(src_ip)
    ↓
How many DIFFERENT values?
```

This distinction is fundamental to scanning, credential abuse, beaconing, and many other detections.

---

# 5.8 `dc()` vs `distinct_count()` vs `estdc()`

Splunk gives you several distinct-count options.

```text
distinct_count(field)
dc(field)
```

are equivalent names for distinct counting.

For very large datasets, Splunk also provides:

```text
estdc(field)
```

which is an **estimated distinct count** [estimated: calculated approximately rather than guaranteeing an exact result].

The trade-off is:

```text
distinct_count()
    ↓
Exact distinct count
    ↓
More memory/work

estdc()
    ↓
Estimated distinct count
    ↓
Lower resource use in appropriate cases
```

Splunk's current documentation specifically recommends considering `estdc` when distinct counting is consuming significant memory or runtime, particularly for a low-cardinality `BY` field or when no `BY` field is used.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/aggregate-functions

Do not replace exact distinct counts blindly. The choice depends on whether approximate results are acceptable for the use case.

---

# 5.9 `values()` — Which Distinct Values?

A count answers:

> How many?

Sometimes you also need:

> Which ones?

For that, Splunk provides:

```text
values(field)
```

Example:

### SPL

```spl
index=windows
| stats values(FileName) BY host
```

### SPL2

```spl2
FROM windows
| stats values(FileName) BY host
```

Suppose the events for `PC-001` contain:

```text
powershell.exe
cmd.exe
powershell.exe
whoami.exe
```

`values(FileName)` returns the distinct values:

```text
cmd.exe
powershell.exe
whoami.exe
```

The current Splunk documentation states that `values()` returns distinct values as a multivalue [multivalue: one field containing multiple values] result and that the ordering is lexicographical [lexicographical: ordered according to character sequence rather than numeric magnitude].

---

# 5.10 `list()` — Keep the Values in Event Order

Splunk also provides:

```text
list(field)
```

This is different from `values()`.

`list()` returns the values in the order of the incoming events.

For example:

```text
Event 1 → powershell.exe
Event 2 → cmd.exe
Event 3 → powershell.exe
Event 4 → whoami.exe
```

Then:

```spl
| stats list(FileName) BY host
```

can preserve:

```text
powershell.exe
cmd.exe
powershell.exe
whoami.exe
```

while:

```spl
| stats values(FileName) BY host
```

returns only the distinct values.

The current Splunk documentation notes that `list()` is limited to the first 100 values by default, while `values()` returns distinct values and orders them lexicographically.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/multivalue-and-array-functions

So remember:

```text
values()
    ↓
Which DISTINCT values?

list()
    ↓
Which values, in input order?
```

This is the closest useful Splunk replacement for the original chapter's discussion of LogScale `collect()`, but it is not a perfect one-to-one translation.

---

# 5.11 `collect()` vs Splunk `values()` / `list()`

The source material uses LogScale:

```text
collect()
```

Splunk does not use that same aggregation function for this purpose.

Instead, the closest event-value aggregation tools are:

```text
values(field)
list(field)
```

The choice depends on the question.

If you want:

```text
unique process names
```

use:

```spl
values(FileName)
```

If you want:

```text
the sequence of observed values
```

use:

```spl
list(FileName)
```

Do not assume that `collect()` and `values()` have identical behavior. The data model and output semantics are different.

---

# 5.12 `sum()` — Total

Now suppose you want to know:

> How many bytes did each host send?

Use:

### SPL

```spl
index=network
| stats sum(bytes_out) AS TotalBytes BY host
```

### SPL2

```spl2
FROM network
| stats sum(bytes_out) AS TotalBytes BY host
```

The result could be:

```text
host     TotalBytes
-------------------
WEB01    18234992
WEB02    442110
```

This is the same general pattern:

```text
GROUP
  ↓
host

CALCULATE
  ↓
sum(bytes_out)
```

---

# 5.13 `avg()`, `min()`, and `max()`

These work similarly.

### Average

```spl
| stats avg(bytes_out) AS AverageBytes BY host
```

### Minimum

```spl
| stats min(bytes_out) AS MinimumBytes BY host
```

### Maximum

```spl
| stats max(bytes_out) AS MaximumBytes BY host
```

SPL2:

```spl2
| stats avg(bytes_out) AS AverageBytes BY host
```

```spl2
| stats min(bytes_out) AS MinimumBytes BY host
```

```spl2
| stats max(bytes_out) AS MaximumBytes BY host
```

The basic questions are:

```text
avg()
→ What is typical?

min()
→ What is the smallest?

max()
→ What is the largest?
```

---

# 5.14 `range()` — Spread

Splunk also provides:

```text
range(field)
```

which returns:

```text
maximum - minimum
```

Example:

```spl
index=network
| stats range(bytes_out) AS ByteRange BY host
```

This is useful when you care about the spread of numeric values.

However, there is an important time-analysis distinction.

For elapsed time between the earliest and latest event, use the time-oriented functions:

```text
earliest()
latest()
earliest_time()
latest_time()
```

rather than assuming that `range(_time)` is always the best expression for every time question.

Time-based aggregation will be covered in more detail in Chapter 6.

---

# 5.15 Earliest, Latest, First, and Last

This is an important Splunk-specific section because it is easy to confuse these functions.

Splunk provides:

```text
earliest(field)
latest(field)
first(field)
last(field)
```

They do not all mean the same thing.

### `earliest()`

Returns the chronologically earliest value.

### `latest()`

Returns the chronologically latest value.

### `first()`

Returns the first value encountered according to the order in which the events are processed by the aggregation.

### `last()`

Returns the last value encountered according to the order in which the events are processed.

Therefore:

```text
first()
last()
```

are about **input order**.

Whereas:

```text
earliest()
latest()
```

are explicitly **time-oriented**.

For time-based investigations, this distinction matters greatly.

Splunk's current documentation specifically recommends `earliest()` and `latest()` when you want values based on event time; `first()` and `last()` should not be substituted when the question is chronological.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/time-functions

---

# 5.16 Finding the First and Last Time

You can also use:

```text
earliest(_time)
latest(_time)
```

to reason about the time range represented by a group.

For example:

### SPL

```spl
index=auth
| stats earliest(_time) AS FirstSeen latest(_time) AS LastSeen BY user
```

### SPL2

```spl2
FROM auth
| stats earliest(_time) AS FirstSeen, latest(_time) AS LastSeen BY user
```

Then you can calculate the duration between them later.

We will use this idea more heavily in Chapter 6.

---

# 5.17 Multiple Aggregations in One Search

Real SOC queries rarely ask only one question.

Suppose you want:

- number of events
- number of unique processes
- total bytes

You can calculate all of them in one `stats` command.

### Traditional SPL

```spl
index=network
| stats
    count AS Events
    dc(process) AS UniqueProcesses
    sum(bytes_out) AS TotalBytes
    BY host
```

### SPL2

```spl2
FROM network
| stats
    count() AS Events,
    dc(process) AS UniqueProcesses,
    sum(bytes_out) AS TotalBytes
    BY host
```

The result becomes:

```text
host    Events   UniqueProcesses   TotalBytes
------------------------------------------------
PC-001  842      37                18234992
PC-009  122      9                 442110
```

Notice the syntax difference:

```text
SPL
multiple aggregations can be separated by spaces

SPL2
multiple aggregations must be comma-delimited
```

The current SPL2 `stats` documentation explicitly documents this requirement.

---

# 5.18 Why `AS` Matters

Without an alias [alias: another name given to a field or result], Splunk may name an aggregation using its expression.

For example:

```spl
| stats sum(bytes_out) BY host
```

can produce a field named:

```text
sum(bytes_out)
```

That works, but it is not a convenient name for later searches.

Instead:

```spl
| stats sum(bytes_out) AS TotalBytes BY host
```

gives:

```text
TotalBytes
```

Then you can write:

```spl
| where TotalBytes > 1000000000
```

This is especially useful when building dashboards, detections, or longer pipelines.

---

# 5.19 Aggregation Results Can Be Filtered

This is one of the most important ideas from the original chapter.

Suppose:

```spl
index=network
| stats sum(bytes_out) AS TotalBytes BY host
```

returns:

```text
host     TotalBytes
-------------------
PC-001   1800000000
PC-002   20000000
PC-003   700000000
```

Now you can filter the **summary**:

```spl
| where TotalBytes > 1000000000
```

Result:

```text
PC-001   1800000000
```

So the pipeline becomes:

```text
FILTER EVENTS
      ↓
AGGREGATE
      ↓
FILTER SUMMARY
      ↓
SORT SUMMARY
```

This is an essential distinction.

Before `stats`, you are working with event-level fields.

After `stats`, you are working with the fields that the aggregation produced.

---

# 5.20 Why Fields Disappear After `stats`

Suppose the original event contains:

```text
_time
host
user
FileName
CommandLine
src_ip
```

Now run:

```spl
| stats count BY host
```

The result contains essentially:

```text
host
count
```

The original event fields are no longer available as ordinary fields in the summary.

That is why this will not work the way a beginner might expect:

```spl
| stats count BY host
| where FileName="powershell.exe"
```

The `FileName` field was not carried into the aggregation result.

Instead, filter first:

```spl
index=windows
| search FileName="powershell.exe"
| stats count BY host
```

This is one of the most important query-order rules in Splunk.

---

# 5.21 The Aggregation Boundary

Think of `stats` as creating a boundary:

```text
EVENT WORLD
────────────────────────
_time
host
user
FileName
CommandLine
src_ip
...
        │
        │ stats
        ▼
SUMMARY WORLD
────────────────────────
host
count
...
```

This is why Chapter 5 is different from Chapters 2–4.

In earlier chapters, you mostly manipulated individual events.

Now you are creating new summary rows.

Once `stats` runs, the data has a new shape.

---

# 5.22 `eventstats` — Aggregate Without Losing the Events

Here is one of the most important Splunk features that belongs in this chapter.

What if you want:

> Calculate an aggregate, but keep every original event?

That is what:

```text
eventstats
```

is for.

Example:

```spl
index=auth
| eventstats dc(src_ip) AS UniqueIPs BY user
```

The original events remain.

But each event gets a new field:

```text
UniqueIPs
```

containing the aggregate for that user's group.

Conceptually:

```text
stats
    ↓
EVENTS → SUMMARY

eventstats
    ↓
EVENTS → EVENTS + SUMMARY FIELD
```

Splunk's official documentation explicitly describes `eventstats` as calculating summary statistics and adding those statistics to the original events.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/eventstats-command/eventstats-command-overview-syntax-and-usage

---

# 5.23 `stats` vs `eventstats`

This distinction is critical.

### `stats`

```spl
index=auth
| stats dc(src_ip) AS UniqueIPs BY user
```

Result:

```text
user    UniqueIPs
-----------------
alice   12
bob     4
```

Events have been transformed into a summary.

### `eventstats`

```spl
index=auth
| eventstats dc(src_ip) AS UniqueIPs BY user
```

Result conceptually remains event-level:

```text
_time    user    src_ip      UniqueIPs
---------------------------------------
10:01    alice   10.0.0.1    12
10:02    alice   10.0.0.2    12
10:03    bob     10.0.0.7     4
```

This allows later logic to compare an individual event against a group-level statistic.

For example:

```spl
index=auth
| eventstats avg(bytes) AS HostAvg BY host
| where bytes > HostAvg
```

This asks:

> Which events are above their host's average?

That pattern is extremely useful for anomaly detection [anomaly detection: identifying activity that differs from an expected baseline].

---

# 5.24 `streamstats` — Running Aggregation

Another related command is:

```text
streamstats
```

Unlike `stats`, which summarizes the entire incoming result set, `streamstats` calculates a running statistic as events are processed.

For example:

```spl
index=auth
| sort 0 _time
| streamstats count AS RunningCount BY user
```

Conceptually:

```text
Event 1 → count 1
Event 2 → count 2
Event 3 → count 3
Event 4 → count 4
```

The count is attached to each event.

SPL2:

```spl2
FROM auth
| sort _time
| streamstats count() AS RunningCount BY user
```

The current SPL2 documentation describes `streamstats` as adding a cumulative statistical value to each result as that result is processed. It also supports windows and reset conditions.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

We will study `streamstats` more deeply later because its ordering, window, reset, and time behavior deserve their own practice.

For now remember:

```text
stats
    ↓
ONE SUMMARY PER GROUP

eventstats
    ↓
ORIGINAL EVENTS + GROUP SUMMARY

streamstats
    ↓
ORIGINAL EVENTS + RUNNING SUMMARY
```

---

# 5.25 A Three-Way Mental Model

This is worth memorizing.

```text
                  AGGREGATION
                       │
        ┌──────────────┼───────────────┐
        │              │               │
        ▼              ▼               ▼
      stats         eventstats      streamstats
        │              │               │
        ▼              ▼               ▼
    collapse        preserve        preserve
    events          events          events
        │              │               │
        ▼              ▼               ▼
    summary         summary added    running
                     to each event   summary
```

This distinction is more important than memorizing individual examples.

---

# 5.26 `top` and `rare` in Traditional SPL

Traditional SPL has commands that are useful for ranking values.

### `top`

Shows the most common values of a field.

For example:

```spl
index=windows
| top limit=10 FileName
```

Conceptually:

```text
FileName            count
-------------------------
powershell.exe      12500
chrome.exe            9200
svchost.exe            8700
...
```

### `rare`

Shows the least common values.

For example:

```spl
index=windows
| rare limit=10 FileName
```

These are useful when you need quick frequency-based exploration.

The traditional SPL command reference identifies `top` as showing the most common values and `rare` as showing the least common values.

---

# 5.27 Important SPL2 Difference: Do Not Assume `top` Is Available

This is a place where the LogScale source must not be copied into Splunk.

The original material presents:

```text
top(FileName)
```

as a LogScale shortcut.

In current SPL2, there is not a native `top` command that you should treat as the direct SPL equivalent.

The current Splunk SPL2 documentation instead shows that a "top values" function can be built from:

```text
stats
→ sort
→ head
```

For example:

```spl2
FROM windows
| stats count() BY FileName
| sort -count
| head 10
```

Structured SPL2 can express the same logic with `GROUP BY`, `ORDER BY`, and `LIMIT`.

This is an important example of why we translate the **operation**, not the original product's syntax.

Official SPL2 example and function documentation:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/functions/custom-command-functions

---

# 5.28 Top Values With SPL2 Structured Syntax

A structured version can be written as:

```spl2
FROM windows
GROUP BY FileName
SELECT FileName, count() AS Events
ORDER BY Events DESC
LIMIT 10
```

The logic is:

```text
GROUP
  ↓
FileName

COUNT
  ↓
How many?

ORDER
  ↓
Largest count first

LIMIT
  ↓
10
```

This is a good example of how the structured SPL2 syntax becomes useful once you understand aggregation.

---

# 5.29 `chart` — Two-Dimensional Aggregation

Another aggregation command worth knowing is:

```text
chart
```

`chart` is designed to create a table structure suitable for visualization and supports row/column splits.

For example, traditional SPL:

```spl
index=web
| chart count BY status host
```

might produce:

```text
status   WEB01   WEB02   WEB03
--------------------------------
200      1200    980     1430
404       35     42       18
500       12     19       21
```

Think of it as:

```text
row dimension
     +
column dimension
     +
aggregation
```

`stats` is generally the core command you should learn first.

`chart` becomes particularly useful when your result is intended to be visualized.

Splunk's documentation groups `chart`, `stats`, and `timechart` under statistical and charting commands.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/search-manual/10.0/calculate-statistics/use-the-stats-command-and-functions

We will treat `timechart` separately in Chapter 6 because it introduces time buckets and time-series analysis.

---

# 5.30 `tstats` — An Important Splunk-Specific Aggregation Command

Because you are learning Splunk internals and TSIDX, there is one command you should know even though it is more advanced:

```text
tstats
```

`stats` normally works from the events returned by the search pipeline.

`tstats` is designed to perform statistical queries against **indexed fields in TSIDX files** and accelerated data-model summaries.

That means:

```text
stats
    ↓
general event/statistical processing

tstats
    ↓
indexed field / TSIDX-oriented statistical processing
```

For example, traditional SPL can use:

```spl
| tstats count WHERE index=windows BY host
```

This can be significantly faster for suitable indexed-field queries because it can avoid scanning raw event data in the same way a normal `stats` search does.

Splunk's current documentation explicitly states that `tstats` performs statistical queries on indexed fields in `tsidx` files.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.4/search-commands/tstats

This is directly related to what you learned earlier about:

```text
TSIDX
indexed fields
walklex
index-time extraction
```

We will study `tstats` in greater depth in the performance/indexed-fields part of the course.

---

# 5.31 Why `tstats` Is Not a Replacement for `stats`

Do not learn:

```text
tstats = faster stats
```

as though the commands were interchangeable.

`tstats` has restrictions.

For example, its `WHERE` clause can use indexed fields, and search-time-only fields are not supported there.

So:

```text
stats
    ↓
can work with normal search-time extracted fields

tstats
    ↓
works from indexed fields / supported data-model summaries
```

That is why the choice depends on the data and the search requirement.

Splunk explicitly documents that search-time-extracted fields are not supported in the `tstats` `WHERE` clause.

---

# 5.32 High-Cardinality Fields

Now we reach an important performance concept.

**Cardinality** [cardinality: the number of distinct values in a field] tells you roughly how many different values exist.

Examples:

```text
host
```

might have:

```text
500 values
```

while:

```text
CommandLine
```

could have:

```text
millions of values
```

This matters for aggregation.

A query such as:

```spl
| stats count BY host
```

usually creates a relatively bounded set of groups.

But:

```spl
| stats count BY CommandLine
```

can create an enormous number of groups if every command line is unique.

This can increase memory consumption and search cost.

Splunk documents memory considerations for functions such as `distinct_count`, `values`, and `list`, and recommends filtering before resource-intensive aggregations when possible.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/eventstats-command/eventstats-command-overview-syntax-and-usage

---

# 5.33 `values()` and `list()` Can Also Be Expensive

Remember:

```text
count()
```

keeps a number.

But:

```text
values()
list()
```

must retain actual field values.

That means they can consume much more memory when groups are large.

For example:

```spl
| stats values(CommandLine) BY host
```

could produce enormous multivalue results.

Use these functions intentionally.

A good workflow is:

```text
FILTER first
    ↓
Reduce the event set
    ↓
Aggregate
    ↓
values/list only when necessary
```

The current Splunk documentation explicitly notes that `distinct_count`, `values`, and `list` can consume substantial memory.

---

# 5.34 Aggregation With Conditional Logic

A powerful technique is to aggregate only events that meet a condition.

For example:

```spl
index=windows
| stats
    count AS TotalEvents
    count(eval(EventCode=4625)) AS FailedLogons
    count(eval(EventCode=4624)) AS SuccessfulLogons
    BY host
```

This gives you multiple security statistics in one row.

SPL2 supports the same concept:

```spl2
FROM windows
| stats
    count() AS TotalEvents,
    count(eval(EventCode=4625)) AS FailedLogons,
    count(eval(EventCode=4624)) AS SuccessfulLogons
    BY host
```

The current SPL2 statistical-function documentation explicitly describes `eval` expressions inside statistical functions, such as:

```spl
stats count(eval(status="404")) AS count_status BY sourcetype
```

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/overview-of-spl2-stats-and-chart-functions

This pattern is extremely useful for SOC dashboards and detection logic.

---

# 5.35 A Real SOC Example — Authentication

Question:

> Which users have the most failed logons?

### SPL

```spl
index=windows EventCode=4625
| stats count AS FailedLogons BY user
| sort -FailedLogons
| head 10
```

### SPL2 pipeline

```spl2
FROM windows
| where EventCode=4625
| stats count() AS FailedLogons BY user
| sort -FailedLogons
| head 10
```

### SPL2 structured form

```spl2
FROM windows
WHERE EventCode=4625
GROUP BY user
SELECT user, count() AS FailedLogons
ORDER BY FailedLogons DESC
LIMIT 10
```

Read the logic:

```text
FAILED LOGONS
      ↓
GROUP BY USER
      ↓
COUNT
      ↓
SORT HIGHEST FIRST
      ↓
TOP 10
```

Notice how Chapters 2–4 and Chapter 5 connect naturally.

---

# 5.36 A Real SOC Example — Unique Source IPs

Question:

> Which hosts received connections from the largest number of unique source IPs?

### SPL

```spl
index=network
| stats dc(src_ip) AS UniqueSourceIPs BY dest
| sort -UniqueSourceIPs
| head 20
```

### SPL2

```spl2
FROM network
| stats dc(src_ip) AS UniqueSourceIPs BY dest
| sort -UniqueSourceIPs
| head 20
```

This is different from:

```spl
| stats count(src_ip) BY dest
```

because `count()` measures field occurrences, while `dc()` measures distinct values.

---

# 5.37 A Real SOC Example — Suspicious Process Families

Question:

> Which hosts executed the largest number of different administrative tools?

### SPL

```spl
index=windows
| search FileName IN ("whoami.exe","ipconfig.exe","nltest.exe","net.exe")
| stats
    count AS Executions
    dc(FileName) AS UniqueTools
    values(FileName) AS Tools
    BY host
| sort -Executions
```

### SPL2

```spl2
FROM windows
| search FileName IN ("whoami.exe","ipconfig.exe","nltest.exe","net.exe")
| stats
    count() AS Executions,
    dc(FileName) AS UniqueTools,
    values(FileName) AS Tools
    BY host
| sort -Executions
```

Now one row tells you:

```text
host
Executions
UniqueTools
Tools
```

This is the point where aggregation becomes useful for real investigations rather than just learning statistics.

---

# 5.38 Common Mistake — Aggregating Before Filtering

Compare:

### Wrong logical order

```spl
index=windows
| stats count BY host
| search FileName="powershell.exe"
```

At this point:

```text
FileName
```

is no longer part of the summary.

### Correct order

```spl
index=windows
| search FileName="powershell.exe"
| stats count BY host
```

So remember:

```text
FILTER EVENT DATA
        ↓
AGGREGATE
        ↓
FILTER SUMMARY DATA
```

The same rule applies to SPL2.

---

# 5.39 Common Mistake — Counting When You Mean Distinct

Compare:

```spl
| stats count(src_ip) BY host
```

with:

```spl
| stats dc(src_ip) BY host
```

Suppose a host received:

```text
10,000 connections
```

from only:

```text
5 source IPs
```

Then:

```text
count(src_ip) = 10,000
dc(src_ip)    = 5
```

Always ask:

> Do I want the number of events, or the number of different values?

---

# 5.40 Common Mistake — Using `first()` When You Mean `latest()`

Suppose:

```text
10:00 → status=online
10:05 → status=offline
```

If the question is:

> What is the latest status?

Use:

```spl
| stats latest(status) BY host
```

Do not assume:

```spl
| stats first(status) BY host
```

means the same thing.

`first()` and `last()` depend on event-processing order.

`latest()` and `earliest()` are chronological functions.

This distinction is documented by Splunk and is especially important when search ordering changes.

---

# 5.41 Common Mistake — Grouping by Extremely High-Cardinality Fields

This:

```spl
| stats count BY CommandLine
```

might produce an enormous number of groups.

Before doing that, ask:

```text
How many unique values does this field have?
```

Fields such as:

```text
host
sourcetype
user
FileName
```

often make natural grouping keys.

Fields such as:

```text
CommandLine
session_id
request_id
transaction_id
```

can have much higher cardinality.

That does not mean you can never group on them.

It means you should understand the resource implications first.

---

# 5.42 Common Mistake — Forgetting That Multivalue Fields Exist

Suppose:

```text
processes = [cmd.exe, powershell.exe, whoami.exe]
```

This is a multivalue field.

Functions such as:

```text
values()
list()
```

are designed to produce multivalue results.

The current SPL2 documentation also notes that some statistical functions treat multivalue fields by processing their individual values.

Do not assume that every field is a single string.

This will become increasingly important when we work with JSON, XML, arrays, and normalized security data.

---

# 5.43 The Complete Aggregation Mental Model

Your query pipeline now has a new branch.

```text
             INVESTIGATION QUESTION
                      │
                      ▼
                FIND EVENTS
                      │
                      ▼
                   FILTER
                      │
                      ▼
             What do I need?
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
      INDIVIDUAL               SUMMARY
        EVENTS                  DATA
          │                       │
          ▼                       ▼
      Chapters 2–4             stats
          │                       │
          │                ┌──────┼──────┐
          │                ▼      ▼      ▼
          │              count  sum     dc
          │                │      │      │
          │                └──────┼──────┘
          │                       ▼
          │                  SUMMARY
          │                       │
          │                       ▼
          │               FILTER / SORT
          │                       │
          │                       ▼
          │                    LIMIT
          │
          ▼
       table /
       fields /
       sort /
       head
```

The single question to ask before writing the aggregation is:

> **Do I want to inspect individual events, or do I want an answer about a collection of events?**

If you want individual events, Chapter 4 tools may be enough.

If you want a summary, aggregation begins.

---

# 5.44 The Three Core Aggregation Commands

For now, the most important commands are:

```text
stats
eventstats
streamstats
```

Think:

```text
stats
→ Collapse into summaries

eventstats
→ Keep events and add summary fields

streamstats
→ Keep events and add running summaries
```

These three commands form the core of Splunk statistical processing.

Splunk's documentation groups `stats`, `eventstats`, and `streamstats` as related statistical commands.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/search-manual/10.0/calculate-statistics/use-the-stats-command-and-functions

---

# 5.45 The Aggregate Functions You Should Know

You do not need to master every statistical function immediately.

Start with:

```text
count()
dc() / distinct_count()
sum()
avg()
min()
max()
range()
values()
list()
earliest()
latest()
```

Then learn:

```text
first()
last()
median()
mode()
stdev()
stdevp()
var()
varp()
perc()
exactperc()
upperperc()
estdc()
estdc_error()
```

The current SPL2 reference includes these functions and separates them into aggregate, event-order, multivalue/array, and time-function categories.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/quick-reference-for-spl2-stats-and-charting-functions

---

# 5.46 SPL vs SPL2 Aggregation Cheat Sheet

| Goal | Traditional SPL | SPL2 pipeline | SPL2 structured |
|---|---|---|---|
| Count all events | `stats count` | `stats count()` | `GROUP BY ... SELECT count()` |
| Count by host | `stats count BY host` | `stats count() BY host` | `GROUP BY host SELECT count(), host` |
| Sum bytes | `stats sum(bytes)` | `stats sum(bytes)` | `SELECT sum(bytes)` |
| Count unique IPs | `stats dc(src_ip)` | `stats dc(src_ip)` | `SELECT dc(src_ip)` |
| Distinct values | `stats values(user)` | `stats values(user)` | `SELECT values(user)` |
| Preserve value sequence | `stats list(user)` | `stats list(user)` | `SELECT list(user)` |
| Add aggregate to events | `eventstats ...` | `eventstats ...` | `eventstats ...` |
| Running aggregate | `streamstats ...` | `streamstats ...` | pipeline command |
| Sort summary | `sort -count` | `sort -count` | `ORDER BY count DESC` |
| Limit summary | `head 10` | `head 10` | `LIMIT 10` |

Notice that SPL2 can use both:

```text
SPL-style pipeline syntax
```

and:

```text
structured `FROM` syntax
```

This is one of the reasons we are learning both SPL and SPL2 rather than treating SPL2 as a completely separate subject.

Splunk's current SPL2 documentation explicitly states that SPL2 supports both SPL syntax and SQL syntax.

Official reference:  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/introduction/understanding-spl2-syntax

---

# 5.47 What We Added Beyond the Original Chapter

The original LogScale chapter gives us a useful aggregation progression, but Splunk has several concepts that cannot simply be copied from LogScale.

For Splunk, this chapter therefore adds:

```text
stats
eventstats
streamstats
dc()
distinct_count()
estdc()
values()
list()
earliest()
latest()
first()
last()
conditional aggregations
chart
tstats
high-cardinality considerations
multivalue aggregation
SPL vs SPL2 comma/parentheses differences
SPL2 structured GROUP BY / SELECT
```

These are important because they describe how aggregation actually works in Splunk rather than trying to force another query language's model into it.

---

# 5.48 Chapter Summary

In this chapter, we moved from event-level searches to summary-level analysis.

The most important concepts are:

### `stats`

Turns many events into summary rows.

```spl
| stats count BY host
```

### `count`

Answers:

> How many events/field occurrences?

```spl
| stats count
```

### `dc()` / `distinct_count()`

Answers:

> How many different values?

```spl
| stats dc(src_ip) BY host
```

### `sum()`

Answers:

> What is the total?

```spl
| stats sum(bytes) BY host
```

### `avg()`, `min()`, `max()`

Answer:

> What is the average, smallest, or largest value?

### `values()`

Answers:

> Which distinct values occurred?

### `list()`

Answers:

> Which values occurred, preserving input order?

### `earliest()` / `latest()`

Answer chronological questions.

### `first()` / `last()`

Depend on event-processing order and should not be confused with chronological functions.

### `eventstats`

Calculates a summary but keeps the original events.

### `streamstats`

Adds a running or windowed summary to each event as it is processed.

### `top` / `rare`

Useful traditional SPL shortcuts for common/rare values.

### `tstats`

Performs statistical queries against indexed fields and TSIDX/data-model summaries for supported use cases.

And the most important workflow is:

```text
FILTER EVENTS
      ↓
AGGREGATE
      ↓
FILTER SUMMARY
      ↓
SORT SUMMARY
      ↓
LIMIT
```

The aggregation boundary is the key idea:

```text
             EVENTS
                │
                │ stats
                ▼
             SUMMARY
```

Before `stats`, you are reasoning about individual events.

After `stats`, you are reasoning about the result of many events.

That distinction is the foundation for the next chapters on time-based analysis, correlation, and detection logic.

---

# Official Splunk Documentation Used for This Chapter

This chapter is based primarily on current official Splunk documentation:

1. **SPL2 `stats` command — Overview, syntax, and usage**  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/stats-command/stats-command-overview-syntax-and-usage

2. **SPL2 `stats` command — Examples**  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/stats-command/stats-command-examples

3. **SPL2 Aggregate Functions**  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/aggregate-functions

4. **SPL2 Multivalue and Array Functions**  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/multivalue-and-array-functions

5. **SPL2 Time Functions**  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/time-functions

6. **SPL2 `eventstats` command**  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/eventstats-command/eventstats-command-overview-syntax-and-usage

7. **SPL2 `streamstats` command**  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

8. **SPL2 syntax — SPL and SQL syntax**  
https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/introduction/understanding-spl2-syntax

9. **Traditional SPL — statistical and charting commands/functions**  
https://help.splunk.com/en/splunk-enterprise/search/search-manual/10.0/calculate-statistics/use-the-stats-command-and-functions

10. **Traditional SPL `tstats`**  
https://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.4/search-commands/tstats

For any syntax difference encountered later, use the current Splunk documentation for the specific Splunk release as the final authority.
