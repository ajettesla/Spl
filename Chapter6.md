# Splunk SPL + SPL2

# Chapter 6 — Time-Based Analysis

In Chapters 2–5, we learned how to find events, search text, select fields, sort results, limit results, and aggregate events.

Now we add **time** as another dimension of the investigation.

A SOC question often changes from:

> How many failed logons are there?

into:

> How many failed logons happened **per minute**?

Or:

> When did the spike begin?

Or:

> Is the current 10-minute count unusual compared with the previous hour?

Or:

> Did activity happen in repeated waves?

These questions require more than simply looking at `_time`.

Splunk provides several time-analysis mechanisms, and traditional SPL and SPL2 do not always use the same syntax.

The most important tools for this chapter are:

```text
Time range
    ↓
earliest / latest / now() / relative_time()

Fixed time buckets
    ↓
bin / bucket
stats ... BY _time span=...
timechart

Rolling calculations
    ↓
streamstats

Time-period comparison
    ↓
timewrap

Session-like grouping
    ↓
transaction in traditional SPL

Time-based display inside summaries
    ↓
sparkline
```

The goal is not to memorize these commands. The goal is to understand **what meaning of time your investigation requires**.

---

# 6.1 Time Is More Than a Field

From Chapter 1, you already know that `_time` is the event timestamp.

Splunk stores timestamps internally as UNIX time [UNIX time: a number representing seconds since the Unix epoch, 1 January 1970 00:00:00 UTC]. The user interface can display that value as a human-readable date and time.

For example:

```text
_time = 2026-09-23 10:15:22
```

Conceptually:

```text
EVENT
  |
  +---- _time = when did it happen?
```

Now we can use `_time` in three different ways:

```text
1. LIMIT the time range
2. GROUP events into time periods
3. COMPARE activity across time
```

Splunk's current documentation describes all three uses: timestamps are used to correlate events, create timeline histograms, and set search time ranges. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/dates-and-time/timestamps-and-time-ranges

---

# 6.2 Time Range Comes First

Before building a time-based analysis, first decide **which events are allowed into the search**.

Traditional SPL:

```spl
index=windows earliest=-1h latest=now
```

SPL2 `search`:

```spl2
search index=windows earliest=-1h latest=now
```

SPL2 `FROM`:

```spl2
FROM windows
WHERE earliest=-1h AND latest=now
```

The `earliest` and `latest` modifiers define the search range.

Examples:

```text
-15m     → last 15 minutes
-1h      → last hour
-24h     → last 24 hours
-7d      → last 7 days
```

You can also snap [snap: round a relative time to a boundary such as the start of an hour or day] a time to a boundary:

```text
-1h@h
```

means approximately:

> One hour ago, snapped to the beginning of that hour.

For example:

```spl
index=windows earliest=-1h@h latest=now
```

Splunk documents `earliest` and `latest` as time modifiers for both SPL and SPL2 searches. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/search-command/search-command-usage citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/from-command/from-command-overview

---

# 6.3 `now()` and `relative_time()`

Sometimes the time condition is part of an expression instead of the search-time range itself.

Splunk provides:

```text
now()
```

and:

```text
relative_time()
```

For example:

```spl
index=windows
| eval one_hour_ago=relative_time(now(), "-1h")
```

Or filter using `_time`:

```spl
index=windows
| where _time > relative_time(now(), "-1h")
```

SPL2 can use the same evaluation functions:

```spl2
FROM windows
WHERE _time > relative_time(now(), "-1h")
```

The important difference is:

```text
earliest/latest
    ↓
Search time range

relative_time()/now()
    ↓
Expression-based time calculation
```

`now()` returns the time at which the search started, expressed as UNIX time. `relative_time()` applies a relative-time specification to a UNIX timestamp. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/evaluation-functions/date-and-time-functions

---

# 6.4 Fixed Time Slices

Now suppose your question is:

> How many failed logons happened every 10 minutes?

You do not want one row per event.

You want:

```text
Time bucket       Count
-----------------------
10:00–10:10          12
10:10–10:20           8
10:20–10:30         164   ← spike
10:30–10:40          11
```

This is a **time bucketing** problem.

A bucket is a fixed interval such as:

```text
1 minute
5 minutes
10 minutes
1 hour
1 day
```

The key idea is:

```text
Many events
     ↓
FIXED TIME INTERVALS
     ↓
Aggregate inside each interval
```

This is the Splunk equivalent of the time-slicing idea from the original CrowdStrike chapter.

---

# 6.5 Traditional SPL: `bin` / `bucket`

Traditional SPL commonly uses `bin` to place continuous values into discrete groups.

For `_time`:

```spl
index=windows
| bin _time span=10m
| stats count AS Events BY _time
```

The result is one row per 10-minute interval.

`bucket` is simply an alias for `bin` in traditional SPL:

```spl
| bucket _time span=10m
```

is equivalent to:

```spl
| bin _time span=10m
```

Splunk's current SPL documentation explicitly states that `bucket` is an alias for `bin`. citehttps://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.4/search-commands/bucket

So, unlike the original LogScale chapter, do **not** learn `bucket()` as a Splunk function.

In traditional SPL, think:

```text
bin command
    ↑
bucket is an alias
```

---

# 6.6 SPL2: Time Span With `stats`

SPL2 adds an important capability that traditional SPL does not have in exactly the same place.

You can specify a time span directly in the `BY` clause of `stats`:

```spl2
search index=windows
| stats count() AS Events BY _time span=10m
```

This produces a count for each 10-minute time interval.

In traditional SPL, the equivalent pattern is:

```spl
index=windows
| bin _time span=10m
| stats count AS Events BY _time
```

This is one of the most useful SPL/SPL2 differences in time analysis.

Splunk's current SPL2 `stats` documentation explicitly documents the `span` option in the `BY` clause and gives the traditional SPL `bin` + `stats` pattern as the equivalent. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/stats-command/stats-command-overview-syntax-and-usage

---

# 6.7 SPL2 `FROM` Syntax for Time Grouping

The structured SPL2 syntax can express the same operation through `GROUP BY`:

```spl2
FROM windows
GROUP BY span(_time, 10m)
SELECT count() AS Events, _time
```

The idea is:

```text
FROM
   ↓
Get events

GROUP BY span(_time, 10m)
   ↓
Create 10-minute time groups

SELECT count()
   ↓
Count events in each group
```

Splunk's SPL2 documentation documents `span(time, span-length)` for time grouping in `GROUP BY`. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/time-functions

---

