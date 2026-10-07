# Splunk SPL + SPL2

# Chapter 7 — Sequence & Correlation Analysis

Chapters 5 and 6 taught us how to answer questions about **quantity** and **time**.

Chapter 5 asked:

> How many failures are there?

Chapter 6 asked:

> How many failures happened in each 5-minute interval?

Now we move to a different kind of question:

> Did **three failures happen before a success**?

Or:

> What happened **immediately before** this process started?

Or:

> Did the number of bytes increase across **several consecutive events**?

Or:

> Did this source IP produce **more than 20 failures in the previous 5 minutes**?

These questions are about the **relationship and order** of events, not only their totals.

That is the purpose of this chapter.

We will build sequence analysis using the native Splunk tools that preserve event-level context:

```text
Event order
    ↓
sort / search order

Previous-event analysis
    ↓
streamstats current=f
autoregress
delta

Running calculations
    ↓
streamstats
accum

Fixed event windows
    ↓
streamstats window=N

Time-based windows
    ↓
streamstats time_window=X

Resetting state
    ↓
reset_before
reset_after
reset_on_change

Per-entity analysis
    ↓
BY user / BY host / BY src_ip

Complex event grouping
    ↓
transaction
```

The goal is not to memorize every option. The goal is to understand what kind of **sequence question** each tool answers.

* * *

# 7.1 What Changes When Order Matters?

Consider six authentication events for one account:

```text
failure
failure
failure
success
failure
success
```

If we use aggregation from Chapter 5:

```spl
index=windows
| stats count(eval(action="failure")) AS failures
        count(eval(action="success")) AS successes
        BY user
```

we might learn:

```text
user    failures    successes
teja        4           2
```

But that does not tell us the sequence.

The important part may be:

```text
failure → failure → failure → SUCCESS
```

The same counts can occur in a harmless order such as:

```text
success → failure → success → failure → failure → failure
```

So:

```text
Aggregation
    ↓
How many?

Sequence analysis
    ↓
In what order?
```

The key change in Chapter 7 is that **the order itself becomes evidence**.

* * *

# 7.2 The Most Important Rule — Sequence Logic Needs a Defined Order

Before using a command to compare a current event with previous events, decide what "previous" means.

Splunk processes the results in the order it receives them. Therefore, sequence commands do not magically know that you intended chronological order.

For event-by-event timeline analysis, a common lab pattern is:

```spl
| sort 0 _time
```

This puts events in ascending [earliest-to-latest] `_time` order.

Then:

```spl
| streamstats ...
```

processes the events in that order.

For example:

```spl
index=windows
| sort 0 _time
| streamstats current=f last(src_ip) AS previous_src_ip
```

Now "previous" means the previous event in chronological order.

Without the ordering step, this:

```spl
| streamstats current=f last(src_ip) AS previous_src_ip
```

may compare an event with whatever result Splunk processed immediately before it, which can be the wrong event for your investigation.

There is one important practical detail:

```text
sort 0 _time
```

is easy to understand and useful for laboratory work, but sorting a very large result set can be expensive. In production searches, always think about the search order and reduce the event set before doing large sequence calculations.

The official `streamstats` reference describes the command as operating on its incoming search results, and the `delta` reference explicitly states that comparison is performed in search order. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.0/search-commands/delta

* * *

# 7.3 `streamstats` — The Core Sequence Tool

`streamstats` adds a statistical calculation to each result as the result is processed.

This is what makes it fundamentally different from `stats`.

From Chapter 5:

```spl
| stats sum(bytes) AS total
```

produces one aggregate [summary] over the result set.

`streamstats` keeps the events and adds the changing value to them:

```spl
| streamstats sum(bytes) AS running_total
```

Example:

```text
_time    bytes    running_total
10:00      100          100
10:01      200          300
10:02      300          600
10:03      400         1000
```

Read it as:

> "When I reach this event, what is the total so far?"

Splunk describes `streamstats` as calculating statistics for each event at the time that event is seen. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

The basic form is:

```spl
| streamstats <stats-function>(<field>) AS <new_field>
```

Examples:

```spl
| streamstats count AS event_number
```

```spl
| streamstats sum(bytes) AS running_bytes
```

```spl
| streamstats avg(response_time) AS running_avg
```

```spl
| streamstats max(bytes) AS running_max
```

You can use more than one statistical function in the same `streamstats` command.

```spl
| streamstats count AS event_number
             sum(bytes) AS running_bytes
             avg(bytes) AS running_avg
```

The output is still event-level data.

That is the central idea of this chapter.

* * *

# 7.4 `streamstats count` — Count Events as They Arrive

The simplest sequence example is:

```spl
| streamstats count AS event_number
```

Suppose the ordered events are:

```text
Event 1
Event 2
Event 3
Event 4
Event 5
```

The result becomes:

```text
Event 1 → event_number=1
Event 2 → event_number=2
Event 3 → event_number=3
Event 4 → event_number=4
Event 5 → event_number=5
```

This creates a running counter.

It is useful for:

- numbering events,
- identifying position in a sequence,
- creating sequence IDs,
- building later correlation logic.

For example:

```spl
index=windows
| sort 0 _time
| streamstats count AS sequence_number
| table _time user action sequence_number
```

The important difference from `stats count` is:

```text
stats count
    ↓
one final count

streamstats count
    ↓
a count attached to every event
```

Splunk's official examples show that `streamstats count` adds 1 to the first event, 2 to the second, and so on. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.5 `current=` — Include or Exclude the Current Event

This option is one of the most important `streamstats` concepts.

The default is:

```spl
current=true
```

That means the current event participates in the calculation.

Example:

```spl
| streamstats sum(bytes) AS total current=true
```

For:

```text
bytes
100
200
300
```

the result is:

```text
bytes    total
100       100
200       300
300       600
```

The current event is included.

Now:

```spl
| streamstats sum(bytes) AS total current=false
```

means:

> Calculate using the events before the current event; do not include the current event itself.

The result is conceptually:

```text
bytes    total
100       NULL
200       100
300       300
```

This is extremely useful when the question is:

> "What was true before this event?"

rather than:

> "What is true including this event?"

Remember:

```text
current=true
    ↓
include this event

current=false
    ↓
exclude this event
```

Splunk documents `current=true` as the default and `current=false` as using the previous event's value for the calculation. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.6 `current=f` for Previous-Event Information

One of the most useful sequence patterns is:

```spl
| streamstats current=f last(field) AS previous_field
```

Suppose:

```text
_time    src_ip
10:00    10.0.0.1
10:01    10.0.0.2
10:02    10.0.0.2
10:03    10.0.0.7
```

Search:

```spl
| sort 0 _time
| streamstats current=f last(src_ip) AS previous_src_ip
```

Conceptually:

```text
_time    src_ip       previous_src_ip
10:00    10.0.0.1     NULL
10:01    10.0.0.2     10.0.0.1
10:02    10.0.0.2     10.0.0.2
10:03    10.0.0.7     10.0.0.2
```

Now you can compare the current value with the previous value.

For example:

```spl
| eval changed=if(src_ip!=previous_src_ip,1,0)
```

This is a general sequence-analysis pattern:

```text
Current event
      ↓
Previous event
      ↓
Compare
      ↓
Detect change
```

The same idea works with:

```text
user
process
parent_process
dest_ip
dest_port
bytes
status
_action
_time
```

* * *

# 7.7 `last()`, `first()`, `latest()`, and `earliest()` Inside `streamstats`

`streamstats` supports event-order functions such as:

```text
first()
last()
earliest()
latest()
```

These do not all mean the same thing.

A useful way to think about them is:

```text
first()
    → first value in the stream window

last()
    → last value in the stream window

earliest()
    → earliest value by event time

latest()
    → latest value by event time
```

For sequence analysis, `last()` with `current=f` is often the easiest way to retrieve the previous event's field:

```spl
| sort 0 _time
| streamstats current=f last(process) AS previous_process
```

For time-aware comparisons, be careful about whether you need:

```text
stream order
or
event timestamp order
```

The commands operate on the search results and on the window you define. Do not assume these concepts are interchangeable.

The official `streamstats` documentation lists event-order functions including `earliest`, `first`, `last`, and `latest`. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.8 `BY` — Maintain Separate State for Each Entity

