# Contributing to SPL + SPL2 Reference

Thank you for contributing! This is a community learning reference. We value accuracy, practical examples, and official-source backing.

## Quick Start

1. **Fork** the repository
2. **Create a branch** for your change: `git checkout -b fix/command-classification-lookup`
3. **Make your change** following the standards below
4. **Test your example** in the stated Splunk product/version
5. **Submit a PR** with a clear description

## Contribution Types

| Type | Requirements |
|------|--------------|
| **New command/example** | Product/version, syntax family, compatibility profile, input/output, official source |
| **Classification fix** | Command, old classification, new classification, official source, mode/option context |
| **SOC pattern** | MITRE technique, data source, SPL/SPL2 query, expected output, evidence standard |
| **Performance tip** | Query, before/after metrics, Job Inspector evidence, version |
| **Lab exercise** | Synthetic data, step-by-step, validation criteria |
| **Documentation fix** | Typo, clarity, broken link, missing cross-reference |

## Evidence Standard (Required)

Every substantial addition MUST include:

```markdown
**Product:** Splunk Enterprise 9.2 / Cloud
**Syntax:** SPL2 (pipeline)
**Profile:** edgeProcessor
**Input:**
```json
{"profile":"linux","action":"login","user":"alice"}
```
**Query:**
```spl2
$pipeline = | from $source
    | where profile == "linux"
    | into $linux_dest;
```
**Output:** Events routed to `$linux_dest`
**Why:** `route` is pipeline-only; `where` filters ingestion stream
**Official Source:** [Edge Processor route command](https://help.splunk.com/en/splunk-cloud-platform/process-data-at-the-edge/use-edge-processors-for-splunk-cloud-platform/route-data-using-pipelines/process-a-subset-of-data-using-an-edge-processor)
```

## Command Classification Updates

When updating a command's classification:

1. **State the mode/option context** — e.g., "dedup with `sortby` is dataset processing"
2. **Cite the official reference** — Splunk Search Manual 10.x or Search Reference 10.x
3. **Note SPL vs SPL2 differences** — e.g., "SPL2 `lookup` has no `local` option"
4. **Update the reference table** in Part 3 and Part 5

## SOC Detection Patterns

New detections must include:

| Field | Required |
|-------|----------|
| MITRE ATT&CK Technique | Yes (e.g., T1110.001) |
| Tactic | Yes (e.g., Credential Access) |
| Data Source | Yes (e.g., Windows Security 4625) |
| SPL Query | Yes (with macros if used) |
| SPL2 Query | Preferred |
| Expected Output | Table schema or description |
| False Positive Considerations | Yes |
| Performance Notes | `tstats` eligible? High cardinality? |

## Code Examples

- Use **synthetic data only** — no real IPs, hosts, users, credentials
- Keep examples **minimal** — 5-15 lines max
- Show **both SPL and SPL2** when both exist
- Mark **pipeline-only commands** clearly (EP/IP only)

## Style Guide

### Formatting
- Commands in backticks: `` `stats` ``
- SPL code blocks: ```spl
- SPL2 code blocks: ```spl2
- JSON in code blocks: ```json
- File paths: `Part3_Command_Classification.md`

### Classification Badges
Use these consistently:
- 🟢 Distributable Streaming
- 🟡 Centralized Streaming
- 🔴 Reporting / Non-Streaming
- 🟣 Generating
- 🟠 Orchestrating
- 🔵 Pipeline-Only (EP/IP)
- ⚪ Custom Command Type

### Links
- Official docs: Full URL to `help.splunk.com`
- Community: Link to `community.splunk.com` thread
- GitHub: Link to repo/file

## Testing Your Changes

Before submitting a PR:

1. **Run the example** in a test Splunk instance (Enterprise/Cloud/EP/IP)
2. **Verify output** matches documented expected result
3. **Check Job Inspector** for execution placement claims
4. **Validate SPL2 syntax** in the target product/profile
5. **Update INDEX.md** if adding new sections

## Review Process

1. Maintainer reviews for evidence standard compliance
2. Technical accuracy check (may request test reproduction)
3. Style/formatting review
4. Merge after approval

## What We Don't Accept

- Examples without official source citations
- Real production data (IPs, hostnames, usernames, credentials)
- Unverified performance claims
- SPL2 syntax not validated in stated profile
- Copied code from other repos without license/attribution check

## Attribution

If adapting from another repository (e.g., Siddhartha Juprod07's youtube repo):
1. Verify license compatibility (MIT preferred)
2. Add attribution in the file header
3. Link to original source

## Questions?

Open an issue with the `question` label. We're happy to clarify standards before you invest time in a PR.

---

**Remember**: This reference teaches *decision-making*, not just syntax. Every contribution should help someone understand **when** and **why**, not just **how**.