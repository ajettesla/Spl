# Splunk SPL + SPL2 — Complete Reference (v2.0)

A comprehensive, structured reference for Splunk Search Processing Language (SPL) and SPL2, designed for SOC analysts, detection engineers, and Splunk architects.

## Why This Exists

Most Splunk resources teach syntax in isolation. This reference teaches **decision-making**: when to use each command, why alternatives exist, and how they behave differently in search vs. ingestion pipelines.

## Structure

```
Part 0 — Quick Start & Navigation
Part 1 — Foundations: Events, Fields & Data Model
Part 2 — SPL Fundamentals: Search, Pipeline & Filtering
Part 3 — Command Classification Taxonomy
Part 4 — SPL vs SPL2: Deep Comparison
Part 5 — Search Commands Reference (Classified)
Part 6 — Time-Based Analysis & Streaming Commands
Part 7 — Sequence & Correlation Analysis
Part 8 — SPL2 Edge/Ingest Processor Pipeline Design
Part 9 — Custom Commands & Python SDK
Part 10 — Performance: tstats, tsidx & Optimization
Part 11 — Real-World SOC Detection Patterns
Part 12 — Reference: Decision Tables & Cheat Sheets
Labs — Hands-on exercises with synthetic data
ROADMAP.md — Future topics and version planning
CONTRIBUTING.md — Contribution guidelines
```

## Key Features

| Feature | Description |
|---------|-------------|
| **Command Classification** | Every command tagged: streaming, non-streaming, distributable, centralized, generating, transforming, dataset-processing, orchestrating |
| **SPL ↔ SPL2 Mapping** | Side-by-side syntax with semantic differences explained |
| **`search` vs `where` Decision Guide** | When to use each, with performance implications |
| **Pipeline vs Search Context** | Same command, different behavior in Edge/Ingest Processor |
| **SOC Patterns** | Real detection logic: brute force, beaconing, exfiltration, lateral movement |
| **Performance Mindset** | tsidx, tstats, indexed fields, search-time vs index-time |

## Quick Navigation

- **New to Splunk?** Start with [Part 1](Part1_Foundations.md) → [Part 2](Part2_SPL_Fundamentals.md)
- **Know SPL, learning SPL2?** Jump to [Part 4](Part4_SPL_vs_SPL2.md)
- **Building detections?** See [Part 11](Part11_SOC_Patterns.md)
- **Edge/Ingest Processor?** See [Part 8](Part8_Edge_Ingest_Processor.md)
- **Command lookup?** Use [Part 5](Part5_Commands_Reference.md) or [Part 12](Part12_Reference.md)

## Command Classification Quick Reference

```
STREAMING (distributed)
├── filter: where, search, regex, match
├── transform: eval, rex, replace, rename, fields
├── enrich: lookup
├── stats: streamstats (windowed), accum, autoregress, delta
└── structure: mvexpand, expand, flatten

STREAMING (centralized)
├── sort, head, tail, dedup
├── streamstats (global/window=0), eventstats
└── stats (without BY), chart, timechart

NON-STREAMING (reporting)
├── stats, chart, timechart, top, rare
├── tstats, sichart, sirare, sitop
└── geom, geostats

GENERATING (dataset processing)
├── search, from, inputlookup, inputcsv
├── makeresults, metadata, typeahead
└── rest, savedsearch

ORCHESTRATING (pipeline control)
├── route, thru, branch
├── if, foreach, map
├── append, appendpipe, appendcols
└── join, selfjoin, lookup (subsearch)

CUSTOM COMMAND TYPES
├── Generating (no input)
├── Streaming (event → event)
├── Reporting (events → summary)
└── Distributable vs Centralized
```

## Prerequisites

- Basic Splunk navigation
- Understanding of events, fields, indexes
- Familiarity with regex concepts

## Evidence Standard (Required for All Examples)

Every substantial example in this reference follows this standard:

| Requirement | Description |
|-------------|-------------|
| **Product & Version** | Splunk Enterprise 9.x / Cloud, Edge Processor, Ingest Processor |
| **Syntax Family** | Classic SPL or SPL2 (explicitly stated) |
| **SPL2 Compatibility Profile** | `splunkd` (Search), `edgeProcessor`, `ingestProcessor`, `federatedSearch` |
| **Input Example** | Small synthetic event showing relevant fields |
| **Exact Query/Pipeline** | Copy-pasteable SPL/SPL2 |
| **Expected Output** | Table or description of result |
| **Why This Command** | Explanation of classification & intent |
| **Execution/Performance Note** | Where it runs, memory impact, boundary crossing |
| **Official Splunk Reference** | Link to current docs |
| **Community/Article Reference** | Only when adding practical context (not overriding docs) |

**Official documentation is the authority.** Community answers and articles are useful for learning patterns and troubleshooting, but must not override current product documentation.

---

## Labs (Hands-On Practice)

This repository includes a synthetic dataset for hands-on validation:

```
labs/
├── data/
│   └── auth_events.csv          # Synthetic auth events (no real data)
├── 01-search-vs-where/          # Lab: when to use each
├── 02-command-execution/        # Lab: Job Inspector evidence
├── 03-statistical-commands/     # Lab: stats vs eventstats vs streamstats
└── 04-spl2-and-pipelines/       # Lab: SPL2 structured vs pipeline vs EP/IP
```

**To use labs:**
1. Import `labs/data/auth_events.csv` into a test Splunk index
2. Run examples after changing the index name
3. Compare result counts and fields
4. Inspect Job Inspector for execution placement
5. Adapt to your own lab only after validating expected output

---

## Contributing

This is a living reference. Issues and PRs welcome for:
- New SPL2 commands/features
- Corrections to command classifications
- Additional SOC patterns
- Performance benchmarks
- Lab exercises with synthetic data

**Contribution requirements:**
1. Provide a reproducible example
2. Identify the product/version/profile
3. Show expected output
4. Cite the official source supporting the claim

## License

MIT — Use freely for learning, teaching, and reference.