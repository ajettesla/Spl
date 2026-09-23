# Splunk SPL + SPL2

# Chapter 3 — Text Searching, Wildcards, and Regular Expressions

## Introduction

In Chapter 2, we learned how to build a search from an investigation question and how to use basic field conditions and pipeline stages.

This chapter goes one step deeper:

> **When I know the field I want to search, how do I decide whether I need an exact value, a wildcard, a pattern, or a regular expression?**

This distinction matters because Splunk has several different mechanisms for text matching, and **SPL and SPL2 do not use exactly the same syntax for every mechanism**.

The main tools we will learn here are:

```text
search
where
LIKE / like()
regex
match()
rex
CASE()
TERM()
IN
```

We will not repeat the basic meaning of `index`, `host`, `source`, `sourcetype`, `_raw`, or `_time` from Chapter 1, or the basic pipeline and Boolean concepts from Chapter 2. Instead, we will build directly on them.

---

# 3.1 First Decide What Kind of Match You Need

Suppose a field contains:

```text
powershell.exe -NoProfile -ExecutionPolicy Bypass
```

There are several different questions you could ask.

### Exact value

> Is the field exactly `powershell.exe`?

### Prefix pattern

> Does the value begin with `powershell`?

### Substring pattern

> Does `powershell` occur anywhere in the value?

### Whole-word pattern

> Does `powershell` occur as a complete word?

### Structured pattern

> Does the value match a more complicated expression?

These are different requirements, so the first step is to choose the matching mechanism rather than immediately writing regex.

A useful decision process is:

```text
What do I know?
      |
      +-- Exact value ----------> field="value"
      |
      +-- Several values --------> IN (...)
      |
      +-- Simple wildcard -------> search wildcard *
      |
      +-- Expression pattern ----> LIKE / like()
      |
      +-- Complex pattern -------> regex / match()
```

---

# 3.2 A Critical Correction: SPL `regex` Is Not LogScale Regex Syntax

This is the example that often causes confusion when moving between platforms.

In CrowdStrike LogScale, you may see a pattern written like:

```text
/powershell/i
```

Do **not** copy that syntax into Splunk.

In traditional Splunk SPL, a regex expression is normally written as a quoted string:

```spl
| regex FileName="(?i)powershell"
```

This is valid SPL syntax according to the Splunk Enterprise 10.4 `regex` command reference.

The syntax documented by Splunk is:

```spl
regex (<field>=<regex-expression> | <field>!=<regex-expression> | <regex-expression>)
```

The regex is an unanchored PCRE expression and quotation marks are required. If no field is specified, the command matches against `_raw` by default.

**Very important:** the example

```spl
| regex FileName="(?i)powershell"
```

is valid. If it does not return anything, that does **not** automatically mean the regex syntax is wrong.

The next question is:

> **Does the current event actually contain a field called `FileName`?**

If `FileName` does not exist or was not extracted for those events, the field-based regex cannot match it.

As a diagnostic test, search the raw event instead:

```spl
index=windows
| regex "(?i)powershell"
```

Because no field is specified, the `regex` command applies the expression to `_raw`.

Or, when the field exists, use:

```spl
index=windows
| regex FileName="(?i)powershell"
```

Splunk documents that the default field for the `regex` command is `_raw`, and that `field=<regex-expression>` keeps results whose field value matches the expression. [Official source: Splunk Enterprise 10.4 Search Reference — `regex` command.]

---

# 3.3 Traditional SPL: `regex` Filters Events

The traditional SPL `regex` command is a **filtering command**.

For example:

```spl
index=windows
| regex CommandLine="(?i)powershell"
```

Think of it as:

```text
Event
  |
  +-- CommandLine matches regex? --> YES --> keep
  |
  +-- No --------------------------> remove
```

Splunk's documentation explicitly distinguishes `regex` from `rex`:

```text
regex
  -> filter results using regex

rex
  -> extract fields or perform sed replacement
```

That distinction should stay in your memory because the two commands look similar but perform different jobs.

---

# 3.4 SPL2: Use `match()` for Regex Filtering

