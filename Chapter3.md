# Splunk SPL + SPL2

# Chapter 3 — Text Searching, Wildcards, and Regex

This chapter is important because text searching is one of the most common things you do during SOC investigations.

You will constantly need to answer questions such as:

- Did the command line contain `powershell`?
- Did the process name start with `win`?
- Did the file name end with `.exe`?
- Does a field contain a particular word?
- Does a field match a specific pattern?
- Can I search for several variations of the same command?
- Do I need an exact value, a wildcard, or a regular expression?
- Should I use `search`, `where`, `regex`, `match()`, `like()`, or `rex`?

The goal of this chapter is **not to memorize a collection of operators**.

Instead, you will learn how Splunk represents each type of text-search requirement and how the same requirement is expressed in traditional SPL and SPL2.

A very important rule for this chapter is:

> **Do not translate LogScale or KQL syntax directly into Splunk. Translate the investigation requirement into the native Splunk operation.**

---

# 3.1 What Does "Text Search" Actually Mean?

Suppose an event contains this command line:

```text
powershell.exe -NoProfile -ExecutionPolicy Bypass
```

You might want to search for:

```text
powershell
```

But there are several different questions you could ask.

### Question 1

Does the command line contain the text `powershell` anywhere?

### Question 2

Does the command line contain `powershell` as a complete word?

### Question 3

Does the command line start with `powershell`?

### Question 4

Does the command line end with `.exe`?

### Question 5

Does the command line match a more complicated pattern?

### Question 6

Does the field match one of several possible values?

These are different search requirements.

Therefore the first skill is:

```text
Investigation requirement
        ↓
Decide what kind of match is required
        ↓
Exact value?
Wildcard?
LIKE pattern?
Regex?
        ↓
Choose the appropriate Splunk command/function
```

Splunk provides several mechanisms for these cases, and they are not interchangeable. The `search` command, `where` command, `regex` command, `rex` command, and functions such as `match()` and `like()` have different purposes.

---

# 3.2 The Most Important Difference From the Previous Chapter

In earlier examples, you may have seen a search such as:

### Traditional SPL

```spl
index=windows EventCode=4625
```

### SPL2

```spl2
FROM windows
WHERE EventCode=4625
```

For text searching, however, you must understand that SPL and SPL2 have **multiple search mechanisms**.

For example:

### Traditional SPL

```spl
index=windows
| search CommandLine=powershell*
```

or:

```spl
index=windows
| regex CommandLine="(?i)powershell"
```

or:

```spl
index=windows
| where match(CommandLine, "(?i)powershell")
```

### SPL2

```spl2
search index=windows CommandLine="powershell*"
```

or:

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell")
```

These are not simply different spellings of one operation.

The first important decision is:

> **Am I performing a search predicate, an expression-based filter, a regex filter, or regex-based extraction?**

That decision becomes increasingly important as your searches become more advanced.

---

# 3.3 `search` — The Basic Text and Field-Value Search

Traditional SPL has a `search` command.

For example:

```spl
index=windows
| search CommandLine=powershell
```

At the beginning of the traditional SPL search, `search` is normally implied:

```spl
index=windows CommandLine=powershell
```

is effectively:

```spl
search index=windows CommandLine=powershell
```

Splunk documents that when a search pipeline starts with `search`, it retrieves matching events from indexes. When `search` occurs later in the pipeline, it filters the results that have already been produced.

SPL2 makes an important syntax distinction: when you explicitly use the `search` command, you write the word `search`.

### SPL

```spl
index=windows CommandLine=powershell
```

### SPL2

```spl2
search index=windows CommandLine=powershell
```

This is one of the easiest places to accidentally mix the languages.

---

# 3.4 Keyword Search vs Field-Value Search

A `search` can look for a general term:

### SPL

```spl
index=windows powershell
```

This searches event content for the term.

Or you can specify a field:

```spl
index=windows CommandLine=powershell
```

The second form says:

> Search the `CommandLine` field for the value/pattern.

SPL2 has the corresponding forms:

```spl2
search index=windows powershell
```

and:

```spl2
search index=windows CommandLine=powershell
```

The `search` command is therefore useful for ordinary keyword searching, phrases, field-value pairs, Boolean expressions, wildcards, and other search expressions.

---

# 3.5 Exact Value Search

Suppose the event contains:

```text
FileName=powershell.exe
```

and you want the exact value.

### SPL

```spl
index=windows FileName="powershell.exe"
```

### SPL2 — `search`

```spl2
search index=windows FileName="powershell.exe"
```

The concept is:

```text
Field
  ↓
FileName

Value
  ↓
powershell.exe
```

Do not confuse exact value matching with regex matching.

For example:

```spl
FileName="powershell.exe"
```

and:

```spl
| regex FileName="(?i)powershell"
```

are different searches.

The first is a field/value search.

The second is a regex filter.

---

# 3.6 Search Is Usually Case-Insensitive

A very important current Splunk behavior is that the `search` command is case-insensitive for keyword searches and field-value values.

For example:

```spl
index=windows user=Administrator
```

can match values with different capitalization such as:

```text
Administrator
administrator
ADMINISTRATOR
```

The field name itself is case-sensitive, so:

```text
User
```

and:

```text
user
```

are not necessarily the same field name.

SPL2 follows the same behavior when you use its `search` command.

This is different from expression-based comparisons such as `where`, which have different case-sensitivity behavior.

This difference is important enough to remember:

```text
search
    ↓
