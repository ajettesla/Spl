# Splunk SPL + SPL2

# Chapter 4 — Selecting, Renaming, Sorting, and Limiting Results

So far, we have learned how to find the events that matter and how to search text and patterns. This chapter moves from **finding events** to **shaping the results** so an analyst can actually work with them.

For example:

### Traditional SPL
```spl
index=windows
| search FileName="powershell.exe"
```

### SPL2
```spl2
FROM windows
WHERE FileName="powershell.exe"
```

Those searches answer:

> Which events match my condition?

Now suppose each event contains many fields and your search returns thousands of events. You may only want a focused result containing time, host, user, process, and command line.

This chapter therefore answers four practical questions:

```text
Which fields should I see?
What should those fields be called?
In what order should the results appear?
How many results should I return?
```

The important Splunk-specific point is that **SPL and SPL2 have several ways to shape results**, and the syntax and semantics are not always identical.

---

# 4.1 Finding Data vs Shaping Data

The filtering concepts were covered in the previous chapters, so we will not repeat them here.

### Traditional SPL
```spl
index=windows
| search FileName="powershell.exe"
```

### SPL2
```spl2
FROM windows
WHERE FileName="powershell.exe"
```

These searches find events.

Now suppose the resulting events contain:

```text
_time
host
source
sourcetype
user
FileName
CommandLine
ProcessId
ParentProcessId
hash
...
```

You may only want:

```text
_time
host
user
FileName
CommandLine
```

That is result shaping.

Think of the workflow as:

```text
FILTER
   ↓
Which events?

FIELD SELECTION
   ↓
Which fields?

RENAME
   ↓
What should the fields be called?

SORT
   ↓
What order?

LIMIT
   ↓
How many rows?
```

---

# 4.2 Traditional SPL: `table`

In traditional SPL, `table` is an easy way to create a focused tabular result.

```spl
index=windows
| search FileName="powershell.exe"
| table _time host user FileName CommandLine
```

This tells Splunk to display these fields in this order.

For example:

```text
_time                  host   user          FileName          CommandLine
--------------------------------------------------------------------------
10:15:22               DC01   administrator powershell.exe    powershell -enc ...
10:18:47               DC02   alice         powershell.exe    powershell -nop ...
```

`table` is especially useful when you are preparing readable output for an investigation, report, or dashboard.

Splunk documents `table` as a command for specifying which fields appear in the results and their order.

---

# 4.3 Traditional SPL: `fields`

Traditional SPL also provides the `fields` command.

To keep fields:

```spl
index=windows
| search FileName="powershell.exe"
| fields _time host user FileName CommandLine
```

To remove fields:

```spl
| fields - _raw
```

or:

```spl
| fields - debug_field temporary_field
```

The useful mental distinction is:

```text
table
   ↓
Focused tabular output

fields
   ↓
Keep or remove fields from results
```

They can look similar in simple searches, but they are not the same command.

---

# 4.4 SPL2: `SELECT`

SPL2 introduces a structured projection [projection: choosing which fields or expressions appear in the result] through the `SELECT` clause.

For example:

```spl2
SELECT _time, host, user, FileName, CommandLine
FROM windows
WHERE FileName="powershell.exe"
```

The same search can be written in the `FROM`-first form:

```spl2
FROM windows
WHERE FileName="powershell.exe"
SELECT _time, host, user, FileName, CommandLine
```

SPL2 supports both forms.

The important idea is:

```text
SELECT field1, field2, field3
```

means:

> Return these fields or expressions.

The current SPL2 documentation defines `SELECT` as a clause used to retrieve specific fields and, in searches, to evaluate expressions and aggregations.

---

# 4.5 An Important SPL2 Difference: `SELECT` vs `fields`

Do not treat these as exact synonyms.

For example:

```spl2
FROM windows
SELECT host, user, FileName
```

The `SELECT` clause restricts the fields returned and can remove unspecified internal fields.

By contrast:

```spl2
FROM windows
| fields host, user, FileName
```

uses the `fields` command, whose documented behavior retains the selected event fields while internal fields such as `_time` and `_raw` are included unless explicitly removed.

This means that `SELECT` can have a more aggressive effect on the result shape.

If `_time` or `_raw` is important, include it explicitly in the `SELECT` clause when needed.

Splunk's current documentation specifically notes this difference.

---

# 4.6 SPL2: `fields`

SPL2 also supports the `fields` command.

Keep fields:

```spl2
FROM windows
| fields _time, host, user, FileName, CommandLine
```

Remove fields:

```spl2
FROM windows
| fields - _raw, _indextime
```

A syntax difference that matters is the field separator.

Traditional SPL commonly uses:

```spl
| fields host user FileName
```

SPL2 requires the field list to be comma-delimited:

```spl2
| fields host, user, FileName
```

SPL2 also requires field names containing special characters to be quoted appropriately.

---

# 4.7 `table` vs `fields` vs `SELECT`

Keep this comparison in mind:

| Purpose | Traditional SPL | SPL2 |
|---|---|---|
| Tabular output | `table _time host user` | `table _time, host, user` |
| Keep fields | `fields _time host user` | `fields _time, host, user` |
| Remove fields | `fields - _raw` | `fields - _raw` |
| Select fields in structured syntax | — | `SELECT _time, host, user FROM windows` |

These overlap, but they solve slightly different problems.

A good mental model is:

```text
table
   ↓
Presentation-oriented table

fields
   ↓
Field inclusion/exclusion

SELECT
   ↓
SPL2 structured field/expression selection
```

---

# 4.8 `SELECT *`

SPL2 supports:

```spl2
SELECT * FROM windows
```

and:

```spl2
FROM windows
SELECT *
```

This is useful when you want to inspect the complete result structure before deciding which fields matter.

You can also use wildcards for similar field names in the `SELECT` clause by enclosing the wildcard expression in single quotes, for example:

```spl2
SELECT 'host*' FROM windows
```

The current SPL2 documentation requires those quotes when the field name itself contains a wildcard.

---

# 4.9 `SELECT DISTINCT`

SPL2 also supports:

```spl2
SELECT DISTINCT host
FROM windows
WHERE FileName="powershell.exe"
```

This returns unique combinations of the selected values.

For example, if the same host produced 500 matching events, the result can contain that host once rather than 500 event rows.

Conceptually:

```text
Many events
    ↓
Repeated host values
    ↓
SELECT DISTINCT host
    ↓
Unique host values
```

This is useful for questions such as:

> Which hosts had this activity?

It belongs here because it is a result-selection operation; aggregation will be covered in Chapter 5.

---

# 4.10 Renaming Fields in Traditional SPL

Sometimes the source field name is not the name you want users to see.

Traditional SPL provides the `rename` command.

For example:

```spl
index=windows
| rename ComputerName AS Endpoint
```

Now the result uses:

```text
Endpoint
```

instead of:

```text
ComputerName
```

You can rename several fields:

```spl
| rename ComputerName AS Endpoint, UserName AS User
```

Renaming here is a search-result operation. It does not rewrite the original indexed event.

---

# 4.11 Renaming Fields in SPL2

SPL2 supports the `AS` keyword when selecting fields or expressions.

For example:

```spl2
SELECT host AS Endpoint,
       user AS User
FROM windows
```

SPL2 also supports the `rename` command:

```spl2
FROM windows
| rename host AS Endpoint
```

Therefore:

```text
Traditional SPL
rename old AS new

SPL2 structured form
SELECT old AS new

SPL2 pipeline form
| rename old AS new
```

Use the structured `SELECT ... AS ...` form when the rest of the query is already using the `FROM` clause style.

The current SPL2 syntax uses `AS` for aliases and also documents the `rename` command.

---

# 4.12 Renaming Calculated Results

Renaming becomes particularly useful once aggregation is introduced.

For example, SPL2 can name an aggregation directly:

```spl2
SELECT count() AS EventCount
FROM windows
```

Instead of leaving the output with a generated expression name, you now have:

```text
EventCount
```

This becomes especially useful with `ORDER BY`, because you can sort on the clean alias.

For example:

```spl2
FROM windows
GROUP BY host
SELECT host, count() AS EventCount
ORDER BY EventCount DESC
```

We will study `GROUP BY` and `count()` in Chapter 5.

---

# 4.13 Sorting Results

Suppose your search returns hundreds of events and you want the newest events first.

Traditional SPL commonly uses:

```spl
| sort -_time
```

The `-` means descending order for that field.

For ascending order:

```spl
| sort +_time
```

Conceptually:

```text
ASC
 ↓
earliest → latest

DESC
 ↓
latest → earliest
```

This is the traditional SPL pipeline form.

---

# 4.14 SPL2: `ORDER BY`

SPL2 adds an `ORDER BY` clause to the structured `FROM` syntax.

For example:

```spl2
FROM windows
WHERE FileName="powershell.exe"
ORDER BY _time DESC
```

This means:

> Sort the results by `_time`, newest first.

Ascending order is:

```spl2
ORDER BY _time ASC
```