This is where we must separate SPL from SPL2 carefully.

Current SPL2 documentation explains that regular expressions are used with the `rex` command and evaluation functions such as `match()` and `replace()`.

For an expression-based regex filter, use `match()`:

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell")
```

Or, after another command has produced results:

```spl2
FROM windows
| where match(CommandLine, "(?i)powershell")
```

`match()` returns `TRUE` when the regex finds a match against any substring of the string value and `FALSE` otherwise.

So the mental model becomes:

```text
Traditional SPL
    regex
      |
      +-- regex filter

SPL / SPL2 expression
    match()
      |
      +-- TRUE / FALSE
```

The official SPL2 documentation specifically lists `match()` as a regular-expression function and allows it in `where` and the `WHERE` clause of `FROM`. [Official source: Splunk Enterprise 10.4 SPL2 Search Reference — comparison and conditional functions; SPL2 and regular expressions.]

---

# 3.5 The Same Investigation in SPL and SPL2

Suppose the requirement is:

> Find events whose `CommandLine` contains `powershell`, ignoring capitalization.

### Traditional SPL

```spl
index=windows
| regex CommandLine="(?i)powershell"
```

### Traditional SPL using `where` + `match()`

```spl
index=windows
| where match(CommandLine, "(?i)powershell")
```

### SPL2 using `WHERE`

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell")
```

### SPL2 using the pipeline `where`

```spl2
FROM windows
| where match(CommandLine, "(?i)powershell")
```

The investigation requirement is identical. The important difference is the command language and the place where the expression is evaluated.

---

# 3.6 Why `(?i)`?

In Splunk regular expressions, `(?i)` is an inline modifier that requests case-insensitive matching.

For example:

```text
(?i)powershell
```

can match:

```text
powershell
PowerShell
POWERSHELL
```

This is different from LogScale's slash-style notation:

```text
/powershell/i
```

For Splunk, keep the regex itself inside the quoted expression:

```spl
"(?i)powershell"
```

Splunk documents regular expressions as PCRE in the Splunk search language. [Official source: Splunk SPL2 Search Manual — About Splunk regular expressions.]

---

# 3.7 Unanchored Regex Means "Find It Anywhere"

The Splunk `regex` command uses an **unanchored** regular expression unless you add anchors yourself.

Therefore:

```spl
| regex CommandLine="(?i)powershell"
```

can match values such as:

```text
powershell.exe
cmd.exe /c powershell
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

You do not need `.*` around the word just to make it search through the field.

This is an important correction to the common beginner pattern:

```text
.*powershell.*
```

For an ordinary unanchored regex search, that is unnecessary.

Use the smallest pattern that expresses the requirement.

The official `regex` command reference describes the expression as an unanchored regular expression, and the `match()` documentation likewise states that `match()` searches for the regex against any substring. [Official sources: Splunk Enterprise 10.4 `regex`; SPL2 comparison and conditional functions.]

---

# 3.8 Exact Value vs Substring Regex

Compare these two searches.

### Exact field value

```spl
index=windows FileName="powershell.exe"
```

This asks for the field value:

```text
powershell.exe
```

### Regex substring search

```spl
index=windows
| regex FileName="(?i)powershell"
```

This asks whether the regex finds `powershell` somewhere in the field.

For example:

```text
powershell.exe
```

can satisfy both.

But:

```text
powershell_ise.exe
```

can satisfy the regex while not satisfying the exact value comparison.

So ask:

```text
Do I know the complete value?
        |
       YES
        |
   Exact search

Do I know only part of the value or a pattern?
        |
       YES
        |
   Wildcard / regex