case-insensitive field-value search

where / expressions
    ↓
expression semantics apply
```

---

# 3.7 `CASE()` — Case-Sensitive Search

By default:

```spl
search host=DC01
```

is not a case-sensitive field-value search.

If you specifically need a case-sensitive search, Splunk provides:

```spl
search host=CASE(DC01)
```

SPL2 also supports `CASE()` in the `search` command.

Think:

```text
Normal search
    ↓
Case-insensitive

CASE()
    ↓
Case-sensitive term/value matching
```

Do not confuse:

```text
CASE()
```

with:

```text
match()
```

`CASE()` is a `search` directive for case-sensitive term matching.

`match()` is a regex evaluation function.

---

# 3.8 `TERM()` — Searching a Complete Indexed Term

Another advanced search feature that fits naturally into this chapter is:

```text
TERM()
```

This is especially useful when the term contains characters that Splunk treats as **minor segmenters** [minor segmenters: characters Splunk can use to split indexed text into smaller searchable terms], such as periods.

An IP address is a classic example:

```text
127.0.0.1
```

A naive keyword search can be affected by how the value was segmented when indexed.

For example:

```spl
search 127.0.0.1
```

can be interpreted as separate terms.

When the value is a complete indexed term and is bounded appropriately, you can use:

```spl
search TERM(127.0.0.1)
```

The purpose is not to perform regex matching.

It is to tell Splunk:

> Treat the content inside `TERM()` as one indexed term.

SPL2's `search` command also supports `TERM()`.

This becomes useful when hunting for:

```text
IP addresses
hostnames
file names
versions
other values containing punctuation
```

Do not use `TERM()` as a replacement for regex. They solve different problems.

---

# 3.9 Wildcards

Wildcards are one of the most important areas where you must understand the difference between `search` and expression-based filtering.

In search expressions, Splunk uses:

```text
*
```

to match zero or more characters.

For example:

```spl
index=windows FileName="powershell*"
```

can match:

```text
powershell.exe
powershell_ise.exe
powershell7.exe
```

Traditional SPL search:

```spl
index=windows FileName="powershell*"
```

SPL2 `search`:

```spl2
search index=windows FileName="powershell*"
```

The `*` is a **search wildcard** here. It is not the same thing as regex `.*`.

---

# 3.10 Wildcard `*` vs Regex `.*`

This distinction is essential.

### Search wildcard

```text
powershell*
```

means:

```text
powershell
followed by zero or more characters
```

### Regex

```text
powershell.*
```

means:

```text
powershell
followed by any character
zero or more times
```

They look similar, but they belong to different matching systems.

Think:

```text
Search syntax
    *
    ↓
Wildcard

Regex syntax
    .*
    ↓
Any character, zero or more times
```

Do not automatically substitute one for the other.

---

# 3.11 Important Wildcard Performance Rule

Splunk's current documentation recommends avoiding wildcard searches that begin with `*`.

For example:

```spl
CommandLine="*powershell"
```

can be expensive because Splunk may need to inspect many possible values before determining whether they end with `powershell`.

A more specific search such as:

```spl
CommandLine="powershell*"
```

is generally more efficient when the requirement really is "starts with PowerShell."

The same general principle applies to SPL2 `search`.

Also avoid broad searches such as:

```text
*
```

when you already know more specific search criteria.

The more specific your search is, the less unnecessary data Splunk needs to consider.

---

# 3.12 SPL2 Introduces Another Wildcard Model

This is one of the most important additions for this chapter.

In SPL2, the wildcard depends on the command.

For the SPL2 `search` command and many other commands, the traditional:

```text
*
```

wildcard is used.

However, in an SPL2 `WHERE` clause and the `where` command, the recommended pattern-matching mechanism is `LIKE`.

For example:

```spl2
FROM windows
WHERE FileName LIKE "powershell%"
```

Here:

```text
%
```

means:

> Match any number of characters.

And:

```text
_
```

means:

> Match exactly one character.

This gives us an important SPL2 distinction:

```text
SPL2 search
    ↓
*

SPL2 WHERE / where
    ↓
LIKE
    ↓
