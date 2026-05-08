---
name: QUALITY.md
description: Design the system's observability and performance contract — logging standard, domain event log vocabulary, metrics catalogue with stable `METRIC-*` IDs, service-level objectives under `SLO-*` IDs with burn-rate alerts, tracing strategy, alert catalogue, performance budgets per flow, capacity envelope, caching strategy, back-pressure and load-shedding policy, and recorded benchmarks. Use when asked to design observability and performance, define SLOs and alerts, write the metrics catalogue, set performance budgets, design the capacity envelope, or produce a QUALITY.md.
---

# Task: Generate QUALITY.md — Observability & Performance Contract

## Objective

Produce a QUALITY.md that serves as the single source of truth for everything the system measures, logs, traces, and alerts on, together with the performance budgets and capacity envelope every hot path must respect. It fixes the logging format and forbidden-fields list, registers the domain event log vocabulary distinct from wire events, catalogues every metric under a stable `METRIC-*` ID with label cardinality stated, defines service-level objectives under `SLO-*` IDs with burn-rate alerts, pins the tracing strategy and span inventory per saga, enumerates alerts with severity and runbook links, attaches per-flow latency and throughput budgets, commits a capacity envelope, specifies caching per location, names back-pressure and load-shedding policies, and records benchmark baselines. An agent reading this document alone can — for any endpoint, saga, or event in the system — name the metrics that observe it, the SLO it serves, the alerts that fire on it, the latency budget it must satisfy, and the capacity envelope that frames it, without opening ARCHITECTURE, INTERFACES, or BEHAVIOR.

QUALITY.md consolidates what older guidance split as OBSERVABILITY.md and PERFORMANCE_MODEL.md, because every downstream reader needs both together: `/operations` wires alert routing and runbooks; every per-unit `/SPEC.md` declares the signals the unit emits and the budgets it satisfies; `/code-review` verifies the declarations; `/system-verification` checks end-to-end that signals actually flow. The defining discipline — and the commonest violation — is **every SLI has an SLO, every SLO has a burn-rate alert, every alert links to a runbook**: orphan signals rot into noise; orphan SLOs decay into aspirations; orphan alerts page humans with no script to follow.

---

## Inputs

1. **ARCHITECTURE.md** (required) — § 2 Components and cross-component flow sections supply the list of components that emit signals and the flows that carry latency budgets; any `ADR-NN` on consistency, caching, or overload framing informs §§ 9–10.
2. **INTERFACES.md** (required) — § 6 Endpoints are the externally-measurable entry points; every `EP-name` that qualifies as a critical flow gets a § 7 Performance Budget row. § 7 Events supply the `EVT-name` set that the tracing strategy in § 5 and metric emission in § 3 anchor to.
3. **BEHAVIOR.md** (required) — § 1 State Machines supply transitions whose rates this document measures; § 2 Sagas supply saga steps whose durations § 7 budgets and whose spans § 5 traces; § 3 Idempotency Keys inform replay metrics; § 6 Event Emission Timing anchors the span inventory of § 5 to exact emission points.
4. **PROPOSAL.md** (optional) — product success criteria inform SLO targets. If PROPOSAL.md states a numeric target (e.g., "p95 push latency under 2 seconds"), cite it verbatim in the relevant `SLO-*`. If no numeric target is stated, propose one in § 13 Open Questions with options and tradeoffs.
5. **DATA.md** (optional — required if produced) — § 8 (or equivalent) sensitive-column inventory is the authoritative forbidden-fields list for § 1 Logging Standard. If DATA.md is not yet produced, derive the forbidden list from DOMAIN's value objects and surface the absent DATA reference as an open question.
6. **ERRORS.md** (optional — required if produced) — § 3 Error Code Registry supplies the codes that § 6 Alerts and § 2 Domain Event Log Vocabulary may reference. Alerts that trigger on specific error codes cite them by `code` string.
7. **OPERATIONS.md** (optional — if not yet produced, § 6 alert routing and runbook links surface as open questions for `/operations` to resolve downstream).
8. **Existing QUALITY.md** (auto-discovered — only if refining) — read fully. Every assigned `METRIC-*` and `SLO-*` ID is permanent. New entries take new unused IDs. Retired entries retain their row with `(retired — superseded by {new-id})` markers.