```

---

# 3.9 Search Wildcards: `*`

In the `search` language, Splunk uses:

```text
*
```

to match an unlimited number of characters.

For example:

### Traditional SPL

```spl
index=windows FileName="powershell*"
```

### SPL2 `search`

```spl2
search index=windows FileName="powershell*"
```

This is a **search wildcard**, not regex.

It can match values such as:

```text
powershell.exe
powershell_ise.exe
powershell7.exe
```

Splunk's current SPL2 wildcard documentation explicitly states that the wildcard depends on the command: the `search` language uses `*`, while `where`/`WHERE` uses `LIKE` with `%` and `_`. [Official source: Splunk Enterprise SPL2 Search Manual — Wildcards.]

---

# 3.10 `*` vs `.*`

This distinction is fundamental.

### Search wildcard

```text
powershell*
```

belongs to the search language.

### Regex

```text
powershell.*
```

belongs to the regex language.

The regex form means:

```text
powershell
+
any character
+
zero or more repetitions
```

Do not use regex syntax when you are writing a normal `search` wildcard, and do not assume that a search wildcard is a regex.

---

# 3.11 SPL2 `LIKE`

SPL2 adds an important expression-based pattern mechanism.

Inside `WHERE`, use `LIKE` rather than the `*` search wildcard.

Example:

```spl2
FROM windows
WHERE FileName LIKE "powershell%"
```

Here:

```text
%
```

means zero or more characters.

And:

```text
_
```

means exactly one character.

For example:

```spl2
FROM windows
WHERE FileName LIKE "win%"
```

matches values beginning with `win`.

This is different from:

```spl2
search index=windows FileName="win*"
```

because `search` and `WHERE` use different pattern semantics.

Splunk's official SPL2 documentation explicitly states that the `WHERE` clause and `where` command use the `LIKE` function with `%` and `_`; `*` is used by other commands such as `search`. [Official source: Splunk Enterprise SPL2 Search Manual — Wildcards; Comparison and Conditional Functions.]

---

# 3.12 `LIKE` and `like()`

The SPL2 predicate:

```spl2
FileName LIKE "powershell%"
```

and the function:

```spl2
like(FileName, "powershell%")
```

represent the same general pattern-matching concept.

For example:

```spl2
FROM windows
WHERE like(FileName, "powershell%")
```

The official documentation notes that the `LIKE` predicate is similar to the `like()` function. [Official source: Splunk Enterprise SPL2 Search Reference — Comparison and Conditional Functions.]

---

# 3.13 Regex Anchors: `^` and `$`

Now we move from "contains" to precise positions.

## `^` — beginning of the string

```text
^powershell
```

means that `powershell` must occur at the beginning.

Traditional SPL:

```spl
index=windows
| regex CommandLine="(?i)^powershell"
```

SPL2:

```spl2
FROM windows
WHERE match(CommandLine, "(?i)^powershell")
```

This can match:

```text
powershell.exe -enc AAAA
powershell -nop
```

but not:

```text
cmd.exe /c powershell
```

---

## `$` — end of the string

```text
\.exe$
```

means that the value must end with `.exe`.

Traditional SPL:

```spl
index=windows
| regex FileName="(?i)\.exe$"
```

SPL2:

```spl2
FROM windows
WHERE match(FileName, "(?i)\.exe$")
```

This can match:

```text
cmd.exe
powershell.exe
chrome.exe
```

but not:

```text
powershell.exe.bak
```

---

# 3.14 Why `\.`?

In regex, a period means:

```text
match any character except a line break
```

So this:

```text
.exe
```

is not precise enough when you mean a literal period.

Use:

```text
\.exe
```

The backslash escapes the period.

Therefore:

```text
\.exe$
```

means:

```text
literal period
+
exe
+
end of the string
```

Splunk's regular-expression documentation specifically describes the need to escape special characters such as the period with a backslash. [Official source: Splunk SPL2 Search Manual — SPL2 and regular expressions.] 

---

# 3.15 Word Boundaries: `\b`

Suppose you search for:

```text
powershell
```

The regex can also match that text inside a larger value such as:

```text
mypowershellscript.exe
```

If the requirement is:

> Find `powershell` as a complete word.

Use:

```text
\bpowershell\b
```

Traditional SPL:

```spl
index=windows
| regex CommandLine="(?i)\bpowershell\b"
```

SPL2:

```spl2
FROM windows
WHERE match(CommandLine, "(?i)\bpowershell\b")
```

The important idea is that `\b` identifies a word boundary [word boundary: a position separating a word character from a non-word character].

This is useful when your requirement is about a complete word rather than an arbitrary substring.

---

# 3.16 Character Classes

Square brackets define a character class.

For example:

```text
[abc]
```

means one character from `a`, `b`, or `c`.

A range can be written as:

```text
[0-9]
```

for one digit.

Common regex character types include:

```text
\d    digit
\w    word character
\s    whitespace
```

These are building blocks rather than complete searches.

For example:

```text
User\s*=\s*\w+
```

can describe text around a `User=` assignment while allowing variable whitespace.

Do not start by memorizing every regex feature. Build from the characters you actually need.

---

# 3.17 Quantifiers: `*`, `+`, `?`

Regex quantifiers control repetition.

```text
*
```

means zero or more.

```text
+
```

means one or more.

```text
?
```

usually means zero or one when used as a quantifier.

For example:

```text
-enc\s+
```

means `-enc` followed by one or more whitespace characters.

This is more precise than using `.*` when the requirement is specifically whitespace.

---

# 3.18 Alternatives With `|`

Inside regex, the pipe character means OR.

For example:

```text
(powershell|pwsh)
```

means:

```text
powershell
OR
pwsh
```

To require `.exe` after either one:

```text
(powershell|pwsh)\.exe
```

Traditional SPL:

```spl
index=windows
| regex FileName="(?i)(powershell|pwsh)\.exe"
```

SPL2:

```spl2
FROM windows
WHERE match(FileName, "(?i)(powershell|pwsh)\.exe")
```

Because `|` is also Splunk's pipeline separator, keep a regex containing `|` inside the quoted regex expression.

Splunk's regular-expression documentation explicitly calls this out. [Official source: Splunk SPL2 Search Manual — SPL2 and regular expressions.]

---

# 3.19 `IN` Can Be Better Than Regex

Suppose you know the complete values you want:

```text
cmd.exe
powershell.exe
pwsh.exe
```

Do not automatically write a regex.

Traditional SPL:

```spl
index=windows
| search FileName IN ("cmd.exe","powershell.exe","pwsh.exe")
```

SPL2 `search`:

```spl2
search index=windows FileName IN ("cmd.exe","powershell.exe","pwsh.exe")
```

`IN` is intended for a list of field-value matches.

Regex becomes more useful when the values share a pattern rather than a short known list.

---

# 3.20 `CASE()` for Case-Sensitive `search`

Normal Splunk `search` behavior is case-insensitive for terms and field values.

When the exact capitalization matters, Splunk provides:

```text
CASE()
```

For example:

```spl
search host=CASE(LOCALHOST)
```

searches for the specified case.

This belongs to the `search` language, not regex.

Therefore keep these separate in your mind:

```text
CASE()
   |
   +-- case-sensitive search term/value