%
_
```

---

# 3.13 `LIKE` in SPL2

The SPL2 expression:

```spl2
FROM windows
WHERE FileName LIKE "powershell%"
```

performs pattern matching.

The same idea can be written with the `like()` function:

```spl2
FROM windows
WHERE like(FileName, "powershell%")
```

The `LIKE` predicate and `like()` function are closely related.

The wildcard meanings are:

```text
%  → zero or more characters
_  → one character
```

Example:

```spl2
WHERE host LIKE "DC%"
```

could match:

```text
DC01
DC02
DC-Backup
DCServer
```

`LIKE` is useful when you need wildcard pattern matching inside an expression.

---

# 3.14 `LIKE` Is Case-Sensitive

This is an important difference from the normal `search` command.

Splunk documents the `like()` function as case-sensitive.

So:

```spl2
WHERE like(FileName, "powershell%")
```

does not automatically mean:

```text
PowerShell.exe
POWERSHELL.exe
```

will match.

For case-insensitive matching, normalize the value first or use regex with an appropriate inline regex option when regex is the better tool.

For example:

```spl2
FROM windows
| eval lower_name=lower(FileName)
| WHERE lower_name LIKE "powershell%"
```

This gives you a useful mental model:

```text
search
    ↓
normally case-insensitive

like()
LIKE
    ↓
case-sensitive

match()
regex
    ↓
regex semantics apply
```

---

# 3.15 Exact Match vs Wildcard vs Regex

This is one of the most important decision points in the entire chapter.

Suppose the requirement is:

> Find `powershell.exe` exactly.

Use:

```spl
FileName="powershell.exe"
```

or:

```spl2
search FileName="powershell.exe"
```

Suppose the requirement is:

> Find values beginning with `powershell`.

Use a search wildcard:

```spl
FileName="powershell*"
```

or SPL2:

```spl2
search FileName="powershell*"
```

Suppose the requirement is:

> Use an expression wildcard pattern.

Use SPL2:

```spl2
WHERE FileName LIKE "powershell%"
```

Suppose the requirement is:

> Find `powershell` anywhere in the value with a regex.

Use:

```spl
| regex FileName="(?i)powershell"
```

or:

```spl
| where match(FileName, "(?i)powershell")
```

The important thing is:

```text
Exact value
    ↓
field="value"

Search wildcard
    ↓
field="value*"

SPL2 LIKE
    ↓
field LIKE "value%"

Regex
    ↓
match(field, "regex")
or regex field="regex"
```

---

# 3.16 Regular Expressions

Now we reach the most powerful pattern-matching mechanism in Splunk:

```text
Regular expressions
```

Splunk regular expressions use Perl-compatible regular expression behavior.

Regex is used in several Splunk operations, including:

```text
regex
rex
match()
replace()
```

and in ingestion/data-processing scenarios.

Regex therefore matters not only for searches but also later when we learn:

- field extraction
- masking
- routing
- filtering
- transformations
- Ingest Processor
- Edge Processor
- `props.conf`
- `transforms.conf`

---

# 3.17 `regex` Command

Traditional SPL has a dedicated command:

```spl
regex
```

Its purpose is to filter results based on a regular expression.

For example:

```spl
index=windows
| regex CommandLine="(?i)powershell"
```

This means:

> Keep events where the `CommandLine` field matches the regex.

The `regex` command can also use:

```spl
regex CommandLine!="(?i)powershell"
```

to keep results that do not match the regex.

If you do not specify a field, `regex` operates on `_raw`:

```spl
index=windows
| regex "(?i)powershell"
```

So:

```text
regex command
    ↓
Filter events using regex
```

This is different from `rex`.

---

# 3.18 `regex` vs `rex`

This distinction is extremely important.

### `regex`

Used to **filter** events.

```spl
index=windows
| regex CommandLine="(?i)powershell"
```

Think:

```text
Does this event match my regex?
        ↓
YES → keep
NO  → remove
```

### `rex`

Used to **extract fields** from a regex pattern, or to perform sed-based replacement.

Example:

```spl
index=windows
| rex "user=(?<username>\S+) src_ip=(?<src_ip>\S+)"
```

Now Splunk creates fields such as:

```text
username
src_ip
```

The regex is being used to create fields.

Think:

```text
regex
  ↓
FILTER

rex
  ↓
EXTRACT / REPLACE
```

Splunk's documentation explicitly distinguishes these two commands.

---

# 3.19 `match()` — Regex Inside an Expression

You can also use regex through the `match()` evaluation function.

Traditional SPL:

```spl
index=windows
| where match(CommandLine, "(?i)powershell")
```

SPL2:

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell")
```

`match()` returns a Boolean result [Boolean: a value that is either true or false].

Conceptually:

```text
match(field, regex)
        ↓
Does the regex match?
        ↓
TRUE / FALSE
```

This is especially useful when the regex is part of a larger logical expression.

For example:

```spl
index=windows
| where match(CommandLine, "(?i)powershell") AND user="administrator"
```

SPL2:

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell")
  AND user="administrator"
```

---

# 3.20 `match()` vs `regex`

Both use regex, but their roles are different.

### `regex`

```spl
| regex CommandLine="(?i)powershell"
```

is a dedicated filtering command.

### `match()`

```spl
| where match(CommandLine, "(?i)powershell")
```

is a Boolean expression.

That difference becomes particularly useful when you need to combine regex with other expressions.

---

# 3.21 Case-Insensitive Regex

A major difference from LogScale is that you should not simply copy:

```text
/powershell/i
```

into a Splunk query.

In Splunk regex expressions, a common way to request case-insensitive matching is the inline regex modifier:

```text
(?i)
```

For example:

```spl
| regex CommandLine="(?i)powershell"
```

or:

```spl
| where match(CommandLine, "(?i)powershell")
```

This can match:

```text
powershell.exe
PowerShell.exe
POWERSHELL.EXE
```

So the basic Splunk mental model is:

```text
LogScale
/powershell/i