# 6.8 Choosing the `span`

`span` answers one question:

> How wide should each time bucket be?

Examples:

```text
span=1m
span=5m
span=10m
span=1h
span=1d
```

The choice is not merely cosmetic.

Suppose this is the event rate:

```text
1-minute buckets:
[0][0][0][95][0][0]
```

But with one-hour buckets:

```text
[95]
```

The short burst disappears into the larger interval.

Therefore:

```text
Small span
    ↓
Short bursts become visible

Large span
    ↓
Longer trends become visible
```

Splunk's `timechart` documentation also warns that automatically selected spans depend on the search range. If you need a specific interval, explicitly specify `span`. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/timechart-command/timechart-command-usage

---

# 6.9 Auto-Selected Span

If you omit `span`, `timechart` chooses a span based on the requested number of bins or the selected time range.

For example, current Splunk documentation lists default spans such as:

```text
Last 15 minutes → 10 seconds
Last 60 minutes → 1 minute
Last 4 hours   → 5 minutes
Last 24 hours  → 30 minutes
Last 7 days    → 1 day
```

These defaults are useful for visualization, but they are not necessarily appropriate for a detection requirement.

For example:

> Detect a 3-minute brute-force burst.

You should deliberately choose a smaller span than the expected burst duration.

Splunk documents these automatically selected spans and the explicit `span` behavior for `timechart`. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/timechart-command/timechart-command-usage

---

# 6.10 `timechart`

This is one of the most important time-analysis commands in Splunk.

Traditional SPL:

```spl
index=windows
| timechart span=10m count
```

SPL2:

```spl2
search index=windows
| timechart span=10m count()
```

The result is a time series with `_time` on the X-axis and the aggregation on the Y-axis.

Think:

```text
stats + time
        ↓
     timechart
        ↓
 time-series result
```

Splunk documents `timechart` as a transforming [transforming: a command that turns event results into a statistical table] command that creates a time series chart and corresponding table. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/timechart-command/timechart-command-overview-and-syntax

---

# 6.11 `timechart` With a Split-By Field

One line answers:

> How many failed logons happened over time?

But you may want:

> How many failed logons happened over time **for each host**?

Traditional SPL:

```spl
index=windows EventCode=4625
| timechart span=10m count BY host
```

SPL2:

```spl2
search index=windows EventCode=4625
| timechart span=10m count() BY host
```

Now each host becomes a series.

Conceptually:

```text
             10:00  10:10  10:20  10:30
DC01          2       3      80       1
DC02          1       2       4       2
WEB01         0       1      60       0
```

This lets you ask not only:

> Did a spike happen?

but also:

> Which host produced the spike?

Splunk documents the split-by behavior for `timechart`. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/timechart-command/timechart-command-overview-and-syntax

---

# 6.12 `timechart` vs `stats` With Time

These are related but serve different purposes.

### Traditional SPL

```spl
| bin _time span=10m
| stats count BY _time
```

returns a table of time buckets.

### Traditional SPL

```spl
| timechart span=10m count
```

is designed specifically for time-series output.

### SPL2

```spl2
| stats count() BY _time span=10m
```

is a statistical grouping by time.

### SPL2

```spl2
| timechart span=10m count()
```

is specifically a time-series chart.

The useful mental model is:

```text
Need a general statistical table?
    ↓
stats + time grouping

Need a time-series chart?
    ↓
timechart
```

---

# 6.13 `bin` Is More General Than Time

Although we are using `bin` for `_time`, it is not limited to timestamps.

SPL2's `bin` command puts continuous numerical values into discrete sets.

For example, numeric values can be grouped into ranges.

In time analysis, you most commonly see:

```spl
| bin _time span=5m
```

or the SPL2 equivalent:

```spl2
| bin _time span=5m
```

Current SPL2 documentation also notes that `timechart` automatically performs time binning, so you generally do not need to run `bin` before `timechart` merely to create its time axis. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/bin-command/bin-command-overview-syntax-and-usage

---

# 6.14 A Real SOC Example — Failed Logon Spike

Question:

> Did failed logons suddenly spike?

Traditional SPL:

```spl
index=windows EventCode=4625
| timechart span=10m count AS FailedLogons
```

SPL2:

```spl2
search index=windows EventCode=4625
| timechart span=10m count() AS FailedLogons
```

Now you can visually inspect:

```text
10:00    4
10:10    7
10:20    5
10:30  238   ← spike
10:40    8
```

A chart makes a burst much easier to see than thousands of individual events.

---

# 6.15 Time-Series Filtering by Host

You can combine the filtering from earlier chapters with time analysis.

Traditional SPL:

```spl
index=windows EventCode=4625 host=DC01
| timechart span=5m count AS FailedLogons
```

SPL2:

```spl2
FROM windows
WHERE EventCode=4625 AND host="DC01"
| timechart span=5m count() AS FailedLogons
```

The workflow is:

```text
FILTER
   ↓
failed logons from DC01

TIME BUCKET
   ↓
5-minute slices

AGGREGATE
   ↓
count
```

This is the combination of Chapters 2, 5, and 6.

---

# 6.16 Time-Series by Multiple Dimensions

You can combine time and another field.

Traditional SPL:

```spl
index=windows EventCode=4625
| timechart span=10m count BY user
```

SPL2:

```spl2
search index=windows EventCode=4625
| timechart span=10m count() BY user
```

Now each user becomes a separate series.

This can reveal a pattern such as:

```text
One user + huge spike
      ↓
Potential brute-force pattern

Many users + simultaneous smaller spikes
      ↓
Potential password-spray pattern
```

These interpretations are investigation hypotheses, not conclusions from the chart alone.

---

# 6.17 Time Zones

Time-zone handling becomes important as soon as you investigate events across regions.

Remember:

```text
_time
   ↓
Stored as UNIX time
```

The Splunk UI displays `_time` in human-readable form according to time-zone context.

The same event therefore represents the same instant even when two analysts see different local clock representations.

For example, an event may represent the same instant as:

```text
UTC  → 04:30
IST  → 10:00
```

The underlying `_time` does not become a different instant merely because the display timezone changes.

Splunk documents that `_time` values are stored as UNIX time and explains how the search time range and display are affected by timezone context. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/dates-and-time/timestamps-and-time-ranges citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/dates-and-time/time-zones

Do not solve timezone problems by blindly adding or subtracting hours from `_time`. First determine whether you are dealing with:

```text
the stored instant
       or
its displayed timezone
       or
an incorrectly parsed source timestamp
```

