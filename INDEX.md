# Index — Splunk SPL + SPL2 Complete Reference (v2.0)

## Quick Start & Navigation
- [README.md](README.md) — Overview, structure, quick navigation, evidence standard, labs
- [INDEX.md](INDEX.md) — This file: complete table of contents

---

## Chapter 1 — Foundations: Events, Fields, Data Model & Search Fundamentals
- [1.1 What Is an Event?](Chapter1.md#12-what-is-an-event)
- [1.2 Where Do Events Come From?](Chapter1.md#13-where-do-events-come-from)
- [1.3 What Is an Index?](Chapter1.md#14-what-is-an-index)
- [1.4 SPL vs SPL2: Same Data, Different Syntax](Chapter1.md#15-spl-vs-spl2-same-data-different-syntax)
- [1.5 Internal Fields: `_raw`, `_time`, `_indextime`, `_cd`, `_bkt`](Chapter1.md#16-internal-fields)
- [1.6 Default Fields: `index`, `host`, `source`, `sourcetype`](Chapter1.md#17-default-fields)
- [1.7 Extracted Fields: Search-Time vs Index-Time](Chapter1.md#18-extracted-fields-search-time-vs-index-time)
- [1.8 Indexed Fields vs Search-Time Fields](Chapter1.md#19-indexed-fields-vs-search-time-fields)
- [1.9 Field Categories Summary Table](Chapter1.md#110-field-categories-summary-table)
- [1.10 Ingestion-Time vs Search-Time Processing](Chapter1.md#111-ingestion-time-vs-search-time-processing)
- [1.11 Filtering, Routing, Masking: Three Different Problems](Chapter1.md#112-filtering-routing-masking-three-different-problems)
- [1.13 Search Fundamentals: Pipeline, `search` vs `where`, SPL2 Syntax](Chapter1.md#113-search-fundamentals-pipeline-search-vs-where-spl2-syntax)
- [1.14 `search` Command vs `where` Command: The Critical Difference](Chapter1.md#114-search-command-vs-where-command-the-critical-difference)
- [1.15 SPL2 Structured Syntax: Clauses in Fixed Order](Chapter1.md#115-spl2-structured-syntax-clauses-in-fixed-order)
- [1.16 Wildcards: `*` in `search` vs `%`/`_` in `LIKE`](Chapter1.md#116-wildcards--in-search-vs--in-like)
- [1.17 Regular Expressions: `regex` vs `match()` vs `rex`](Chapter1.md#117-regular-expressions-regex-vs-match-vs-rex)
- [1.18 Building a Search Step by Step](Chapter1.md#118-building-a-search-step-by-step)
- [1.19 Command Classification Primer (Preview)](Chapter1.md#119-command-classification-primer-preview)
- [1.20 SPL2 Compatibility Profiles](Chapter1.md#120-spl2-compatibility-profiles)
- [1.21 Evidence Standard](Chapter1.md#121-evidence-standard)
- [1.22 Labs (Hands-On Practice)](Chapter1.md#122-labs-hands-on-practice)
- [1.23 What's Next](Chapter1.md#123-whats-next)

---

## Chapter 2 — Command Classification Taxonomy
- [2.1 Why Classification Matters](Chapter2.md#21-why-classification-matters)
- [2.2 The Complete Taxonomy](Chapter2.md#22-the-complete-taxonomy)
- [2.3 Streaming Commands (Distributable) 🟢](Chapter2.md#23-streaming-commands-distributable)
- [2.4 Streaming Commands (Centralized) 🟡](Chapter2.md#24-streaming-commands-centralized)
- [2.5 Non-Streaming / Reporting Commands 🔴](Chapter2.md#25-non-streaming--reporting-commands)
- [2.6 Generating Commands (Dataset Processing) 🟣](Chapter2.md#26-generating-commands-dataset-processing)
- [2.7 Orchestrating Commands (Pipeline Control) 🟠](Chapter2.md#27-orchestrating-commands-pipeline-control)
- [2.8 Custom Command Types ⚪](Chapter2.md#28-custom-command-types)
- [2.9 Command Classification Reference Table](Chapter2.md#29-command-classification-reference-table)
- [2.10 How Classification Affects Performance](Chapter2.md#210-how-classification-affects-performance)
- [2.11 Distributed Streaming vs Centralized Streaming: Decision Guide](Chapter2.md#211-distributed-streaming-vs-centralized-streaming-decision-guide)
- [2.12 Pipeline Ordering Rules](Chapter2.md#212-pipeline-ordering-rules)
- [2.13 Indexer-to-Search-Head Handoff: Visual Execution Model](Chapter2.md#213-indexer-to-search-head-handoff-visual-execution-model)
- [2.14 Mode-Dependent Command Classification (Critical)](Chapter2.md#214-mode-dependent-command-classification-critical)
- [2.15 SPL2 Compatibility Profiles: Pipeline Command Availability](Chapter2.md#215-spl2-compatibility-profiles-pipeline-command-availability)
- [2.16 Decision Guide: Which Classification Do I Need?](Chapter2.md#216-decision-guide-which-classification-do-i-need)
- [2.17 What's Next](Chapter2.md#217-whats-next)

---

## Chapter 3 — SPL vs SPL2: Deep Comparison
- [3.1 Philosophy: Evolution, Not Replacement](Chapter3.md#31-philosophy-evolution-not-replacement)
- [3.2 Generating Commands: `search` vs `FROM`](Chapter3.md#32-generating-commands-search-vs-from)
- [3.3 Filtering: `search` vs `where` vs `WHERE` Clause](Chapter3.md#33-filtering-search-vs-where-vs-where-clause)
- [3.4 Pipeline Syntax: `|` vs Structured Clauses](Chapter3.md#34-pipeline-syntax--vs-structured-clauses)
- [3.5 Field Selection: `fields`/`table` vs `SELECT`](Chapter3.md#35-field-selection-fieldstable-vs-select)
- [3.6 Aggregation: `stats` vs `GROUP BY`/`SELECT`](Chapter3.md#36-aggregation-stats-vs-group-byselect)
- [3.7 Sorting: `sort` vs `ORDER BY`](Chapter3.md#37-sorting-sort-vs-order-by)
- [3.8 Limiting: `head` vs `LIMIT`/`OFFSET`](Chapter3.md#38-limiting-head-vs-limitoffset)
- [3.9 Wildcards: `*` vs `LIKE`/`like()`](Chapter3.md#39-wildcards--vs-like)
- [3.10 Regex: `regex` Command vs `match()` Function](Chapter3.md#310-regex-regex-command-vs-match-function)
- [3.11 Time Grouping: `bin`/`bucket` vs `span` in `stats`/`GROUP BY`](Chapter3.md#311-time-grouping-binbucket-vs-span-in-statsgroup-by)
- [3.12 Time Series: `timechart` in Both](Chapter3.md#312-time-series-timechart-in-both)
- [3.13 Streaming Commands: `streamstats`, `eventstats`, `autoregress`, `delta`](Chapter3.md#313-streaming-commands-streamstats-eventstats-autoregress-delta)
- [3.14 Pipeline-Only Commands: `route`, `thru`, `branch`, `if`](Chapter3.md#314-pipeline-only-commands-route-thru-branch-if)
- [3.15 SPL2 in Edge/Ingest Processor vs Search](Chapter3.md#315-spl2-in-edgeingest-processor-vs-search)
- [3.16 SPL2 Compatibility Profiles](Chapter3.md#316-spl2-compatibility-profiles)
- [3.17 Migration Decision Table](Chapter3.md#317-migration-decision-table)
- [3.18 What's Next](Chapter3.md#318-whats-next)

---

## Chapter 4 — Search Commands Reference & Time-Based Analysis
- [4.1 Classification Badges](Chapter4.md#41-classification-badges)
- [4.2 Search Commands Reference (Classified)](Chapter4.md#42-search-commands-reference-classified)
  - [4.2.1 Generating Commands 🟣](Chapter4.md#421-generating-commands)
  - [4.2.2 Filtering Commands 🟢](Chapter4.md#422-filtering-commands)
  - [4.2.3 Streaming Transform Commands 🟢](Chapter4.md#423-streaming-transform-commands)
  - [4.2.4 Streaming Stats Commands 🟢](Chapter4.md#424-streaming-stats-commands)
  - [4.2.5 Centralized Streaming Commands 🟡](Chapter4.md#425-centralized-streaming-commands)
  - [4.2.6 Reporting / Non-Streaming Commands 🔴](Chapter4.md#426-reporting--non-streaming-commands)
  - [4.2.7 Orchestrating Commands 🟠](Chapter4.md#427-orchestrating-commands)
  - [4.2.8 Dataset Processing Commands (SPL2)](Chapter4.md#428-dataset-processing-commands-spl2)
  - [4.2.9 Structure Commands 🟢](Chapter4.md#429-structure-commands)
  - [4.2.10 Lookup & Enrichment Commands](Chapter4.md#4210-lookup--enrichment-commands)
  - [4.2.11 Geospatial Commands 🔴](Chapter4.md#4211-geospatial-commands)
  - [4.2.12 Machine Learning Commands 🔴](Chapter4.md#4212-machine-learning-commands)
  - [4.2.13 Pipeline-Only Commands (Edge/Ingest Processor) 🔵](Chapter4.md#4213-pipeline-only-commands-edgeingest-processor)
  - [4.2.14 Custom Command Types ⚪](Chapter4.md#4214-custom-command-types)
- [4.3 Quick Lookup Table](Chapter4.md#43-quick-lookup-table)
- [4.4 Time-Based Analysis & Streaming Commands](Chapter4.md#44-time-based-analysis--streaming-commands)
  - [4.5 Time as a First-Class Dimension](Chapter4.md#45-time-as-a-first-class-dimension)
  - [4.6 Search Time Range](Chapter4.md#46-search-time-range)
  - [4.7 Time Bucketing: `bin`/`bucket` vs `span`](Chapter4.md#47-time-bucketing-binbucket-vs-span)
  - [4.8 `timechart` Deep Dive](Chapter4.md#48-timechart-deep-dive)
  - [4.9 `stats` with Time Span (SPL2 Exclusive)](Chapter4.md#49-stats-with-time-span-spl2-exclusive)
  - [4.10 `streamstats`: Running Calculations](Chapter4.md#410-streamstats-running-calculations)
  - [4.11 `eventstats`: Aggregate Without Losing Events](Chapter4.md#411-eventstats-aggregate-without-losing-events)
  - [4.12 `autoregress`: Previous Event Values](Chapter4.md#412-autoregress-previous-event-values)
  - [4.13 `delta`: Change Between Events](Chapter4.md#413-delta-change-between-events)
  - [4.14 `accum`: Simple Running Sum](Chapter4.md#414-accum-simple-running-sum)
  - [4.15 Time Windows: `time_window` vs `window`](Chapter4.md#415-time-windows-time_window-vs-window)
  - [4.16 Reset Logic](Chapter4.md#416-reset-logic)
  - [4.17 Rolling Statistics After `timechart`](Chapter4.md#417-rolling-statistics-after-timechart)
  - [4.18 Time Zone Handling](Chapter4.md#418-time-zone-handling)
  - [4.19 Complete Time Analysis Patterns](Chapter4.md#419-complete-time-analysis-patterns)
- [4.20 What's Next](Chapter4.md#420-whats-next)

---

## Chapter 5 — Sequence & Correlation Analysis + SOC Detection Patterns
- [5.1 Why Sequence Matters](Chapter5.md#51-why-sequence-matters)
- [5.2 The Golden Rule: Order Must Be Defined](Chapter5.md#52-the-golden-rule-order-must-be-defined)
- [5.3 Failure → Success Patterns (Brute Force)](Chapter5.md#53-failure--success-patterns-brute-force)
- [5.4 Password Spray: Distributed Failed Logons](Chapter5.md#54-password-spray-distributed-failed-logons)
- [5.5 Consecutive Event Detection](Chapter5.md#55-consecutive-event-detection)
- [5.6 Process Tree Analysis (Parent-Child)](Chapter5.md#56-process-tree-analysis-parent-child)
- [5.7 Beaconing Detection (Regular Intervals)](Chapter5.md#57-beaconing-detection-regular-intervals)
- [5.8 Lateral Movement Patterns](Chapter5.md#58-lateral-movement-patterns)
- [5.9 Data Exfiltration: Cumulative Volume](Chapter5.md#59-data-exfiltration-cumulative-volume)
- [5.10 Privilege Escalation: Token Manipulation](Chapter5.md#510-privilege-escalation-token-manipulation)
- [5.11 Persistence: Scheduled Tasks](Chapter5.md#511-persistence-scheduled-tasks)
- [5.12 Defense Evasion: Clear Logs](Chapter5.md#512-defense-evasion-clear-logs)
- [5.13 Credential Access: LSASS Dump](Chapter5.md#513-credential-access-lsass-dump)
- [5.14 Discovery: System Info](Chapter5.md#514-discovery-system-info)
- [5.15 Collection: Archive Files](Chapter5.md#515-collection-archive-files)
- [5.16 Command & Control: DNS Tunneling](Chapter5.md#516-command--control-dns-tunneling)
- [5.17 Exfiltration Over Web Service](Chapter5.md#517-exfiltration-over-web-service)
- [5.18 MITRE ATT&CK Mapping Table](Chapter5.md#518-mitre-attck-mapping-table)
- [5.19 Detection Engineering Best Practices](Chapter5.md#519-detection-engineering-best-practices)
- [5.20 Performance-Aware Detection](Chapter5.md#520-performance-aware-detection)
- [5.21 Tool Decision Guide](Chapter5.md#521-tool-decision-guide)
- [5.22 Complete Detection Examples](Chapter5.md#522-complete-detection-examples)
- [5.23 Performance Considerations](Chapter5.md#523-performance-considerations)
- [5.24 What's Next](Chapter5.md#524-whats-next)

---

## Chapter 6 — Edge/Ingest Processor Pipeline Design + Custom Commands
- [6.1 Two-Layer Configuration: Partition + Pipeline](Chapter6.md#61-two-layer-configuration-partition--pipeline)
- [6.2 Pipeline Syntax](Chapter6.md#62-pipeline-syntax)
- [6.3 Event Flow Model: 8 Stages](Chapter6.md#63-event-flow-model-8-stages)
- [6.4 Command Classification for Pipelines](Chapter6.md#64-command-classification-for-pipelines)
- [6.5 `where`: Filtering in Pipelines](Chapter6.md#65-where-filtering-in-pipelines)
- [6.6 `rex`/`spath`: Extraction](Chapter6.md#66-rexspath-extraction)
- [6.7 `eval`/`replace`: Transformation & Masking](Chapter6.md#67-evalreplace-transformation--masking)
- [6.8 `route`: Divert Subset to Another Destination](Chapter6.md#68-route-divert-subset-to-another-destination)
- [6.9 `thru`: Copy + Continue Main Path](Chapter6.md#69-thru-copy--continue-main-path)
- [6.10 `branch`: Multiple Independent Paths](Chapter6.md#610-branch-multiple-independent-paths)
- [6.11 `route` vs `thru` vs `branch` vs `if`: Decision Guide](Chapter6.md#611-route-vs-thru-vs-branch-vs-if-decision-guide)
- [6.12 `dedup`: Deduplication](Chapter6.md#612-dedup-deduplication)
- [6.13 `stats`: Aggregation in Pipelines](Chapter6.md#613-stats-aggregation-in-pipelines)
- [6.14 `if`: Conditional Processing](Chapter6.md#614-if-conditional-processing)
- [6.15 `fields`/`rename`: Field Management](Chapter6.md#615-fieldsrename-field-management)
- [6.16 `lookup`: Enrichment](Chapter6.md#616-lookup-enrichment)
- [6.17 `decrypt` / `ocsf`](Chapter6.md#617-decryptocsf)
- [6.18 `expand` / `flatten` / `mvexpand`](Chapter6.md#618-expandflattenmvexpand)
- [6.19 Complete Pipeline Design Patterns](Chapter6.md#619-complete-pipeline-design-patterns)
- [6.20 Partition vs `where`: Critical Distinction](Chapter6.md#620-partition-vs-where-critical-distinction)
- [6.21 Pipeline Debugging & Testing](Chapter6.md#621-pipeline-debugging--testing)
- [6.21 Custom Commands & Python SDK](Chapter6.md#621-custom-commands--python-sdk)
- [6.22 Custom Command Architecture](Chapter6.md#622-custom-command-architecture)
- [6.23 Generating Commands (No Input)](Chapter6.md#623-generating-commands-no-input)
- [6.24 Streaming Commands (Event → Event)](Chapter6.md#624-streaming-commands-event--event)
- [6.25 Reporting Commands (Events → Summary)](Chapter6.md#625-reporting-commands-events--summary)
- [6.26 Distributable vs Centralized](Chapter6.md#626-distributable-vs-centralized)
- [6.27 Python SDK](Chapter6.md#627-python-sdk)
- [6.28 Intersplunk vs Streaming Command Protocol](Chapter6.md#628-intersplunk-vs-streaming-command-protocol)
- [6.29 Checkpointing & State](Chapter6.md#629-checkpointing--state)
- [6.30 Configuration](Chapter6.md#630-configuration)
- [6.31 Testing & Debugging](Chapter6.md#631-testing--debugging)
- [6.32 What's Next](Chapter6.md#632-whats-next)

---

## Chapter 7 — Performance, Reference & Cheat Sheets
- [7.1 How Splunk Stores Data: tsidx Files](Chapter7.md#71-how-splunk-stores-data-tsidx-files)
- [7.2 Indexed Fields vs Search-Time Fields: Performance Impact](Chapter7.md#72-indexed-fields-vs-search-time-fields-performance-impact)
- [7.3 `tstats`: TSIDX-Based Statistics](Chapter7.md#73-tstats-tsidx-based-statistics)
- [7.4 `walklex`: Exploring the Lexicon](Chapter7.md#74-walklex-exploring-the-lexicon)
- [7.5 Search Modes: Fast, Smart, Verbose](Chapter7.md#75-search-modes-fast-smart-verbose)
- [7.6 Filtering Early: The Cardinality Principle](Chapter7.md#76-filtering-early-the-cardinality-principle)
- [7.7 `stats` vs `eventstats` vs `streamstats` Memory](Chapter7.md#77-stats-vs-eventstats-vs-streamstats-memory)
- [7.8 High-Cardinality Fields: `values`, `list`, `dc`](Chapter7.md#78-high-cardinality-fields-values-list-dc)
- [7.9 Summary Indexing & `sistats`](Chapter7.md#79-summary-indexing--sistats)
- [7.10 Data Model Acceleration](Chapter7.md#710-data-model-acceleration)
- [7.11 Search Job Inspector: Reading the Metrics](Chapter7.md#711-search-job-inspector-reading-the-metrics)
- [7.12 Optimization Checklist](Chapter7.md#712-optimization-checklist)
- [7.13 Reference: Decision Tables & Cheat Sheets](Chapter7.md#713-reference-decision-tables--cheat-sheets)
  - [7.13.1 `search` vs `where` vs `WHERE` Decision Table](Chapter7.md#7131-search-vs-where-vs-where-decision-table)
  - [7.13.2 Wildcard & Pattern Matching Cheat Sheet](Chapter7.md#7132-wildcard--pattern-matching-cheat-sheet)
  - [7.13.3 Regex Syntax: SPL vs SPL2](Chapter7.md#7133-regex-syntax-spl-vs-spl2)
  - [7.13.4 Aggregation Function Quick Reference](Chapter7.md#7134-aggregation-function-quick-reference)
  - [7.13.5 Streaming Command Options Quick Reference](Chapter7.md#7135-streaming-command-options-quick-reference)
  - [7.13.6 Time Window Cheat Sheet](Chapter7.md#7136-time-window-cheat-sheet)
  - [7.13.7 Command Classification Quick Lookup](Chapter7.md#7137-command-classification-quick-lookup)
  - [7.13.8 SPL → SPL2 Migration Cheat Sheet](Chapter7.md#7138-spl--spl2-migration-cheat-sheet)
  - [7.13.9 Pipeline vs Search Context Behavior](Chapter7.md#7139-pipeline-vs-search-context-behavior)
  - [7.13.10 Performance Decision Tree](Chapter7.md#71310-performance-decision-tree)
  - [7.13.11 Common Mistakes & Anti-Patterns](Chapter7.md#71311-common-mistakes--anti-patterns)
  - [7.13.12 Glossary](Chapter7.md#71312-glossary)
  - [7.13.13 File Inventory](Chapter7.md#71313-file-inventory)
  - [7.13.14 Version History](Chapter7.md#71314-version-history)
  - [7.13.15 Contributing](Chapter7.md#71315-contributing)

---

## Labs (Hands-On Practice)
- [Lab Data](labs/data/README.md) — Synthetic dataset schema and import instructions
- [Lab 01: `search` vs `where`](labs/01-search-vs-where/README.md) — When to use each
- [Lab 02: Command Execution](labs/02-command-execution/README.md) — Indexer vs Search Head placement
- [Lab 03: Statistical Commands](labs/03-statistical-commands/README.md) — stats vs eventstats vs streamstats
- [Lab 04: SPL2 & Pipelines](labs/04-spl2-and-pipelines/README.md) — Structured vs Pipeline vs EP/IP

---

## Reference Files
- [SPL2_EP_IP.md](SPL2_EP_IP.md) — Edge Processor pipeline design reference (detailed)
- [labs/data/README.md](labs/data/README.md) — Synthetic dataset schema and import instructions