Splunk
(?i)powershell
```

Do not mix the syntaxes.

---

# 3.22 Regex Is Usually Unanchored Unless You Anchor It

A regex such as:

```text
(?i)powershell
```

searches for that pattern within the string.

So it can match:

```text
powershell.exe
cmd.exe /c powershell
mypowershellscript.exe
```

If you want the regex to describe the entire field, you need anchors.

This leads to two critical regex characters:

```text
^
$
```

---

# 3.23 `^` — Start of String

The regex:

```text
^powershell
```

means:

> The value must begin with `powershell`.

Example:

```spl
index=windows
| regex CommandLine="(?i)^powershell"
```

These can match:

```text
powershell.exe -enc AAAA
powershell -nop
```

But:

```text
cmd.exe /c powershell
```

does not match because it begins with:

```text
cmd.exe
```

Mental model:

```text
^
↓
START HERE
```

---

# 3.24 `$` — End of String

The regex:

```text
\.exe$
```

means:

> The value must end with `.exe`.

Example:

```spl
index=windows
| regex FileName="(?i)\.exe$"
```

This matches:

```text
powershell.exe
cmd.exe
chrome.exe
```

but not:

```text
powershell.exe.bak
```

because the string does not end at `.exe`.

Mental model:

```text
$
↓
END HERE
```

---

# 3.25 Why Do We Escape `.`?

In regex:

```text
.
```

has a special meaning:

> Match any character.

For example:

```text
a.b
```

could match:

```text
aab
axb
a1b
a-b
```

But a Windows extension contains a literal period:

```text
.exe
```

Therefore we write:

```text
\.exe
```

The backslash tells the regex engine that the period should be treated literally.

So:

```text
\.exe$
```

means:

```text
literal .
+
exe
+
end of string
```

---

# 3.26 What Does `.*` Mean?

You will see:

```text
.*
```

constantly in regex.

It contains two pieces.

```text
.
```

means:

> Any character.

And:

```text
*
```

means:

> Zero or more occurrences.

Together:

```text
.*
```

means roughly:

> Any number of characters.

For example:

```text
powershell.*-enc
```

can match:

```text
powershell -enc AAAA
powershell.exe -enc AAAA
powershell -NoProfile -enc AAAA
```

because the `.*` allows content between:

```text
powershell
```

and:

```text
-enc
```

However, broad `.*` patterns can make a search less precise. Do not use it automatically.

---

# 3.27 Building Regex From the Requirement

Use this mental model:

| Requirement | Regex |
|---|---|
| Find `powershell` anywhere | `powershell` |
| Find `powershell` at the start | `^powershell` |
| Find `.exe` at the end | `\.exe$` |
| Find PowerShell then later `-enc` | `powershell.*-enc` |
| Match a literal period | `\.` |
| Find a complete word | `\bpowershell\b` |
| Match one of two values | `(powershell|pwsh)` |

The point is not to memorize the table.

The point is:

```text
Requirement
    ↓
Describe the pattern
    ↓
Encode the pattern
```

---

# 3.28 Word Boundaries — `\b`

Suppose:

```text
CommandLine=mypowershellscript.exe
```

and you search:

```text
powershell
```

the substring exists inside the larger word.

If your requirement is:

> Find `powershell` as a complete word.

you can use:

```text
\bpowershell\b
```

For example:

```spl
index=windows
| regex CommandLine="(?i)\bpowershell\b"
```

or:

```spl
index=windows
| where match(CommandLine, "(?i)\bpowershell\b")
```

The `\b` indicates a word boundary [word boundary: a position between a word character and a non-word character, according to the regex engine].

Think:

```text
powershell.exe
^^^^^^^^^^
complete word before punctuation

mypowershellscript.exe
   ^^^^^^^^^^
inside a larger word
```

---

# 3.29 Multiple Alternatives With `|`

Inside a regex, the pipe character means OR.

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

To require `.exe` after either alternative:

```text
(powershell|pwsh)\.exe
```

Traditional SPL:

```spl
index=windows
| regex FileName="(?i)(powershell|pwsh)\.exe"
```

Or with `match()`:

```spl
index=windows
| where match(FileName, "(?i)(powershell|pwsh)\.exe")
```

SPL2:

```spl2
FROM windows
WHERE match(FileName, "(?i)(powershell|pwsh)\.exe")
```

---

# 3.30 Why Parentheses Matter

Compare:

```text
powershell|pwsh\.exe
```

with:

```text
(powershell|pwsh)\.exe
```

The second clearly means:

```text
either powershell
OR pwsh