Suppose your logs contain two users:

```text
teja
john
teja
john
teja
```

If you run:

```spl
| streamstats count AS count
```

there is one overall counter:

```text
teja → 1
john → 2
teja → 3
john → 4
teja → 5
```

But often a security investigation needs a counter per entity.

Use:

```spl
| streamstats count AS attempts BY user
```

Now the state is maintained separately for each user:

```text
teja → 1
john → 1
teja → 2
john → 2
teja → 3
```

The same pattern works for:

```spl
| streamstats count AS attempts BY src_ip
```

or:

```spl
| streamstats count AS attempts BY host
```

or:

```spl
| streamstats count AS attempts BY user src_ip
```

Think:

```text
BY user
    ↓
one calculation for each user

BY src_ip
    ↓
one calculation for each source IP

BY user src_ip
    ↓
one calculation for each user + source-IP combination
```

This is the sequence equivalent of the grouping concept you learned in Chapter 5, but the calculation now evolves event by event rather than returning only one final row per group.

* * *

# 7.9 `window=` — Use a Fixed Number of Previous Events

Without a `window`, `streamstats` accumulates over the available stream, subject to Splunk's stream-window limits.

With:

```spl
| streamstats window=3 sum(bytes) AS total
```

the calculation uses a maximum of three events.

Suppose:

```text
bytes
100
200
300
400
500
```

Then:

```text
100
100 + 200
100 + 200 + 300
200 + 300 + 400
300 + 400 + 500
```

produces:

```text
bytes    total
100       100
200       300
300       600
400       900
500      1200
```

The window slides forward one event at a time.

Think:

```text
Event 1
[100]

Event 2
[100, 200]

Event 3
[100, 200, 300]

Event 4
[200, 300, 400]

Event 5
[300, 400, 500]
```

This is a **fixed event window** [a window containing a fixed number of events].

Splunk defines `window=N` as the number of events used for the calculation, with `window=0` meaning all previous and current events by default. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.10 `window=` + `current=false`

Now combine the two concepts.

```spl
| streamstats window=3 current=false sum(bytes) AS previous_total
```

`window=3` says:

> Use at most three events.

`current=false` says:

> Do not include the current event.

Suppose:

```text
Event   bytes
1       100
2       200
3       300
4       400
5       500
```

The relevant previous-event sets become approximately:

```text
Event 1 → no previous event
Event 2 → [100]
Event 3 → [100, 200]
Event 4 → [100, 200, 300]
Event 5 → [200, 300, 400]
```

So:

```text
Event   bytes   previous_total
1       100         NULL
2       200          100
3       300          300
4       400          600
5       500          900
```

This is useful when you want a baseline that does not include the value you are evaluating.

For example:

```text
current value
    compared with
previous values
```

This pattern is common in anomaly [behavior that differs from an expected pattern] detection.

* * *

# 7.11 `global=` — Shared Window vs Separate Windows

`global` matters when `window` is used.

The classic SPL syntax is:

```spl
global=true
```

or:

```spl
global=false
```

The default is:

```spl
global=true
```

Splunk defines this option as controlling whether a single window is used or separate windows are used according to the `BY` clause. With `global=false` and a non-zero `window`, each `BY` group gets its own window. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

For example:

```spl
| streamstats window=5 global=false avg(bytes) BY user
```

means:

> Maintain a five-event window separately for each user.

Conceptually:

```text
TEJA
    [teja event 1]
    [teja event 2]
    [teja event 3]
    ...

JOHN
    [john event 1]
    [john event 2]
    [john event 3]
    ...
```

By contrast, with:

```spl
| streamstats window=5 global=true avg(bytes) BY user
```

the window is global [shared] rather than separately maintained for each `BY` group.

This option becomes important when you are designing a per-user or per-IP sliding calculation.

A useful rule is:

```text
BY
    ↓
Which entity gets the result?

global
    ↓
Is the finite window shared or separate?

window
    ↓
How many events does the window hold?
```

The official Splunk example for a per-group five-event window uses:

```spl
| streamstats avg(foo) BY bar window=5 global=f
```

citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.12 `time_window=` — Window Based on Time

`window=5` means five events.

Sometimes that is not what you want.

Security questions are often phrased like:

> How many failures happened during the previous five minutes?

Use:

```spl
| streamstats count AS failures_last_5m time_window=5m
```

For this type of analysis, the events must be ordered in ascending or descending `_time`.

For ascending time:

```text
10:00
10:01
10:03
10:04
10:06
```

the window represents the relevant previous time period as the search moves forward.

For descending time, the direction is reversed, so the window represents the next five minutes relative to the current event.

Splunk documents these direction-dependent semantics and requires ordered `_time` values for `time_window`. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

A common lab query is:

```spl
index=windows EventCode=4625
| sort 0 _time
| streamstats count AS failures_last_5m time_window=5m BY user
| where failures_last_5m > 20
```

Read it as:

> For each user, walk through the events chronologically and calculate how many matching events are in the preceding five-minute window.

This is different from `timechart span=5m`.

`timechart` creates fixed buckets:

```text
10:00–10:05
10:05–10:10
```

`time_window=5m` follows each event:

```text
current event
     ← previous 5 minutes
```

That distinction matters when activity occurs across bucket boundaries.

* * *

# 7.13 Combining `time_window=` and `window=`

You can use both.

Example:

```spl
| streamstats time_window=5m window=1000 count AS recent_events
```

This means:

> Use the five-minute time window, but do not keep more than 1000 events in the calculation.

The two limits therefore serve different purposes:

```text
time_window=5m
    ↓
time boundary

window=1000
    ↓
maximum event count
```

Splunk documents that `window` can be combined with `time_window` to specify the maximum number of events in the time window. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.14 `reset_before=` — Reset Before Calculating the Event

`reset_before` takes a Boolean [true/false] expression.

Classic SPL syntax is:

```spl
reset_before="("<eval-expression>")"
```

For example:

```spl
| streamstats count AS sequence_count
    reset_before="(action=\"success\")"
```

The reset condition is evaluated before `streamstats` produces the result for the event.

Suppose the ordered actions are:

```text
failure
failure
failure
success
failure
failure
```

The success event triggers the reset before its calculation.

Think:

```text
previous state
      ↓
condition TRUE?
      ↓
RESET
      ↓
calculate current event
```

Splunk documents that `reset_before` resets all accumulated statistics before the calculation for the matching event. If a window is present, the window is also reset. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.15 Why Are the Quotes Escaped in `reset_before`?

This is worth learning because it is a common syntax problem.

You may write:

```spl
match(_raw,"oracle.*")
```

normally.

But inside the quoted `reset_before` expression:

```spl
reset_before="(match(_raw,\"oracle.*\"))"
```

the inner quotes around `oracle.*` are escaped.

Why?

Because there are two levels of quotes:

```text
outer expression:
reset_before=" ... "

inner string:
match(_raw,"oracle.*")
```

The inner quotes would otherwise terminate the outer string.

So:

```spl
"oracle.*"
```

becomes:

```spl
\"oracle.*\"
```

inside the outer quoted expression.

Splunk's own `streamstats` documentation gives examples using escaped quotation marks inside `reset_after` expressions. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

Therefore this is valid classic SPL:

```spl
index=*
| streamstats reset_before="(match(_raw,\"oracle.*\"))"
    window=3
    sum(pid) AS sumpid
| table _time sumpid pid _raw
```

Do not add escaping when the regex is not nested inside another quoted expression:

```spl
| where match(_raw,"oracle.*")
```

Here:

```text
match(_raw, "oracle.*")
```

is already the complete expression.

* * *

# 7.16 `reset_after=` — Reset After Calculating the Event

`reset_after` is similar, but the reset happens after the calculation for the current event.

Example:

```spl
| streamstats count AS sequence_count
    reset_after="(action=\"success\")"
```

Think:

```text
calculate current event
      ↓
condition TRUE?
      ↓
RESET
      ↓
next event starts fresh
```

Suppose the events are:

```text
failure
failure
failure
success
failure
failure
```

The success event belongs to the sequence that just ended.

That makes `reset_after` particularly useful for:

```text
failures → success
```

patterns.