(?i)
   |
   +-- case-insensitive regex
```

Splunk's official documentation describes `CASE()` as a directive for case-sensitive term and field-value matching. [Official source: Splunk Search Manual — Use CASE() and TERM() to match phrases.] 

---

# 3.21 `TERM()` for Indexed Terms With Punctuation

Another search feature worth knowing before we leave this chapter is:

```text
TERM()
```

`TERM()` tells Splunk to treat the contents as a single indexed term, which can be useful when the value contains minor segmenters [minor segmenters: characters Splunk can recognize as boundaries between indexed terms], such as periods.

A common example is:

```spl
search TERM(127.0.0.1)
```

This is not regex.

It is a search-time instruction about how Splunk should match an indexed term.

Do not use `TERM()` simply because a value contains a dot; it has specific indexing and tokenization semantics.

Splunk's current Search Manual documents `TERM()` for terms containing minor segmenters such as periods or underscores and explains its boundaries and limitations. [Official source: Splunk Search Manual — Use CASE() and TERM() to match phrases.]

---

# 3.22 `regex` vs `match()`

These two often confuse beginners because both use regex.

### Traditional SPL `regex`

```spl
| regex CommandLine="(?i)powershell"
```

Its purpose is to remove results that do not match the expression.

### SPL/SPL2 `match()`

```spl
| where match(CommandLine, "(?i)powershell")
```

or:

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell")
```