then .exe
```

The first has different operator precedence [operator precedence: the rules that determine which part of an expression is interpreted together first].

Therefore, when using alternatives, group them clearly:

```text
(powershell|pwsh)\.exe
```

---

# 3.31 Character Classes

Square brackets define a character class.

Examples:

```text
[abc]
```

means one character from:

```text
a
b
c
```

And:

```text
[0-9]
```

means one digit.

You may also encounter:

```text
\d
```

for a digit,

```text
\w
```

for a word character, and:

```text
\s
```

for whitespace.

---

# 3.32 Regex Quantifiers: `*`, `+`, `?`

You should understand the basic difference.

```text
*
```

means:

> Zero or more.

```text
+
```

means:

> One or more.

```text
?
```

means:

> Zero or one in the usual quantifier sense.

For example:

```text
-enc\s+
```

means:

```text
-enc
+
one or more whitespace characters
```

---

# 3.33 Regex Is About Describing a Pattern

This is the mindset you should develop.

Don't think:

> "I need to remember this strange syntax."

Instead think:

> "What pattern describes the activity I'm hunting?"

For example:

```text
Requirement
Find command lines containing powershell.

Pattern
powershell
```

```text
Requirement
Find the complete word powershell.

Pattern
\bpowershell\b
```

```text
Requirement
Find command lines starting with powershell.

Pattern
^powershell
```

```text
Requirement
Find values ending in .exe.

Pattern
\.exe$
```

```text
Requirement
Find powershell followed later by -enc.

Pattern
powershell.*-enc
```

---

# 3.34 A Real SOC Example — PowerShell Encoded Commands

One common investigation is looking for PowerShell commands using encoded commands.

A typical command line may contain:

```text
powershell.exe -enc AAAABBBBCCCC
```

Traditional SPL:

```spl
index=windows
| regex CommandLine="(?i)powershell.*-(enc|encodedcommand)"
```

Or with `match()`:

```spl
index=windows
| where match(CommandLine, "(?i)powershell.*-(enc|encodedcommand)")
```

SPL2:

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell.*-(enc|encodedcommand)")
```

This expresses:

```text
Process-related data
      ↓
PowerShell
      ↓
Encoded-command indicator
```

---

# 3.35 Be Careful With Broad Regex

A regex such as:

```text
powershell.*-enc
```

may match unexpected strings because `.*` is broad.

Always ask:

> "Is my pattern specific enough for the investigation?"

Sometimes a more carefully designed regex is better than adding `.*` everywhere.

Don't use regex just because it is powerful.

Use the smallest pattern that accurately describes the requirement.

---

# 3.36 Exact Match vs Word Match vs Substring Match

This is worth remembering.

### Exact value

```spl
FileName="powershell.exe"
```

The field value should equal the requested value under search semantics.

### Substring / regex search

```spl
| where match(CommandLine, "(?i)powershell")
```

Find `powershell` within the value.

### Whole-word regex

```spl
| where match(CommandLine, "(?i)\bpowershell\b")
```

Find `powershell` as a word.

### Starts with

```spl
| where match(CommandLine, "(?i)^powershell")
```

The value must start with `powershell`.

### Ends with

```spl
| where match(FileName, "(?i)\.exe$")
```

The value must end with `.exe`.

These are different hunting requirements.

---

# 3.37 `IN` — Several Known Values

Sometimes regex is not necessary.

Suppose you want:

```text
cmd.exe
powershell.exe
pwsh.exe
```

Traditional SPL:

```spl
index=windows FileName IN ("cmd.exe","powershell.exe","pwsh.exe")
```

SPL2 `search`:

```spl2
search index=windows FileName IN ("cmd.exe","powershell.exe","pwsh.exe")
```

This is clearer than a long chain of OR expressions when the possibilities are known values.

Do not use regex just because regex is available.

---

# 3.38 Wildcards With `IN`

The `search` language also supports wildcard patterns inside `IN`.

For example:

```spl
index=windows status IN (4*,5*)
```

This can match status values beginning with `4` or `5`.

SPL2 `search` supports the same search-style wildcard behavior.

This gives you three useful approaches:

```text
One exact value
    ↓
field="value"

Several exact values
    ↓
field IN (...)

Several patterns
    ↓
field IN ("prefix*","other*")
```

---

# 3.39 `NOT` vs `!=`

This is another important Splunk detail.

Consider:

```spl
search fieldA!="value2"
```

This is a field/value comparison.

Whereas:

```spl
search NOT fieldA="value2"
```

is the negation of the search condition.

Splunk documents that these can differ when the field is missing.

Therefore:

```text
!=
    ↓
Field/value comparison

NOT
    ↓
Negation of the search condition
```

This becomes especially important when investigating fields that may not be extracted consistently.

---

# 3.40 Missing Fields

You may sometimes need to ask:

> Which events do not contain this field?

A common search technique is:

```spl
| search NOT field=*
```

This uses the wildcard to test whether the field has a value in search semantics, so the negation can identify events where that field is absent/null.

Do not casually replace it with:

```spl
field!=*
```

because Splunk documents different behavior for `NOT field=*` and `field!=*`.

---

# 3.41 `where` — Expression-Based Filtering

Traditional SPL also has:

```spl
where
```

For example:

```spl
index=windows
| where EventCode=4625
```