Splunk documents that `reset_after` resets accumulated statistics after the matching event has received its calculated result. With `window`, the window is reset too. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.17 `reset_before` vs `reset_after`

This difference is easier to understand with a table.

Suppose:

```text
1  failure
2  failure
3  success
4  failure
```

### `reset_before`

```spl
| streamstats count AS c
    reset_before="(action=\"success\")"
```

Conceptually:

```text
failure → c=1
failure → c=2
success → RESET, then calculate success
failure → c=2
```

The matching success starts a new state.

### `reset_after`

```spl
| streamstats count AS c
    reset_after="(action=\"success\")"
```

Conceptually:

```text
failure → c=1
failure → c=2
success → c=3, then RESET
failure → c=1
```

The matching success finishes the old state.

Remember:

```text
reset_before
    → matching event STARTS the new state

reset_after
    → matching event ENDS the old state
```

This distinction is central to sequence detection.

* * *

# 7.18 `reset_on_change=` — Reset When the `BY` Value Changes

`reset_on_change=true` resets the accumulated statistics when the group-by fields change.

Example:

```spl
| streamstats count BY shift reset_on_change=true
```

If the stream is:

```text
DAY
DAY
NIGHT
NIGHT
NIGHT
DAY
NIGHT
```

the counter becomes:

```text
DAY    1
DAY    2
NIGHT  1
NIGHT  2
NIGHT  3
DAY    1
NIGHT  1
```

The important word here is **change**.

This is not simply:

> "Give me one lifetime counter per user."

Instead:

> "Reset when the value of the `BY` fields changes in the incoming stream."

Therefore the order matters.

For example:

```text
teja
teja
john
john
teja
```

causes:

```text
teja  1
teja  2
john  1
john  2
teja  1
```

because the stream changed from `teja` to `john`, and later back to `teja`.

Splunk documents this exact behavior and notes that only events having all the group-by fields can trigger the reset. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

This is very different from simply using:

```spl
| streamstats count BY user
```

which maintains separate aggregation state for the users without resetting just because a different user's event arrived.

* * *

# 7.19 `allnum=` — Numeric Safety for Statistical Calculations

The classic SPL option:

```spl
allnum=true
```

tells `streamstats` to compute numeric statistics for a field only when all values being considered are numeric.

Default:

```spl
allnum=false
```

Example:

```spl
| streamstats allnum=true avg(bytes) AS avg_bytes
```

Suppose the values contain:

```text
100
200
"unknown"
400
```

The presence of a non-numeric value matters when `allnum=true`.

This option is mostly useful when the input data is inconsistent and you explicitly want a numeric calculation only when every value in the relevant calculation is numeric.

For normal numeric fields such as:

```text
bytes
duration
latency
pid
port
```

you may not need it often.

The official SPL reference defines `allnum=true` exactly this way. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.20 The `streamstats` Options — One Table

At this point, the major classic SPL options can be summarized like this:

```text
current
    Include the current event?

window
    How many events?

time_window
    How much time?

BY
    Which entity/group?

global
    Shared window or separate windows
    when window is used?

reset_before
    Reset before the matching event is calculated

reset_after
    Reset after the matching event is calculated

reset_on_change
    Reset when the BY values change

allnum
    Require numeric values for numeric statistics
```

A compact mental model is:

```text
current     → THIS event?
window      → HOW MANY events?
time_window → HOW MUCH TIME?
BY          → WHO?
global      → SHARED or SEPARATE window?
reset_*     → WHEN does state restart?
allnum      → ARE all values numeric?
```

The current classic SPL reference lists these options and their defaults. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.21 `streamstats` Can Run Multiple Calculations at Once

You do not need a separate command for each statistic.

For example:

```spl
| streamstats count AS events_seen
             sum(bytes) AS bytes_so_far
             avg(bytes) AS avg_bytes
             max(bytes) AS max_bytes
```

Now every event can contain:

```text
events_seen
bytes_so_far
avg_bytes
max_bytes
```

This is powerful for sequence analysis because you can create several pieces of state from the same stream.

Example:

```spl
index=proxy
| sort 0 _time
| streamstats count AS events_seen
             sum(bytes_out) AS total_bytes
             avg(bytes_out) AS avg_bytes
| table _time src_ip bytes_out events_seen total_bytes avg_bytes
```

* * *

# 7.22 `streamstats` With Conditional Aggregation

Because `streamstats` accepts statistical aggregation expressions, you can count only events that meet a condition.

For example:

```spl
| streamstats count(eval(action="failure")) AS failures_so_far
```

This counts only failure events in the running calculation.

You can also maintain multiple states:

```spl
| streamstats count(eval(action="failure")) AS failures_so_far
             count(eval(action="success")) AS successes_so_far
```

This produces:

```text
action     failures_so_far    successes_so_far
failure          1                    0
failure          2                    0
success          2                    1
failure          3                    1
success          3                    2
```

This is useful, but remember:

> A running total of failures is not automatically a **consecutive** failure count.

If you need "failures since the last success," combine the conditional count with a reset condition.

* * *

# 7.23 Detecting "Failures Then Success"

This is one of the most useful SOC patterns in this chapter.

Suppose your events contain:

```text
action=failure
action=failure
action=failure
action=success
```

We want:

> Count failures until a success occurs, then reset.

A production-shaped classic SPL pattern is:

```spl
index=windows
(action="failure" OR action="success")
| sort 0 _time
| streamstats count(eval(action="failure")) AS failures
    reset_after="(action=\"success\")"
    BY user
| where action="success" AND failures>=3
| table _time user failures action
```

Read this as:

```text
1. Search only the relevant events.
2. Put them in chronological order.
3. Count failures for each user.
4. When success occurs, use that success as the end marker.
5. Reset after the success.
6. Keep successful events that had at least 3 failures before them.
```

For:

```text
failure
failure
failure
success
```

the success event can carry:

```text
failures=3
```

which makes it detectable.

This is the SPL equivalent of the sequence idea:

```text
FAILURE → FAILURE → FAILURE → SUCCESS
```

Notice that we did not repeat the `stats` concepts from Chapter 5. We reused `count()` inside a streaming context.

* * *

# 7.24 Per-User Sequence Detection

Security sequences almost always belong to an entity.

The entity might be:

```text
user
host
src_ip
dest_ip
process
account
```

Therefore:

```spl
| streamstats ... BY user
```

is often essential.

For example:

```spl
index=windows
(action="failure" OR action="success")
| sort 0 _time
| streamstats count(eval(action="failure")) AS failures
    reset_after="(action=\"success\")"
    BY user
| where action="success" AND failures>=5
```

Without `BY user`, failures from different accounts could contribute to one shared sequence.

For example:

```text
alice failure
bob   failure
alice failure
bob   failure
alice success
```

If you treat the whole stream as one state, the events from Bob can contaminate Alice's sequence.

The rule is:

```text
If the question is:
"Did THIS user do X then Y?"
    ↓
BY user
```

Likewise:

```text
"Did THIS source IP do X then Y?"
    ↓
BY src_ip
```

and:

```text
"Did THIS host do X then Y?"
    ↓
BY host
```

* * *

# 7.25 Previous Event With `streamstats`

Suppose you want to answer:

> What was the previous process before the current process?

Use:

```spl
index=windows
| sort 0 _time
| streamstats current=f last(process_name) AS previous_process
| table _time process_name previous_process
```

Example:

```text
_time    process_name    previous_process
10:00    explorer.exe    NULL
10:01    powershell.exe  explorer.exe
10:02    cmd.exe         powershell.exe
10:03    rundll32.exe    cmd.exe
```

Now you can search for a suspicious transition:

```spl
| where process_name="rundll32.exe"
    AND previous_process="cmd.exe"
```

The sequence itself becomes the detection condition:

```text
cmd.exe
   ↓
rundll32.exe
```

This is often more informative than looking for either process independently.

* * *

# 7.26 Previous Event With `autoregress`

Splunk also has the `autoregress` command.

It copies one or more previous values from a field into the current event.

Basic syntax:

```spl
| autoregress field AS newfield
```

For example:

```spl
| autoregress src_ip AS previous_src_ip
```

This copies the previous `src_ip` into `previous_src_ip`.

You can also request a specific earlier event:

```spl
| autoregress src_ip AS old_src_ip p=3
```