Its purpose is to return a Boolean result that can be used inside an expression.

Think:

```text
regex
  |
  +-- command that filters

match()
  |
  +-- function that returns TRUE/FALSE
```

The difference is not the regex itself; it is the role of the regex in the search.

---

# 3.23 `regex` vs `rex`

This distinction is just as important.

### `regex` — filtering

```spl
index=windows
| regex CommandLine="(?i)powershell"
```

Question:

> Does this event match the regex?

### `rex` — extraction

```spl
index=windows
| rex field=CommandLine "(?<shell>powershell|pwsh)"
```

Question:

> Can I extract information from the field using this regex?

The result of the second command is a new field named `shell`.

The official Splunk 10.4 `rex` reference explicitly distinguishes extraction/replacement from the filtering behavior of `regex`. [Official source: Splunk Enterprise 10.4 Search Reference — `rex` and `regex`.]

---

# 3.24 `rex` Does Not Mean "Filter With Regex"

A common mistake is to expect:

```spl
| rex field=CommandLine "(?i)powershell"
```

to behave like:

```spl
| regex CommandLine="(?i)powershell"
```

They have different purposes.

`rex` is useful when you have a pattern and want to extract a named capture group.

For example:

```spl
| rex field=CommandLine "(?<shell>powershell|pwsh)"
```

creates `shell` when the expression matches.

If your goal is only to keep matching events, use `regex` in traditional SPL or `match()` in an expression.

---

# 3.25 A Reliable Way to Test a Regex

When a regex appears not to work, do not immediately change the regex five times.

Test the layers separately.

## Test 1 — Does the field exist?

```spl
index=windows
| table FileName CommandLine
```

Check whether `FileName` is actually present in the returned events.

## Test 2 — Look directly at `_raw`

```spl
index=windows
| table _raw FileName CommandLine
```

If the text is visible in `_raw` but the field is missing, this is a **field extraction issue**, not a regex issue.

## Test 3 — Test the regex against `_raw`

```spl
index=windows
| regex "(?i)powershell"
```

If this works, the regex is matching the raw event.

## Test 4 — Test it against the field

```spl
index=windows
| regex CommandLine="(?i)powershell"
```

If Test 3 works and Test 4 does not, investigate the field extraction.

This troubleshooting sequence is much better than blindly changing the pattern.

---

# 3.26 Why Your Original Example Can Appear Not to Work

Consider:

```spl
| regex FileName="(?i)powershell"
```

There are several separate possibilities:

```text
Regex syntax wrong?
       |
       +-- No. The SPL syntax is valid.

FileName exists?
       |
       +-- Maybe not.

FileName contains PowerShell?
       |
       +-- Maybe the value is stored in another field.

Text exists only in _raw?
       |
       +-- Then search _raw or extract the field first.

Running SPL2 instead of traditional SPL?
       |
       +-- Use match() in WHERE/where.
```

This is why a failed regex test does not automatically prove that the regex is invalid.

The official SPL `regex` syntax is valid exactly as shown above; the surrounding data and language context determine whether it produces matches. [Official source: Splunk Enterprise 10.4 `regex` command.]

---

# 3.27 A Complete PowerShell Example

Now we can build a real investigation without repeating the basic search lessons from Chapter 2.

Requirement:

> Find process events where the command line contains PowerShell and later contains `-enc` or `-encodedcommand`, ignoring capitalization.

### Traditional SPL with `regex`

```spl
index=windows
| regex CommandLine="(?i)powershell.*-(enc|encodedcommand)"
```

### Traditional SPL with `where` and `match()`

```spl
index=windows
| where match(CommandLine, "(?i)powershell.*-(enc|encodedcommand)")
```

### SPL2

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell.*-(enc|encodedcommand)")
```

Break the pattern down:

```text
(?i)
  |
  +-- ignore case

powershell
  |
  +-- find PowerShell

.*
  |
  +-- allow intervening characters

-
  |
  +-- hyphen

