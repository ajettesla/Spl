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

# 6.18 Moving Baselines — The Splunk Way

The original CrowdStrike chapter introduced a `window()` function for calculating a moving baseline.

Splunk does **not** have a one-to-one `window()` function with that same syntax.

The native Splunk tool for running and moving calculations is:

```text
streamstats
```

Splunk documents `streamstats` as a command that adds cumulative or windowed statistical values to each result as the results are processed. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

---

# 6.19 Building a Rolling Baseline

Suppose we already have one count per 10-minute bucket:

```text
_time     current
-----------------
10:00       10
10:10       12
10:20        9
10:30       11
10:40       13
10:50       80
```

We want the average of the previous six buckets as a baseline.

Traditional SPL can use:

```spl
| streamstats window=6 current=f avg(current) AS baseline
```

The important part is:

```text
window=6
```

and:

```text
current=f
```

`current=f` [false: do not include the current result in the calculation] means the current bucket is not included in its own baseline.

SPL2 can use the corresponding `streamstats` syntax:

```spl2
| streamstats window=6 current=false avg(current) AS baseline
```

Current SPL2 documentation defines `window` as the number of events/results used for the calculation and `current=false` as excluding the current result from the calculation. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

---

# 6.20 Turning a Timechart Into a Baseline Search

A practical pattern is:

### Traditional SPL

```spl
index=windows EventCode=4625
| timechart span=10m count AS current
| streamstats window=6 current=f avg(current) AS baseline
```

The logic is:

```text
Raw events
   ↓
10-minute buckets
   ↓
current count
   ↓
previous 6 buckets
   ↓
baseline average
```

### SPL2

```spl2
search index=windows EventCode=4625
| timechart span=10m count() AS current
| streamstats window=6 current=false avg(current) AS baseline
```

Now each row can conceptually contain:

```text
_time     current    baseline
--------------------------------
10:00       10          null
10:10       12          10
10:20        9          11
10:30       11        10.33
10:40       13        10.5
10:50       80        11
```

The spike is now visible as:

```text
current  = 80
baseline = 11
```

This is the Splunk equivalent of the rolling-baseline concept from the original chapter.

---

# 6.21 `streamstats window` vs `time_window`

Traditional SPL has two different window concepts in `streamstats`:

```text
window=N
```

and:

```text
time_window=<time span>
```

`window=N` is based on the number of results.

`time_window` is based on time.

For example:

```spl
| streamstats window=6 avg(bytes) AS avg_bytes
```

uses a result-count window.

Whereas:

```spl
| streamstats time_window=1h avg(bytes) AS avg_bytes
```

uses a one-hour time window.

The current SPL2 `streamstats` documentation lists `window` but explicitly identifies `time_window` as an SPL option without an SPL2 equivalent.

Therefore:

```text
Traditional SPL
    window
    time_window

SPL2
    window
    no time_window equivalent in the current command reference
```

This is a good example of why we are studying SPL and SPL2 together instead of assuming every command option exists in both languages. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

---

# 6.22 Moving Average With `trendline` in Traditional SPL

Traditional SPL also has a dedicated command:

```text
trendline
```

It computes moving averages such as:

```text
sma
ema
wma
```

where:

```text
sma = simple moving average
ema = exponential moving average
wma = weighted moving average
```

For example:

```spl
index=windows EventCode=4625
| timechart span=10m count AS current
| trendline sma5(current) AS baseline
```

This is another native way to create a moving baseline in traditional SPL.

We will not treat `trendline` as an SPL2 equivalent here; the current official reference for the command is in the traditional SPL search-command documentation.

Splunk documents `trendline` as a command for moving averages. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.2/search-commands/trendline

---

# 6.23 `sparkline()` — A Small Time Series Inside a Summary

Sometimes you do not want a full timechart.

You want a summary table such as:

```text
host     events   trend
--------------------------
DC01       810    ▁▂▂▃▇▂
DC02       220    ▁▁▂▂▃▁
DC03       140    ▁▂▁▁▁▁
```

SPL2 provides `sparkline()` as a statistical function.

For example:

```spl2
FROM windows
| stats count() AS Events,
        sparkline(count(), 10m) AS Trend
  BY host
```

The idea is:

```text
stats summary
    +
small time trend
```

Current SPL2 documentation defines `sparkline()` as generating time-based trend displays within search results and supports it with `stats` and `streamstats`. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/statistical-and-charting-functions/time-functions

---

# 6.24 Comparing the Same Time Period Across Days or Weeks

Another time question is:

> Is today's activity similar to yesterday's?

or:

> How does this week compare with the previous weeks?

The native Splunk tool for this is:

```text
timewrap
```

For example:

### Traditional SPL

```spl
index=windows EventCode=4625
| timechart span=1h count
| timewrap 1d
```

### SPL2

```spl2
search index=windows EventCode=4625
| timechart span=1h count()
| timewrap 1d
```

`timewrap` takes the output from `timechart` and creates separate series for previous periods.

This answers a different question from a rolling baseline:

```text
timechart + streamstats
    ↓
How unusual is the current value compared with recent buckets?

 timechart + timewrap
    ↓
How does this period compare with the same position in earlier periods?
```

Splunk's current SPL2 documentation defines `timewrap` as wrapping timechart output so different periods become different series. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/timewrap-command/timewrap-command-overview-syntax-and-usage

---

# 6.25 `delta` — Difference Between Nearby Results

Another useful time-oriented operation is measuring the difference between adjacent results.

Traditional SPL provides:

```spl
| delta field AS change
```

For example:

```spl
index=windows
| timechart span=10m count AS current
| delta current AS change
```

Conceptually:

```text
previous = 100
current  = 140
change   = 40
```

The important detail is that `delta` compares results in the **search order**, not automatically in chronological order. Therefore, if the order matters, make the ordering explicit before using it.

Splunk documents `delta` as calculating the difference between the current result and a previous result in search order. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.0/search-commands/delta

We will use `streamstats` more heavily for sequence analysis later, so `delta` is an additional tool rather than the center of this chapter.

---

# 6.26 Fixed Buckets vs Rolling Windows vs Period Comparison

At this point, we have several different meanings of "time."

### Fixed interval

Question:

> How many events happened every 10 minutes?

Use:

```text
SPL
bin + stats

SPL2
stats ... BY _time span=10m

Both
 timechart span=10m ...
```

### Rolling calculation

Question:

> Is this bucket unusual compared with the previous six buckets?

Use:

```text
streamstats window=6
```

### Historical period comparison

Question:

> How does today compare with the same time yesterday?

Use:

```text
timechart + timewrap
```

These are different analytical questions.

---

# 6.27 Session-Like Activity: Not a Direct `session()` Translation

The original CrowdStrike chapter used a `session()` function to create groups based on gaps in activity.

Do not copy that function name into Splunk.

Traditional SPL provides the `transaction` command for grouping conceptually related events across time, including constraints such as:

```text
maxpause
maxspan
maxevents
startswith
endswith
```

For example:

```spl
index=web
| transaction clientip maxpause=5m maxspan=30m
```

This means events for the same `clientip` can be grouped into a transaction when the time constraints are satisfied.

Splunk documents `maxpause` as the maximum allowed pause between events and `maxspan` as the maximum overall transaction duration. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.4/search-commands/transaction

---

# 6.28 `transaction` Is Not the Same as `stats`

This distinction is extremely important.

`stats` answers questions such as:

```text
How many events?
What is the average?
What is the sum?
```

It collapses the events into a statistical result.

`transaction` is intended to group related events into transaction objects while retaining event-level information such as the combined raw events and transaction duration/event count.

Splunk explicitly recommends using `stats` instead of `transaction` when you only need aggregate statistics and grouping by fields is sufficient, because `stats` is generally more efficient. `transaction` becomes useful when grouping depends on time gaps, start/end conditions, reused identifiers, or when the combined raw event content is important. citehttps://help.splunk.com/en/splunk-cloud-platform/search/search-manual/10.4.2604/group-and-correlate-events/about-transactions

Therefore:

```text
Need statistics?
    ↓
stats

Need complex event grouping across time?
    ↓
transaction
```

For SPL2, do not invent a `session()` command simply because the CrowdStrike example had one. We will use the documented SPL2 commands, especially `streamstats`, for session-like logic where appropriate.

---

# 6.29 A Real SOC Example — Failed-Logon Attack Waves