Now the value comes from the third previous event.

Or copy a range:

```spl
| autoregress bytes p=2-4
```

which produces fields such as:

```text
bytes_p2
bytes_p3
bytes_p4
```

This makes `autoregress` especially useful when your question is:

> "What were the values two, three, or four events ago?"

The official Splunk reference documents `p=<int>` and `p=<int>-<int>` for previous-event offsets. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.6/search-commands/autoregress

* * *

# 7.27 `streamstats current=f last()` vs `autoregress`

These tools overlap, but the learning idea is different.

### `streamstats`

```spl
| streamstats current=f last(bytes) AS previous_bytes
```

asks:

> What was the previous value within the stream calculation?

### `autoregress`

```spl
| autoregress bytes AS previous_bytes
```

asks:

> Copy the previous event's value into the current event.

For deeper offsets:

```spl
| autoregress bytes p=3
```

asks:

> What was the value three events ago?

A useful practical rule is:

```text
Need running statistics or state?
    ↓
streamstats

Need previous/Nth previous field values?
    ↓
autoregress
```

* * *

# 7.28 `delta` — Difference From a Previous Result

`delta` calculates the difference between the current result and a previous result for a numeric field.

Syntax:

```spl
| delta field AS change
```

Example:

```spl
index=*
| sort 0 _time
| delta bytes AS byte_change
| table _time bytes byte_change
```

Suppose:

```text
bytes
100
150
220
200
```

Then, in ascending search order:

```text
100 → NULL
150 → 50
220 → 70
200 → -20
```

The calculation is:

```text
current value - previous value
```

The previous result is determined by **search order**, so ordering matters.

If the result order is reversed, the sign is reversed as well.

The official reference also provides:

```spl
| delta bytes AS change p=2
```

to compare the current result with the result two positions earlier in search order. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.0/search-commands/delta

* * *

# 7.29 `delta` for Network Traffic Trends

Suppose a host sends:

```text
1000
1200
1400
1800
```

Use:

```spl
| sort 0 _time
| delta BytesSent AS change
```

Now:

```text
change
NULL
200
200
400
```

You can then detect increases:

```spl
| where change > 0
```

or unusually large jumps:

```spl
| where change > 10000
```

This is different from `stats sum()`.

`sum()` tells you the amount.

`delta` tells you the **change between adjacent results**.

That difference is important when analyzing behavior rather than totals.

* * *

# 7.30 Detecting a Rising Sequence

Now we can combine:

```text
delta
+
streamstats
```

Suppose `BytesSent` is:

```text
100
200
300
400
250
```

First:

```spl
| sort 0 _time
| delta BytesSent AS change
```

produces:

```text
NULL
100
100
100
-150
```

Now create a rise flag:

```spl
| eval rising=if(change>0,1,0)
```

Then calculate a running number of consecutive rises using reset logic.

A simple pattern is:

```spl
| streamstats count(eval(rising=1)) AS rises
```

But this is **not yet a consecutive counter** because it keeps counting all rises.

To count runs of consecutive rises, reset when the rise condition fails:

```spl
| streamstats count(eval(rising=1)) AS consecutive_rises
    reset_after="(rising=0)"
```

Then:

```spl
| where consecutive_rises>=3
```

can identify points where the stream has contained three rising events within the current run.

The exact placement of the reset condition matters, so always inspect a small test dataset before turning sequence logic into a production detection.

* * *

# 7.31 `accum` — A Simpler Running Sum

Splunk also provides:

```spl
| accum field
```

`accum` calculates a running sum for a numeric field.

Example:

```spl
| accum bytes AS total_bytes
```

For:

```text
100
200
300
```

you get:

```text
bytes    total_bytes
100      100
200      300
300      600
```

`accum` is therefore a specialized [designed for one specific purpose] running-sum command.

The official syntax is:

```spl
accum <field> [AS <newfield>]
```

and the field must contain numeric values. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.6/search-commands/accum

Compare:

```text
accum
    → running sum only

streamstats
    → running sum
      running count
      running average
      min/max
      event-order functions
      windows
      resets
```

Therefore `streamstats` is the more flexible sequence tool, while `accum` is simple when you only need a cumulative numeric sum.

* * *

# 7.32 Running Cumulative Exfiltration

Suppose an endpoint sends:

```text
2 MB
3 MB
5 MB
4 MB
7 MB
```

No single event is huge.

But the cumulative amount may be important.

Use:

```spl
index=network
| sort 0 _time
| streamstats sum(bytes_out) AS total_bytes_out BY src_ip
| where total_bytes_out > 500000000
```

The detection is based on the amount accumulated so far.

The same idea can use:

```spl
| accum bytes_out AS total_bytes_out
```

if you do not need per-IP state or other stream statistics.

* * *

# 7.33 `streamstats` After `timechart`

This is a powerful combination from Chapter 6.

Suppose Chapter 6 gives us:

```spl
index=windows EventCode=4625
| timechart span=10m count AS current
```

Now we have one result per time bucket:

```text
_time    current
10:00      10
10:10      12
10:20       9
10:30      80
```

We can feed those results into `streamstats`:

```spl
| streamstats window=3 current=f avg(current) AS baseline
```

Now `streamstats` operates on the **timechart results**.

This gives:

```text
_time    current    baseline
10:00      10         NULL
10:10      12          10
10:20       9          11
10:30      80        10.33
```

This is an important concept:

```text
raw events
   ↓
timechart
   ↓
time buckets
   ↓
streamstats
   ↓
rolling calculation over those buckets
```

Therefore `streamstats` does not have to operate directly on raw events. It can process the results produced by another transforming command.

Splunk's `streamstats` documentation explicitly describes this use with results produced by `stats`, `chart`, and `timechart`. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.34 Fixed Event Window vs Time Window

You now have two fundamentally different windows.

### Event-count window

```spl
| streamstats window=5 avg(bytes) AS avg_bytes
```

Question:

> What is the average over the latest five events?

### Time window

```spl
| streamstats time_window=5m avg(bytes) AS avg_bytes
```

Question:

> What is the average over the relevant five-minute period?

These are not interchangeable.

Imagine:

```text
10:00  event
10:00  event
10:01  event
10:10  event
10:11  event
```

Five events may span eleven minutes.

Five minutes may contain only three events.

So:

```text
window=5
    → event count defines the window

time_window=5m
    → time defines the window
```

* * *

# 7.35 Exact Difference Between `timechart span` and `time_window`

This distinction is one of the most useful things in SOC detection design.

### `timechart span=5m`

Creates fixed boundaries:

```text
10:00 ───── 10:05
10:05 ───── 10:10
10:10 ───── 10:15
```

An event at:

```text
10:04:59
```

and another event at:

```text
10:05:01
```

are put into different buckets.

### `streamstats time_window=5m`

The window follows the current event.

For an event at:

```text
10:07:00
```

you examine the relevant five-minute period around that point, based on the search order.

There are no fixed five-minute bucket boundaries in the calculation.

Therefore:

```text
timechart
    → fixed slices

time_window
    → moving time window
```

Use fixed slices for dashboards and bucket-based trends.

Use moving windows when the detection should not depend on arbitrary bucket boundaries.

* * *

# 7.36 Detecting a Burst Without Bucket-Edge Problems

Suppose login failures occur:

```text
10:03
10:04
10:05
10:06
10:07
10:07
10:08
```

A five-minute `timechart` can split that activity across:

```text
10:00–10:05
10:05–10:10
```

But:

```spl
| streamstats count AS recent_failures time_window=5m
```

lets every event ask:

> How many failures are in the relevant five-minute moving window?

This is much closer to a detection requirement such as:

> Alert whenever a user has more than 20 failures in the previous five minutes.

* * *

# 7.37 Building a Sequence State With `streamstats`

One powerful use of `streamstats` is creating a state number.

Suppose:

```text
start
activity
activity
end
start
activity
end
```

Create a marker:

```spl
| eval new_state=if(action="start",1,0)
```

Then:

```spl
| streamstats sum(new_state) AS state_id
```

gives each event a state number.

Conceptually:

```text
start      → 1
activity   → 1
activity   → 1
end        → 1
start      → 2
activity   → 2
end        → 2
```

Now you have:

```text
state_id
```

that can be used later for grouping.