The `where` command evaluates an expression and returns only events where the expression evaluates to true.

It is more powerful than a basic `search` predicate because it can work with expressions and compare fields.

For example:

```spl
index=network
| where src_port=dest_port
```

This compares one field against another field.

The `search` command does not interpret the right-hand field name as another field in the same way.

---

# 3.42 Search vs `where`

Think of them like this.

### `search`

Best suited for:

```text
Index/event retrieval
Keyword searches
Field=value predicates
Wildcards
IN
CASE
TERM
Basic Boolean search syntax
```

### `where`

Best suited for:

```text
Boolean expressions
Field-to-field comparison
Evaluation functions
Arithmetic
LIKE
match()
Other expression-based logic
```

For example:

```spl
index=network
| search src_port=443
```

asks for a field/value search.

But:

```spl
index=network
| where src_port=dest_port
```

asks for an expression comparing two fields.

That difference is fundamental.

---

# 3.43 SPL2 `WHERE` Clause

SPL2 puts the same expression-based filtering idea directly into the `FROM` command:

```spl2
FROM network
WHERE src_port=dest_port
```

You can also use the standalone SPL2 `where` command in a pipeline:

```spl2
FROM network
| where src_port=dest_port
```

The SPL2 documentation describes the `where` command as equivalent to the `WHERE` clause in the `from` command.

---

# 3.44 `AND`, `OR`, and Parentheses

When you combine conditions, pay attention to precedence [precedence: the order in which expressions are evaluated].

For example:

```spl
index=windows
| search user=administrator OR user=system EventCode=4688
```

Do not rely on memory about how a complex Boolean expression will be grouped.

Use parentheses when the intended logic matters:

```spl
index=windows
| search (user=administrator OR user=system) EventCode=4688
```

For `where`/expression logic, parentheses are especially important.

SPL2 documents different evaluation precedence between `search` and `where` expressions, so explicit parentheses are the safest habit for complicated Boolean logic.

---

# 3.45 Regex `|` vs Search Pipeline `|`

This is a common beginner mistake.

In a Splunk search:

```text
|
```

normally means:

```text
pass results to the next command
```

But inside a regex:

```text
|
```

means:

```text
OR
```

For example:

```text
(powershell|pwsh)
```

contains regex OR.

Because the same character is also Splunk's pipeline separator, the regex expression must be quoted as one expression so that the pipe remains part of the regex.

---

# 3.46 Backslashes and Escaping

Regex uses:

```text
\
```

as an escape character.

For example:

```text
\.
```

means a literal period.

But Splunk search syntax also has its own handling of backslashes and quotation marks.

This becomes especially noticeable with Windows paths.

For example:

```text
C:\Windows\System32
```

may require additional escaping depending on where the expression is written.

The important lesson is:

```text
There are two things to understand:

1. Splunk string/search escaping
2. Regex escaping
```

Do not assume that every backslash is consumed by the same layer.

---

# 3.47 PCRE and PCRE2

Splunk regular-expression support is based on the Perl-compatible regular-expression family.

For current Splunk search documentation, regex is described using PCRE behavior.

For current Splunk data-processing pipelines, such as Edge Processor and Ingest Processor, Splunk documents PCRE2 as the current regex engine.

This matters because older examples online may use RE2 syntax.

Therefore:

```text
Older pipeline examples
        ↓
May use RE2

Current pipelines
        ↓
PCRE2
```

Do not blindly copy old pipeline regex examples without checking which engine the documentation refers to.

---

# 3.48 `rex` — Regex Extraction

The `rex` command is extremely important and deserves to be introduced here even though we will study extraction in greater depth later.

Suppose the event contains:

```text
user=john src_ip=10.10.10.5
```

You can extract the values using named capture groups.

Traditional SPL:

```spl
index=security
| rex "user=(?<username>\S+) src_ip=(?<src_ip>\S+)"
```

Now Splunk creates fields such as:

```text
username
src_ip
```

This is regex being used to **extract information** rather than merely filter events.

---

# 3.49 `rex` Against a Specific Field

Instead of the entire raw event, you can specify a field:

```spl
index=security
| rex field=CommandLine "user=(?<username>\S+)"
```

In SPL2, the field option is written before the regex expression:

```spl2
FROM security
| rex field=CommandLine "user=(?<username>\S+)"
```

Current SPL2 documentation also places `max_match` and `offset_field` before the regex expression.

---

# 3.50 `rex` Can Also Perform Sed Replacement

`rex` is not only for extraction.

It also supports:

```text
mode=sed
```

for replacement/substitution.

For example:

```spl2
| rex field=password mode=sed "s/Secret123/********/g"
```

This is useful for string replacement and becomes relevant to our later masking study.

The mental model is:

```text
regex
   ↓
match / filter

rex
   ↓
extract

rex mode=sed
   ↓
replace
```

---

# 3.51 Search-Time Masking vs Ingest-Time Masking

Do not confuse a search-time operation such as:

```spl2
| rex field=password mode=sed "s/.*/********/g"
```

with permanent ingestion-time masking.

A search-time transformation affects the result being processed by the search. It does not by itself rewrite the original indexed event.

