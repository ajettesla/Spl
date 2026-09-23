# Splunk SPL + SPL2

# Chapter 2 — Writing Your First Splunk SPL and SPL2 Query

Now that we understand what an event, field, index, `_raw`, `_time`, `host`,
`source`, and `sourcetype` are, we can start writing searches.

The goal of this chapter is **not to memorize syntax**.

Instead, we want to understand how to think when building a Splunk search.

This chapter covers **both traditional SPL and SPL2**.

That is important because the two languages share many ideas, but their syntax
and capabilities are not identical.

A useful way to think about them is:

```text
Traditional SPL
    ↓
Search-processing commands connected with |

SPL2
    ↓
Supports SPL-style searches
AND
SQL-style FROM / WHERE / SELECT syntax
```

Splunk's current SPL2 documentation describes SPL2 as supporting both SPL and
SQL-style syntax. The `search` and `from` commands are the two common ways to
start a search. [Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/introduction/understanding-spl2-syntax)

---

# 2.1 The Basic Idea

A Splunk search is essentially a set of instructions that tells Splunk:

**"Start with the data I am interested in, narrow it down, and then do
something with the results."**

For example, suppose we want failed Windows logons.

### Traditional SPL

```spl
index=windows EventCode=4625
```

### SPL2 — SPL-style syntax

```spl2
search index=windows EventCode=4625
```

### SPL2 — `FROM` syntax

```spl2
FROM windows
WHERE EventCode=4625
```

All three express the same basic investigation idea:

```text
Start with Windows data
        ↓
Find EventCode 4625
        ↓
Return matching events
```

The important difference is that SPL2 gives you an additional structured
`FROM ... WHERE ...` form.

Splunk documents `search` as a generating command and `from` as another
generating command. An index is a kind of dataset [dataset: a collection of
data that can be searched]. `search` uses `index=<name>`, while `from` can use
the index name directly. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/getting-started/quick-start-write-and-run-a-basic-spl2-search/start-searching-data-using-spl2))

---

# 2.2 Your First Traditional SPL Search

Let's start with traditional SPL.

```spl
index=windows
```

This means:

> Search the `windows` index.

You can make it more specific:

```spl
index=windows EventCode=4625
```

This means:

> Search the `windows` index for events where `EventCode` is `4625`.

You can add another field:

```spl
index=windows EventCode=4625 LogonType=3
```

Now the idea becomes:

```text
index=windows
        ↓
Find Windows events

EventCode=4625
        ↓
Find failed logon events

LogonType=3
        ↓
Keep events for that logon type
```

One important SPL detail is that, at the beginning of a traditional SPL search,
the `search` command is normally implied.

Therefore these are equivalent:

```spl
index=windows EventCode=4625
```

and:

```spl
search index=windows EventCode=4625
```

Splunk's documentation explicitly describes this behavior for the beginning of
a search pipeline. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/search-manual/10.2/search-primer/search-command-primer))

---

# 2.3 Your First SPL2 Search

Now let's write the same investigation in SPL2.

## SPL2 using `search`

```spl2
search index=windows EventCode=4625
```

This is the most familiar form if you already know traditional SPL.

The major syntax difference is:

```text
SPL:
index=windows

SPL2:
search index=windows
```

The SPL2 `search` command must be explicitly specified when used as the
generating command. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/search-command/search-command-overview-and-syntax))

---

## SPL2 using `FROM`

SPL2 also provides a SQL-style way to start the search:

```spl2
FROM windows
```

Then filter it:

```spl2
FROM windows
WHERE EventCode=4625
```

This can be read as:

```text
FROM windows
     ↓
Use the windows dataset

WHERE EventCode=4625
     ↓
Keep events matching the condition
```

The `from` command can also use clauses such as `WHERE`, `GROUP BY`,
`SELECT`, `ORDER BY`, and `LIMIT`. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/from-command/from-command-syntax))

---

# 2.4 Three Ways to Think About the Same Search

For the rest of this chapter, keep this mental model:

### Traditional SPL

```spl
index=windows EventCode=4625
```