This is a useful SPL technique when you want to emulate [reproduce the behavior of] a "partition at marker" concept without pretending that Splunk has a direct `partition()` command identical to another query language.

* * *

# 7.38 `transaction` — Group Related Events

Another important SPL sequence tool is:

```spl
transaction
```

A transaction groups related events into a transaction [a logical collection of related events].

Example:

```spl
sourcetype=access_combined
| transaction clientip maxpause=5m maxspan=30m
```

This says:

```text
same clientip
    +
no pause larger than 5 minutes
    +
total transaction span no larger than 30 minutes
```

Splunk documents `maxpause` as the maximum allowed gap between events and `maxspan` as the maximum overall transaction duration. citehttps://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/9.4/search-commands/transaction

`transaction` is useful when the actual grouping logic depends on event relationships and time gaps.

But do not use it just because you need a count.

If the question is:

> How many events did each source IP generate?

use:

```spl
| stats count BY clientip
```

rather than:

```spl
| transaction clientip
```

Splunk recommends `stats` when simple aggregation and grouping are sufficient because `transaction` has more processing overhead and is intended for more complex event grouping. citehttps://help.splunk.com/en/splunk-enterprise/manage-knowledge-objects/knowledge-management-manual/10.6/transactions/about-transactions

* * *

# 7.39 Important `transaction` Options

The most important traditional SPL options are:

```text
maxspan
    maximum total transaction duration

maxpause
    maximum gap between events

maxevents
    maximum number of events

startswith
    condition that starts a transaction

endswith
    condition that ends a transaction
```

Example:

```spl
index=web
| transaction clientip maxpause=5m maxspan=30m
```

Or:

```spl
index=auth
| transaction user startswith="login" endswith="logout"
```

The `startswith` and `endswith` expressions can use search expressions or Boolean `eval` expressions, according to Splunk's transaction documentation. citehttps://help.splunk.com/en/splunk-enterprise/manage-knowledge-objects/knowledge-management-manual/10.2/transactions/search-for-transactions

* * *

# 7.40 `transaction` vs `streamstats`

These commands answer different sequence questions.

### `streamstats`

Best when you want:

```text
running state
previous value
rolling count
rolling sum
time window
reset state
per-entity state
```

Example:

```spl
| streamstats current=f last(src_ip) AS previous_src_ip
```

### `transaction`

Best when you want:

```text
groups of related events
time-gap-based grouping
start/end transaction boundaries
combined transaction fields
```

Example:

```spl
| transaction user maxpause=5m maxspan=30m
```

Think:

```text
Need a calculation on EVERY event?
    ↓
streamstats

Need to BUILD A GROUP of related events?
    ↓
transaction
```

* * *

# 7.41 A Real SOC Example — Brute Force Followed by Success

Let's build the full logic.

Question:

> Did a user have five or more failures followed by a successful login within one sequence?

Start with the relevant events:

```spl
index=windows
(action="failure" OR action="success")
```

Order them:

```spl
| sort 0 _time
```

Maintain failure state per user:

```spl
| streamstats count(eval(action="failure")) AS failures
    reset_after="(action=\"success\")"
    BY user
```

Keep successful events where the failure count was high:

```spl
| where action="success" AND failures>=5
```

Full query:

```spl
index=windows
(action="failure" OR action="success")
| sort 0 _time
| streamstats count(eval(action="failure")) AS failures
    reset_after="(action=\"success\")"
    BY user
| where action="success" AND failures>=5
| table _time user failures action
```

The mental model is:

```text
user timeline
     ↓
failure
failure
failure
failure
failure
success
     ↓
failures=5 on success
     ↓
DETECTION
     ↓
reset
```

This is the kind of question that simple aggregation cannot answer reliably because the ordering and reset boundary are part of the logic.

* * *

# 7.42 Another SOC Example — Previous Process

Question:

> What process launched immediately before PowerShell?

```spl
index=windows
| sort 0 _time
| streamstats current=f last(process_name) AS previous_process
| where process_name="powershell.exe"
| table _time host user previous_process process_name
```

Now you can investigate:

```text
previous_process = winword.exe
process_name     = powershell.exe
```

versus:

```text
previous_process = explorer.exe
process_name     = powershell.exe
```

The sequence provides context that a simple:

```spl
process_name="powershell.exe"
```

search does not.

* * *

# 7.43 SOC Example — Beacon Timing

Suppose you want to see how regular outbound connections are.

Use:

```spl
index=network
| sort 0 _time
| streamstats current=f last(_time) AS previous_time BY src_ip dest_ip
| eval interval=_time-previous_time
| where isnotnull(interval)
| stats avg(interval) AS avg_interval
        stdev(interval) AS stdev_interval
        count AS connections
        BY dest_ip
```

The sequence is:

```text
event 1 → previous_time=NULL
event 2 → interval=event2_time-event1_time
event 3 → interval=event3_time-event2_time
...
```

Very regular intervals may deserve investigation.

For example:

```text
60
61
60
60
61
```

looks much more regular than:

```text
4
127
32
311
19
```

The query does not prove that the activity is malicious. It creates a behavioral signal [a measurable property useful for investigation].

* * *

# 7.44 SOC Example — Rising Data Transfers

Question:

> Did outbound transfer size increase across several consecutive events?

Start with:

```spl
index=network
| sort 0 _time
| delta bytes_out AS change
```

Create a rise indicator:

```spl
| eval rising=if(change>0,1,0)
```

Then inspect the sequence:

```spl
| table _time bytes_out change rising
```

For:

```text
100
200
300
400
250
```

you see:

```text
bytes_out    change    rising
100          NULL      0
200           100      1
300           100      1
400           100      1
250          -150      0
```

Now the three consecutive positive changes are visible.

This is an example of combining:

```text
delta
    ↓
change

eval
    ↓
turn change into a flag

streamstats
    ↓
maintain sequence state
```

* * *

# 7.45 Sequence Logic With `match()`

`match()` is an evaluation function [a function that evaluates a condition] that returns a Boolean result.

Basic form:

```spl
match(field,"regex")
```

For example:

```spl
| where match(_raw,"oracle")
```

means:

> Keep the event when `_raw` matches the regex `oracle`.

Because `match()` is a condition, it can also be used directly in `streamstats` reset logic:

```spl
| streamstats reset_before="(match(_raw,\"oracle.*\"))"
```

Here:

```text
match()
    ↓
TRUE / FALSE
    ↓
reset_before
```

You are not storing the Boolean in a field. You are using it as the Boolean condition expected by `reset_before`.

This explains why:

```spl
| where match(_raw,"oracle")
```

works, and:

```spl
| eval result=match(_raw,"oracle")
```

may fail in classic SPL because the Boolean result is being assigned directly to a field.

When you need to convert the Boolean to another value:

```spl
| eval result=if(match(_raw,"oracle"),1,0)
```

The three patterns are therefore:

```spl
| where match(_raw,"oracle")
```

```spl
| streamstats reset_before="(match(_raw,\"oracle\"))"
```

```spl
| eval result=if(match(_raw,"oracle"),1,0)
```

The first two use the Boolean as a condition.

The third converts the condition into a numeric value.

* * *

# 7.46 Building a Resettable Running Counter

Suppose your logs contain:

```text
type=login
type=login
type=login
type=reset
type=login
type=login
```

You want the count to restart when `type=reset`.

Use:

```spl
| streamstats count AS events_seen
    reset_before="(type=\"reset\")"
```

Or, if the reset marker should finish the old state:

```spl
| streamstats count AS events_seen
    reset_after="(type=\"reset\")"
```

The choice depends on where the marker belongs:

```text
reset_before
    → marker starts the next sequence

reset_after
    → marker closes the current sequence
```

This is one of the cleanest ways to understand resets.

* * *

# 7.47 Building a Sliding Detection

Suppose you want:

> Alert when an IP generates 10 or more events in the previous 60 seconds.

Use:

```spl
index=network
| sort 0 _time
| streamstats count AS events_last_60s
    time_window=60s
    BY src_ip
| where events_last_60s>=10
```

Now every event carries the current count of recent activity.

This gives you the exact event where the threshold is crossed rather than only a fixed bucket.

The workflow is:

```text
events
   ↓
sort chronologically
   ↓
per-IP time window
   ↓
count recent events
   ↓
threshold
```