That distinction prevents a large class of timestamp mistakes.

---

# 6.18 `streamstats` — The Core Idea

`streamstats` is the main Splunk command for **running [calculated progressively as each event/result is processed] and moving [calculated over a limited recent set] statistics**.

The most important idea is:

```text
stats
    ↓
Looks at the result set as a whole
    ↓
Produces aggregate results

streamstats
    ↓
Processes results one by one
    ↓
Adds the calculation to each result
```

Splunk describes `streamstats` as a centralized streaming command: it calculates statistics for each event at the time the event is seen. The calculation can also operate on the results produced by commands such as `stats`, `chart`, or `timechart`. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

A simple example:

```spl
index=windows
| streamstats count AS event_number
| table _time event_number
```

Conceptually:

```text
Event 1 → event_number = 1
Event 2 → event_number = 2
Event 3 → event_number = 3
Event 4 → event_number = 4
```

Unlike `stats`, the original rows remain available.

---

# 6.19 `stats` vs `streamstats` vs `eventstats`

These commands are related, but they answer different questions.

Suppose the events are:

```text
_time   user   bytes
------  -----  -----
10:00   teja   100
10:01   teja   200
10:02   john   300
10:03   teja   400
```

### `stats`

```spl
| stats sum(bytes) BY user
```

Result:

```text
user   sum(bytes)
-----  ----------
teja   700
john   300
```

`stats` **collapses [reduces many events into aggregate rows]** the result set.

Question:

> What is the total bytes for each user?

### `eventstats`

```spl
| eventstats sum(bytes) AS total_bytes BY user
```

Conceptually:

```text
_time   user   bytes   total_bytes
------  -----  -----   -----------
10:00   teja   100     700
10:01   teja   200     700
10:02   john   300     300
10:03   teja   400     700
```

`eventstats` calculates the aggregate over the relevant result set and adds it back to the matching events.

Question:

> What is this user's total, while keeping every event?

### `streamstats`

```spl
| sort 0 _time
| streamstats sum(bytes) AS running_bytes BY user
```

Result:

```text
_time   user   bytes   running_bytes
------  -----  -----   -------------
10:00   teja   100     100
10:01   teja   200     300
10:02   john   300     300
10:03   teja   400     700
```

Question:

> What was the running total for this user at each event?

The key distinction is:

```text
stats
    → aggregate the whole result set

eventstats
    → aggregate the whole result set + attach result to events

streamstats
    → aggregate progressively as each result is processed
```

Splunk's current documentation makes the same distinction: `stats` works on the result set as a whole, `eventstats` adds aggregate statistics inline, and `streamstats` calculates at the time each event is seen. citehttps://help.splunk.com/en/splunk-enterprise/search/search-manual/10.2/calculate-statistics/use-the-stats-command-and-functions

---

# 6.20 `streamstats` Syntax

Traditional SPL supports these options:

```spl
| streamstats
    [reset_on_change=<bool>]
    [reset_before="("<eval-expression>")"]
    [reset_after="("<eval-expression>")"]
    [current=<bool>]
    [window=<int>]
    [time_window=<span-length>]
    [global=<bool>]
    [allnum=<bool>]
    <stats-aggregation>...
    [BY <field-list>]
```

The options must appear before the aggregation terms. The `BY` clause is placed after the aggregation in classic SPL. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

SPL2 uses a different reset syntax:

```spl2
| streamstats
    [current=<bool>]
    [reset before <eval-expression>
     after <eval-expression>
     onchange]
    [window=<int>]
    <aggregation>...
    [BY <field-list>]
```

In current SPL2, these classic-SPL options have no equivalent:

```text
allnum
global
time_window
```

SPL2 combines the reset controls into one `reset` clause and does not require the same escaping of quotation marks and parentheses that classic SPL requires. citehttps://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

---

# 6.21 Option 1 — `current`

`current` controls whether the **current result** is included in the calculation.

Default:

```spl
current=true
```

For example:

```spl
| streamstats sum(bytes) AS total
```

is equivalent to:

```spl
| streamstats current=true sum(bytes) AS total
```

Suppose:

```text
event   bytes
-----   -----
1       100
2       200
3       300
4       400
```

With `current=true`:

```text
event   bytes   total
-----   -----   -----
1       100     100
2       200     300
3       300     600
4       400     1000
```

The current event contributes to its own result.

### `current=false`

```spl
| streamstats current=false sum(bytes) AS previous_total
```

Now the current event is excluded:

```text
event   bytes   previous_total
-----   -----   --------------
1       100     null
2       200     100
3       300     300
4       400     600
```

This is useful when you want to compare the current event with what existed **before** it.

For example:

```spl
| streamstats current=false last(src_ip) AS previous_ip
```

can provide the previous `src_ip` in the stream.

Official SPL documentation defines `current=true` as including the current event and `current=false` as using values from the previous event/result. The default is `true`. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

---

# 6.22 Option 2 — `window`

`window` controls the maximum number of events/results used by the calculation.

Example:

```spl
| streamstats window=3 sum(bytes) AS total_bytes
```

Suppose:

```text
event   bytes
-----   -----
1       100
2       200
3       300
4       400
5       500
```

With `current=true`:

```text
event  window contents     total_bytes
-----  -----------------   -----------
1      100                 100
2      100,200             300
3      100,200,300         600
4      200,300,400         900
5      300,400,500         1200
```

The important point is:

```text
window=3
    ↓
Current result + previous 2 results
```

It is a **row-count window [a window based on the number of results]**, not a time window.

The `window` value must be non-negative. The documented default is `0`, meaning use the accumulated previous and current results, subject to Splunk's configured stream-window limits. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

---

# 6.23 `current` + `window` Together

These two options become especially important together.

Suppose:

```text
bytes = 10, 20, 30, 40, 50
```

### `window=3 current=true`

```spl
| streamstats window=3 current=true sum(bytes) AS total
```

Result:

```text
10 → 10
20 → 30
30 → 60
40 → 90
50 → 120
```

The calculation is:

```text
10
10+20
10+20+30
20+30+40
30+40+50
```

### `window=3 current=false`

```spl
| streamstats window=3 current=false sum(bytes) AS previous_total
```

Result:

```text
10 → null
20 → 10
30 → 30
40 → 60
50 → 90
```

Now the current event is excluded:

```text
Event 4:
previous three = 10+20+30 = 60
current 40 is excluded
```

This combination is extremely useful when constructing baselines [normal reference values] or comparing the current value with recent history.

---

# 6.24 Option 3 — `BY`