The current SPL2 documentation defines `ORDER BY` as a clause that sorts on one or more expressions.

---

# 4.15 SPL2 Still Has the `sort` Command

Do not assume that `ORDER BY` replaced `sort` everywhere.

SPL2 also supports the `sort` command in a pipeline:

```spl2
FROM windows
| sort -_time
```

So SPL2 gives you two useful styles:

```text
Structured FROM syntax
    ↓
ORDER BY _time DESC

Pipeline syntax
    ↓
| sort -_time
```

The first belongs to the `FROM` clause hierarchy; the second is a pipeline command.

---

# 4.16 Sorting by Multiple Fields

Suppose you want:

1. Newest event first.
2. For equal timestamps, host ascending.

Traditional SPL:

```spl
| sort -_time host
```

SPL2 pipeline form:

```spl2
| sort -_time, host
```

SPL2 structured form:

```spl2
ORDER BY _time DESC, host ASC
```

This gives you:

```text
Primary sort
    ↓
_time DESC

Tie-breaker
    ↓
host ASC
```

---

# 4.17 Lexicographical Sorting

Sorting is not always numeric just because the values look numeric.

For example, string ordering can produce something like:

```text
10
100
20
3
```

because lexicographical [lexicographical: ordered according to character sequence rather than numeric magnitude] comparison is being used.

This matters for fields such as ports, identifiers, or other values that may be stored as strings.

SPL2 documents lexicographical sorting and also provides sort-type functions for certain data types.

---

# 4.18 Sorting IP Addresses in SPL2

An IP address is a useful example because string ordering is not the same as numerical IP ordering.

SPL2 supports a type-oriented sort expression such as:

```spl2
| sort ip(src_ip)
```

This tells Splunk to compare the field using IP-address semantics rather than ordinary string ordering.

The lesson is:

```text
Field appearance
      ≠
Data type used for sorting
```

When sort order matters, know what type of value you are sorting.

---

# 4.19 Limiting Results With `head`

Traditional SPL uses:

```spl
| head 10
```

For example:

```spl
index=windows
| search FileName="powershell.exe"
| sort -_time
| head 10
```

This means:

```text
Find events
   ↓
Sort newest first
   ↓
Return the first 10
```

SPL2 also supports the `head` command:

```spl2
FROM windows
| sort -_time
| head 10
```

So `head` is one command that remains available in both SPL and SPL2.

---

# 4.20 Why Sorting Before `head` Matters

Compare:

```spl
| head 10
| sort -_time
```

with:

```spl
| sort -_time
| head 10
```

They do not express the same operation.

The first performs:

```text
Take 10
   ↓
Sort those 10
```

The second performs:

```text
Sort the results
   ↓
Take 10
```

If the requirement is:

> Give me the 10 newest matching events.

the second order is the appropriate expression of that requirement.

This is a general query principle:

> **Command order matters.**

---

# 4.21 SPL2: `LIMIT`

SPL2 provides another way to limit rows when using the structured `FROM` syntax:

```spl2
FROM windows
WHERE FileName="powershell.exe"
ORDER BY _time DESC
LIMIT 10
```

Now the query reads naturally:

```text
FROM
 ↓
WHERE
 ↓
ORDER BY
 ↓
LIMIT
```

The current SPL2 documentation defines `LIMIT` as a clause that sets the maximum number of rows returned.

---

# 4.22 SPL2: `OFFSET`

SPL2 also supports:

```spl2
OFFSET
```

It skips rows before returning results.

For example:

```spl2
FROM windows
ORDER BY _time DESC
LIMIT 10
OFFSET 20
```

Conceptually:

```text
Rows 1–20
   ↓
skipped

Rows 21–30
   ↓
returned
```

This is useful for pagination [pagination: dividing a large result set into smaller sequential sections].

It is not something you will need in every SOC investigation, but it is part of the current SPL2 search syntax and belongs in your fundamentals.

---

# 4.23 `LIMIT` vs `head`

The general concept is the same, but the syntax belongs to different parts of SPL2.

### Traditional SPL
```spl
index=windows
| sort -_time
| head 10
```

### SPL2 pipeline
```spl2
FROM windows
| sort -_time
| head 10
```

### SPL2 structured syntax
```spl2
FROM windows
ORDER BY _time DESC
LIMIT 10
```

This is one of the clearest examples of SPL2 giving you both a pipeline style and a structured clause style.

---

# 4.24 `ORDER BY` + `LIMIT` vs `sort` + `head`

These forms can express the same basic intent:

```spl2
FROM windows
ORDER BY _time DESC
LIMIT 10
```

and:

```spl2
FROM windows
| sort -_time
| head 10
```

But they belong to different syntactic models.

Think:

```text
SPL-style pipeline
    ↓
commands connected by |

SPL2 structured search
    ↓
FROM / WHERE / SELECT / ORDER BY / LIMIT
```

This distinction becomes increasingly important when we reach grouping and aggregation.

---

# 4.25 A Complete Traditional SPL Example

Suppose the investigation question is:

> Show the 10 newest PowerShell executions.

```spl
index=windows
| search FileName="powershell.exe"
| fields _time host user FileName CommandLine
| sort -_time
| head 10
```

Read it as:

```text
FIND
 ↓
PowerShell events

FIELDS
 ↓
Useful fields

SORT
 ↓
Newest first

LIMIT
 ↓
10 events
```

---

# 4.26 A Complete SPL2 Pipeline Example

The pipeline-oriented SPL2 form is:

```spl2
FROM windows
| where FileName="powershell.exe"
| fields _time, host, user, FileName, CommandLine
| sort -_time
| head 10
```

The mental workflow is almost identical to traditional SPL.

---

# 4.27 A Complete Structured SPL2 Example

The same investigation can be written using the structured `FROM` clauses:

```spl2
FROM windows
WHERE FileName="powershell.exe"
SELECT _time, host, user, FileName, CommandLine
ORDER BY _time DESC
LIMIT 10
```

Now each part has a defined role:

```text
FROM
    What dataset?

WHERE
    Which events?

SELECT
    Which fields?

ORDER BY
    What order?

LIMIT
    How many rows?
```

This structured syntax is one of the important additions that makes SPL2 different from traditional SPL.

---

# 4.28 Selecting Fields Does Not Filter Events

Keep this distinction clear.

Traditional SPL:

```spl
| search FileName="powershell.exe"
```

filters events.

But:

```spl
| fields FileName
```

changes which fields remain in the result.

SPL2 follows the same conceptual distinction:

```spl2
WHERE FileName="powershell.exe"
```

filters events, while:

```spl2
SELECT FileName
```

selects a field for the result.

Therefore:

```text
FILTER
→ Which events?

FIELD SELECTION
→ Which fields?

LIMIT
→ How many rows?
```

---

# 4.29 A Useful SOC Output Pattern

For event-level investigations, a common pattern is:

### Traditional SPL
```spl
index=windows
| search FileName="powershell.exe"
| table _time host user FileName CommandLine
| sort -_time
| head 20
```

### SPL2 pipeline
```spl2
FROM windows
| where FileName="powershell.exe"
| fields _time, host, user, FileName, CommandLine
| sort -_time
| head 20
```

### SPL2 structured
```spl2
FROM windows
WHERE FileName="powershell.exe"
SELECT _time, host, user, FileName, CommandLine
ORDER BY _time DESC
LIMIT 20
```

This is the event-level pattern we will build on in Chapter 5.

---

# 4.30 A New SPL2 Concept: Clause Hierarchy

This belongs in the fundamentals because SPL2's structured syntax has an explicit clause order.

For a normal `FROM` search, the current documentation defines the hierarchy broadly as:

```text
FROM
 ↓
JOIN
 ↓
WHERE
 ↓
GROUP BY
 ↓
SELECT
 ↓
HAVING
 ↓
ORDER BY
 ↓
LIMIT
 ↓
OFFSET
```

You do not need every clause in every search.

However, when you use the structured `FROM` syntax, the clauses have to be placed in the supported order.

For example:

```spl2
FROM windows
WHERE EventCode=4625
ORDER BY _time DESC
LIMIT 10
```

This is not just a stylistic preference; it follows the documented SPL2 clause hierarchy.

This becomes especially important when Chapter 5 introduces:

```text
GROUP BY
SELECT aggregation
HAVING
ORDER BY
LIMIT
```

---

# 4.31 One Important Performance Distinction

Do not confuse:

```text
I only want to DISPLAY 10 rows
```

with:

```text
I only want Splunk to RETRIEVE or PROCESS 10 events.
```

Field selection and row limiting are different operations.

For example:

```spl
index=windows
| table _time host user
```

does not mean only 10 events were retrieved.

Similarly, a search can perform earlier commands before the limiting step is reached.

Splunk documents `head` as a way to limit retrieved events and recommends using a sample set when you are only testing whether a search retrieves the expected events.

We will study search optimization separately instead of mixing performance tuning into this chapter.

---

# 4.32 Common Beginner Mistakes