### SPL2 using SPL syntax

```spl2
search index=windows EventCode=4625
```

### SPL2 using SQL-style syntax

```spl2
FROM windows
WHERE EventCode=4625
```

Conceptually:

```text
             SAME INVESTIGATION
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
       SPL       SPL2 search   SPL2 FROM
        │           │           │
 index=...     search index=... FROM ...
```

Do not treat these as three unrelated searches.

They are three ways of expressing the same basic search idea.

---

# 2.5 Why the Pipeline `|` Matters

You will see this symbol everywhere in traditional SPL:

```text
|
```

It is called the **pipe** or **pipeline operator**.

It takes the results produced by one command and passes them to the next
command.

For example:

```spl
index=windows EventCode=4625
| stats count() by user
```

Read it from left to right:

```text
Search Windows failed-logon events
             ↓
Create statistics
             ↓
Count events by user
```

The same pipeline concept exists in SPL2.

For example:

```spl2
search index=windows EventCode=4625
| stats count() by user
```

SPL2 also allows a `FROM` search followed by pipeline commands:

```spl2
FROM windows
WHERE EventCode=4625
| stats count() by user
```

Therefore:

```text
Command 1
   |
   ↓
Command 2
   |
   ↓
Command 3
   |
   ↓
Final results
```

This pipeline model is one of the most important ideas to understand in
Splunk.

---

# 2.6 Building a Search Step by Step

Suppose your investigation question is:

**"Which users have failed Windows logons?"**

Do not immediately try to write the complete search.

Break the question into pieces.

## Step 1 — What data are we interested in?

Windows:

```spl
index=windows
```

or:

```spl2
FROM windows
```

## Step 2 — Which event?

Failed logon:

```spl
index=windows EventCode=4625
```

or:

```spl2
FROM windows
WHERE EventCode=4625
```

## Step 3 — What do we want to do with the results?

Count by user:

```spl
index=windows EventCode=4625
| stats count() by user
```

or:

```spl2
search index=windows EventCode=4625
| stats count() by user
```

The SPL2 `FROM` form can also use aggregation [aggregation: calculating a
summary such as count, sum, or average] directly:

```spl2
FROM windows
WHERE EventCode=4625
GROUP BY user
SELECT count() AS count, user
```

The exact form you use can depend on whether you are writing an SPL-style
pipeline or using the SQL-style clauses of `from`. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/from-command/from-command-syntax))

---

# 2.7 Think in Questions, Not Syntax

This is an important habit.

Do not start an investigation by thinking:

> "What Splunk syntax do I need?"

Instead ask:

**"What question am I trying to answer?"**

For example:

### Question

Which computers have run PowerShell?

Think:

```text
I need process-related data
        ↓
I need PowerShell executions
        ↓
I need to group or display the computer
```

Traditional SPL:

```spl
index=windows process="powershell.exe"
| stats count by host
```

SPL2 using `search`:

```spl2
search index=windows process="powershell.exe"
| stats count by host
```

SPL2 using `FROM`:

```spl2
FROM windows
WHERE process="powershell.exe"
| stats count by host
```

The syntax is secondary.

The investigation logic comes first.

---

# 2.8 Filtering With `=`

The most basic field comparison you will use constantly is:

```text
field=value
```

For example:

```spl
index=windows user=administrator
```

This means:

> Find Windows events where the `user` field has the value
> `administrator`.

Another example:

```spl
index=windows EventCode=4625
```

This means:

> Find Windows events where `EventCode` equals `4625`.

With SPL2:

```spl2
search index=windows user="administrator"
```

Or:

```spl2
FROM windows
WHERE user="administrator"
```

There is an important syntax difference here.

In traditional SPL's `search` expression, string values can often be written
without quotation marks:

```spl
user=administrator
```

In SPL2's `FROM ... WHERE` syntax, string values are enclosed in double
quotation marks:

```spl2
WHERE user="administrator"
```

Splunk documents double quotation marks as the standard way to enclose string
values in SPL2. The SPL2 `search` command retains backward-compatible SPL
search behavior. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/wildcards-quotes-and-escape-characters/quotation-marks))