* * *

# 7.48 Why `streamstats` Is Called "Streaming"

The word "streaming" describes how Splunk processes the results.

Think of an event stream:

```text
EVENT 1 → process
EVENT 2 → process
EVENT 3 → process
EVENT 4 → process
```

At each event, `streamstats` updates the calculation.

For example:

```text
event 1 → count=1
event 2 → count=2
event 3 → count=3
event 4 → count=4
```

It does not wait until every event is summarized into one row.

That is why `streamstats` can preserve event-level context while adding statistical state.

* * *

# 7.49 Sequence Analysis After `stats`

A very important pattern is:

```text
raw events
    ↓
stats/timechart
    ↓
ordered result rows
    ↓
streamstats
```

For example:

```spl
index=windows EventCode=4625
| timechart span=10m count AS failures
| streamstats window=6 current=f avg(failures) AS baseline
```

Now the sequence is not individual authentication events.

It is:

```text
10-minute bucket 1
10-minute bucket 2
10-minute bucket 3
...
```

This is why `streamstats` is useful in both:

```text
raw-event analysis
```

and:

```text
time-bucket analysis
```

The unit of the sequence is whatever rows the command receives.

* * *

# 7.50 The Unit of the Window

This is one of the most important concepts in the entire chapter.

Ask:

> What exactly is one "event" for my current `streamstats` command?

Sometimes it is a raw log event:

```text
firewall event
firewall event
firewall event
```

Sometimes it is a time bucket:

```text
10:00 bucket
10:10 bucket
10:20 bucket
```

For example:

```spl
| streamstats window=5 count
```

after raw events means:

> five raw results.

But:

```spl
| timechart span=10m count
| streamstats window=5 count
```

means:

> five timechart results, approximately five time buckets.

Therefore the same `window=5` can represent completely different things depending on what came before it.

This is a powerful mental model:

```text
streamstats window
    ↓
counts ROWS
```

not automatically minutes, hours, users, or sessions.

* * *

# 7.51 Common Mistake — Using `streamstats` Without Thinking About Order

This is the most common sequence-analysis problem.

Bad:

```spl
index=*
| streamstats current=f last(src_ip) AS previous_src_ip
```

Better for a chronological lab analysis:

```spl
index=*
| sort 0 _time
| streamstats current=f last(src_ip) AS previous_src_ip
```

Now:

```text
previous
```

has a clear meaning.

Remember:

```text
current=f
```

does not mean "previous event in time."

It means:

> Exclude the current result from the calculation.

The previous result is determined by the stream/search order.

* * *

# 7.52 Common Mistake — Assuming `BY` Means "Lifetime Per-Entity Sequence"

This:

```spl
| streamstats count BY user
```

creates per-user running state.

But if you use:

```spl
| streamstats count BY user reset_on_change=true
```

the meaning changes.

Now a change in the `BY` value resets the state.

For example:

```text
teja
teja
john
john
teja
```

produces:

```text
teja  1
teja  2
john  1
john  2
teja  1
```

So:

```text
BY user
    ≠
reset_on_change=true
```

They control different parts of the behavior.

* * *

# 7.53 Common Mistake — Confusing `window` With `time_window`

These are not synonyms.

```spl
window=5
```

means:

```text
five events
```

while:

```spl
time_window=5m
```

means:

```text
five minutes
```

For example:

```text
100 events in 1 minute
```

could fit into:

```spl
window=100
```

but not necessarily into:

```spl
time_window=1m
```

Likewise:

```text
5 events spread over 30 minutes
```

can fit into:

```spl
window=5
```

but not into:

```spl
time_window=5m
```

Choose the window based on the question.

* * *

# 7.54 Common Mistake — Including the Current Event in Its Own Baseline

Suppose you want to decide whether:

```text
current=100
```

is unusual compared with the previous values.

Do not let the current `100` influence the baseline if your definition is "previous values only."

Use:

```spl
| streamstats window=6 current=f avg(value) AS baseline
```

The mental model is:

```text
CURRENT
   ↓
should I compare it against itself?
   ↓
NO
   ↓
current=f
```

This prevents the value being evaluated from partially hiding its own anomaly.

* * *

# 7.55 Common Mistake — Using `transaction` for Everything

This is common when people first learn correlation.

For example:

```spl
| transaction user
```

may be tempting whenever multiple events belong to the same user.

But if your question is:

> Count failures by user.

Use:

```spl
| stats count BY user
```

If your question is:

> Maintain a running failure state by user.

Use:

```spl
| streamstats ...
    BY user
```

If your question is:

> Group related events according to time gaps and start/end boundaries.

Then:

```spl
| transaction ...
```

may be appropriate.

Use the simplest command that matches the question.

* * *

# 7.56 Common Mistake — Treating `delta` as a Time Function

`delta` is not automatically a timestamp command.

It calculates:

```text
current numeric value
-
previous numeric value
```

If you apply it to `_time`:

```spl
| sort 0 _time
| delta _time AS time_change
```

then the result is the numerical difference between timestamps.

That can be useful for inter-event gaps, but the important behavior is still:

```text
numeric current
-
numeric previous
```

If your field is not numeric, `delta` does not generate the desired output field.

The official documentation defines `delta` as operating on a numeric field and comparing results in search order. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.0/search-commands/delta

* * *

# 7.57 Measuring Inter-Event Time

This is a useful sequence pattern.

```spl
index=network
| sort 0 _time
| delta _time AS gap_seconds
| table _time src_ip dest_ip gap_seconds
```

Conceptually:

```text
event 1 → NULL
event 2 → 10
event 3 → 62
event 4 → 11
```

This tells you how much time separated neighboring results.

You can then investigate regular timing:

```spl
| where gap_seconds>=55 AND gap_seconds<=65
```

or:

```spl
| stats avg(gap_seconds) AS avg_gap
        stdev(gap_seconds) AS stdev_gap
        BY dest_ip
```

Regular repeated intervals can be an investigation lead, especially for automated network behavior.

* * *

# 7.58 `streamstats` Memory and Limits

`streamstats` maintains information in memory to calculate its windows.

Classic SPL has limits that affect how many results can be retained for these calculations, including:

```text
max_stream_window
maxresultrows
max_mem_usage_mb
```

The documented default `max_stream_window` behavior limits the number of events used when no explicit window is supplied; the classic reference documents a default of 10,000 events from `limits.conf`. `window` is also subject to the configured result limits. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

You normally do not change these settings while learning.

The important lesson is:

> A very large or unbounded `streamstats` calculation can have memory implications.

For large production searches, limit the event set, choose a sensible window, and test the search.

* * *

# 7.59 SPL vs SPL2 — Important Syntax Difference

This course studies both classic SPL and SPL2, but this chapter is primarily written for the classic SPL syntax used in Splunk Search.

Classic SPL:

```spl
| streamstats current=f window=5 avg(bytes) AS avg_bytes
```

SPL2 uses the same command concept but can express options with the SPL2 syntax documented for that command.

For example, SPL2 uses:

```spl2
| streamstats current=false window=5 avg(bytes) AS avg_bytes
```

For reset expressions, classic SPL has the nested quoted syntax:

```spl
reset_after="(action=\"success\")"
```

SPL2 uses the newer syntax:

```spl2
reset after action="success"
```

Do not mix the two syntaxes blindly.

The current Splunk documentation maintains separate references for classic SPL `streamstats` and SPL2 `streamstats`. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats citehttps://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

* * *

# 7.60 The CQL Sequence Concepts and Their SPL Equivalents

The previous sequence chapter was organized around concepts such as previous-event lookup, accumulation, partitioning, and sliding windows.

The SPL equivalents are not one-to-one commands.

Use this conceptual mapping:

```text
Previous event
    ↓
streamstats current=f last(field)
autoregress

Running total
    ↓
streamstats sum(field)
accum

Difference from previous event
    ↓
delta
streamstats current=f last(field)

Fixed N-event window
    ↓
streamstats window=N

Moving time window
    ↓
streamstats time_window=X

Partition/reset logic
    ↓
streamstats reset_before/reset_after
streamstats reset_on_change
sequence/state IDs
transaction

Per-entity sequence
    ↓
BY user / BY host / BY src_ip
```