Ingest-time masking is a separate processing stage that happens before or during delivery to a destination.

That distinction will become important when we study:

```text
props.conf
transforms.conf
SEDCMD
INGEST_EVAL
Ingest Processor
Edge Processor
```

---

# 3.52 A Practical Decision Process

When searching a string, ask these questions in order.

### Question 1 — Do I need an exact value?

Use:

```text
field="value"
```

### Question 2 — Do I need several known values?

Use:

```text
field IN ("value1","value2")
```

### Question 3 — Do I need a simple prefix/suffix pattern?

Consider a search wildcard:

```text
field="prefix*"
```

or, in SPL2 expression filtering:

```text
field LIKE "prefix%"
```

### Question 4 — Do I need a complex pattern?

Use regex:

```text
match(field,"regex")
```

or the traditional SPL `regex` command when a dedicated regex filter is appropriate.

### Question 5 — Do I need to extract data from text?

Use:

```text
rex
```

This decision process is more useful than memorizing a table of operators.

---

# 3.53 A Real Investigation Example

Imagine your SOC receives an alert:

> "Investigate possible encoded PowerShell execution."

Start with a broad but relevant process search.

### Traditional SPL

```spl
index=windows
| search FileName="powershell*"
```

Then look for encoded-command indicators:

```spl
index=windows
| search FileName="powershell*"
| where match(CommandLine, "(?i)-(enc|encodedcommand)")
```

Or combine the pattern:

```spl
index=windows
| where match(CommandLine, "(?i)powershell.*-(enc|encodedcommand)")
```

### SPL2

```spl2
FROM windows
WHERE match(CommandLine, "(?i)powershell.*-(enc|encodedcommand)")
```

Now the query expresses the actual investigation:

```text
Process event/data
        ↓
PowerShell
        ↓
Encoded-command parameter
```

---

# 3.54 Why Search Method Matters for Performance

Not all matching methods have the same search behavior.

A highly specific search such as:

```spl
index=windows EventCode=4625 user=administrator
```

gives Splunk useful indexed/searchable information early.

A broad regex over `_raw` such as:

```spl
index=windows
| regex "(?i)powershell"
```

may require more work because the regex is being applied to event text.

This does not mean regex is bad.

It means you should narrow the dataset and use exact/searchable predicates where you can before using expensive pattern matching.

The same principle matters later when we build production SOC detections and ingestion pipelines.

---

# 3.55 Common Beginner Mistakes

## Mistake 1 — Copying LogScale regex syntax

Do not automatically write:

```text
/powershell/i
```

in Splunk.

Use a Splunk-compatible regex expression such as:

```spl
| regex CommandLine="(?i)powershell"
```

or:

```spl
| where match(CommandLine, "(?i)powershell")
```

---

## Mistake 2 — Treating `*` and `.*` as the same

Search wildcard:

```text
powershell*
```

Regex:

```text
powershell.*
```

They belong to different matching systems.

---

## Mistake 3 — Using regex when `=` is enough

If you know:

```text
FileName=powershell.exe
```

there is usually no reason to write an elaborate regex.

Use:

```spl
FileName="powershell.exe"
```

---

## Mistake 4 — Using `search` when you need field-to-field comparison

Do not assume this means what it looks like:

```spl
| search src_port=dest_port
```

The `search` command interprets the right side according to search-expression semantics rather than as the name of another field.

Use:

```spl
| where src_port=dest_port
```

when you want a field-to-field expression.

---

## Mistake 5 — Forgetting `NOT` vs `!=`

These can produce different results when fields are missing.

Know whether you mean:

```text
field exists and is not value
```

or:

```text
the event should not satisfy this condition
```

---

## Mistake 6 — Using a leading wildcard unnecessarily

Avoid:

```text
*something
```

when a more specific search can express the requirement.

Prefix wildcards can have a performance cost.

---

## Mistake 7 — Assuming SPL2 `WHERE` uses `*`

For SPL2 expression filtering:

```spl2
WHERE FileName LIKE "powershell%"
```

is the expression-based wildcard form.

Do not confuse it with:

```spl2
search FileName="powershell*"
```

---

## Mistake 8 — Confusing `regex` and `rex`

Remember:

```text
regex
    ↓
FILTER

rex
    ↓
EXTRACT / REPLACE
```

---

# 3.56 The Important Tools to Remember

| Tool | Main purpose |
|---|---|
| `search` | Retrieve/filter events using search expressions |
| `where` | Filter with Boolean/evaluation expressions |
| `regex` | Filter events with regex in traditional SPL |
| `match()` | Return true/false from a regex expression |
| `rex` | Extract fields or perform sed replacement |
| `LIKE` / `like()` | Pattern matching in SPL2 expressions; available in evaluation contexts |
| `IN` | Match one field against multiple values |
| `CASE()` | Force case-sensitive `search` matching |
| `TERM()` | Treat indexed content as one term |

These tools overlap in some situations, but they are not interchangeable.

---

# 3.57 SPL vs SPL2 Cheat Sheet

