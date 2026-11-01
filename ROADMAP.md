# Roadmap — SPL + SPL2 Reference v2.x

## Version 2.1 (Q1 2027)
- [ ] **SPL2 Federated Search profile** — command compatibility matrix
- [ ] **MLTK integration** — `fit`, `apply`, `anomalydetection` with SPL2 syntax
- [ ] **Dashboard Studio SPL2** — token handling, dynamic options, drilldown
- [ ] **Metrics & `mstats`** — SPL2 syntax for metrics indexes
- [ ] **SOAR/Phantom integration** — SPL2 in playbook actions

## Version 2.2 (Q2 2027)
- [ ] **Data Model Acceleration deep dive** — `tstats` with DM, `summariesonly`
- [ ] **Summary indexing patterns** — `sistats`, `collect`, `marker` with SPL2
- [ ] **KV Store & `lookup` SPL2** — `inputlookup`/`outputlookup` in pipelines
- [ ] **Custom visualizations** — SPL2 data shaping for D3/React components

## Version 2.3 (Q3 2027)
- [ ] **Multi-cluster search** — `splunk_server`, `splunk_server_group` in SPL2
- [ ] **Search Job Inspector automation** — REST API for performance CI/CD
- [ ] **Edge Processor advanced** — `branch`/`route`/`thru` nesting, `if`/`elseif`/`else`
- [ ] **Ingest Processor `logs_to_metrics`** — metric extraction patterns

## Version 3.0 (2027+)
- [ ] **SPL3 / future language** — Track Splunk roadmap
- [ ] **AI-assisted search** — Natural language → SPL/SPL2
- [ ] **Real-time detection** — Streaming analytics with `streamstats` windows
- [ ] **Cross-product detection** — Unified SPL2 across Search, EP, IP, SOAR

---

## Topic Backlog (Unscheduled)

### Foundations
- [ ] `tsidx` internals: `walklex`, `dbinspect`, bucket lifecycle
- [ ] Index-time vs search-time extraction: `props.conf`, `transforms.conf`, `INGEST_EVAL`
- [ ] Field aliases, calculated fields, tags, event types
- [ ] Time zone handling: `_time`, `strftime`, `relative_time`, `snap`

### Analytics
- [ ] `xyseries`, `contingency`, `correlate`, `predict`, `trendline`
- [ ] `geostats`, `geom`, `cluster`, `kmeans`
- [ ] `join` optimization: `lookup` vs `join` vs `stats` + `eval`
- [ ] Subsearch limits: `maxout`, `maxtime`, `timeout`
- [ ] `map` vs `foreach` vs `apply` for iterative logic

### SPL2 Language
- [ ] SPL2 modules: `import`, `export`, `module` syntax
- [ ] SPL2 variables: `let`, `set`, `params`
- [ ] SPL2 functions: user-defined, lambda, higher-order
- [ ] SPL2 type system: `cast`, `typeof`, schema validation

### Ingestion Pipelines
- [ ] EP/IP partition design patterns
- [ ] Pipeline testing: sample data, unit tests, CI/CD
- [ ] `decrypt`/`ocsf` real-world patterns
- [ ] Multi-destination routing with `branch`
- [ ] Schema normalization: `ocsf`, `EQL`, `Sigma` export

### Custom Commands
- [ ] Streaming command protocol v2
- [ ] Checkpointing for incremental APIs
- [ ] State management: `kvstore`, `summary index`, `collect`
- [ ] Packaging: `app.conf`, `commands.conf`, `setup.xml`

### Performance & Operations
- [ ] Search concurrency: `max_concurrent`, `base_max_concurrent`
- [ ] Resource pools: `srchJobsQuota`, `srchDiskQuota`
- [ ] `tstats` with accelerated data models
- [ ] Search head clustering: knowledge bundle replication

### Detection Engineering
- [ ] MITRE ATT&CK coverage matrix
- [ ] Detection-as-code: versioning, testing, CI/CD
- [ ] Risk-based alerting: `risk_object`, `risk_score`
- [ ] Notable event management: suppression, enrichment, closure

### Developer Experience
- [ ] VS Code extension for SPL/SPL2
- [ ] Linting: `splint`, custom rules
- [ ] Unit testing framework for searches
- [ ] Documentation generation from macros/lookups

---

## Maintenance

- [ ] Quarterly review: Splunk release notes for new commands
- [ ] Annual audit: Dead links, deprecated syntax, version compatibility
- [ ] Community PR triage: Weekly

---

## Completed in v2.0

- [x] Command classification taxonomy (6 categories + mode-dependent)
- [x] SPL vs SPL2 deep comparison (40+ syntax mappings)
- [x] SPL2 Compatibility Profiles (`splunkd`, `edgeProcessor`, `ingestProcessor`)
- [x] Edge/Ingest Processor pipeline design patterns
- [x] `search` vs `where` decision guide with 10-capability table
- [x] 14 MITRE-mapped SOC detection patterns
- [x] `tstats`, `tsidx`, `walklex` performance guide
- [x] Indexer-to-search-head handoff visual model
- [x] 12-part structured reference with INDEX.md navigation
- [x] Evidence standard for all examples
- [x] Labs structure with synthetic dataset