---

# 2.9 Filtering With the `search` Command

In traditional SPL, you can put filtering conditions at the beginning:

```spl
index=windows EventCode=4625
```

You can also use `search` later in the pipeline:

```spl
index=windows
| search EventCode=4625
```

The idea is:

```text
index=windows
     ↓
Retrieve events
     ↓
search EventCode=4625
     ↓
Filter the results already produced
```

This distinction becomes important later when we learn search optimization.

In SPL2, the same `search` command can be used as the generating command or
as a filtering command later in the pipeline:

```spl2
search index=windows
| search EventCode=4625
```

Splunk documents the SPL2 `search` command as a generating command when first
in the search and as a filtering command when used later in the pipeline.
([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/search-command/search-command-overview-and-syntax))

---

# 2.10 `WHERE` in SPL and SPL2

This is an important place where beginners can become confused.

Traditional SPL also has a `where` command:

```spl
index=windows
| where EventCode=4625
```

SPL2 has the `where` command as well:

```spl2
search index=windows
| where EventCode=4625
```

And SPL2 `FROM` has an equivalent `WHERE` clause:

```spl2
FROM windows
WHERE EventCode=4625
```

So there are three useful forms to recognize:

```text
SPL:
| where condition

SPL2:
| where condition

SPL2 FROM:
WHERE condition
```

The SPL2 `where` command evaluates a predicate [predicate: a condition that
produces TRUE or FALSE] and keeps results that evaluate to TRUE. Splunk also
states that the `where` command is identical in meaning to the `WHERE` clause
of the `from` command. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/where-command/where-command-overview-syntax-and-usage))

---

# 2.11 `search` vs `where`

Do not think of `search` and `where` as simply two names for the same thing.

They overlap, but their expression capabilities are different.

The `search` command uses a simpler search expression model.

For example:

```spl
index=windows EventCode=4625
```

The `where` command uses expressions that can compare fields and calculate
values.

For example:

```spl
index=windows
| where src_ip=dest_ip
```

This compares one field with another.

That distinction also exists in SPL2. Splunk documents that the `where`
command has a more powerful expression language than the `search` command and
can compare two fields. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/search-command/search-command-usage))

A useful beginner mental model is:

```text
search
  ↓
Fast, familiar event filtering language

where
  ↓
Expression-based filtering
```

Do not memorize this as a performance promise for every search. The actual
execution and optimization depend on the search and data.

---

# 2.12 Multiple Filters and AND Logic

You can add multiple conditions.

Traditional SPL:

```spl
index=windows EventCode=4625 LogonType=3
```

The search terms are effectively ANDed.

SPL2 `search`:

```spl2
search index=windows EventCode=4625 LogonType=3
```

SPL2 `WHERE`:

```spl2
FROM windows
WHERE EventCode=4625 AND LogonType=3
```

Notice the difference.

In the SPL/SPL2 `search` command, multiple search terms have an implied AND.

In the SPL2 `WHERE` clause, you explicitly use logical operators such as
`AND` and `OR`.

Splunk documents this distinction for the SPL2 `search` command and the
`where`/`WHERE` expression syntax. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/search-command/search-command-usage))

---

# 2.13 OR Logic

Suppose you want to search for either `4624` or `4625`.

Traditional SPL:

```spl
index=windows (EventCode=4624 OR EventCode=4625)
```

SPL2 `search`:

```spl2
search index=windows (EventCode=4624 OR EventCode=4625)
```

SPL2 `WHERE`:

```spl2
FROM windows
WHERE EventCode=4624 OR EventCode=4625
```

Read this as:

```text
Windows events
      ↓
EventCode 4624
      OR
EventCode 4625
```

Parentheses become important when you combine AND and OR conditions.

For example:

```spl
index=windows user=administrator (EventCode=4624 OR EventCode=4625)
```

This means:

```text
user must be administrator
AND
EventCode must be 4624 OR 4625
```

Splunk documents explicit grouping with parentheses for compound search
expressions. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/search-command/search-command-usage))