The important lesson is:

> Learn the **analytical idea**, then choose the SPL command that implements it.

Do not expect another query language's command name to exist unchanged in Splunk.

* * *

# 7.61 The Three Most Important Sequence Patterns

After completing this chapter, you should recognize three major patterns immediately.

## Pattern 1 — Previous event

Question:

> What happened immediately before this event?

```spl
| sort 0 _time
| streamstats current=f last(field) AS previous_field
```

or:

```spl
| autoregress field AS previous_field
```

## Pattern 2 — Running state

Question:

> What has happened so far?

```spl
| sort 0 _time
| streamstats count AS events_so_far
```

or:

```spl
| streamstats sum(bytes) AS bytes_so_far
```

## Pattern 3 — Moving window

Question:

> What happened in the latest N events or X minutes?

```spl
| streamstats window=5 count AS recent_events
```

or:

```spl
| streamstats time_window=5m count AS recent_events
```

Once these three patterns become familiar, most of the remaining options are ways to control:

```text
who
how far
when to reset
whether to include current
```

* * *

# 7.62 A Complete Sequence Analysis Workflow

A practical SOC workflow is:

```text
                INVESTIGATION QUESTION
                         |
                         v
                   FILTER EVENTS
                         |
                         v
                 DEFINE EVENT ORDER
                         |
                         v
                    PER ENTITY?
                    /          \
                  YES           NO
                   |             |
                 BY field       global stream
                   |             |
                   +------┬------+
                          |
                          v
                  WHAT SEQUENCE?
                          |
        +-----------------+------------------+
        |                 |                  |
        v                 v                  v
    PREVIOUS          RUNNING            WINDOW
        |                 |                  |
        v                 v                  v
 streamstats        streamstats       window/time_window
 autoregress        accum
 delta
        |                 |                  |
        +-----------------+------------------+
                          |
                          v
                       RESET?
                    /           \
                  YES            NO
                   |              |
               reset_*            continue
                   |
                   v
                  TEST
                   |
                   v
              INVESTIGATE
```

This is the mental workflow for sequence searches.

* * *

# 7.63 A Full Example — Brute Force Detection With a Time Window

Suppose the requirement is:

> Detect when a user has at least 10 failed logins in the previous five minutes.

Use:

```spl
index=windows action="failure"
| sort 0 _time
| streamstats count AS failures_last_5m
    time_window=5m
    BY user
| where failures_last_5m>=10
| table _time user failures_last_5m
```

The sequence is:

```text
raw events
    ↓
only failures
    ↓
chronological order
    ↓
five-minute moving window
    ↓
separate state per user
    ↓
threshold
```

This is a direct example of turning the natural-language requirement into SPL.

* * *

# 7.64 A Full Example — Failure Sequence Ending in Success

Requirement:

> Detect five or more failed logins followed by a successful login for the same user.

```spl
index=windows
(action="failure" OR action="success")
| sort 0 _time
| streamstats count(eval(action="failure")) AS failures
    reset_after="(action=\"success\")"
    BY user
| where action="success" AND failures>=5
| table _time user failures action
```

Notice the differences:

```text
time window detection
    → time_window

sequence ending at an event
    → reset_after

per-account sequence
    → BY user
```

This is why the `streamstats` options should be learned as a system rather than as isolated parameters.

* * *

# 7.65 A Full Example — Previous Destination

Requirement:

> Find events where a connection to one destination immediately followed a connection to another destination.

```spl
index=network action="connection"
| sort 0 _time
| streamstats current=f last(dest_ip) AS previous_dest_ip BY src_ip
| where dest_ip="10.10.10.20" AND previous_dest_ip="10.10.10.10"
| table _time src_ip previous_dest_ip dest_ip
```

Read it as:

```text
For each source IP:
    find the previous destination
    compare it with the current destination
    keep the desired transition
```

This is sequence analysis through **transition detection [detecting a change from one state to another]**.

* * *

# 7.66 A Full Example — Cumulative Data Transfer

Requirement:

> Find the point where an endpoint's cumulative outbound transfer exceeds 500 MB.

```spl
index=network direction="outbound"
| sort 0 _time
| streamstats sum(bytes) AS total_bytes BY host
| where total_bytes>500000000
| table _time host bytes total_bytes
```

The important value is not just:

```text
bytes
```

but:

```text
total_bytes
```

which represents state accumulated across the sequence.

* * *

# 7.67 A Full Example — Three Consecutive Increases

Requirement:

> Find a host where outbound transfer size increased three times in a row.

A useful investigation pattern is:

```spl
index=network
| sort 0 _time
| delta bytes_out AS change
| eval rising=if(change>0,1,0)
| streamstats count AS consecutive_rises
    reset_after="(rising=0)"
    BY host
| where consecutive_rises>=3
| table _time host bytes_out change rising consecutive_rises
```

The workflow is:

```text
delta
    ↓
current vs previous

eval
    ↓
rise or not?

streamstats
    ↓
count the current run

reset_after
    ↓
break the run when the increase stops

where
    ↓
threshold
```

Always test a small synthetic dataset first, because sequence boundaries are easy to get wrong.

* * *

# 7.68 Testing Sequence Searches With `makeresults`

When learning sequence commands, synthetic [artificially created for testing] events are extremely useful.

For example:

```spl
| makeresults count=6
| streamstats count AS n
| eval action=case(
    n=1,"failure",
    n=2,"failure",
    n=3,"failure",
    n=4,"success",
    n=5,"failure",
    n=6,"failure"
)
| table n action
```

You can then experiment with:

```spl
| sort 0 n
| streamstats count(eval(action="failure")) AS failures
    reset_after="(action=\"success\")"
| table n action failures
```

This lets you understand exactly what each event receives.

For learning `streamstats`, this is often better than starting with thousands of real logs.

* * *

# 7.69 A Practice Table You Should Be Able to Predict

Consider:

```text
n   action
1   failure
2   failure
3   failure
4   success
5   failure
6   failure
7   success
```

Try:

```spl
| streamstats count(eval(action="failure")) AS failures
    reset_after="(action=\"success\")"
```

Before running it, predict the `failures` field.

The important reasoning is:

```text
n=1 → failure count increases
n=2 → failure count increases
n=3 → failure count increases
n=4 → success sees the accumulated failures
       then reset
n=5 → new sequence
n=6 → new sequence
n=7 → success sees the new accumulated failures
       then reset
```

This kind of manual prediction is one of the fastest ways to actually understand `streamstats`.

* * *

# 7.70 Choosing the Right Command

When you see a sequence question, ask:

### "What was immediately before this?"

Use:

```text
streamstats current=f last()
autoregress
```

### "How much has accumulated so far?"

Use:

```text
streamstats sum()
accum
```

### "How much did the value change?"

Use:

```text
delta
```

### "How many events are in the last N events?"

Use:

```text
streamstats window=N
```

### "How many events are in the previous X minutes?"

Use:

```text
streamstats time_window=X
```

### "When should the calculation start again?"

Use:

```text
reset_before
reset_after
reset_on_change
```

### "Does the sequence belong to one entity?"

Use:

```text
BY user
BY host
BY src_ip
```

### "Do I need to group related events into transactions?"

Use:

```text
transaction
```

This decision tree is more valuable than memorizing isolated command syntax.

* * *

# 7.71 Classic SPL `streamstats` Syntax Reference

The main classic SPL form is:

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
    <stats-agg-term>...
    [BY <field-list>]
```

The important defaults are:

```text
current=true
global=true
reset_before=false
reset_after=false
reset_on_change=false
allnum=false
window=0
time_window=None
```

Classic SPL documentation also notes that when no explicit `window` or `time_window` is supplied, the effective stream window is bounded by the configured `max_stream_window` setting. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.72 Option-by-Option Summary

```text
current=true
    Include the current event.

current=false
    Exclude the current event.

window=5
    Use at most five events.

time_window=5m
    Use a five-minute time-based window.

BY user
    Maintain the calculation by user.

global=false
    With a window, maintain separate windows for BY groups.

reset_before=...
    Reset before the matching event is calculated.

reset_after=...
    Reset after the matching event is calculated.

reset_on_change=true
    Reset when the BY field values change.

allnum=true
    Require numeric values for numeric calculations.