`BY` creates independent aggregations for different field values.

Example:

```spl
| streamstats count AS user_event_number BY user
```

Data:

```text
event   user
-----   ----
1       teja
2       john
3       teja
4       john
5       teja
```

Result:

```text
event   user   user_event_number
-----   -----  -----------------
1       teja   1
2       john   1
3       teja   2
4       john   2
5       teja   3
```

Without `BY`:

```spl
| streamstats count AS event_number
```

you get:

```text
1
2
3
4
5
```

With `BY user`, each user's running calculation is separate.

Multiple fields are also possible:

```spl
| streamstats count BY user src_ip
```

That creates a separate stream for each `user + src_ip` combination.

Splunk documents `BY` as the grouping clause for one or more fields. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

---

# 6.25 Option 4 — `global`

`global` matters when you combine:

```text
BY + window
```

Classic SPL defaults to:

```spl
global=true
```

With `global=true`, a single event window is shared across the result stream.

With:

```spl
global=false
```

and a non-zero `window`, each `BY` group has its own event window. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

Consider:

```text
event   host   bytes
-----   ----   -----
1       A      10
2       B      20
3       A      30
4       B      40
```

### `global=true`

```spl
| streamstats window=2 global=true sum(bytes) AS total BY host
```

A single last-two-event window moves through the whole stream:

```text
event  window       host  total
-----  -----------  ----  -----
1      A:10         A     10
2      A:10,B:20    B     20
3      B:20,A:30    A     30
4      A:30,B:40    B     40
```

Only the values belonging to the current `BY` group inside that shared window contribute to that group's aggregation.

### `global=false`

```spl
| streamstats window=2 global=false sum(bytes) AS total BY host
```

Each host maintains its own two-event window:

```text
event  host  host window    total
-----  ----  ------------   -----
1      A     A:10           10
2      B     B:20           20
3      A     A:10,A:30      40
4      B     B:20,B:40      60
```

This option is advanced. Learn it after `BY` and `window`.

---

# 6.26 Option 5 — `reset_on_change`

Classic SPL:

```spl
| streamstats count BY shift reset_on_change=true
```

This resets the running calculation when a `BY` field changes between incoming results.

Example:

```text
event   shift
-----   -----
1       DAY
2       DAY
3       NIGHT
4       NIGHT
5       NIGHT
6       DAY
7       NIGHT
```

Result:

```text
event  shift   count
-----  ------  -----
1      DAY       1
2      DAY       2
3      NIGHT     1
4      NIGHT     2
5      NIGHT     3
6      DAY       1
7      NIGHT     1
```

Notice that the count does not simply mean "the number of DAY events seen so far." Instead, it resets whenever the group-by value changes.

This is especially useful when the input is already ordered into meaningful runs.

Important:

```text
BY user
reset_on_change=true
```

looks at whether the `user` value changes in the incoming sequence.

It is therefore sensitive to **result order**. With multiple `BY` fields, the reset can be triggered when the group defined by those fields changes; official SPL also notes that an event must have all of the group-by fields to trigger the reset.

---

# 6.27 Option 6 — `reset_before`

`reset_before` resets the accumulated statistics **for the event where the condition is true, before calculating that event**.

Classic SPL syntax:

```spl
| streamstats reset_before="("<eval-expression>")" sum(bytes) AS total
```

Example:

```spl
| streamstats reset_before="(action=\"REBOOT\")" sum(bytes) AS total
```

Suppose:

```text
event   action          bytes
-----   --------------  -----
1       LOGON           100
2       DOWNLOAD        400
3       REBOOT           50
4       LOGON           100
```

Result:

```text
event  action      bytes  total
-----  ----------  -----  -----
1      LOGON       100    100
2      DOWNLOAD    400    500
3      REBOOT       50     50
4      LOGON        100    150
```

At `REBOOT`, the previous accumulated state is discarded first.

That is the meaning of:

```text
reset BEFORE calculation for this event
```

Splunk documents that `reset_before` resets the aggregation on the event whose expression evaluates to true. If `window` is also used, the window is reset as well. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

---

# 6.28 Option 7 — `reset_after`

`reset_after` waits until the current result has been calculated, then resets the aggregation for the next result.

```spl
| streamstats reset_after="(action=\"REBOOT\")" sum(bytes) AS total
```

Same input:

```text
event   action       bytes
-----   -----------  -----
1       LOGON        100
2       DOWNLOAD     400
3       REBOOT       50
4       LOGON        100
```

Result:

```text
event  action      bytes  total
-----  ----------  -----  -----
1      LOGON       100    100
2      DOWNLOAD    400    500
3      REBOOT       50    550
4      LOGON        100    100
```

The `REBOOT` event is included in the old calculation.

Then the next event starts from a fresh state.

So remember:

```text
reset_before
    → reset first
    → calculate current event

reset_after
    → calculate current event
    → reset for next event
```

This distinction is one of the most important `streamstats` concepts. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

---

# 6.29 `reset_before` / `reset_after` With `match()`

`reset_before` and `reset_after` need a Boolean expression [an expression that evaluates to true or false].

That is where `match()` is useful.

```spl
match(_raw,"oracle")
```

returns a Boolean condition:

```text
TRUE
or
FALSE
```

So you can use:

```spl
| streamstats reset_before="(match(_raw,\"oracle\"))" window=3 sum(pid) AS sumpid
```

Classic SPL requires the reset expression to be enclosed in the reset argument, and quotation marks inside that expression must be escaped.

For example:

```spl
reset_before="(match(_raw,\"oracle.*\"))"
```

The quotation marks around `oracle.*` are escaped because they are nested inside another quoted expression.

Splunk's official examples demonstrate this exact escaping pattern for `match()` inside a `reset_before`/`reset_after` expression. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

A complete example:

```spl
index=* source="/var/log/secure"
| sort 0 _time
| streamstats reset_before="(match(_raw,\"oracle.*\"))" window=3 sum(pid) AS sumpid
| table _time pid sumpid _raw
```

Conceptually:

```text
events before Oracle
        ↓
running 3-event sum
        ↓
Oracle event found
        ↓
RESET
        ↓
Oracle event starts new calculation
```

You can use a simple field condition when regex is unnecessary:

```spl
reset_before="(action=\"REBOOT\")"
```

Use `match()` when you need a regular-expression condition.

---

# 6.30 Option 8 — `time_window`

`time_window` creates a window based on elapsed time rather than a fixed number of results.

Classic SPL:

```spl
| streamstats time_window=5m count AS events_5m
```