---

# 2.14 Using `!=`

Sometimes you want to exclude a value.

Traditional SPL:

```spl
index=windows user!="administrator"
```

SPL2:

```spl2
search index=windows user!="administrator"
```

SPL2 `WHERE`:

```spl2
FROM windows
WHERE user!="administrator"
```

There is an important subtlety.

A condition such as:

```text
user!="administrator"
```

does not mean "return every event where the value is not administrator,
including events where the field does not exist."

For the `search` command, Splunk documents that `field!="value"` returns events
where the field exists and is not equal to that value. To include events where
the field is absent, `NOT field="value"` has different behavior. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/search-command/search-command-examples))

This difference becomes important when building detections.

---

# 2.15 Numeric Comparisons

Splunk can also compare numeric values.

Examples:

```spl
index=windows ProcessId>1000
```

```spl
index=windows ProcessId<1000
```

```spl
index=windows ProcessId>=1000
```

```spl
index=windows ProcessId<=1000
```

The same comparison operators are available in SPL2 search expressions and
SPL2 `where` expressions.

For example:

```spl2
search index=windows ProcessId>1000
```

or:

```spl2
FROM windows
WHERE ProcessId>1000
```

The basic operators are:

| Meaning | Operator |
|---|---|
| Equal | `=` |
| Not equal | `!=` |
| Greater than | `>` |
| Less than | `<` |
| Greater than or equal | `>=` |
| Less than or equal | `<=` |

In SPL2 evaluation expressions, `=` and `==` are equivalent for equality, but
the `search` command retains its field-value search syntax. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/evaluation-functions/overview-of-spl2-eval-functions))

---

# 2.16 Searching for Text

This is an area where you should understand the difference between:

```text
searching the raw event
```

and:

```text
searching a specific field
```

Suppose the raw event contains:

```text
User failed authentication from 10.10.10.50
```

A traditional SPL keyword search can be:

```spl
index=windows "failed authentication"
```

Because you did not specify a field, the search is looking for the text in the
event's raw data.

Splunk's search primer explains that keyword and phrase searches at the
beginning of a search normally match against `_raw`. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/search-manual/10.2/search-primer/search-command-primer))

SPL2 can use:

```spl2
search index=windows "failed authentication"
```

You can also use a field:

```spl
index=windows Message="failed authentication"
```

or:

```spl2
search index=windows Message="failed authentication"
```

The important idea is:

```text
Keyword / phrase
      ↓
Search event text

field=value
      ↓
Search a specific field
```

---

# 2.17 Searching With Wildcards

Wildcards are extremely useful.

The most familiar wildcard in traditional SPL and in the SPL2 `search` command
is:

```text
*
```

It represents multiple characters.

For example:

```spl
index=windows host=DC*
```

can match hosts such as:

```text
DC01
DC02
DC-PRIMARY
```

provided they match the search expression.

Another example:

```spl
index=windows CommandLine="*powershell*"
```

This searches for values containing `powershell`.

SPL2 `search` supports the same `*` wildcard style:

```spl2
search index=windows CommandLine="*powershell*"
```

However, **do not assume that `*` works the same way in every SPL2 command**.

For SPL2 `WHERE`, wildcard pattern matching uses the `LIKE` operator or the
`like()` function, where `%` matches multiple characters and `_` matches one
character.

For example:

```spl2
FROM windows
WHERE CommandLine LIKE "%powershell%"
```

or:

```spl2
FROM windows
WHERE like(CommandLine, "%powershell%")
```

This distinction is documented by Splunk. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/wildcards-quotes-and-escape-characters/wildcards))

Think:

```text
SPL / SPL2 search
    *
    ↓
wildcard characters

SPL2 WHERE / LIKE
    %
    ↓
multiple characters

SPL2 WHERE / LIKE
    _
    ↓
single character
```

---

# 2.18 Regex Searching

Regular expressions [regex: patterns used to match text] are another important
way to search text.

Suppose you want to find command lines that contain some form of
`powershell`.

Traditional SPL can use the `regex` command:

```spl
index=windows
| regex CommandLine="(?i)powershell"
```

Here:

```text
CommandLine
     ↓
Field being examined

(?i)
     ↓
Case-insensitive matching

powershell
     ↓
Pattern
```

SPL2 also supports regex through the `match()` function.

For example:

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell")
```

Or within an `eval` expression:

```spl2
search index=windows
| eval is_ps=if(match(CommandLine, "(?i)powershell"), 1, 0)
```

Splunk documents `match()` as a regular-expression function usable with
`eval`, `where`, and the `WHERE` clause of `from`. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/evaluation-functions/comparison-and-conditional-functions))

We will study regex in a dedicated later chapter, because it becomes important
for both SOC hunting and data processing.

---

# 2.19 A Complete Investigation Example

Let's say your SOC receives this question:

**"Find failed Windows logons from an external source IP against the
Administrator account."**

Break the question into parts.

## Step 1 — Start with the relevant dataset

Traditional SPL:

```spl
index=windows
```

SPL2:

```spl2
FROM windows
```

## Step 2 — Find failed logons

Traditional SPL:

```spl
index=windows EventCode=4625
```

SPL2:

```spl2
FROM windows
WHERE EventCode=4625
```

## Step 3 — Identify the account

Traditional SPL:

```spl
index=windows EventCode=4625 TargetUserName=Administrator
```

SPL2:

```spl2
FROM windows
WHERE EventCode=4625
  AND TargetUserName="Administrator"
```

## Step 4 — Add an IP condition

For example, if your environment has a normalized `src_ip` field:

Traditional SPL:

```spl
index=windows EventCode=4625 TargetUserName=Administrator src_ip="10.10.10.*"
```

SPL2:

```spl2
FROM windows
WHERE EventCode=4625
  AND TargetUserName="Administrator"
  AND src_ip LIKE "10.10.10.%"
```

The exact fields are data-source dependent. Do not assume that every Windows
event has a normalized `src_ip` field.

The important lesson is the method:

```text
Question
   ↓
Identify dataset
   ↓
Identify event type
   ↓
Add field filters
   ↓
Add text/IP conditions
   ↓
Process the results
```

---

# 2.20 Adding Statistics

Now suppose the question changes:

**"How many failed logons occurred for each user?"**

Traditional SPL:

```spl
index=windows EventCode=4625
| stats count by TargetUserName
```

SPL2 using an SPL-style pipeline:

```spl2
search index=windows EventCode=4625
| stats count by TargetUserName
```

SPL2 using `FROM`:

```spl2
FROM windows
WHERE EventCode=4625
GROUP BY TargetUserName
SELECT count() AS count, TargetUserName
```

The result conceptually becomes:

| TargetUserName | count |
|---|---:|
| administrator | 791 |
| john | 24 |
| alice | 7 |

The important thing is what happened:

```text
Many events
     ↓
Filter EventCode=4625
     ↓
Group by user
     ↓
Count the events
     ↓
Small summary table
```

The SPL/SPL2 `stats` command calculates aggregate statistics such as count,
sum, and average. A `BY` field creates one result row for each distinct value.
([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/stats-command/stats-command-overview-syntax-and-usage))

---

# 2.21 Displaying Only the Fields You Need

Often you do not need every field.

Traditional SPL commonly uses:

```spl
index=windows EventCode=4625
| fields _time host TargetUserName src_ip
```

SPL2 can use the `fields` command in the pipeline:

```spl2
search index=windows EventCode=4625
| fields _time host TargetUserName src_ip
```

SPL2 `FROM` can also use `SELECT` to project [project: choose which fields or
expressions appear in the result]:

```spl2
FROM windows
WHERE EventCode=4625
SELECT _time, host, TargetUserName, src_ip
```

This gives us another important SPL2 idea:

```text
Traditional SPL:
| fields ...

SPL2:
| fields ...