(enc|encodedcommand)
  |
  +-- either parameter name
```

This is a pattern-based investigation, which is exactly where regex becomes useful.

---

# 3.28 A More Precise Pattern

The previous expression can be broad because `.*` allows almost anything between the two pieces.

If you know that there should be whitespace before the option, a more precise pattern might be:

```text
powershell.*\s-(enc|encodedcommand)\b
```

Traditional SPL:

```spl
index=windows
| regex CommandLine="(?i)powershell.*\s-(enc|encodedcommand)\b"
```

SPL2:

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell.*\s-(enc|encodedcommand)\b")
```

Do not automatically make a regex more complicated. Add precision only when the investigation requirement justifies it.

---

# 3.29 Performance: Be Specific Before You Use Regex

Regex is flexible, but you should not use it for every search.

For example, when you know the exact value:

```spl
index=windows FileName="powershell.exe"
```

is conceptually simpler than:

```spl
index=windows
| regex FileName="(?i)^powershell\.exe$"
```

Likewise, if a search wildcard is enough:

```spl
index=windows FileName="powershell*"
```

there is no need to replace it with a regex merely because regex is more powerful.

Splunk's wildcard guidance recommends being as specific as possible and warns that poorly chosen wildcards, particularly broad leading wildcards, can increase the search cost. [Official source: Splunk Search Manual — Wildcards.]

The same general rule applies to regex design:

```text
Specific search
     |
     +-- easier to understand
     +-- easier to test
     +-- often less work

Broad regex
     |
     +-- more accidental matches
     +-- more difficult to debug
```

---

# 3.30 Current Regex Engine Note

There are two related but different contexts in current Splunk documentation.

For Splunk search language documentation, regular expressions are documented as PCRE [PCRE: Perl Compatible Regular Expressions].

For current Edge Processor and Ingest Processor pipelines, Splunk moved to PCRE2 [PCRE2: the newer major version of the Perl-compatible regular-expression engine]. Splunk's pipeline documentation states that from June 5, 2025, RE2 support ended and pipelines use PCRE2.

Therefore, when you copy an old example from the internet, first determine whether it is:

```text
Traditional SPL search
       |
       +-- search regex semantics

Edge / Ingest pipeline
       |
       +-- current pipeline regex semantics
```

Do not treat every regex example found online as version-neutral.

---

# 3.31 KQL Comparison — Only the Concept Matters

You may already know KQL concepts such as:

```text
contains
startswith
endswith
has
matches regex
```

The useful way to translate them into Splunk is by requirement, not by name.

| Requirement | Traditional SPL | SPL2 |
|---|---|---|
| Exact value | `field="value"` | `search field="value"` |
| Prefix search with search syntax | `field="value*"` | `search field="value*"` |
| Expression wildcard | `where like(field,"value%")` | `WHERE field LIKE "value%"` |
| Regex filter | `regex field="regex"` | `WHERE match(field,"regex")` |
| Several known values | `field IN (...)` | `search field IN (...)` |

The important lesson is that there is not always a one-to-one operator mapping.

---

# 3.32 Practical Decision Table

Use this as the working reference for the chapter.

| What you want | Use |
|---|---|
| One exact known value | `field="value"` |
| A few known values | `IN (...)` |
| Prefix/suffix-style search with `search` | `*` wildcard |
| Expression pattern in SPL2 `WHERE` | `LIKE` / `like()` |
| Complex pattern filter in traditional SPL | `regex` |
| Complex pattern as a Boolean expression | `match()` |
| Extract values from text | `rex` |
| Replace text at search time | `rex mode=sed` |
| Case-sensitive `search` term/value | `CASE()` |
| Match a single indexed term containing minor breakers | `TERM()` |

---

# 3.33 Common Mistakes

## Mistake 1 — Copying LogScale regex delimiters

Do not use:

```text
/powershell/i
```

as your Splunk regex syntax.

Use:

```spl
"(?i)powershell"
```

inside the Splunk command/function.

---

## Mistake 2 — Assuming `regex` is the SPL2 equivalent of traditional SPL