This means:

> Calculate using the events inside a five-minute time window.

The direction of the incoming results matters:

```text
Ascending `_time`
    → the window represents the previous time period

Descending `_time`
    → the window represents the next time period
```

So "previous" or "next" is determined by the order in which `streamstats` receives the results, not merely by how the timestamps look on screen.

For example:

```text
_time   event
------  -----
10:00   A
10:01   B
10:04   C
10:07   D
```

With:

```spl
| streamstats time_window=5m count AS events_5m
```

the calculation is driven by the timestamps rather than by "last N rows."

This is very different from:

```spl
| streamstats window=5 count
```

because five events do not necessarily represent five minutes.

### Important requirement

For `time_window`, the events must be sorted in ascending or descending `_time` order.

For a normal chronological investigation:

```spl
| sort 0 _time
```

is usually the clearest choice.

Classic SPL supports `time_window`. Current SPL2 `streamstats` does not have an equivalent option. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats citehttps://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

---

# 6.31 `window` vs `time_window`

This distinction deserves its own rule.

### `window=5`

```spl
| streamstats window=5 count AS count_5
```

means:

```text
Last 5 RESULTS
```

### `time_window=5m`

```spl
| streamstats time_window=5m count AS count_5m
```

means:

```text
Events in the relevant 5-MINUTE TIME WINDOW
```

These are not interchangeable.

Example:

```text
10:00  event 1
10:00  event 2
10:01  event 3
10:01  event 4
10:02  event 5
10:15  event 6
```

At 10:15:

```text
window=5
    → can still use five recent results

time_window=5m
    → uses only events inside the five-minute time range
```

This distinction is crucial for security detections.

If your rule says:

> "5 attempts"

use an event-count window.

If your rule says:

> "5 attempts in 5 minutes"

use a time-based window.

---

# 6.32 Combining `window` and `time_window`

Classic SPL allows both:

```spl
| streamstats time_window=5m window=100 count AS events
```

This gives you a time-based window while also limiting the maximum number of events considered.

Think:

```text
5 minutes
    +
maximum 100 results
```

This can be useful when a very high-volume source produces many events in a short period.

Current Splunk documentation explicitly states that `window` can be used with `time_window` to specify a maximum number of events in the time window. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

---

# 6.33 Option 9 — `allnum`

`allnum` is a classic-SPL option.

```spl
| streamstats allnum=true avg(bytes) AS avg_bytes
```

It controls whether numerical statistics are calculated when all values in the relevant set are numeric.

Example data:

```text
event   bytes
-----   -----
1       100
2       200
3       N/A
4       400
```

With:

```spl
| streamstats window=3 allnum=true avg(bytes) AS avg_bytes
```

the calculation is only produced for a numerical set when every value being considered is numeric.

Without `allnum=true`, numerical statistics can operate according to the normal statistical-function behavior.

The practical lesson is:

```text
allnum=false
    → default numeric behavior

allnum=true
    → require all values in the calculation to be numeric
```

This option is useful when data quality is inconsistent and you do not want mixed-type data participating silently in a numerical calculation.

SPL2 does not provide an `allnum` option for `streamstats`. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

---

# 6.34 Option 10 — Aggregation Functions

`streamstats` is not limited to `sum()`.

You can use statistical and event-order functions.

Common examples:

```spl
| streamstats count AS n
| streamstats sum(bytes) AS running_bytes
| streamstats avg(bytes) AS running_avg
| streamstats min(bytes) AS min_bytes
| streamstats max(bytes) AS max_bytes
| streamstats range(bytes) AS byte_range
| streamstats stdev(bytes) AS byte_stdev
```

You can also use event-order functions:

```spl
| streamstats current=false last(src_ip) AS previous_ip
```

and:

```spl
| streamstats first(action) AS first_action
```

Multivalue functions include:

```spl
| streamstats values(src_ip) AS seen_ips
```

and:

```spl
| streamstats list(src_ip) AS ip_sequence
```

Current Splunk documentation lists aggregate functions such as `avg`, `count`, `distinct_count`, `estdc`, percentiles, `max`, `median`, `min`, `mode`, `range`, `stdev`, `sum`, `var`; event-order functions such as `earliest`, `first`, `last`, `latest`; and multivalue functions such as `list` and `values`. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

The exact function behavior follows the corresponding Splunk statistical-function definition.

---

# 6.35 `count` in `streamstats`

A common beginner question is:

```spl
| streamstats count
```

What does it mean?

It creates a running event counter.

Example:

```spl
| makeresults count=5
| streamstats count AS event_number
```

Result:

```text
event_number
------------
1
2
3
4
5
```

This is different from:

```spl
| stats count
```

which gives one final number:

```text
count
-----
5
```

SPL2 uses:

```spl2
| streamstats count()
```

while classic SPL commonly uses:

```spl
| streamstats count
```

Both represent the same conceptual operation in their respective syntaxes. Current SPL2 documentation explicitly demonstrates `streamstats count()` as a running count. citehttps://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/streamstats-command/streamstats-command-examples

---

# 6.36 Processing Order Is Critical

`streamstats` is order-sensitive.

Suppose:

```text
_time
-----
10:00
10:01
10:02
10:03
```

and you run:

```spl
| streamstats current=false last(_time) AS previous_time
```

"previous" means:

> Previous result in the stream.

It does not automatically mean:

> Earlier timestamp in chronological order.

Therefore, for chronological analysis, use:

```spl
| sort 0 _time
```

before `streamstats` when necessary.

For example:

```spl
index=windows EventCode=4625
| sort 0 _time
| streamstats current=false last(_time) AS previous_time BY src_ip
| eval gap=_time-previous_time
| table _time src_ip previous_time gap
```

Now `previous_time` represents the previous event in chronological order for that `src_ip`.

This is especially important when using:

```text
current=false
time_window
reset_on_change
previous-event logic
```

because all of these depend on the sequence of incoming results.

---

# 6.37 Using `streamstats` After `timechart`

A powerful pattern is:

```text
raw events
    ↓
timechart
    ↓
one row per time bucket
    ↓
streamstats
    ↓
running / rolling calculation over those buckets
```

Example:

```spl
index=windows EventCode=4625
| timechart span=10m count AS current
| streamstats window=6 current=false avg(current) AS baseline
```

Now `streamstats` is no longer processing individual Windows events.

It is processing the **10-minute time buckets produced by `timechart`**.

Example:

```text
_time   current   baseline
------  -------   --------
10:00   10        null
10:10   12        10
10:20    9        11
10:30   11        10.33
10:40   13        10.5
10:50   80        11
```

At 10:50:

```text
current  = 80
baseline = 11
```

This makes the spike much easier to identify.

This is an important `streamstats` principle:

> `streamstats` works on whatever results arrive from the previous command.

Splunk explicitly documents that `streamstats` can process results produced by transforming commands such as `stats`, `chart`, and `timechart`. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

---

# 6.38 Example — Rolling Average

Suppose these are already time buckets:

```text
bucket   events
------   ------
1        10
2        12
3         9
4        11
5        13
6        80
```

Search:

```spl
| streamstats window=3 avg(events) AS rolling_avg
```

Result:

```text
bucket  events  rolling_avg
------  ------  -----------
1       10      10
2       12      11
3        9      10.33
4       11      10.67
5       13      11
6       80      34.67
```

If you do not want the current bucket in the baseline:

```spl
| streamstats window=3 current=false avg(events) AS baseline
```

Result conceptually:

```text
bucket  events  baseline
------  ------  --------
1       10      null
2       12      10
3        9      11
4       11      10.33
5       13      10.67
6       80      11
```

This is often better for anomaly comparison because the current spike does not inflate its own baseline.

---

# 6.39 Example — Conditional Sliding Event Count

A very useful SOC pattern is to turn a condition into a numeric indicator and then use `streamstats`.

Suppose each event has:

```text
action = success
or
action = failure
```

Create:

```spl
| eval is_failure=if(action="failure",1,0)
```

Then:

```spl
| streamstats window=5 sum(is_failure) AS failures_last_5
```

Now every event carries the number of failures in the last five results.

Example:

```text
action    is_failure    failures_last_5
-------   -----------   ---------------
failure       1                 1
failure       1                 2
success       0                 2
failure       1                 3
failure       1                 4
success       0                 3
```

To exclude the current event:

```spl
| streamstats window=5 current=false sum(is_failure) AS previous_failures
```

This pattern is conceptually close to a sliding-window detection:

```text
event
  ↓
turn condition into 1/0
  ↓
streamstats window=N
  ↓
count/sum recent matching events
```

For a real time-based rule, use:

```spl
| sort 0 _time
| streamstats time_window=5m sum(is_failure) AS failures_5m
```

This asks:

> How many failed events occurred in the relevant five-minute window?

---

# 6.40 Example — Previous Event With `last()`

`streamstats` is useful when the current event must be compared with an earlier event.

Example:

```spl
index=network
| sort 0 _time
| streamstats current=false last(src_ip) AS previous_src_ip
| eval changed=if(src_ip!=previous_src_ip,1,0)
| table _time src_ip previous_src_ip changed
```

Another common example:

```spl
| streamstats current=false last(_time) AS previous_time BY src_ip
| eval gap=_time-previous_time
```

Now:

```text
_time   src_ip       previous_time   gap
------  -----------  --------------  ----
10:00   10.0.0.5     null            null
10:01   10.0.0.5     10:00           60
10:04   10.0.0.5     10:01          180
10:15   10.0.0.5     10:04          660
```

The `gap` field tells you how long it had been since the previous event for that IP.

This is a building block for session-like analysis.

---

# 6.41 Example — Creating Session-Like Groups With `streamstats`

Suppose you define a new session whenever the gap between events is greater than five minutes.

A practical pattern is:

```spl
index=web
| sort 0 src_ip _time
| streamstats current=false last(_time) AS previous_time BY src_ip
| eval gap=_time-previous_time
| eval new_session=if(isnull(previous_time) OR gap>300,1,0)
| streamstats sum(new_session) AS session_id BY src_ip
| table _time src_ip gap new_session session_id
```

Conceptually:

```text
same IP
   ↓
measure gap
   ↓
gap > 300 seconds?
   ↓
yes → new_session=1
no  → new_session=0
   ↓
running sum
   ↓
session_id
```

This gives you session-like groups without using `transaction`.

For larger or more complex event-correlation problems, compare this approach carefully with `transaction`, because `transaction` has explicit grouping controls such as `maxpause` and `maxspan`.

---

# 6.42 Example — Reset After a Reboot

This is a clean example of state [the accumulated value carried from previous events].

```spl
index=system
| streamstats reset_after="(action=\"REBOOT\")" sum(bytes) AS total_bytes BY host
| table _time host action bytes total_bytes
```

Suppose:

```text
host  action       bytes
----  -----------  -----
A     LOGON        100
A     DOWNLOAD     400
A     REBOOT        50
A     LOGON        100
```

Result:

```text
host  action       bytes  total_bytes
----  -----------  -----  -----------
A     LOGON        100    100
A     DOWNLOAD     400    500
A     REBOOT        50     550
A     LOGON        100    100
```

The reboot event is included in the old total because the reset is `after`.

Change it to `reset_before`:

```spl
| streamstats reset_before="(action=\"REBOOT\")" sum(bytes) AS total_bytes BY host
```

and the reboot starts the new total:

```text
host  action       bytes  total_bytes
----  -----------  -----  -----------
A     LOGON        100    100
A     DOWNLOAD     400    500
A     REBOOT        50     50
A     LOGON        100    150
```

This is the cleanest way to remember the difference.

---

# 6.43 Example — Reset When the Source Changes

Suppose events arrive in order by `src_ip` and you want to reset whenever the IP changes.

```spl
| sort 0 src_ip _time
| streamstats count BY src_ip reset_on_change=true AS sequence
```

Conceptually:

```text
src_ip       sequence
-----------  --------
10.0.0.1        1
10.0.0.1        2
10.0.0.1        3
10.0.0.2        1
10.0.0.2        2
10.0.0.3        1
```

`reset_on_change` is about a change in the incoming `BY` value, so ordering must match the way you want the runs to be interpreted.

Splunk's official example demonstrates the same DAY/NIGHT behavior. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

---

# 6.44 SPL vs SPL2 — `streamstats` Options

For this command, the differences are worth memorizing.

| Feature | Classic SPL | SPL2 |
|---|---|---|
| `current` | ✅ | ✅ |
| `window` | ✅ | ✅ |
| `BY` | ✅ | ✅ |
| `reset_before` | ✅ | ✅, changed syntax |
| `reset_after` | ✅ | ✅, changed syntax |
| `reset_on_change` | ✅ | ✅, expressed as `onchange` |
| `time_window` | ✅ | ❌ |
| `global` | ✅ | ❌ |
| `allnum` | ✅ | ❌ |