SPL2 FROM:
SELECT ...
```

SPL2's `SELECT` clause can retrieve specific fields and can also contain
expressions and aggregate functions in searches. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/from-command/from-command-syntax))

---

# 2.22 A Very Important Mental Model

When writing Splunk searches, think in three stages.

## Stage 1 — What data am I looking at?

Traditional SPL:

```spl
index=windows
```

SPL2:

```spl2
search index=windows
```

or:

```spl2
FROM windows
```

This answers:

> **Which dataset am I searching?**

---

## Stage 2 — What events am I interested in?

Traditional SPL:

```spl
index=windows EventCode=4625
```

SPL2:

```spl2
search index=windows EventCode=4625
```

or:

```spl2
FROM windows
WHERE EventCode=4625
```

This answers:

> **Which events should remain?**

---

## Stage 3 — What do I want to do with them?

For example:

```spl
| stats count by TargetUserName
```

or:

```spl
| fields _time host TargetUserName
```

or:

```spl
| sort -count
```

In SPL2, the `FROM` form can also express these operations using clauses such
as:

```spl2
GROUP BY
SELECT
ORDER BY
LIMIT
```

The result is:

```text
DATASET
   ↓
FILTER
   ↓
TRANSFORM / AGGREGATE
   ↓
DISPLAY / ORDER
   ↓
RESULT
```

This is the basic structure behind much more advanced searches.

---

# 2.23 SPL vs SPL2 — The Core Differences So Far

Here is the comparison you should remember.

| Task | Traditional SPL | SPL2 |
|---|---|---|
| Start with index | `index=windows` | `search index=windows` |
| Start with dataset using `FROM` | Not the same SPL `from` behavior | `FROM windows` |
| Filter in search expression | `EventCode=4625` | `search EventCode=4625` |
| Filter with command | `\| where EventCode=4625` | `\| where EventCode=4625` |
| `FROM` filter | Not applicable in this form | `FROM windows WHERE EventCode=4625` |
| Pipeline | `\|` | `\|` |
| Equality | `=` | `=` in search; expressions also support `==` |
| AND | Often implied in search terms | Implied in `search`; explicit in `WHERE` |
| OR | `OR` | `OR` |
| Wildcard in search | `*` | `*` with `search`; `%` / `_` with `LIKE` in `WHERE` |
| Regex | `regex` / regex functions | `match()` and regex-capable commands/functions |
| Aggregation | `\| stats ...` | `\| stats ...` or `GROUP BY` / `SELECT` with `FROM` |
| Select fields | `\| fields ...` | `\| fields ...` or `SELECT ...` |

This table is a learning guide, not a statement that every SPL and SPL2
command has a one-to-one replacement.

---

# 2.24 Don't Try to Memorize Complete Searches

This is one of the biggest mistakes beginners make.

They memorize:

```spl
index=windows EventCode=4625
| stats count by TargetUserName
```

But then someone asks:

> "Show me successful logons by host during the same period."

Instead, learn the structure:

```text
Dataset
   ↓
Filter
   ↓
Filter
   ↓
Optional transformation
   ↓
Aggregation / display
```

Then you can build a new search:

```spl
index=windows EventCode=4624
| stats count by host
```

Or:

```spl2
FROM windows
WHERE EventCode=4624
GROUP BY host
SELECT count() AS count, host
```

The syntax becomes a tool instead of something you have to memorize.

---

# 2.25 A Note About Search Performance

You will hear a lot about "search optimization" in Splunk.

At this stage, remember one basic rule:

**Be specific about the data you need.**

For example:

```spl
index=windows EventCode=4625
```

is more useful than an unnecessarily broad search across many datasets.

Likewise, in SPL2:

```spl2
FROM windows
WHERE EventCode=4625
```

is more focused than:

```spl2
FROM windows
```

However, do not assume that adding a field condition automatically means
that the field is indexed or that every condition is equally efficient.

That depends on how the field is represented and how Splunk executes the search.

Splunk's documentation explains that default fields such as `index`,
`source`, and `sourcetype` can be used to narrow searches, and that indexed
fields can provide specific search advantages. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/optimizing-searches/search-using-default-fields))

We will study this properly later when we cover:

```text
indexed fields
tsidx
search-time extraction
index-time extraction
```

---

# 2.26 Search-Time Filtering vs Ingest-Time Filtering

This distinction is extremely important for the rest of our course.

When you write:

```spl
index=windows EventCode=4625
```

you are performing a **search** against data that is already in Splunk.

You are not changing the stored event.

Conceptually:

```text
Already indexed data
        ↓
       SPL
        ↓
     Results