| Requirement | Traditional SPL | SPL2 |
|---|---|---|
| Start a search | `index=windows` | `FROM windows` |
| Explicit search command | `search index=windows` | `search index=windows` |
| Exact field value | `FileName="cmd.exe"` | `search FileName="cmd.exe"` |
| Search wildcard | `FileName="powershell*"` | `search FileName="powershell*"` |
| Expression filter | `\| where ...` | `WHERE ...` or `\| where ...` |
| Regex filter | `\| regex field="pattern"` | Use `match()` in an expression |
| Regex Boolean test | `\| where match(field,"regex")` | `WHERE match(field,"regex")` |
| Regex extraction | `\| rex ...` | `\| rex field=... ...` |
| Several values | `field IN (...)` | `search field IN (...)` |
| Expression wildcard | `\| where like(field,"a%")` where supported | `WHERE field LIKE "a%"` |
| Case-sensitive search | `CASE()` | `CASE()` with `search` |
| Indexed-term matching | `TERM()` | `TERM()` with `search` |

The important point is that SPL2 contains both a **search language** and a more explicit **expression language**, and those mechanisms have different semantics.

---

# 3.58 The Most Important Mental Model

At this point, think about text searching this way:

```text
                         SEARCH REQUIREMENT
                                │
                                ▼
                        What do I know?
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
         Exact value       Several values       Pattern
              │                 │                 │
              ▼                 ▼                 ▼
          field=value       field IN (...)      ┌─────────┐
                                                 │         │
                                                 ▼         ▼
                                             Wildcard    Regex
                                                 │         │
                                                 │    ┌────┴────┐
                                                 │    │         │
                                                 ▼    ▼         ▼
                                                *  match()   regex
                                                   /rex
```

And remember:

```text
search
    ↓
Search language

where
    ↓
Expression language

regex
    ↓
Regex filtering

match()
    ↓
Regex Boolean expression

rex
    ↓
Regex extraction/replacement

LIKE
    ↓
Expression-based wildcard pattern
```

---

# 3.59 Chapter Summary

By the end of this chapter, you should understand:

- Text searching is not one single operation in Splunk.
- Traditional SPL and SPL2 both provide `search` and expression-based filtering, but their syntax and semantics differ.
- A normal search field/value comparison is different from regex matching.
- `search` is useful for retrieving/filtering events and supports keyword search, field/value expressions, Boolean logic, wildcards, `IN`, `CASE()`, and `TERM()`.
- `where` evaluates Boolean expressions.
- `search` and `where` are not interchangeable.
- `regex` is a traditional SPL filtering command.
- `match()` performs regex matching as a Boolean expression.
- `rex` extracts fields or performs sed-style replacement.
- `*` is a search wildcard.
- In SPL2 expression filtering, `LIKE` uses `%` for multiple characters and `_` for one character.
- Regex uses constructs such as `.`, `*`, `+`, `?`, `^`, `$`, `\b`, `\`, `()`, `[]`, and `|`.
- `(?i)` is a common regex inline modifier for case-insensitive matching.
- `TERM()` helps when searching indexed terms containing minor segmenters such as periods.
- `CASE()` enables case-sensitive matching for `search`.
- `IN` is useful when you have several known values.
- `NOT` and `!=` are not always equivalent, especially when fields are missing.
- Prefix wildcards can hurt search performance, so be as specific as possible.
- Current Splunk pipelines use PCRE2, so older RE2 pipeline examples should not be copied blindly.

The most important lesson is:

> **Do not memorize syntax first. Identify the kind of match you need, then choose the Splunk operation that naturally expresses it.**

---

# What We Have Covered So Far

```text
CHAPTER 1
Understanding Splunk Events, Fields, and Data
        │
        ├── Event
        ├── index
        ├── host
        ├── source
        ├── sourcetype
        ├── _raw
        ├── _time
        └── Indexed vs search-time fields
        │
        ▼
CHAPTER 2
Writing Your First SPL + SPL2 Search
        │
        ├── Search structure
        ├── Pipelines
        ├── Filtering
        ├── AND / OR / NOT
        ├── Comparisons
        ├── WHERE
        └── Basic search logic
        │
        ▼
CHAPTER 3
Text Searching, Wildcards, and Regex
        │
        ├── search
        ├── where
        ├── regex
        ├── match()
        ├── rex
        ├── LIKE / like()
        ├── wildcards
        ├── IN
        ├── CASE()
        ├── TERM()
        └── Regex fundamentals
        │
        ▼
CHAPTER 4
Field Extraction and Transformation
```

---

# Official Splunk Documentation Used for This Chapter

This chapter follows the current official Splunk documentation for:

- Search command — Overview, syntax, examples, and usage
- Where command — Overview and syntax
- SPL and regular expressions
- About Splunk regular expressions
- Wildcards
- Comparison and conditional functions
- Predicate expressions
- `rex` command — Overview, syntax, and examples
- `CASE()` and `TERM()` search behavior
- Current regex behavior for SPL2 pipelines

For this course, current official Splunk documentation takes precedence over older blog posts, third-party tutorials, and examples written for older Splunk releases.