Suppose the question is:

> Did one source IP generate repeated failed-logon waves?

Traditional SPL can use `transaction`:

```spl
index=windows EventCode=4625
| transaction src_ip maxpause=5m
| table _time src_ip duration eventcount
```

This is asking Splunk to create time-gap-based groups.

But if the question is simply:

> How many failures came from each source IP?

do not use `transaction`.

Use:

```spl
index=windows EventCode=4625
| stats count AS Attempts BY src_ip
| sort -Attempts
```

This is a key decision:

```text
Count / aggregate
    ↓
stats

Time-gap transaction
    ↓
transaction
```

---

# 6.30 Session-Like Logic With `streamstats`

There is another important pattern that will matter in later sequence analysis.

You can use `streamstats` to maintain state across events.

For example:

```spl
| streamstats current=f last(_time) AS previous_time BY src_ip
```

Conceptually:

```text
Current event
     ↓
What was the previous event time for this IP?
```

You can then calculate a gap:

```spl
| eval gap=_time-previous_time
```

This creates a building block for detecting activity gaps without using `transaction`.

SPL2 also supports `streamstats`, including `BY` grouping and reset conditions. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

The more advanced session-building patterns will belong in the later **sequence/correlation** chapter, so we will not duplicate them here.

---

# 6.31 A Practical Time Analysis Workflow

When you have a time-based SOC question, follow this process.

```text
1. Define the search time range
        ↓
2. Filter to the relevant events
        ↓
3. Decide what "time" means
        ↓
4. Choose the correct time operation
        ↓
5. Aggregate or compare
        ↓
6. Visualize or investigate the abnormal periods
```

For example:

> Find failed logon spikes during the last 24 hours.

Think:

```text
LAST 24 HOURS
       ↓
FAILED LOGONS
       ↓
10-MINUTE BUCKETS
       ↓
COUNT
       ↓
TIMECHART
```

Traditional SPL:

```spl
index=windows EventCode=4625 earliest=-24h
| timechart span=10m count AS FailedLogons
```

SPL2:

```spl2
search index=windows EventCode=4625 earliest=-24h
| timechart span=10m count() AS FailedLogons
```

---

# 6.32 Common Mistake — Treating `span` as Decoration

This is one of the most important lessons in the chapter.

These searches are both valid:

```spl
| timechart span=1m count
```

and:

```spl
| timechart span=1h count
```

But they ask different questions about the data.

If the behavior occurs in a three-minute burst, a one-hour span may hide the burst inside a large aggregate.

Therefore:

```text
Detection interval
      ↓
Choose span deliberately
```

Splunk's timechart documentation confirms that `span` controls the size of the time bins and that automatic spans depend on the time range. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/timechart-command/timechart-command-usage

---

# 6.33 Common Mistake — Assuming `bucket` Is a Function Like LogScale

Do not write a LogScale-style mental model such as:

```text
bucket(...)
```

and expect that to be the normal Splunk syntax.

Traditional SPL uses:

```spl
| bin _time span=10m
```

and `bucket` is an alias:

```spl
| bucket _time span=10m
```

SPL2 can use:

```spl2
| bin _time span=10m
```

or, more naturally for statistics:

```spl2
| stats count() BY _time span=10m
```

or:

```spl2
FROM windows
GROUP BY span(_time, 10m)
SELECT count(), _time
```

Splunk explicitly documents these SPL/SPL2 differences. citehttps://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.4/search-commands/bucket citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/stats-command/stats-command-overview-syntax-and-usage

---

# 6.34 Common Mistake — Using `timechart` Without Thinking About Series

Consider:

```spl
| timechart span=10m count BY host
```

If your environment has thousands of hosts, you may create a very large number of series.

`timechart` supports series limits and other options for controlling the number of split-by series returned.

For example, the current SPL2 documentation exposes a `limit` option for split-by series.

Do not automatically chart every possible value of a high-cardinality field [high cardinality: a field with a very large number of distinct values].

Instead ask:

```text
Which dimension actually helps the investigation?
```

Splunk documents split-by series limits and the `limit` option for `timechart`. citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/timechart-command/timechart-command-overview-and-syntax

---

# 6.35 Common Mistake — Confusing Event Time With Index Time