```

Use the official syntax rather than older examples copied from unrelated query languages. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.73 `streamstats` vs `stats`

You learned `stats` in Chapter 5.

Keep this comparison permanently:

```text
stats
    ↓
aggregate the result set
    ↓
fewer rows

streamstats
    ↓
calculate while processing
    ↓
original rows remain
```

Example:

```spl
| stats count BY user
```

answers:

> How many events did each user produce?

But:

```spl
| streamstats count BY user
```

answers:

> At this point in each user's event stream, how many events have been seen?

And:

```spl
| streamstats window=5 count BY user
```

answers:

> For each event, how many events are represented by the latest five-event window for that user?

The question changes because the processing model changes.

* * *

# 7.74 `streamstats` vs `eventstats`

This distinction is also important.

`eventstats` calculates aggregate statistics and adds them back to the events.

`streamstats` calculates them progressively as the event stream is processed.

Think:

```text
eventstats
    → final group-level statistic
    → attached to events

streamstats
    → running/windowed statistic
    → changes from event to event
```

For example:

```spl
| eventstats avg(bytes) AS avg_bytes BY host
```

may attach one average per host to the events.

Whereas:

```spl
| streamstats avg(bytes) AS running_avg BY host
```

changes as new events arrive.

This distinction follows the broader command behavior documented by Splunk. citehttps://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats

* * *

# 7.75 Performance Thinking

Sequence searches can become expensive because they retain state across results.

Good habits include:

```text
Filter first
    ↓
Keep only events that matter
    ↓
Sort only the reduced set when necessary
    ↓
Use the smallest useful window
    ↓
Use BY only for entities that matter
    ↓
Return only the fields you need
```

For example, prefer:

```spl
index=windows EventCode=4625 user="teja"
| sort 0 _time
| streamstats time_window=5m count AS failures
```

over a broad:

```spl
index=*
| sort 0 _time
| streamstats time_window=5m count
```

when the investigation is specifically about one authentication pattern.

Sequence commands are powerful, but they should be applied deliberately.

* * *

# 7.76 Common Syntax Problem — Boolean Expressions in Reset Conditions

A very common mistake is:

```spl
| streamstats reset_before=match(_raw,"oracle")
```

Classic SPL expects the reset expression in its documented quoted syntax.

Use:

```spl
| streamstats reset_before="(match(_raw,\"oracle\"))"
```

Likewise:

```spl
| streamstats reset_after="(action=\"success\")"
```

The rule is:

```text
normal match()
    → match(_raw,"oracle")

inside classic streamstats reset expression
    → reset_before="(match(_raw,\"oracle\"))"
```

This is a parser [the component that interprets query syntax] issue, not a different meaning for `match()`.

* * *

# 7.77 Common Mistake — Believing `match()` Stores TRUE/FALSE in a Field

In classic SPL:

```spl
| eval result=match(_raw,"oracle")
```

can produce the error:

```text
Fields cannot be assigned a boolean result.
```

But this is valid:

```spl
| where match(_raw,"oracle")
```

because `where` expects a Boolean condition.

And this is valid:

```spl
| eval result=if(match(_raw,"oracle"),1,0)
```

because `if()` converts the Boolean condition into a value.

The sequence principle is:

```text
match()
    → condition

if()
    → convert condition into a value

where
    → use condition to filter

reset_before/reset_after
    → use condition to control state
```

* * *

# 7.78 What This Chapter Added

You already knew from Chapter 6 how to make time buckets and basic rolling calculations.

Chapter 7 adds **event relationships**.

You now have:

```text
Previous event
    → streamstats current=f last()
    → autoregress

Difference
    → delta

Running state
    → streamstats
    → accum

Event-count window
    → streamstats window=N

Time window
    → streamstats time_window=X

Reset state
    → reset_before
    → reset_after
    → reset_on_change

Per-entity state
    → BY

Complex grouping
    → transaction
```

These tools let you move from:

```text
How many?
```

to:

```text
What happened before this?
What happened after that?
How many happened before the reset?
How much accumulated before the threshold?
How many happened in the previous five minutes?
Did the same user perform these actions in this order?
```

That is the core transition from aggregation to sequence analysis.

* * *

# 7.79 The Complete Mental Model

```text
                   INVESTIGATION QUESTION
                            |
                            v
                   IS ORDER IMPORTANT?
                      /            \
                    NO              YES
                    |                |
                 stats          DEFINE ORDER
                                  |
                                  v
                           PER ENTITY?
                          /          \
                        YES           NO
                         |             |
                       BY field      one stream
                         |             |
                         +------┬------+
                                |
                                v
                         WHAT DO I NEED?
                                |
       +------------------------+------------------------+
       |                        |                        |
       v                        v                        v
   PREVIOUS                 RUNNING                   WINDOW
       |                        |                        |
       v                        v                        v
streamstats               streamstats              window=N
current=f                  sum/count/etc            time_window=X
autoregress                accum
delta
       |                        |                        |
       +------------------------+------------------------+
                                |
                                v
                           NEED RESET?
                         /              \
                       YES               NO
                        |                 |
                    reset_*              continue
                        |
                        v
                     FILTER
                        |
                        v
                  SECURITY RESULT
```

The key question for Chapter 7 is:

> **What relationship between events am I trying to prove?**

* * *

# 7.80 Chapter Summary

Sequence analysis is different from ordinary aggregation because **event order becomes part of the evidence**.

The core SPL tool is:

```spl
streamstats
```

For previous-event analysis:

```spl
| streamstats current=f last(field) AS previous_field
```

or:

```spl
| autoregress field AS previous_field
```

For differences:

```spl
| delta field AS change
```

For running totals:

```spl
| streamstats sum(field) AS running_total
```

or:

```spl
| accum field AS running_total
```

For the last N events:

```spl
| streamstats window=5 ...
```

For the previous X amount of time:

```spl
| streamstats time_window=5m ...
```

For per-entity calculations:

```spl
| streamstats ... BY user
```

For resets:

```spl
reset_before
reset_after
reset_on_change
```

For transaction-like event grouping:

```spl
| transaction ...
```

The most important distinctions are:

```text
stats
    → final aggregate

streamstats
    → running/windowed aggregate on each result

window
    → number of events

time_window
    → amount of time

current
    → include or exclude the current event

BY
    → separate entity/group

global
    → shared or separate finite window

reset_before
    → matching event starts the new state

reset_after
    → matching event ends the old state

reset_on_change
    → reset when BY values change
```

* * *

# 🧠 What You Should Remember

If you remember only these points:

```text
Order matters?
    ↓
Think sequence commands.

Need the previous value?
    ↓
streamstats current=f last()
or autoregress.

Need the difference?
    ↓
delta.

Need a running total/count?
    ↓
streamstats.

Need the last N events?
    ↓
window=N.

Need the previous X minutes?
    ↓
time_window=X.

Need separate state per user/IP/host?
    ↓
BY.

Need to restart the state?
    ↓
reset_before / reset_after / reset_on_change.

Need related events grouped into transactions?
    ↓
transaction.
```

And the most important habit:

```text
FILTER
   ↓
ORDER
   ↓
PER-ENTITY GROUPING
   ↓
SEQUENCE STATE
   ↓
RESET / WINDOW
   ↓
DETECTION CONDITION
```

Once you can translate a SOC requirement into those steps, sequence-based SPL becomes much easier to construct.

* * *

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
Sequence & Correlation Analysis
        ↓
CHAPTER 8
Arrays & Nested Data
```

Chapter 7 therefore builds directly on the concepts already learned rather than repeating Chapters 1–6.

* * *

# Official Splunk Documentation Used for This Chapter

The chapter follows the current Splunk documentation for:

- `streamstats`
- `autoregress`
- `delta`
- `accum`
- `transaction`
- SPL2 `streamstats`
- `stats` and time-series concepts inherited from Chapter 6

Official references:

1. https://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.0/search-commands/streamstats
2. https://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.6/search-commands/autoregress
3. https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.0/search-commands/delta
4. https://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.6/search-commands/accum
5. https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.4/search-commands/transaction
6. https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/streamstats-command/streamstats-command-overview-syntax-and-usage

For this course, current official Splunk documentation takes precedence over older community examples when syntax or behavior differs.