```

But later we will learn ingestion-time processing:

```text
Incoming event
      ↓
Filter
      ↓
Mask
      ↓
Route
      ↓
Index / destination
```

This is a completely different operation.

For example:

### Search-time filtering

```spl
index=windows EventCode=4625
```

means:

> Return only matching events to my search results.

### Ingest-time filtering

could mean:

> Do not send certain incoming events to the index at all.

This distinction is central to understanding:

```text
props.conf
transforms.conf
SEDCMD
_TCP_ROUTING
Heavy Forwarders
Ingest Processor
Edge Processor
```

We will study those separately.

---

# 2.27 SPL2 Is Also Used in Processing Pipelines

Another important point is that SPL2 is not limited to searches.

Splunk uses SPL2 in products such as Ingest Processor and Edge Processor.

In a processing pipeline, the general model is:

```text
Incoming data
      ↓
FROM $source
      ↓
Processing
      ↓
INTO $destination
```

For example, Splunk documents pipeline syntax around:

```spl2
$pipeline =
| from $source
| <processing>
| into $destination;
```

In these pipelines, commands can be used to filter or transform incoming data
before it reaches a destination. ([Official Splunk documentation](https://help.splunk.com/en/data-management/process-data-at-ingest-time/use-ingest-processor/working-with-pipelines/ingest-processor-pipeline-syntax))

This is why learning SPL2 has value beyond search.

But remember:

```text
SPL search
    ↓
Find/analyze existing data

SPL2 processing pipeline
    ↓
Process data before it reaches its destination
```

We will keep those two uses separate so that the concepts do not become
confused.

---

# 2.28 Common Beginner Mistakes

## Mistake 1 — Thinking SPL2 replaces SPL completely

It does not.

SPL2 supports SPL-style searching and also provides SQL-style syntax.

You will encounter both.

---

## Mistake 2 — Thinking `FROM` is just another name for `search`

They start searches in different ways.

For example:

```spl
index=windows
```

versus:

```spl2
FROM windows
```

The `FROM` command has its own dataset-oriented syntax and supports clauses
such as `WHERE`, `GROUP BY`, `SELECT`, `ORDER BY`, and `LIMIT`. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/from-command/from-command-overview))

---

## Mistake 3 — Assuming `WHERE` and `search` behave identically

They overlap, but they use different expression models.

For example:

```spl2
search field=value
```

is different from:

```spl2
where field=value
```

The `where` command supports more powerful expressions, including comparisons
between fields. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/where-command/where-command-overview-syntax-and-usage))

---

## Mistake 4 — Using `*` everywhere

A search such as:

```spl
index=windows CommandLine="*powershell*"
```

can be useful, but broad wildcard searches can be expensive.

In SPL2 `WHERE`, remember that wildcard pattern matching uses `LIKE` with `%`
and `_`, not the same `*` syntax used by the `search` command. ([Official Splunk documentation](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/wildcards-quotes-and-escape-characters/wildcards))

---

## Mistake 5 — Confusing search-time filtering with data filtering

This:

```spl
index=windows EventCode=4625
```

does not delete the other events from the index.

It only filters what your search returns.

That is completely different from ingest-time filtering.

---

# 2.29 The Most Important Mindset

Do not think:

> "I need to remember a Splunk query."

Think:

> **"I need to translate an investigation question into search operations."**

For example:

```text
Investigation question
        ↓
What data?
        ↓
What event?
        ↓
What fields?
        ↓
What conditions?
        ↓
What calculation?
        ↓
What output?
```

Then express that logic in either SPL or SPL2.

For example:

```text
Question:
Which users generated failed Windows logons?

        ↓

DATASET
windows

        ↓