## Mistake 1 — Thinking `table` filters events

```spl
| table host user
```

does not decide which events survive.

It decides which fields appear in the table.

---

## Mistake 2 — Treating `SELECT` and `fields` as identical

They overlap, but SPL2 documents an important difference: the `SELECT` clause can remove unspecified internal fields, while `fields` has different retention behavior for internal fields.

Be deliberate when `_time` and `_raw` matter.

---

## Mistake 3 — Sorting after `head`

If you need the newest 10:

```spl
| sort -_time
| head 10
```

not:

```spl
| head 10
| sort -_time
```

---

## Mistake 4 — Treating `LIMIT` as another spelling of `head`

They can express a similar result limit, but:

```spl2
LIMIT 10
```

belongs to the structured SPL2 `FROM` syntax, whereas:

```spl2
| head 10
```

is a pipeline command.

---

## Mistake 5 — Forgetting SPL2 comma rules for `fields`

Traditional SPL commonly permits:

```spl
| fields host user FileName
```

SPL2 requires:

```spl2
| fields host, user, FileName
```

---

## Mistake 6 — Assuming numeric-looking strings sort numerically

A field containing the string `443` is not automatically the same as a numeric field containing `443` for sorting purposes.

Know the data type when order matters.

---

# 4.33 What You Should Remember

If you remember only these ideas:

```text
FILTER
→ Which events?

TABLE / FIELDS / SELECT
→ Which fields?

RENAME / AS
→ What should the fields be called?

SORT / ORDER BY
→ What order?

HEAD / LIMIT
→ How many?
```

And remember the three main forms:

### Traditional SPL
```spl
index=windows
| search ...
| fields ...
| sort -_time
| head 10
```

### SPL2 pipeline
```spl2
FROM windows
| where ...
| fields ...
| sort -_time
| head 10
```

### SPL2 structured
```spl2
FROM windows
WHERE ...
SELECT ...
ORDER BY _time DESC
LIMIT 10
```

Do not learn SPL2 as a simple replacement dictionary for SPL. SPL2 adds a structured clause model that becomes increasingly useful as your searches move into grouping and aggregation.

---

# 4.34 Chapter Summary

In this chapter, we moved from finding events to shaping the results.

We learned the major result-shaping tools in both languages.

### Traditional SPL

```text
table
fields
rename
sort
head
```

### SPL2

```text
SELECT
SELECT DISTINCT
fields
rename / AS
ORDER BY
sort
LIMIT
OFFSET
head
```

We also learned that:

- `table` is useful for focused tabular output.
- `fields` keeps or removes fields.
- `SELECT` performs structured field/expression selection in SPL2.
- `SELECT` can remove unspecified internal fields, so `_time` and `_raw` must be included when needed.
- `rename` changes field names in the result.
- `AS` can create clearer field names, especially with SPL2 `SELECT`.
- `sort` controls order in a pipeline.
- `ORDER BY` controls order in the structured SPL2 syntax.
- `head` limits pipeline results.
- `LIMIT` limits rows in structured SPL2 syntax.
- `OFFSET` skips rows and is useful for pagination.
- `SELECT DISTINCT` returns unique combinations of selected values.
- Sorting before limiting matters when the requirement is "newest N" or "largest N".
- Lexicographical ordering can differ from numerical ordering.
- Field selection and event filtering are different operations.
- The SPL2 `FROM` clause has a documented hierarchy that will matter when we introduce aggregation.

The central workflow is:

```text
FIND
 ↓
FILTER
 ↓
SELECT FIELDS
 ↓
RENAME
 ↓
SORT
 ↓
LIMIT
```

---

# 4.35 Official Splunk Documentation

This chapter follows the current official Splunk documentation for:

- SPL2 `FROM` syntax and clause hierarchy
- SPL2 `SELECT` and `SELECT DISTINCT`
- SPL2 `fields`
- SPL2 `ORDER BY`
- SPL2 `LIMIT`
- SPL2 `OFFSET`
- SPL2 `head`
- SPL2 `rename` and `AS`
- Traditional SPL `table`
- Traditional SPL `fields`
- Traditional SPL `sort`
- Traditional SPL `head`

The current Splunk documentation should take precedence over older tutorials when syntax differs between releases.

# Chapter 5

**Grouping and Aggregation**

Next we move from:

```text
individual events
```

to:

```text
counts
statistics
frequencies
grouped results
```

with:

```text
SPL
stats
chart
timechart

SPL2
GROUP BY
SELECT count(), sum(), avg(), ...
HAVING
```