For traditional SPL:

```spl
| regex CommandLine="(?i)powershell"
```

For SPL2 expression filtering:

```spl2
| where match(CommandLine, "(?i)powershell")
```

or:

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell")
```

---

## Mistake 3 — Using regex when an exact search is enough

Prefer:

```spl
FileName="powershell.exe"
```

over an elaborate regex when you already know the exact value.

---

## Mistake 4 — Using `.*` when unanchored matching already solves the problem

This:

```text
powershell
```

already searches for the pattern within the string for `regex` and `match()` semantics.

You do not automatically need:

```text
.*powershell.*
```

---

## Mistake 5 — Forgetting to test the field

If:

```spl
| regex FileName="(?i)powershell"
```

returns nothing, inspect:

```spl
| table _raw FileName CommandLine
```

before changing the regex.

---

## Mistake 6 — Confusing a search wildcard with regex

```text
*
```

in `search` is not the same language as:

```text
.*
```

in regex.

---

# 3.34 Chapter Summary

At the end of this chapter, you should be able to answer the most important question before writing a text search:

> **What kind of match do I actually need?**

You should now understand:

- Traditional SPL `regex` is a filtering command.
- The syntax `| regex FileName="(?i)powershell"` is valid traditional SPL.
- If that search produces no results, check whether `FileName` exists and contains the value; the default target for `regex` is `_raw` when no field is specified.
- SPL2 uses regular expressions through `match()` and related functions; do not copy the traditional SPL `regex` command blindly into SPL2.
- `(?i)` is a Splunk regex modifier for case-insensitive matching.
- Splunk regex matching is normally unanchored, so a pattern such as `powershell` can match a substring.
- `^` anchors a regex to the start of a string.
- `$` anchors a regex to the end of a string.
- `\b` expresses a word boundary.
- `\.` matches a literal period.
- `|` inside a regex expresses OR.
- `*` in the `search` language is a wildcard; it is not the regex `.*` construct.
- SPL2 `WHERE` uses `LIKE` / `like()` with `%` and `_` for expression-based wildcard matching.
- `IN` is preferable when you simply have a short list of known values.
- `CASE()` provides case-sensitive matching for `search`.
- `TERM()` provides special indexed-term matching for values containing minor segmenters.
- `regex` filters; `rex` extracts or replaces; `match()` evaluates a regex as TRUE/FALSE.
- Search debugging should distinguish a bad regex from a missing/unextracted field.
- Current Splunk pipeline regex documentation uses PCRE2, so old RE2 pipeline examples need version checking.

The central mental model is:

```text
                     TEXT SEARCH REQUIREMENT
                              |
                 +------------+-------------+
                 |                          |
            Known value                Pattern
                 |                          |
           +-----+-----+              +-----+------+
           |           |              |            |
        Exact         List        Simple        Complex
           |           |          wildcard       regex
           |           |              |            |
           v           v              v            v
         field=      IN(...)        * / LIKE     regex / match()

                           |
                           v
                    Need to extract?
                           |
                           v
                          rex
```

The next chapter can now move into **field extraction in practice**, where we will work directly with `_raw` and learn how regex becomes an extraction tool rather than only a filtering tool. That will give us the bridge into `rex`, search-time extraction, `props.conf`, `transforms.conf`, indexed extraction, and eventually ingest-time filtering, routing, and masking.

---

# Official Splunk Documentation Basis

This chapter is based on the current official Splunk documentation relevant to Splunk Enterprise 10.4 and current SPL2 behavior, especially:

- Splunk Enterprise 10.4 Search Reference — `regex` command
- Splunk Enterprise 10.4 Search Reference — `rex` command
- Splunk Enterprise Search Manual — SPL and regular expressions
- Splunk Enterprise SPL2 Search Manual — SPL2 and regular expressions
- Splunk Enterprise SPL2 Search Reference — `match()` and comparison/conditional functions
- Splunk Enterprise SPL2 Search Manual — Wildcards
- Splunk Search Manual — `CASE()` and `TERM()`
- Splunk Search Manual — Wildcards