EVENT
EventCode=4625

        ↓

GROUP
TargetUserName

        ↓

CALCULATE
count
```

Traditional SPL:

```spl
index=windows EventCode=4625
| stats count by TargetUserName
```

SPL2:

```spl2
FROM windows
WHERE EventCode=4625
GROUP BY TargetUserName
SELECT count() AS count, TargetUserName
```

Same investigation.

Different expression.

---

# 2.30 Chapter Summary

At this point, you should understand how a basic Splunk search is constructed.

The general pattern is:

```text
Identify the dataset
        ↓
Filter events
        ↓
Process the results
        ↓
Display useful information
```

Traditional SPL example:

```spl
index=windows EventCode=4625
| stats count by TargetUserName
```

SPL2 using SPL-style syntax:

```spl2
search index=windows EventCode=4625
| stats count by TargetUserName
```

SPL2 using `FROM`:

```spl2
FROM windows
WHERE EventCode=4625
GROUP BY TargetUserName
SELECT count() AS count, TargetUserName
```

The most important concepts from this chapter are:

### `index`

Identifies the Splunk index you are searching in traditional SPL and in the
SPL2 `search` syntax.

### `search`

The main keyword-based search command. In traditional SPL it is commonly
implicit at the beginning of the search; in SPL2 it can be used explicitly as
a generating command.

### `FROM`

The SPL2 dataset-oriented generating command.

### `|`

Passes results from one command to the next.

### `WHERE`

Filters using expressions in SPL2 and is also available as a command in the
pipeline.

### `=`, `!=`, `>`, `<`, `>=`, `<=`

Basic comparison operators.

### `AND` and `OR`

Logical operators used to combine conditions.

### `*`

Common wildcard in SPL-style searches and the SPL2 `search` command.

### `LIKE`

SPL2 pattern matching using `%` and `_`.

### Regex

Pattern-based text matching, available through commands/functions such as
`regex` and `match()`.

### `stats`

Produces aggregate [aggregate: summarized from many events] results such as
counts, sums, and averages.

### `fields` / `SELECT`

Used to control which fields appear in results.

---

# What We Have Covered So Far

```text
CHAPTER 1
Understanding Splunk Events, Fields, and Data
        │
        ├── What is an event?
        ├── What is a field?
        ├── _raw
        ├── _time
        ├── index
        ├── host
        ├── source
        ├── sourcetype
        ├── indexed fields
        └── search-time fields
                │
                ▼
CHAPTER 2
Writing Your First SPL + SPL2 Query
        │
        ├── SPL fundamentals
        ├── SPL2 search syntax
        ├── SPL2 FROM syntax
        ├── Pipeline |
        ├── search
        ├── where / WHERE
        ├── Equality
        ├── AND / OR
        ├── !=
        ├── Numeric comparisons
        ├── Wildcards
        ├── Regex
        ├── stats
        ├── fields / SELECT
        └── Investigation workflow
                │
                ▼
CHAPTER 3
Text Searching, Wildcards, Regex, and Field Matching
```

The next chapters will build on this foundation rather than asking you to
memorize isolated commands.

---

# Official Splunk Documentation Used for This Chapter

1. Understanding SPL2 syntax  
   https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/introduction/understanding-spl2-syntax

2. Start searching data using SPL2  
   https://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/getting-started/quick-start-write-and-run-a-basic-spl2-search/start-searching-data-using-spl2

3. `search` command — overview and syntax  
   https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/search-command/search-command-overview-and-syntax

4. `from` command — syntax  
   https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/from-command/from-command-syntax

5. `where` command — overview, syntax, and usage  
   https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/where-command/where-command-overview-syntax-and-usage

6. Wildcards in SPL2  
   https://help.splunk.com/en/splunk-enterprise/search/spl2-search-manual/wildcards-quotes-and-escape-characters/wildcards

7. `stats` command  
   https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/stats-command/stats-command-overview-syntax-and-usage

8. Splunk search command primer  
   https://help.splunk.com/en/splunk-enterprise/search/search-manual/10.2/search-primer/search-command-primer