Classic SPL:

```spl
| streamstats reset_before="(action=\"REBOOT\")" window=5 current=false sum(bytes) AS total BY host
```

SPL2:

```spl2
| streamstats current=false reset before action="REBOOT" window=5 sum(bytes) AS total BY host
```

Classic SPL reset syntax:

```text
reset_before="("<eval-expression>")"
reset_after="("<eval-expression>")"
reset_on_change=true
```

SPL2 reset syntax:

```text
reset before <expression>
      after <expression>
      onchange
```

SPL2 no longer uses separate classic-SPL reset arguments and does not have equivalents for `allnum`, `global`, or `time_window`. citehttps://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

---

# 6.45 A Complete `streamstats` Option Map

Keep this table as your quick reference.

```text
current=true
    → include the current result

current=false
    → exclude the current result

window=N
    → use a result-count window

time_window=5m
    → use a time-based window
      (classic SPL only)

BY field
    → maintain separate calculations by group

global=true
    → one shared window across BY groups
      (classic SPL only)

global=false
    → separate event window for each BY group
      (classic SPL only)

reset_before=<condition>
    → reset before calculating the matching event

reset_after=<condition>
    → calculate the matching event, then reset

reset_on_change=true
    → reset when a BY value changes
      (classic SPL)

allnum=true
    → require numerical values for numerical statistics
      (classic SPL only)
```

The official SPL reference lists these classic options and their defaults; the current SPL2 reference lists the reduced option set and changed reset syntax. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats citehttps://help.splunk.com/en/splunk-cloud-platform/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

---

# 6.46 Common `streamstats` Mistakes

### Mistake 1 — Thinking `streamstats` is the same as `stats`

Wrong mental model:

```text
streamstats = stats
```

Better:

```text
stats
    → final/grouped aggregate

streamstats
    → aggregate attached progressively to each result
```

### Mistake 2 — Forgetting order

This:

```spl
| streamstats current=false last(_time) AS previous_time
```

depends on the order of incoming results.

If you need chronological behavior:

```spl
| sort 0 _time
```

first.

### Mistake 3 — Confusing `window` with `time_window`

```text
window=5
    → five results

time_window=5m
    → five minutes
```

### Mistake 4 — Using the current result in its own baseline

This:

```spl
| streamstats window=6 avg(current) AS baseline
```

includes the current result.

For a previous-bucket baseline:

```spl
| streamstats window=6 current=false avg(current) AS baseline
```

### Mistake 5 — Forgetting that `window` also interacts with reset

When `reset_before`, `reset_after`, or `reset_on_change` is combined with `window`, the window is reset when the accumulation is reset. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

### Mistake 6 — Expecting `time_window` in SPL2

Classic SPL:

```spl
| streamstats time_window=5m count
```

SPL2:

```text
No direct streamstats time_window option
```

Do not copy classic-SPL options into SPL2 without checking the command reference.

### Mistake 7 — Writing a Boolean condition incorrectly in classic SPL

This:

```spl
reset_before=reset_flag
```

is not the documented classic syntax.

Use a Boolean expression in the reset clause, for example:

```spl
reset_before="(action=\"REBOOT\")"
```

or:

```spl
reset_before="(match(_raw,\"oracle.*\"))"
```

Classic SPL requires the documented reset expression format. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

---

# 6.47 Performance and Memory

`streamstats` keeps state in memory while calculating running or windowed statistics.

The important configuration limits include:

```text
max_stream_window
maxresultrows
max_mem_usage_mb
```

Current Splunk documentation lists `max_stream_window` with a default of 10,000 for `streamstats`. The search processor also uses `maxresultrows` and `max_mem_usage_mb` to control the amount of result and memory state it can maintain. citehttps://help.splunk.com/en/splunk-enterprise/administer/admin-manual/10.4/configuration-file-reference/10.4.0-configuration-file-reference/limits.conf citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

For example:

```spl
| streamstats window=500000 sum(bytes)
```

should not be treated as harmless simply because the syntax is valid.

Large windows can consume significant memory.

Practical habits:

```text
Filter first
    ↓
Reduce unnecessary events
    ↓
Choose the smallest useful window
    ↓
Use BY only when it helps the detection
```

Do not change `limits.conf` just to make an oversized search work unless you understand the memory impact.

---

# 6.48 Real SOC Example — Brute-Force Detection

Question:

> Did a source IP generate many failed logons in a short period?

Start with the event filter:

```spl
index=windows EventCode=4625
| sort 0 _time
```

Create a failure indicator:

```spl
| eval is_failure=1
```

Count recent failures by IP:

```spl
| streamstats time_window=5m sum(is_failure) AS failures_5m BY src_ip
```

Now each event can carry the number of failures associated with that IP in the current five-minute window.

You can then threshold [set a value above which an alert or investigation is triggered]:

```spl
| where failures_5m >= 10
```

The important architecture is:

```text
failed-logon events
       ↓
sort chronologically
       ↓
one event = one failure
       ↓
5-minute streamstats window
       ↓
group BY src_ip
       ↓
count failures
       ↓
threshold
```

This is an example of why `streamstats` is useful for security analytics.

---

# 6.49 `reset_after` Can Use a Calculated Field

A reset expression can reference a field produced by `streamstats`.

For example, reset after the running count reaches three:

```spl
| streamstats count AS n reset_after="(n>=3)"
```

Conceptually:

```text
event 1 → n=1
event 2 → n=2
event 3 → n=3 → reset after this event
event 4 → n=1
event 5 → n=2
event 6 → n=3 → reset after this event
```

This is useful when the reset condition depends on the result of the aggregation itself, rather than directly on an input field.

The expression capability is documented for `reset_after`, including references to fields returned by `streamstats`. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.3/search-commands/streamstats

---

# 6.50 Real SOC Example — Password Spray Pattern

Password spraying [trying a small number of passwords against many accounts] is different from repeated failures against one account.

Suppose the important fields are:

```text
src_ip
user
action
_time
```

You can build a per-IP rolling count:

```spl
index=windows EventCode=4625
| sort 0 _time
| eval failure=1
| streamstats time_window=10m sum(failure) AS failures_10m BY src_ip
```

Then investigate IPs with a high number of failures:

```spl
| where failures_10m >= 20
```

A second dimension can then be examined, such as the number of distinct users:

```spl
| eventstats dc(user) AS unique_users BY src_ip
| where failures_10m >= 20 AND unique_users >= 10
```

The detection logic is conceptually:

```text
many failures
      +
many targeted users
      +
same source IP
      +
short time interval
      ↓
password-spray hypothesis
```

The search does not prove malicious activity by itself. It identifies a pattern for investigation.

---

# 6.51 Fixed Buckets vs `streamstats` Windows vs `timewrap`

Now compare the major time-analysis mechanisms.

### Fixed bucket

Question:

> How many events happened every 10 minutes?

Use:

```spl
| bin _time span=10m
| stats count AS Events BY _time
```

or:

```spl
| timechart span=10m count
```

### Running / rolling calculation

Question:

> What is the current value compared with recent results?

Use:

```spl
| streamstats window=6 ...
```

or:

```spl
| streamstats time_window=10m ...
```

in classic SPL.

### Historical period comparison

Question:

> How does today compare with yesterday?

Use:

```spl
| timechart span=1h count
| timewrap 1d
```

The three operations answer different questions:

```text
Fixed clock interval
    → bin / stats span / timechart

Recent rolling state
    → streamstats

Same position in an earlier period
    → timewrap
```

---

# 6.52 The Complete `streamstats` Mental Model

Think of `streamstats` as a stateful calculator moving through results.

```text
             EVENT 1
                |
                v
        calculate current state
                |
                v
         output EVENT 1 + state
                |
             EVENT 2
                |
                v
        update current state
                |
                v
         output EVENT 2 + state
                |
             EVENT 3
                |
                v
        update current state
                |
                v
         output EVENT 3 + state
```

The state can be:

```text
count
sum
average
minimum
maximum
previous value
latest value
list of values
distinct count
...
```

And the state can be controlled by:

```text
current
window
BY
reset_before
reset_after
reset_on_change
time_window
global
allnum
```

The most important mental model is:

```text
streamstats
    =
process each incoming result
    +
maintain statistical state
    +
write the current state back to that result
```

---

# 6.53 `streamstats` Learning Sequence

For learning, do not memorize all options at once.

Use this progression:

```text
1. streamstats count
        ↓
2. streamstats sum(bytes)
        ↓
3. current=true / false
        ↓
4. window=3
        ↓
5. BY user
        ↓
6. window + BY
        ↓
7. reset_before
        ↓
8. reset_after
        ↓
9. reset_on_change
        ↓
10. time_window
        ↓
11. global
        ↓
12. allnum
        ↓
13. previous-event logic
        ↓
14. rolling baseline
        ↓
15. sliding-window detection
        ↓
16. session-like correlation
```

Once these are comfortable, `streamstats` becomes much easier to use in SOC searches.

---

# 6.54 Chapter Summary

### Search time range

```spl
index=windows earliest=-1h latest=now
```

### Fixed time bucket

```spl
| bin _time span=10m
| stats count AS Events BY _time
```

### Time-series chart

```spl
| timechart span=10m count
```

### Running calculation

```spl
| streamstats count AS n
```

### Running sum

```spl
| streamstats sum(bytes) AS running_bytes
```

### Exclude current result

```spl
| streamstats current=false sum(bytes) AS previous_total
```

### Fixed event-count window

```spl
| streamstats window=5 avg(bytes) AS avg_bytes
```

### Separate calculation by group

```spl
| streamstats window=5 sum(bytes) AS total BY src_ip
```

### Reset before a condition

```spl
| streamstats reset_before="(action=\"REBOOT\")" sum(bytes) AS total
```

### Reset after a condition

```spl
| streamstats reset_after="(action=\"REBOOT\")" sum(bytes) AS total
```

### Reset when the BY value changes

```spl
| streamstats count BY user reset_on_change=true
```

### Time-based window — classic SPL

```spl
| streamstats time_window=5m count AS events_5m
```

### Rolling baseline

```spl
| timechart span=10m count AS current
| streamstats window=6 current=false avg(current) AS baseline
```

### Previous event

```spl
| streamstats current=false last(_time) AS previous_time BY src_ip
```

### Conditional sliding count

```spl
| eval failure=if(action="failure",1,0)
| streamstats window=5 sum(failure) AS failures_last_5
```

The key decision is:

```text
What question am I asking?

How many overall?
    → stats

How many in each fixed time bucket?
    → bin + stats / timechart

What is the running or recent value?
    → streamstats

What happened immediately before this result?
    → streamstats current=false + last()

When should the accumulated state restart?
    → reset_before / reset_after / reset_on_change

How many occurred in a time interval?
    → time_window in classic SPL

How does this period compare with an earlier period?
    → timewrap
```

The central idea remains:

> **`streamstats` is about carrying statistical state from one result to the next.**

---

# 6.55 Progress So Far

```text
CHAPTER 1
Understanding Events & Fields
        ↓
CHAPTER 2
Writing Your First Query
        ↓
CHAPTER 3
Text Searching & Regex
        ↓
CHAPTER 4
Selecting, Sorting & Limiting
        ↓
CHAPTER 5
Grouping & Aggregation
        ↓
CHAPTER 6
Time-Based Analysis
        ↓
CHAPTER 7
Sequence & Correlation
(test, streamstats, state, event relationships)
```

Chapter 7 should build on the `streamstats` concepts learned here rather than re-teaching the command from the beginning.

---

# Official Splunk Documentation Used for This Chapter

This chapter was expanded and checked against current official Splunk documentation for `streamstats`, including classic SPL and SPL2 option differences. The examples above distinguish documented syntax from illustrative SOC detection patterns.


The chapter follows the current official Splunk documentation for:

- SPL and SPL2 timestamps and time ranges
- Time zones
- `earliest`, `latest`, `now()`, and `relative_time()`
- `bin` / `bucket`
- `stats` time spans in SPL2
- `timechart`
- `streamstats`
- `trendline` in traditional SPL
- `sparkline()` in SPL2
- `timewrap`
- `transaction`
- SPL vs SPL2 syntax differences

Official references:

1. https://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/dates-and-time/timestamps-and-time-ranges
2. https://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/dates-and-time/time-zones
3. https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/evaluation-functions/date-and-time-functions
4. https://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.4/search-commands/bucket
5. https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/stats-command/stats-command-overview-syntax-and-usage
6. https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/timechart-command/timechart-command-overview-and-syntax
7. https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/timechart-command/timechart-command-usage
8. https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage
9. https://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.2/search-commands/trendline
10. https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/time-functions
11. https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/timewrap-command/timewrap-command-overview-syntax-and-usage
12. https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.4/search-commands/transaction

For this course, current official Splunk documentation takes precedence over older community examples when SPL and SPL2 syntax differ.