From Chapter 1 we learned:

```text
_time
```

is the event timestamp, while:

```text
_indextime
```

is associated with when Splunk indexed the event.

Time-based detections normally use `_time`, because the investigation usually asks:

> When did the activity happen?

But ingestion-delay investigations may compare:

```spl
| eval delay_sec=_indextime-_time
```

Do not use `_indextime` for a normal event timeline unless your actual question concerns ingestion or indexing delay.

---

# 6.36 Common Mistake — Filtering After Time Aggregation

Suppose you want failed logons from one host.

Prefer filtering the raw events before the time aggregation when the filter describes the event itself.

Traditional SPL:

```spl
index=windows EventCode=4625 host=DC01
| timechart span=10m count
```

not:

```spl
index=windows EventCode=4625
| timechart span=10m count BY host
| search host=DC01
```

The second approach can be useful in some circumstances, but it creates the time series for more hosts before filtering it.

The general principle is the same one we learned in Chapter 5:

```text
Filter the raw events as early as practical
       ↓
Then aggregate
```

---

# 6.37 The Complete Mental Model

Now your investigation can look like this:

```text
             INVESTIGATION QUESTION
                       |
                       v
                DEFINE TIME RANGE
                       |
                       v
                    FILTER
                       |
                       v
             WHAT DOES "TIME" MEAN?
                       |
       +---------------+----------------+
       |               |                |
       v               v                v
   FIXED SLICE     ROLLING VIEW    PERIOD COMPARISON
       |               |                |
       v               v                v
bin / bucket      streamstats       timewrap
stats span        window            + timechart
 timechart
       |
       +---------------+----------------+
                       |
                       v
                  AGGREGATE
                       |
                       v
                 INVESTIGATE SPIKES
```

And for session-like event grouping in traditional SPL:

```text
EVENTS
   ↓
FILTER
   ↓
transaction
   ↓
maxpause / maxspan / startswith / endswith
   ↓
TRANSACTION RESULTS
```

The central decision is:

> **What exactly does “time” mean in my question?**

---

# 6.38 Chapter Summary

In this chapter, we moved from ordinary event searching to time-based analysis.

### Search time range

Traditional SPL:

```spl
index=windows earliest=-1h latest=now
```

SPL2:

```spl2
FROM windows
WHERE earliest=-1h AND latest=now
```

### Fixed time grouping

Traditional SPL:

```spl
| bin _time span=10m
| stats count AS Events BY _time
```

SPL2:

```spl2
| stats count() AS Events BY _time span=10m
```

or:

```spl2
FROM windows
GROUP BY span(_time, 10m)
SELECT count() AS Events, _time
```

### Time-series chart

Traditional SPL:

```spl
| timechart span=10m count
```

SPL2:

```spl2
| timechart span=10m count()
```

### Rolling baseline

Traditional SPL:

```spl
| streamstats window=6 current=f avg(current) AS baseline
```

SPL2:

```spl2
| streamstats window=6 current=false avg(current) AS baseline
```

### Historical period comparison

```text
timechart + timewrap
```

### Session-like grouping

Traditional SPL:

```spl
| transaction src_ip maxpause=5m
```

The important distinction is:

```text
Fixed clock slice
    → bin / stats span / timechart

Rolling calculation
    → streamstats

Compare with previous periods
    → timewrap

Time-gap transaction grouping
    → transaction (traditional SPL)
```

---

# What You Should Remember

If you remember only these points:

```text
_time
→ When did the event happen?

earliest / latest
→ Which time range should be searched?

span
→ How wide is each time bucket?

bin / bucket
→ Put values into fixed groups in traditional SPL

stats ... BY _time span=...
→ Native SPL2 time grouping

timechart
→ Time-series visualization + aggregation

streamstats
→ Running / moving calculations

timewrap
→ Compare different time periods

transaction
→ Group related events across time in traditional SPL
```

And the most important question is:

> **Is my investigation asking about a fixed interval, a recent rolling baseline, a historical comparison, or a time-gap-based session?**

Once that decision becomes automatic, time-based SPL and SPL2 searches become much easier to construct.

---

# Progress So Far

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

---

# Official Splunk Documentation Used for This Chapter

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