Read set size: 3 required artifacts + PROPOSAL, DATA, ERRORS when present + optional prior QUALITY. Read all required inputs end-to-end. Truncated reads cause two specific failure modes: metrics cataloged against endpoints or sagas that do not exist (INTERFACES / BEHAVIOR not fully read) and SLOs whose targets contradict PROPOSAL's success criteria (PROPOSAL not fully read).

---

## Workflow

Quality-contract construction proceeds in eight phases: logging standard, event log vocabulary, metrics catalogue, SLOs, tracing, alerts, performance and capacity, caching and back-pressure. Phases are sequential — later phases cite IDs introduced by earlier phases — but revisit earlier phases if a later one reveals a missing metric or an un-derivable SLO.

### Phase 1: Logging Standard & Event Log Vocabulary

Produce § 1 Logging Standard and § 2 Domain Event Log Vocabulary together — they share the redaction contract.

§ 1 fixes format (structured JSON default), required fields, forbidden fields, log-level policy, retention, correlation, and redaction — see the template in Output Format for exact fields. The forbidden-fields list is derived from DATA.md § 8 (or DOMAIN's value objects if DATA not yet produced); missing any sensitive column is a data-leak bug. Retention cites OPERATIONS; absence surfaces as an open question.

§ 2 catalogues **domain log events** — operational log events the system writes, distinct from wire events owned by INTERFACES § 7. Name them in dotted lowercase verb-phrase form (`repo.push.accepted`, `user.login.failed`, `saga.push.compensated`). Each entry anchors to BEHAVIOR: cite `SAGA-{name} step {N}` or `SM-{entity}: {from} → {to}` where emission occurs. Un-anchored events drift. Sampling rates below 100% name the justification (volume, cost) and whether sampling is head or tail.

### Phase 2: Metrics Catalogue

§ 3 is the master metrics catalogue — one row per metric, alphabetical by `METRIC-*`. Derive from three sources: every `EP-name` in INTERFACES § 6 gets request-count, duration-histogram, and in-flight gauge; every `SAGA-{name}` in BEHAVIOR § 2 gets duration-histogram, compensation-count, and timeout-count; every non-trivial `SM-{entity}: {from} → {to}` transition worth counting gets a counter.

ID form is `METRIC-{kebab-case-verb-or-noun}` (`METRIC-push-duration-ms`, `METRIC-saga-push-compensation-count`). Columns are defined in the Output Format template. See Rules §§ 1, 2, 5, 6 for stability, SLI-to-SLO contract, cardinality, and unit constraints.

### Phase 3: Service-Level Objectives

§ 4 enumerates every SLO. ID form: `SLO-{kebab-case}` — `SLO-push-availability`, `SLO-push-latency-p95`. Each block carries SLI definition (formula over § 3 metrics only), numeric windowed target, implied error budget stated explicitly, burn-rate alert ID (fast/slow pair — `14.4× over 1h` and `6× over 6h` is the standard), scope, and `UC-NN` references. PROPOSAL's numeric targets are cited verbatim; silence surfaces in § 13 Open Questions with conservative/aggressive options.

### Phase 4: Tracing Strategy

§ 5 pins propagation format (W3C `traceparent` default), required tags on every span (`service`, `tenant_id`, `request_id`, `route`, `saga_id`, `outcome`), sampling strategy (typical: 100% tail on errors, ~1% head on success, 100% on `flag_trace=1`), and retention (cite OPERATIONS).

Span inventory: every `SAGA-{name}` in BEHAVIOR § 2 gets one root span plus one per happy-path step using naming convention `{saga}.{step-N}.{verb}` (`push.1.validate`, `push.2.accept`). State-machine transitions worth tracing get `{sm}.{from}-to-{to}`. High-cardinality dimensions land on spans, not metrics.

### Phase 5: Alert Catalogue

§ 6 derives alerts from three sources: one burn-rate alert per `SLO-*` in § 4 (fast/slow thresholds); one saturation alert per monitored capacity dimension from § 8 (queue depth, CPU, memory, connection pool); one health alert per critical component in ARCHITECTURE § 2 whose failure is not auto-recoverable (primary DB unreachable, message bus down).

ID form: `ALERT-{kebab-case}` (`ALERT-push-availability-burn-rate`, `ALERT-db-connection-pool-saturation`). Each block fills the fields in the Output Format template. Triggers are query-level with thresholds and duration, never prose. Severity follows OPERATIONS routing policy: `sev1` pages on-call, `warning` posts to a channel. Missing `RUNBOOK-*` names surface in § 13 pending `/operations`.

### Phase 6: Performance Budgets & Capacity Envelope

§ 7 attaches a perf budget to every critical flow — `EP-*` named hot in ARCHITECTURE, `SAGA-*` user-facing or SLO-backed, or cross-surface scenario `SCN-*`. Each block fills target load, p50/p95/p99 latency, component/step breakdown whose slices sum to the total (with an explicit overhead line), throughput targets, and degradation mode — a concrete mechanism citing § 10, never prose like "graceful degradation".

§ 8 Capacity Envelope states concurrent users (baseline / peak / year-out), RPS per critical endpoint (baseline / peak / N-month projection), data-volume growth (cite DATA for storage assumptions), an explicit hot-path list, and per-component resource envelopes (CPU cores, memory GB, disk I/O MB/s, network Mbps) citing ARCHITECTURE § 2 components exactly.

### Phase 7: Caching, Back-Pressure, Benchmarks

§ 9 Caching Strategy. One block per caching location (edge CDN, API in-process, database query cache). Each names what is cached, key format, concrete TTL (`60s`, `5m`, `24h`) or event-driven (`invalidated on EVT-push-accepted`), invalidation trigger citing `EVT-*` where applicable, fallback on miss, and coherency guarantee (strong / eventual bounded by TTL / best-effort).

§ 10 Back-Pressure & Load Shedding. Per-component back-pressure (HTTP 429 with Retry-After, bounded-latency slow response, circuit breaker open) citing `ERR_CODE` from ERRORS.md; explicit load-shedding priority ladder (typical: anonymous → non-interactive → low-tier tenant → interactive → high-tier); per-downstream circuit-breaker policies (open / half-open / closed thresholds and caller behaviour when open).

§ 11 Benchmarks. One block per measured flow with date, methodology (`k6`, `wrk`, or custom; concurrency; duration; percentile window), p50/p95/p99 results in ms, saturation RPS (< 1% error), and signed delta to § 7 target. If none recorded: "No benchmarks recorded — produced by `/system-verify` and recorded back on regeneration."

### Phase 8: Relationship, Open Questions, Validation

§ 12 Relationship to Other Artifacts is one bullet per relationship in the fixed order: ARCHITECTURE, INTERFACES, BEHAVIOR, ERRORS, DATA, OPERATIONS, SECURITY, SPEC-level, `/system-verify`. § 13 Open Questions collects genuine ambiguity — missing PROPOSAL targets, absent OPERATIONS runbook names, un-classified label cardinality — with options, tradeoffs, and a recommendation.

Before finalising, run the Quality Checklist below end-to-end. Update frontmatter counts to match the body exactly. `status` is `complete` if § 13 reads "All questions resolved.", `has_open_questions` otherwise.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Stable IDs forever

`METRIC-*` and `SLO-*` IDs are assigned once and never renumbered, never reused, never silently deleted. Retired entries retain their row with `(retired — superseded by {new-id})` markers so downstream citations keep resolving. The same rule applies to `ALERT-*` IDs in § 6.

### 2. Measurement-to-commitment chain is complete

Every `is_sli: yes` row in § 3 has a matching `SLO-*` in § 4; every `SLO-*` cites at least one `ALERT-*` in § 6 whose trigger is a burn-rate query; every `ALERT-*` in § 6 names a `RUNBOOK-*` from OPERATIONS (or surfaces the missing runbook as an open question). Orphan SLIs, orphan SLOs, and blind pages are each modelling failures.

### 3. Every critical flow has a perf budget

Every `EP-*` marked hot in ARCHITECTURE or named in an SLO, and every user-facing `SAGA-*` in BEHAVIOR § 2, has a § 7 budget block. Orphan flows drift into latency regression.

### 4. Perf-budget breakdown sums

For each flow in § 7, the component / step slices sum to the total (with a named overhead line if slices sum under total). A breakdown that does not sum is a modelling bug.

### 5. Label cardinality is stated

Every label on every metric in § 3 has its cardinality estimated as a number or explicitly flagged as unbounded with a mitigation strategy (exemplars, traces, histogram-over-hash). High-cardinality dimensions (`user_id`, `request_id`, `saga_id`) are forbidden as metric labels; they live on traces and logs.

### 6. Units are concrete

Every metric row has a concrete unit — `ms`, `bytes`, `requests`, `ratio`, `count`. Never `value`, `amount`, blank, or `(varies)`. Durations are in milliseconds; sizes in bytes; percentages are ratios in the range 0–1 unless the metric explicitly declares otherwise.

### 7. SLO targets are numeric and windowed

Targets are expressed as a numeric value plus a time window — `99.9% over 30-day rolling window`, `p95 < 2000 ms over 7 days`. Prose targets ("high availability", "fast response") are modelling failures; either derive a numeric target from PROPOSAL or surface the ambiguity in § 13.

### 8. Every saga is traced

Every `SAGA-{name}` in BEHAVIOR § 2 has a span entry in § 5 Span Inventory. Un-traced sagas cannot be debugged in production — the commonest source of outages that never resolve.

### 9. Forbidden fields cover DATA § 8 in full

The forbidden-fields list in § 1 is a superset of DATA.md's sensitive inventory. Missing any sensitive column is a data-leak bug, detectable by diffing § 1 against DATA § 8. If DATA.md has not been produced, surface the absence as an open question and derive a best-effort list from DOMAIN's value objects.

### 10. Caches are concrete on key, TTL, and invalidation

Every cache entry in § 9 names a concrete key format, a concrete TTL (duration or event-driven), and a concrete invalidation trigger. `(indefinite)` TTL without an event-driven invalidation is a memory leak.

### 11. Degradation modes are named mechanisms

Every budget in § 7 names the degradation mode on breach — `503`, bounded queue of depth N, shed anonymous traffic, reduce pagination limit. "Graceful degradation" alone is not a mechanism. Back-pressure citations link to § 10 for concrete behaviour.

### 12. No wire-schema design

QUALITY.md names metrics, SLOs, alerts, and budgets. It does not design HTTP status codes for error responses (ERRORS.md), wire shape of events (INTERFACES.md), or database schema (DATA.md). A QUALITY.md that starts enumerating JSON fields or table columns is out of scope — delete that content.

### 13. No implementation syntax

No code snippets (try / catch / async / await). No stack-specific SDK or product names beyond the tracing propagation format (`W3C traceparent`) and the load-generator names in § 11 — those belong to OPERATIONS.

### 14. Single YAML frontmatter block

One YAML block at the top containing common fields (`skill`, `date`, `status`) and quality-specific counts (`metrics`, `slos`, `alerts`, `perf_budgets`, `traced_flows`, `open_questions`). Never emit a second block. Counts match the body exactly.

### 15. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "some", "reasonable". Use exact IDs, exact durations, exact numbers. Unresolvable ambiguity surfaces in § 13 Open Questions with options, tradeoffs, and a recommendation.

---

## Output Format

```markdown
---
skill: QUALITY.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
metrics: {N}
slos: {N}
alerts: {N}
perf_budgets: {N}
traced_flows: {N}
open_questions: {N}
---

# QUALITY — {ProductName}

> Observability and performance contract. Every metric, SLO, alert, trace span,
> perf budget, capacity target, cache, back-pressure rule, and benchmark lives
> here. Downstream artifacts cite by stable `METRIC-*`, `SLO-*`, `ALERT-*` IDs.
> Runbook contents and alert routing detail live in OPERATIONS.md; error-code
> definitions in ERRORS.md; wire shape of endpoints and events in INTERFACES.md.

## § 1. Logging Standard

- **Format:** {structured JSON | other — with rationale}
- **Required fields on every log line:** `ts`, `level`, `service`, `request_id`, `trace_id`, `span_id`, `user_id_hash`, `tenant_id`, `error_code`, `message`, {additional fields}
- **Forbidden fields:** {column or field name} — {PII | secret | payload body | per DATA § 8}
- **Log levels policy:**
  - `debug` — {when, one example}
  - `info` — {when, one example}
  - `warn` — {when, one example}
  - `error` — {when, one example}
  - `critical` — {when, one example}
- **Retention:** {duration} in {store} (cite OPERATIONS § {N})
- **Correlation:** every request carries a `request_id`; every saga step from BEHAVIOR § 2 adds a `span_id`; cross-service propagation via W3C `traceparent`.
- **Redaction rules applied before logging:** {rule list — `user_id` hashed via HMAC-SHA-256 with rotating key; `authorization` header stripped; fields in DATA § 8 are never logged}.

---

## § 2. Domain Event Log Vocabulary

| Event name | When emitted | Fields | Sampling | Primary consumers |
|------------|--------------|--------|----------|-------------------|
| `{name.dotted.lowercase}` | `SAGA-{name} step {N}` / `SM-{entity}: {from} → {to}` | {field-role list} | `{100%}` / `{N%}` | `{SRE / security / product / support}` |

(One row per event, alphabetical. If none beyond access logs: "No domain log
 events beyond access logs.")

---

## § 3. Metrics Catalogue

| `METRIC-*` | type | unit | labels | cardinality | is_sli | emitted_by | source |
|------------|------|------|--------|-------------|--------|-----------|--------|
| `METRIC-{kebab-case}` | `{counter / gauge / histogram / summary}` | `{ms / bytes / requests / ratio / count}` | `{label1, label2}` | `{label1: ~N, label2: bounded-set-of-M}` | `{yes / no}` | `{ComponentName from ARCHITECTURE § 2}` | `{HTTP middleware / SAGA wrapper / code span}` |

(Alphabetical by `METRIC-*`. One row per metric. No blanks.)

---

## § 4. Service-Level Objectives

### `SLO-{kebab-case}`

- **SLI definition:** `{formula over § 3 metrics — e.g., METRIC-push-requests{outcome="success"} / METRIC-push-requests}`
- **Target:** `{numeric}% over {window}` or `p{NN} < {N} ms over {window}`
- **Error budget:** `{implied — e.g., 0.1% per 30 days ≈ 43.2 min}`
- **Burn-rate alert:** `ALERT-{id}` — fast threshold `{14.4× over 1h}`; slow threshold `{6× over 6h}`
- **Scope:** `{per EP-name / per SAGA-name / per tenant / global}`
- **Related use cases:** `UC-{NN}`, `UC-{NN}`

(Repeat for every SLO.)

---

## § 5. Tracing Strategy

- **Propagation format:** W3C `traceparent` (or {other — with rationale}).
- **Required tags on every span:** `service`, `tenant_id`, `request_id`, `route` (if HTTP-entered), `saga_id` (if saga-bound), `outcome` (`success` / `error:{ERR_CODE}` / `timeout`).
- **Sampling:** head {N}%; tail 100% on traces where any span has `error`; 100% when `flag_trace=1`.
- **Retention:** {duration} in hot store; {duration} in cold store (cite OPERATIONS § {N}).

### Span inventory

| Span name | Emitted by | Tags added beyond required | Notes |
|-----------|-----------|----------------------------|-------|
| `{saga-name}.{N}.{step-verb}` | `SAGA-{name} step {N}` | `{tag list}` | `{note}` |
| `{sm-name}.{from}-to-{to}` | `SM-{entity}: {from} → {to}` | `{tag list}` | `{note}` |

(Every `SAGA-*` in BEHAVIOR § 2 appears. If no sagas: "No sagas in BEHAVIOR.md;
 only HTTP request spans trace end-to-end.")

---

## § 6. Alert Catalogue

### `ALERT-{kebab-case}`

- **Name:** `{short human-readable}`
- **Trigger:** `{PromQL-equivalent query + threshold + duration — e.g., rate(METRIC-push-errors[5m]) / rate(METRIC-push-requests[5m]) > 0.01 for 10m}`
- **Severity:** `{sev1 / sev2 / sev3 / warning}`
- **Routing:** `{team / channel — cite OPERATIONS routing table}`
- **Runbook:** `RUNBOOK-{name}` (cite OPERATIONS § {N})
- **Owner:** `{component from ARCHITECTURE § 2 / team}`
- **Related SLO:** `SLO-{id}` or `(n/a)`

(Repeat for every alert — burn-rate from § 4, saturation from § 8, health
 per critical component.)

---

## § 7. Performance Budgets

### `EP-{name}` or `SAGA-{name}` or `SCN-{name}`

- **Target load:** baseline `{N}` RPS, peak `{M}` RPS, concurrency `{K}`.
- **Total latency budget:** p50 `{N}` ms; p95 `{N}` ms; p99 `{N}` ms at target load.
- **Breakdown by component / step:**
  - `{ComponentName / Step-N}` — `{N}` ms ({description})
  - `{ComponentName / Step-N}` — `{N}` ms ({description})
  - Overhead / network — `{N}` ms
- **Throughput target:** sustained `{N}` RPS; peak `{M}` RPS.
- **Degradation mode:** `{HTTP 429 with Retry-After | queue of max depth N | shed anonymous | reduce pagination to {limit}}` — cite § 10.

(Repeat for every critical flow. Breakdown slices sum to total.)

---

## § 8. Capacity Envelope

- **Concurrent users:** baseline `{N}`; peak `{M}`; year-out projection `{P}`.
- **RPS per critical endpoint:**
  - `EP-{name}`: baseline `{N}`, peak `{M}`, `{months}-month projection `{P}`
- **Data-volume growth:** baseline `{N} GB/month`; projected total at `{months}` months: `{P} GB` (cite DATA).
- **Hot paths (must stay fast):** `EP-*`, `SAGA-*` — comma-separated list.
- **Resource envelopes per component:**
  - `{ComponentName}` — CPU `{N}` cores, memory `{N}` GB, disk I/O `{N}` MB/s, network `{N}` Mbps

(Component names exact to ARCHITECTURE § 2.)

---

## § 9. Caching Strategy

### `{location — e.g., edge CDN / API in-process / DB query cache}`

- **What is cached:** `{entity / query result / rendered page / session token}`
- **Key format:** `{key template — e.g., repo:{repo_id}:head}`
- **TTL:** `{duration}` or `{event-driven: invalidated on EVT-{name}}`
- **Invalidation trigger:** `{EVT-name}` or `{TTL expiry only}`
- **Fallback on miss:** `{compute-and-populate / fetch-from-origin / stale-while-revalidate}`
- **Coherency guarantee:** `{strong / eventual bounded by TTL / best-effort}`

(Repeat per location. If none: "No caches — every read path hits the system
 of record.")

---

## § 10. Back-Pressure & Load Shedding

### Per-component back-pressure

- `{ComponentName}` signals overload by `{HTTP 429 with Retry-After | slow response bounded at N ms | circuit breaker open}`; cites `ERR_CODE {code}` from ERRORS.md when emitting an error response.

(Repeat per component. If none: "No explicit back-pressure — all
 load-shedding happens at the edge.")

### Load-shedding order

Shed in this priority order (first-shed at top):

1. `{category — e.g., anonymous non-interactive traffic}`
2. `{category — e.g., background batch jobs}`
3. `{category — e.g., low-tier tenants}`
4. `{category — e.g., authenticated interactive}`
5. `{never-shed — e.g., high-tier tenants, payment flows}`

### Circuit-breaker policies

- `{Downstream dependency}` — open at `{failure rate over window}`; half-open probe every `{interval}`; closed on `{N}` consecutive successes. When open, caller `{fails fast with ERR_CODE / falls back to cache / queues with TTL N}`.

(Repeat for every downstream dependency whose failure is anticipated.)

---

## § 11. Benchmarks

### `{EP-name / SAGA-name / SCN-name}`

- **Date measured:** `{YYYY-MM-DD}`
- **Methodology:** load generator `{k6 / wrk / custom}`, concurrency `{N}`, duration `{M}` minutes, percentile window `{rolling W}`.
- **Measured:** p50 `{N}` ms; p95 `{N}` ms; p99 `{N}` ms.
- **RPS at saturation (< 1% error):** `{N}` RPS.
- **Delta to § 7 target:** `{-N ms under budget | +N ms over budget}` — `{action if over}`.

(Repeat per benchmarked flow. If none: "No benchmarks recorded — produced by
 `/system-verify` and recorded back here on regeneration.")

---

## § 12. Relationship to Other Artifacts

- **ARCHITECTURE.md** owns structural decisions; this document quantifies their quality. Every component named in §§ 3, 6, 8, 10 exists in ARCHITECTURE § 2.
- **INTERFACES.md** owns per-endpoint wire shapes; every `EP-name` in §§ 3, 6, 7 exists in INTERFACES § 6. Every `EVT-name` cited here exists in INTERFACES § 7.
- **BEHAVIOR.md** owns state machines and sagas; every `SAGA-name` and `SM-name` cited here exists in BEHAVIOR §§ 1–2.
- **ERRORS.md** owns error codes; every `ERR_CODE` cited here exists in ERRORS § 3. The `log_level` column in ERRORS aligns with § 1's log-level policy.
- **DATA.md** owns the sensitive-column inventory; § 1's forbidden-fields list is a superset of DATA § 8.
- **OPERATIONS.md** owns runbook contents and alert-routing detail; every `RUNBOOK-*` cited in § 6 exists there, and every routing entry aligns with OPERATIONS.
- **SECURITY.md** owns threat-driven alerting (detection rules for STRIDE threats); those alerts are cataloged in SECURITY, not here — this document catalogues reliability and performance alerts.
- **SPECs** cite the `METRIC-*`, `SLO-*`, and § 7 budget entries the unit must satisfy. `/review` verifies the declarations. `/system-verify` re-measures and updates § 11.

---

## § 13. Open Questions

- [ ] {Question — e.g., "PROPOSAL.md states 'fast push latency' without numbers. Propose p95 < 2000 ms over 7 days as `SLO-push-latency-p95` target?"}
  - **Option A:** `p95 < 1000 ms` — aggressive; matches industry leaders; requires significant engineering investment.
  - **Option B:** `p95 < 2000 ms` — conservative; matches current measured baselines per § 11; leaves headroom for feature work.
  - **Recommendation:** Option B, with a ratchet plan to tighten to 1500 ms in the next quarter once § 11 measurements confirm stable sub-1500 ms p95.

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Logging standard (format, required fields, forbidden fields, log-level policy, retention, correlation, redaction)
- Domain event log vocabulary distinct from INTERFACES wire events
- Metrics catalogue with stable `METRIC-*` IDs, label cardinality, SLI flag, emitter, source
- Service-level objectives with stable `SLO-*` IDs, numeric windowed targets, error budgets, burn-rate alerts, scope, use-case traceability
- Tracing strategy — propagation, span inventory per saga and traced SM transition, required tags, sampling, retention
- Alert catalogue with stable `ALERT-*` IDs, trigger, severity, routing, runbook, owner, related SLO
- Performance budgets per critical flow with component/step breakdown and named degradation mode
- Capacity envelope — concurrent users, RPS projections, data-volume growth, hot-path list, resource envelopes per component
- Caching strategy per location (what, key, TTL, invalidation, fallback, coherency)
- Back-pressure per component, load-shedding priority ladder, circuit-breaker policies
- Benchmark baselines with methodology and delta-to-budget
- Cross-artifact relationships with ARCHITECTURE, INTERFACES, BEHAVIOR, ERRORS, DATA, OPERATIONS, SECURITY, SPEC-level
- Genuinely ambiguous SLO targets or capacity numbers surfaced in § 13 Open Questions

### Out of scope

- Runbook contents (step-by-step on-call procedures) — owned by `/operations`. This document names alerts and their intended runbook IDs.
- Log, metric, and trace storage infrastructure (choice of Elasticsearch vs OpenSearch, Prometheus vs VictoriaMetrics, Tempo vs Jaeger) — owned by `/operations`.
- Alert-routing configuration syntax (PagerDuty integrations, Slack webhook URLs) — owned by `/operations`.
- Error code definitions (`code → http_status → user_action → retryable → log_level`) — owned by `/errors`. This document cites codes by string when referencing them in alerts or logs.
- Wire shape of domain events and endpoints (field names, casing, versioning) — owned by `/interfaces`.
- Threat-driven alerting (STRIDE-derived detection rules) — owned by `/security`. Security alerts may be cross-referenced here with a one-line pointer.
- Database schema and index design that achieves the perf budget — owned by `/data`. This document names the budget; `/data` provides the mechanism.
- Implementation syntax — metric client SDKs, try/catch, async/await, goroutines, thread pools — owned by `/spec` and `/implement`.
- UI-level performance (render frame budgets, bundle size budgets, Core Web Vitals) — owned by the per-surface IA skills. This document covers server-side and end-to-end flow budgets.
- Cost modelling, FinOps, resource pricing — not part of the ADD workflow (may live in a separate operational document).

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `metrics`, `slos`, `alerts`, `perf_budgets`, `traced_flows`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "some", "reasonable")
- [ ] § 13 Open Questions is present (empty with "All questions resolved." or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files beyond the referenced registries (ARCHITECTURE § 2, INTERFACES §§ 6–7, BEHAVIOR §§ 1–2, ERRORS § 3, DATA § 8, OPERATIONS)
- [ ] All thirteen sections § 1 through § 13 are present with their exact headings
- [ ] § 1 Logging Standard names format, required fields, forbidden fields, log-levels policy with one example per level, retention, correlation, and redaction rules
- [ ] § 1 forbidden-fields list is a superset of DATA § 8 (or the absence of DATA is surfaced in § 13)
- [ ] § 2 Domain Event Log Vocabulary lists every event as a row with name, emission point, fields, sampling, primary consumers; sorted alphabetically
- [ ] § 3 Metrics Catalogue has every row with `METRIC-*`, type, unit, labels, cardinality, is_sli, emitted_by, source — no blanks; sorted alphabetically by `METRIC-*`
- [ ] Every metric row's `type` is one of `counter`, `gauge`, `histogram`, `summary`; `unit` is concrete (`ms`, `bytes`, `requests`, `ratio`, `count` — never blank or `value`)
- [ ] Every metric label's cardinality is stated as a number, a bounded set, or an explicit unbounded mitigation (exemplars, traces, histogram-over-hash)
- [ ] High-cardinality dimensions (`user_id`, `request_id`, `saga_id`) do not appear as metric labels
- [ ] Every metric row with `is_sli: yes` has a matching `SLO-*` in § 4
- [ ] § 4 every SLO block has ID, SLI definition, numeric windowed target, error budget, burn-rate alert ID, scope, related use cases
- [ ] Every SLO cites at least one `ALERT-*` in § 6 whose trigger is a burn-rate query
- [ ] § 5 Tracing Strategy names propagation format, required tags, sampling strategy, retention, and a span inventory
- [ ] Every `SAGA-*` in BEHAVIOR § 2 has at least one span row in § 5 span inventory
- [ ] § 6 every alert block has ID, name, trigger (numeric and query-level, not prose), severity, routing, runbook, owner, related SLO
- [ ] Every alert links to a `RUNBOOK-*` by name (or surfaces the missing runbook in § 13)
- [ ] § 7 every critical flow has target load, total latency budget (p50/p95/p99), component/step breakdown whose slices sum to the total (with explicit overhead line), throughput target, and named degradation mode
- [ ] § 8 Capacity Envelope names concurrent users (baseline/peak/projection), RPS per critical endpoint, data-volume growth, hot-path list, resource envelopes per ARCHITECTURE § 2 component
- [ ] § 9 every cache entry has location, what-is-cached, key format, concrete TTL (or event-driven), invalidation trigger, fallback, coherency guarantee
- [ ] § 10 has per-component back-pressure, an explicit load-shedding priority ladder, and circuit-breaker policies per anticipated downstream failure
- [ ] § 11 Benchmarks has a block per recorded baseline (or states "No benchmarks recorded" verbatim)
- [ ] § 12 Relationship to Other Artifacts names ARCHITECTURE, INTERFACES, BEHAVIOR, ERRORS, DATA, OPERATIONS, SECURITY, and SPEC-level artifacts
- [ ] Every `METRIC-*`, `SLO-*`, and `ALERT-*` ID is unique and stable; retired IDs retain their row with `(retired — superseded by {new-id})`
- [ ] Every `EP-name` cited exists in INTERFACES § 6; every `EVT-name` cited exists in INTERFACES § 7; every `SAGA-name` and `SM-name` cited exists in BEHAVIOR; every `ERR_CODE` cited exists in ERRORS § 3; every component name cited exists in ARCHITECTURE § 2; every `UC-NN` cited exists in USE_CASES.md
- [ ] No implementation syntax (grep-check: `try`, `catch`, `async`, `await`, `goroutine`, `thread`, `mutex`) and no wire-schema design (grep-check: `JSON body`, `camelCase`, `snake_case`, `column`, `table`)
- [ ] Frontmatter counts match the body: `metrics`, `slos`, `alerts`, `perf_budgets`, `traced_flows`, `open_questions`
- [ ] `status` is `complete` if § 13 is "All questions resolved." and `has_open_questions` otherwise
- [ ] Document length is within the 400–1200 line target (hard cap 1500)
