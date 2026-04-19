---
name: OPERATIONS.md
description: Consolidate the system's operations contract — deployment topology per environment, CI/CD pipelines with promotion and rollback, environment strategy, config catalogue with stable `CFG_NAME` IDs, secrets inventory with rotation policies, integration catalogue with failure behaviour, runbooks for every alert, disaster recovery with concrete RTO/RPO, observability infrastructure, change management, and an initial-setup runbook. Use when asked to design operations and deployment, plan CI/CD and environments, write runbooks and the config catalogue, document integration failure behaviour, or produce an OPERATIONS.md.
---

# Task: Generate OPERATIONS.md — Deployment, Config, Runbooks, Integrations

## Objective

Produce an OPERATIONS.md that serves as the single source of truth for how the system runs, ships, is configured, integrates, and is recovered: per-environment deployment topology (compute, region, network, data plane, per-component placement, scaling behaviour), CI/CD pipelines with stages, gates, artifacts, promotion, and rollback, environment strategy (config differences, data strategy, access control), a config catalogue under stable `CFG_NAME` IDs with owner and scope per entry, a secrets inventory with storage, rotation policy, read/rotate principals, and audit trail, an integration catalogue with auth, SLA, failure behaviour, and privacy notes for every external dependency, a runbook entry for every alert named in QUALITY, a disaster-recovery plan with concrete RTO/RPO numbers and a step-by-step restore procedure, observability infrastructure (where logs, metrics, traces land — *what* is emitted lives in QUALITY), change management, and an initial-setup runbook executable as-is by a new operator. An agent reading this document alone can — for any environment, alert, config var, secret, or external integration — name the compute shape, the config delta, the rotation schedule, the failure mode, or the on-call response, without opening ARCHITECTURE, INTERFACES, DATA, QUALITY, or SECURITY.

OPERATIONS.md consolidates what older guidance split into DEPLOYMENT.md, RUNBOOK.md, CONFIG_CATALOG.md, and INTEGRATION_CATALOG.md, because every reader doing ops work reads them together: the on-call engineer opens a runbook, needs the config var to flip, sees which integration is degraded, and checks the rollback path — all in one page. The defining discipline — and the commonest violation — is **every alert in QUALITY § 6 has a § 7 runbook entry here, every config var has a stable `CFG_NAME`, every external integration has a named failure behaviour, every production secret has a rotation policy**. Orphan alerts page humans with no script; orphan config vars rename without deprecation and break production; orphan integrations fail silently at 3am; secrets that never rotate are the easiest persistent-access path an attacker has.

---

## Inputs

1. **ARCHITECTURE.md** (required) — § 2 Components supplies the list of things that run in the deployment topology; § 6 (or equivalent) deployment-shape headline frames § 1; every `ADR-NN` on runtime choice (Kubernetes vs Cloud Run vs bare metal, single-region vs multi-region, push vs pull deployment) is cited verbatim here.
2. **INTERFACES.md** (required) — § 1 Boundaries supplies the inventory of external dependencies that drive § 6 Integration Catalogue; § 6 Endpoints supply the inbound surfaces whose ingress / egress shape § 1 Network; § 7 Events supply the event bus dependencies.
3. **DATA.md** (required) — datastore list, provisioning shape, migration mechanism, and retention windows drive the data-plane rows of § 1, the backup/restore in § 8, and the per-environment data strategy in § 3.
4. **QUALITY.md** (required) — § 6 Alert Catalogue is the authoritative list of alerts; every `ALERT-*` there has a matching runbook entry in § 7 here. § 9 Observability Infrastructure references storage locations whose concrete choices live here.
5. **SECURITY.md** (optional — required if produced) — § 9 Secret & Credential Management names secret classes and rotation policies; § 5 and § 11 drive audit trail and abuse-prevention config vars; § 13 Incident Response references paging destinations this document pins.
6. **BEHAVIOR.md** (optional) — § 2 Sagas frame integration-failure compensation paths that § 6 Integration Catalogue cites in its Failure-Behaviour rows.
7. **Existing OPERATIONS.md or deployment docs** (auto-discovered — only if refining) — read fully. Every assigned `CFG_NAME`, `RUNBOOK-*`, and integration entry is permanent. New entries take new unused IDs. Retired entries retain their row with `(retired — superseded by {new-id})`.

Read set size: 4 required artifacts + SECURITY and BEHAVIOR when present + optional prior OPERATIONS. Read all required inputs end-to-end. Truncated reads cause three specific failure modes: runbooks missing for alerts QUALITY catalogs (QUALITY § 6 not fully read — produces blind paging), integration catalogue missing services INTERFACES § 1 names (INTERFACES not fully read — produces "why did this fail at 3am" surprises), and backup strategy missing datastores DATA lists (DATA not fully read — produces data-loss blind spots).

---

## Workflow

Operations-contract construction proceeds in six phases: deployment and CI/CD, config and secrets, integrations, runbooks, disaster recovery and observability, change management and setup. Phases are sequential — later phases cite IDs introduced by earlier phases — but revisit earlier phases if a later one reveals a missing environment, an un-routed alert, or an un-rotated secret.

### Phase 1: Deployment Topology, CI/CD, Environment Strategy

Produce §§ 1–3 together — they share the environment axis.

§ 1 Deployment Topology has one block per environment (typically `dev` / `staging` / `prod`; additional environments like `sandbox`, `preview`, `dr` get their own block when they exist). Each block fills the Output Format template: purpose and scope (who accesses, what traffic), compute (cloud provider, service type — Kubernetes / Cloud Run / Lambda / EC2 / bare metal — instance sizes), region strategy (single-region / multi-region / active-passive), network (VPCs, subnets, firewalls, ingress, egress policies), data plane (managed DBs, object storage, queues, caches — every row citing DATA), per-component mapping (which ARCHITECTURE § 2 component runs in which compute resource), scaling behaviour (horizontal / vertical, auto-scaling triggers). Every component from ARCHITECTURE § 2 appears in at least one environment's per-component mapping, or surfaces in § 13 as a missing-placement open question. A Mermaid diagram or an external link per environment illustrates the layout.

§ 2 CI/CD Pipelines has one block per pipeline. Each block fills repo and branch, stages (lint → test → build → deploy) with the tool family per stage (not specific product versions — those are `/implement` concerns), gates (required checks, approvals, review counts), artifacts (Docker image tag format, release version scheme — `v{major}.{minor}.{patch}` plus commit SHA), promotion model (staging → prod: auto with guardrails / manual with two-approver / canary with percentage ramp), rollback mechanism (image tag pinning / feature-flag flip / blue-green switch / DB-migration reverse script). Every environment in § 1 is reachable by exactly one pipeline path (or has an explicit "bootstrap-only, no CI deploy" annotation).

§ 3 Environment Strategy has three fixed subsections. Config difference matrix: a table of what varies per environment, citing `CFG_NAME` entries from § 4 (domains, rate limits, feature flags, retention windows). Data strategy per environment: `dev` = synthetic, `staging` = production clone (scrubbed — cite § 5 and SECURITY § 10 for scrubbing rules), `prod` = production. Access control per environment: who can deploy to what, who can read production logs, who can execute § 7 runbook commands — cites SECURITY § 8 roles where available.

### Phase 2: Config Catalogue & Secrets Inventory

§ 4 is the master config catalogue — one row per env var, config flag, and feature flag, alphabetical by `CFG_NAME`. ID form: `CFG_{SHOUTY_SNAKE_CASE}` — the name *is* the ID. Columns follow the Output Format template: name, type (`string` / `int` / `bool` / `duration` / `url` / `enum:value1|value2`), default, scope (`process` / `tenant` / `user` / `global`), secret? (`yes` / `no` — if `yes`, also appears in § 5), per-env override? (`yes` / `no`), owner (team from ARCHITECTURE § 2 or product/platform/security), used by (components from ARCHITECTURE § 2 that read it), purpose (one sentence, domain-level).

Feature flags — typed `bool`, conventionally named `CFG_FEATURE_*` or `CFG_FLAG_*` — carry an explicit retirement date (or rationale for permanence — `permanent: kill-switch for payment dependency outage`). Flags without retirement become tech debt.

§ 5 Secrets Inventory has one block per secret class. Each block fills storage (secret-manager path / env-var-injected-from-secret-manager — product-name choices live in § 9 for observability storage; for secrets, cite by abstract class `secret manager` in this document), owner (team/role), rotation policy (frequency, trigger — calendar-triggered `90 days` / event-triggered `on offboarding` / breach-triggered `on any incident classified sev2 or above`; `never` is an unacceptable answer for a production secret), access control (who can read, who can rotate, who can decrypt), audit trail (where access is logged — cite QUALITY § 2 `EVT-*` when available). SECURITY § 9 drives this section where present; in its absence, derive from the integration catalogue (every OAuth client, every API key, every signing key gets a row).

### Phase 3: Integration Catalogue

§ 6 has one block per external service the system depends on. Derive the master list from INTERFACES § 1 Boundaries (every external boundary is a candidate) plus BEHAVIOR § 2 Sagas (every saga step that calls out to a third party). Each block fills provider (company or product name — here specific product names are legitimate because the integration *is* with that specific product), purpose (what the system uses it for, in domain terms), auth (API key / OAuth / mTLS — cite SECURITY § 9 when present), endpoints used (from the external service's API — name the paths or webhook event types), rate limits and quotas provided by the provider, SLA published by the provider, failure behaviour when the provider is unavailable (degrade mode — specific mechanism: fail fast with `ERR_CODE`, queue with TTL, fall back to cached last-known-good, disable the feature — cite `SAGA-{name}` from BEHAVIOR when compensating logic lives there), cost model (pay-per-request / subscription tier / free tier limits), data sent (what the system shares with the provider — a privacy statement citing DATA § 8 sensitivity classes), contact and escalation (provider support tier, escalation email, account owner).

"Will figure it out when it happens" is not a failure behaviour. A row with no failure behaviour is a modelling bug — either specify a mechanism or explicitly mark `acceptable to fail open: {business justification}`.

### Phase 4: Runbook

§ 7 has one entry per alert in QUALITY § 6 Alert Catalogue (one-to-one by `ALERT-*` ID), plus component-level entries for symptoms that do not map to a single alert (`database is slow but no alert fired`, `deployment wedged mid-rollout`). Each runbook entry fills the Output Format template: severity (`sev1` / `sev2` / `sev3` — matches the alert's severity), symptoms (what the on-call sees in the dashboard, pager text, or user report), quick verification (exact commands or queries to confirm the issue is real and not a false positive), first response (immediate actions to stabilise — typically "check dashboard link X, verify Y, if true then Z"), diagnostic steps (numbered investigation sequence with decision branches — "If metric X > threshold, jump to step 7"), fix patterns (common fixes for common causes, each citing the specific `CFG_NAME` to flip or the specific component to restart), escalation (when and to whom — named team rotation or individual role), related alerts and metrics (cite `METRIC-*`, `SLO-*`, `ALERT-*` from QUALITY).

Every runbook entry has a stable `RUNBOOK-{name}` ID used by QUALITY § 6 to link from the alert. Missing `RUNBOOK-*` names surface in § 13.

If § 7 threatens to exceed the body's size budget, extract per-component runbooks into `runbooks/{component}.md` files and keep § 7 as an index — one row per alert with a file link. Do not extract when the skill's body is still well under budget; inline is the reader's preferred shape.

### Phase 5: Disaster Recovery, Observability Infrastructure

§ 8 Disaster Recovery pins concrete numbers. RTO (recovery time objective — how long until the system is back up) and RPO (recovery point objective — how much data loss is acceptable) are stated in minutes or hours per scenario, citing ARCHITECTURE § 6 headlines where they exist. "Best effort" is not an RTO. Backup strategy: what is backed up (every datastore from DATA — one row each), frequency (`hourly snapshot + daily full + weekly cold`), retention (`30 days hot, 1 year cold`), storage location (separate region, separate cloud account, separate provider for resilience). Restore procedure: numbered steps with expected duration per step, validation at the end. Failure scenarios covered (region loss, datastore corruption, supply-chain compromise, simultaneous admin-credential compromise and database corruption — name at minimum three scenarios). DR drills: frequency (`quarterly tabletop, annual full`), scope, responsibility.

§ 9 Observability Infrastructure is the *where*, distinct from QUALITY's *what*. Log aggregation (tool family — `search-based log platform`, not specific vendor unless pinned — retention duration). Metric storage (tool family, retention, cardinality envelope). Trace storage (tool family, retention, sampling strategy — cite QUALITY § 5). Dashboard platform (tool family, access control citing § 3). When specific product choices are pinned (a deliberate ADR says `Grafana + Prometheus + Tempo`), state them and cite the `ADR-NN`; when unpinned, name the category and surface the choice in § 13.

### Phase 6: Change Management, Initial Setup, Validation

§ 10 Change Management covers deployment windows (explicit freeze periods — `no production deploys Fri 14:00 UTC through Mon 08:00 UTC`), change notification (who needs to know — dependency-team channel, customer-facing status page), rollback criteria (concrete: `p95 latency > 2× baseline for 5 minutes triggers automatic rollback`; or `any sev1 pager fires triggers automatic rollback`), post-incident review (cadence, template, action-item tracking).

§ 11 Initial Setup Runbook is the zero-to-running-service procedure for a new operator. Prerequisites (accounts, tools, access grants), ordered setup steps (each a numbered command or click-path), smoke tests at the end (exact commands whose success confirms the setup worked), links to § 7 for ongoing ops. Test the procedure end-to-end mentally: a reader with no prior context should be able to follow it.

§ 12 Relationship to Other Artifacts is one bullet per relationship in the fixed order: ARCHITECTURE, INTERFACES, DATA, QUALITY, SECURITY, BEHAVIOR, ERRORS, SPEC-level, `/system-verify`.

§ 13 Open Questions collects genuine ambiguity — staging sizing (same as prod for fidelity vs cost-optimised for budget), pinned vs unpinned tool choices, integration contact channels when provider contracts are in flux — each with options, tradeoffs, and a recommendation.

Before finalising, run the Quality Checklist below end-to-end. Update frontmatter counts to match the body exactly. `status` is `complete` if § 13 reads "All questions resolved.", `has_open_questions` otherwise.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Every alert in QUALITY § 6 has a § 7 runbook entry

QUALITY § 6 is authoritative for the alert inventory. Every `ALERT-*` there is matched one-to-one by a `RUNBOOK-*` entry in § 7 here. An alert with no runbook routes to a pager with no script — the commonest operational failure mode for young systems. Exceptions must be explicit: `ALERT-diagnostic-only` annotated `No runbook — diagnostic alert, never paged, fires only to a quiet channel`.

### 2. Every config var has a stable `CFG_NAME`

`CFG_NAME` IDs are SHOUTY_SNAKE_CASE strings that *are* the config var's name (`CFG_DATABASE_URL`, `CFG_FEATURE_FORKS`). The `CFG_` prefix covers every config var, including feature flags. Once assigned, a `CFG_NAME` never renames without a deprecation window: both old and new rows appear in § 4 simultaneously for at least one release cycle, the old marked `(deprecated — use CFG_NEW_NAME; removed YYYY-MM-DD)`. Silent rename breaks every deployment that reads the old name.

### 3. Every external integration has a named failure behaviour

Every row in § 6 carries a concrete mechanism for how the system behaves when the external service is unavailable: fail fast with a specific `ERR_CODE`, queue with a specific TTL, fall back to cached last-known-good with a specific staleness bound, disable the feature with a specific user-facing message, compensate via a specific `SAGA-{name}`. "Graceful degradation" alone is not a mechanism. "Will figure it out when it happens" is a modelling failure.

### 4. Every production secret has a rotation policy

Every row in § 5 carries a rotation policy with a concrete cadence or trigger: calendar (`90 days`), event (`on offboarding`, `on privilege change`), or incident (`immediate on any sev2 or above touching the secret's component`). `never` is forbidden as a rotation policy for production secrets; long-lived credentials whose rotation is a business decision must carry an explicit `(rotation deferred: {rationale, review date YYYY-MM-DD})` annotation and surface in § 13.

### 5. Every feature flag has a retirement date or rationale for permanence

Feature flags (typed `bool`, conventionally named `CFG_FEATURE_*` or `CFG_FLAG_*`) either carry a retirement date (`retire 2026-09-30 once migration is complete`) or an explicit permanence rationale (`permanent: kill-switch for payment provider outage`). Flags that never retire silently become tech debt that blocks refactoring. The retirement date is a commitment — missing it is a § 13 escalation.

### 6. DR targets are concrete numbers

§ 8 RTO and RPO are stated as numeric values with units per scenario: `RTO 30 minutes`, `RPO 15 minutes`, `RTO 4 hours` for a cross-region rebuild. "Best effort", "as soon as possible", "minimal data loss" are modelling failures. If the business has not yet committed to numbers, state a concrete proposal and surface the acceptance question in § 13.

### 7. Every component from ARCHITECTURE § 2 has a § 1 placement in at least one environment

§ 1's per-component mapping is complete relative to ARCHITECTURE § 2. A component that ARCHITECTURE names but § 1 never places is unreachable in production or a docs drift. Surface the missing placement in § 13 or correct ARCHITECTURE; never silently omit.

### 8. Config difference matrix cites by `CFG_NAME`

§ 3's config-difference matrix references config vars by `CFG_NAME` only — never by restated value or by prose name. "`CFG_RATE_LIMIT_RPS`: dev=1000, staging=1000, prod=100" is valid; "`Rate limit`: higher in dev than prod" is not. The matrix is machine-parseable; prose is not.

### 9. Integration rows cite `ERR_CODE` and `SAGA-{name}` when applicable

Failure behaviour that returns an error to a caller names the specific `ERR_CODE` from ERRORS.md. Failure behaviour that involves a multi-step compensation names the specific `SAGA-{name}` from BEHAVIOR § 2. Generic "returns an error" or "rolls back" is unverifiable. If ERRORS or BEHAVIOR has not yet produced the referenced ID, surface in § 13.

### 10. Observability infrastructure names retention and access control

§ 9 entries state concrete retention durations (`30 days hot, 1 year cold`), not prose (`long enough`). Access control states a role or team name from SECURITY § 8 when available, or a specific grant list when SECURITY has not been produced (and surfaces the pending SECURITY dependency in § 13).

### 11. Initial setup runbook is executable as-is

§ 11 is a zero-context operator's sole guide. Every step is a concrete command or click-path; every prerequisite is a named account, tool, or access grant; every smoke test is a specific command with an expected output. "Set up the environment" is not a step; "Install `git` (`brew install git` on macOS, `apt install git` on Debian)" is. If the skill's agent cannot mentally walk the procedure to green, § 11 is incomplete.

### 12. No wire-schema design

OPERATIONS names deployment shape, config var names, secret classes, integration contracts, and runbook procedures. It does not design HTTP request/response field names (INTERFACES), error code strings (ERRORS), or database column names (DATA). An OPERATIONS.md that enumerates JSON field shapes for an integration is out of scope — cite the `EP-name` or INTERFACES § N reference and keep the shape there.

### 13. No implementation syntax

No code snippets (try / catch / async / await / goroutine). Shell commands in § 7 runbooks and § 11 setup are legitimate because they *are* the operational interface; application code is not. Specific SDK product-version pins are `/implement` concerns; OPERATIONS pins the tool *family* or — when an ADR commits — the product name citing the `ADR-NN`.

### 14. Single YAML frontmatter block

One YAML block at the top containing common fields (`skill`, `date`, `status`) and operations-specific counts (`environments`, `components_deployed`, `config_vars`, `integrations`, `runbook_entries`, `open_questions`). Never emit a second block. Counts match the body exactly.

### 15. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "various", "and so on", "industry-standard", "best practice", "reasonable", "sufficient". Use exact `CFG_NAME` strings, exact durations (`90 days`, not `periodic`), exact numeric thresholds (`p95 > 2× baseline for 5 minutes`, not `significant latency regression`), exact commands. Unresolvable ambiguity surfaces in § 13 Open Questions with options, tradeoffs, and a recommendation.

---

## Output Format

```markdown
---
skill: OPERATIONS.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
environments: {N}
components_deployed: {N}
config_vars: {N}
integrations: {N}
runbook_entries: {N}
open_questions: {N}
---

# OPERATIONS — {ProductName}

> Consolidated operations contract. Every environment, CI/CD pipeline, config
> var, secret, external integration, runbook, DR target, observability store,
> and setup procedure lives here. Downstream artifacts cite by stable
> `CFG_NAME` and `RUNBOOK-*` IDs. What to emit (logs, metrics, traces) and what
> to alert on lives in QUALITY.md; wire shapes in INTERFACES.md; threat model
> in SECURITY.md; datastore schema and retention in DATA.md.

## § 1. Deployment Topology

### Environment: `{dev | staging | prod | ...}`

- **Purpose & scope:** {who accesses, what traffic — "engineering-only, synthetic data, no customer traffic" / "customer-facing, all traffic"}
- **Compute:** {cloud provider / service type — Kubernetes cluster `{name}` on {provider}; Cloud Run services; Lambda functions; bare-metal pool} — instance sizes per node pool
- **Region strategy:** {single-region `{region}` / multi-region active-active `{region list}` / active-passive with failover from `{primary}` to `{secondary}`}
- **Network:**
  - VPC(s): `{vpc-name}` in `{region}`
  - Subnets: `{public / private / data}` with CIDR ranges
  - Firewalls / security groups: `{summary of ingress / egress rules}`
  - Ingress: `{load balancer + WAF — named by family, cite ADR-NN if product-pinned}`
  - Egress policies: `{allow-list of external domains / open egress with monitoring}`
- **Data plane:**
  - `{managed DB / object store / queue / cache}` — `{provider shape — e.g., "PostgreSQL 15, primary + 1 read replica, 200 GB"}` (cite DATA § {N})
- **Per-component mapping:**
  - `{ComponentName from ARCHITECTURE § 2}` → `{compute resource}` — `{instance count / size}`
- **Scaling behaviour:**
  - `{ComponentName}`: `{horizontal / vertical}`, `{trigger — CPU > 70% sustained 5m / queue depth > 100 / manual}`, `{min / max}`

```mermaid
flowchart LR
    Client --> LB[Load Balancer]
    LB --> Component1[{ComponentName}]
    Component1 --> DB[({DatabaseName})]
```

(Repeat the full block for every environment.)

---

## § 2. CI/CD Pipelines

### Pipeline: `{pipeline-name}`

- **Repo / branch:** `{repo-url}` — `{main / release-*}`
- **Stages:**
  1. `{stage name}` — tool family `{linter / test runner / builder / deployer}` — purpose
  2. `{stage name}` — tool family — purpose
- **Gates:**
  - `{required check — e.g., "all tests pass"}`
  - `{required approval — e.g., "two-approver review for main-branch merges"}`
- **Artifacts:**
  - `{Docker image tag format — e.g., "{service}:{version}-{commit-sha}"}`
  - `{release version scheme — e.g., "v{major}.{minor}.{patch} via SemVer"}`
- **Promotion model:** `{dev → staging: auto on main merge / staging → prod: manual with two-approver / canary with 10% ramp over 30m before full rollout}`
- **Rollback mechanism:** `{image tag pinning via manifest revert / feature-flag flip CFG_NAME / blue-green switch / DB-migration reverse script}`

(Repeat per pipeline. Every environment in § 1 is reachable by exactly one
 pipeline path or annotated `bootstrap-only — no CI deploy`.)

---

## § 3. Environment Strategy

### § 3.1. Config Difference Matrix

| `CFG_NAME` | dev | staging | prod |
|------------|-----|---------|------|
| `CFG_{NAME}` | `{value}` | `{value}` | `{value}` |

(Only config vars that differ across environments appear here. All are cited
 by `CFG_NAME` and defined in § 4.)

### § 3.2. Data Strategy per Environment

- **dev:** `{synthetic seed data / empty / developer-provided fixtures}`
- **staging:** `{production clone scrubbed per SECURITY § 10 / subset of production / synthetic at scale}`
- **prod:** `{live production data}`

### § 3.3. Access Control per Environment

- **dev:** `{all engineers can deploy / read / execute runbooks}`
- **staging:** `{engineers can deploy with PR approval; ops can execute destructive runbook commands}`
- **prod:** `{on-call engineers + named roles from SECURITY § 8}`

---

## § 4. Config Catalogue

| `CFG_NAME` | type | default | scope | secret? | per-env override? | owner | used by | purpose |
|------------|------|---------|-------|---------|-------------------|-------|---------|---------|
| `CFG_{NAME}` | `{string / int / bool / duration / url / enum:A\|B}` | `{value / none}` | `{process / tenant / user / global}` | `{yes / no}` | `{yes / no}` | `{team}` | `{ComponentName, ComponentName}` | `{one sentence}` |
| `CFG_FEATURE_{NAME}` | `bool` | `false` | `{global / tenant / user}` | `no` | `yes` | `{team}` | `{ComponentName}` | `{one sentence}`; retire `{YYYY-MM-DD}` or `permanent: {rationale}` |

(Alphabetical by `CFG_NAME`. One row per config var or feature flag. No blanks.
 Retired entries retain their row with `(deprecated — use CFG_NEW_NAME; removed YYYY-MM-DD)`.)

---

## § 5. Secrets Inventory

### `{SecretClass — e.g., "Service-to-service JWT signing key"}`

- **Storage:** `{secret-manager path — e.g., "secret-manager://prod/jwt-signing-key"}`
- **Owner:** `{team / role}`
- **Rotation policy:** `{cadence — "90 days" / trigger — "on offboarding" / incident — "immediate on any sev2 or above touching ComponentName"}`
- **Principals with read access:** `{role list from SECURITY § 8}`
- **Principals with rotate access:** `{role list}`
- **Audit trail:** `{EVT-name from QUALITY § 2 or explicit "no audit trail yet — § 13"}`

(Repeat per secret class. Cover at minimum: service-to-service credentials,
 encryption keys, webhook signing keys, third-party API tokens. Drive this
 section from SECURITY § 9 when produced.)

---

## § 6. Integration Catalogue

### `{ProviderName — e.g., "Stripe"}`

- **Purpose:** `{what the system uses it for, in domain terms — "payment capture and refund for UC-{NN}"}`
- **Auth:** `{API key / OAuth 2.0 client credentials / mTLS — cite SECURITY § 9 secret class when produced}`
- **Endpoints used:** `{path list or webhook event types — e.g., "POST /v1/charges, POST /v1/refunds; webhooks: charge.succeeded, charge.failed"}`
- **Rate limits / quotas:** `{provider-documented — e.g., "100 requests/sec per account, burst 500"}`
- **SLA:** `{provider-published — e.g., "99.99% monthly uptime per provider terms"}`
- **Failure behaviour:** `{concrete mechanism — "fail fast with ERR_PAYMENT_UNAVAILABLE, user sees retry prompt; SAGA-checkout compensates by releasing the cart hold after 15 minutes"}`
- **Cost model:** `{pay-per-request — "$0.029 + $0.30 per charge" / subscription — "tier $X/month up to N requests" / free tier — "N requests/month free"}`
- **Data sent:** `{what is shared — "payment instrument token, charge amount, currency, idempotency key; no PII beyond billing address when provided"}` (cite DATA § 8)
- **Contact / escalation:** `{provider support tier + email + account owner on our side}`

(Repeat per external integration. Derive the list from INTERFACES § 1 external
 boundaries plus BEHAVIOR § 2 sagas with external steps.)

---

## § 7. Runbook

### `RUNBOOK-{name}`

- **Linked alert:** `ALERT-{name}` (cite QUALITY § 6) — or `(component-symptom only — no alert)`
- **Severity:** `sev1 / sev2 / sev3`
- **Symptoms:** `{what the on-call sees — "dashboard X shows Y > threshold for 5 minutes; pager text reads Z; users report W"}`
- **Quick verification:** `{exact commands or queries to confirm — e.g., "kubectl get pods -n payments | grep CrashLoopBackOff"}`
- **First response:** `{immediate stabilising action — e.g., "check status page at URL; if provider-side, post to status-channel and wait; if our side, continue"}`
- **Diagnostic steps:**
  1. `{step with decision branch — "run X; if output contains Y, jump to step 4; else continue"}`
  2. `{step}`
- **Fix patterns:**
  - **Cause: `{common cause}`** — fix: `{specific action — e.g., "flip CFG_RATE_LIMIT_RPS to 50, wait 2 minutes, observe METRIC-push-duration-ms"}`
- **Escalation:** `{when and to whom — "if unresolved after 30 minutes, page {team-name}; if data-integrity suspected, page security on-call"}`
- **Related alerts / metrics:** `METRIC-{name}`, `SLO-{name}`, `ALERT-{name}` (cite QUALITY)

(Repeat per alert from QUALITY § 6. Every `ALERT-*` has exactly one `RUNBOOK-*`
 here. Component-level entries for symptoms without alerts are separately
 listed. If § 7 exceeds the body's size budget, extract per-component runbooks
 to `runbooks/{component}.md` and keep a one-row-per-runbook index here.)

---

## § 8. Disaster Recovery

- **RTO / RPO per scenario:**
  - `{scenario — e.g., "single-AZ loss"}`: RTO `{N minutes}`, RPO `{N minutes}` (cite ARCHITECTURE § {N})
  - `{scenario — e.g., "full region loss"}`: RTO `{N hours}`, RPO `{N minutes}`
  - `{scenario — e.g., "datastore corruption"}`: RTO `{N hours}`, RPO `{last successful backup}`
- **Backup strategy:**
  - `{DatastoreName from DATA}`: frequency `{hourly snapshot + daily full + weekly cold}`, retention `{30 days hot, 1 year cold}`, location `{separate region / separate cloud account}`
- **Restore procedure:**
  1. `{step — "identify last-known-good backup timestamp; verify integrity via {command}"}` — expected duration `{N minutes}`
  2. `{step — "halt writes on primary by flipping CFG_FEATURE_READ_ONLY_MODE"}` — expected duration `{N minutes}`
  3. `{step}` — expected duration `{N}`
  - **Validation at end:** `{commands to confirm system is healthy post-restore — e.g., "run smoke test suite X; verify checksum of sample dataset"}`
- **Failure scenarios covered:** `{region loss, datastore corruption, supply-chain compromise, admin-credential compromise combined with datastore corruption}`
- **DR drills:** frequency `{quarterly tabletop + annual full-restore rehearsal}`, scope `{what is drilled}`, responsibility `{role or team}`

---

## § 9. Observability Infrastructure

- **Log aggregation:** tool family `{search-based log platform — name product if ADR-NN pins one}`, retention `{30 days hot, 1 year cold}`, access control `{role from SECURITY § 8 or grant list}`
- **Metric storage:** tool family `{time-series database — name product if pinned}`, retention `{13 months}`, cardinality envelope `{target total active series: N}` (cite QUALITY § 5 for label cardinality rules)
- **Trace storage:** tool family `{distributed-tracing backend}`, retention `{7 days hot, 30 days cold}`, sampling `{cite QUALITY § 5}`
- **Dashboard platform:** tool family `{dashboards-as-code platform}`, access control `{role from SECURITY § 8}`

(Where a specific product is committed by an ADR, state it and cite `ADR-NN`.
 Where unpinned, name the category and surface the choice in § 13.)

---

## § 10. Change Management

- **Deployment windows:** `{explicit freeze — "no production deploys Fri 14:00 UTC through Mon 08:00 UTC; no deploys on major holidays"}`
- **Change notification:** `{who / how — "dependency-team channel on every prod deploy; customer-facing status page on sev1-eligible changes"}`
- **Rollback criteria:** `{concrete triggers — "any sev1 pager fires within 10m of deploy → automatic rollback; p95 latency > 2× 24h baseline for 5m → automatic rollback"}`
- **Rollback procedure:** `{step-by-step — "trigger pipeline rollback via command X; confirm image tag reverted via Y; post to change-log"}`
- **Post-incident review:** cadence `{within 5 business days of any sev1 or sev2}`, template `{link or inline outline}`, action-item tracking `{issue tracker lane / review cadence}`

---

## § 11. Initial Setup Runbook

> Zero-to-running-service for a new operator. Executable as-is.

### § 11.1. Prerequisites

- Accounts: `{cloud provider account with role R; source-repo access with role R; secret-manager read-grant to secret class C}`
- Tools: `{named tool — install command per OS}`
- Access grants: `{named grants — "VPN profile; on-call schedule invite; status-page writer role"}`

### § 11.2. Ordered Setup Steps

1. `{step with exact command — "clone repo: git clone {url}"}`
2. `{step — "authenticate to cloud: {command}"}`
3. `{step — "bootstrap local config: cp .env.example .env and populate CFG_DATABASE_URL from secret-manager path X"}`
4. `{step}`

### § 11.3. Smoke Tests

- `{command}` — expected output: `{string / condition}`
- `{command}` — expected output: `{string}`

### § 11.4. Links

- On-call duties: § 7 Runbook
- Config reference: § 4 Config Catalogue
- Incident response: SECURITY § 13 (when produced)

---

## § 12. Relationship to Other Artifacts

- **ARCHITECTURE.md** owns components and `ADR-NN` runtime decisions; § 1 places every ARCHITECTURE § 2 component; § 8 DR targets cite ARCHITECTURE § 6 headlines.
- **INTERFACES.md** owns boundaries and endpoints; § 6 Integration Catalogue covers every external boundary from INTERFACES § 1; § 1 Network ingress shapes match the surfaces in INTERFACES § 6.
- **DATA.md** owns datastore schema and retention; § 1 Data plane and § 8 Backup strategy cite DATA by datastore name; § 3.2 data strategy per environment respects DATA retention policies.
- **QUALITY.md** owns what to emit and what to alert on; § 7 has one `RUNBOOK-*` per `ALERT-*` in QUALITY § 6; § 9 observability infrastructure stores what QUALITY specifies.
- **SECURITY.md** owns threats, controls, and secret classes; § 5 secrets inventory aligns with SECURITY § 9; § 3 access control cites SECURITY § 8 roles; § 6 integration auth cites SECURITY § 9 when produced.
- **BEHAVIOR.md** owns sagas and compensations; § 6 integration failure behaviour cites `SAGA-{name}` when compensating logic lives there.
- **ERRORS.md** owns error-code strings; § 6 integration failure-behaviour rows and § 7 runbook fix patterns cite `ERR_CODE` by string.
- **SPECs** for ops-touching units cite `CFG_NAME`, `RUNBOOK-*`, and integration entries their unit reads or owns; `/review` verifies the declarations.
- **/system-verify** bootstraps § 11 to spin up a full stack and checks § 7 runbooks are executable end-to-end.

---

## § 13. Open Questions

- [ ] `{Question — e.g., "Is staging sized same as prod for regression fidelity, or cost-optimised?"}`
  - **Option A:** `{description — e.g., "Same-as-prod staging ensures perf regressions surface before customer traffic."}` — `{tradeoff: cost roughly doubles}`
  - **Option B:** `{description — e.g., "Cost-optimised staging (¼ prod scale) with synthetic perf tests in a dedicated perf environment."}` — `{tradeoff: occasional perf regression reaches prod}`
  - **Recommendation:** `{suggestion and reasoning}`

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Deployment topology per environment (compute, region, network, data plane, per-component placement, scaling behaviour)
- CI/CD pipelines (stages, gates, artifacts, promotion, rollback)
- Environment strategy (config difference matrix, data strategy, access control)
- Config catalogue with stable `CFG_NAME` IDs, owner, scope, secret flag, per-env override, purpose
- Feature flags with retirement date or permanence rationale
- Secrets inventory (storage, owner, rotation policy, access control, audit trail — names and policies, not values)
- Integration catalogue (provider, auth, endpoints, rate limits, SLA, failure behaviour, cost model, data sent, contact)
- Runbook (one `RUNBOOK-*` per QUALITY `ALERT-*` plus component-level entries) with severity, symptoms, verification, first response, diagnostic steps, fix patterns, escalation
- Disaster recovery with concrete RTO / RPO per scenario, backup strategy per datastore, step-by-step restore procedure, DR drills
- Observability infrastructure (where logs / metrics / traces land, retention, access control — what to emit lives in QUALITY)
- Change management (deployment windows, rollback criteria, post-incident review)
- Initial-setup runbook executable as-is by a new operator
- Cross-artifact relationships with ARCHITECTURE, INTERFACES, DATA, QUALITY, SECURITY, BEHAVIOR, ERRORS, SPEC-level
- Genuinely ambiguous sizing, tooling, or policy decisions surfaced in § 13 Open Questions

### Out of scope

- What signals to emit (log fields, metric definitions, trace span shapes) — owned by `/quality`. This document stores them; QUALITY specifies them.
- SLO definitions, alert conditions, burn-rate formulas — owned by `/quality`. This document routes alerts to runbooks; QUALITY defines the alerts.
- Schema and migration content (tables, columns, indexes, migration scripts) — owned by `/data`. This document schedules backups and restores; DATA owns the structure.
- Wire shape of endpoints, events, and versioning syntax — owned by `/interfaces`.
- Threat model, STRIDE analysis, mitigation catalogue — owned by `/security`. This document references secret classes and roles; SECURITY models the threats.
- Error code definitions (code → http_status → user_action → retryable) — owned by `/errors`. This document cites codes by string.
- Implementation syntax and SDK version pins (`async`/`await`, specific library versions) — owned by `/spec` and `/implement`.
- Per-surface UI flows (sign-in screens, error screens, onboarding flow) — owned by IA skills.
- Infrastructure-as-code implementation (Terraform modules, Helm charts, CDK stacks) — owned by `/spec` and `/implement`. This document specifies the deployment shape; `/spec` declares the IaC unit.
- Cost modelling, FinOps, rightsizing analysis — not part of the ADD workflow.
- SOC2 / ISO27001 / HIPAA audit evidence packaging — this document informs audits; the audit workflow itself is not owned here.

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `environments`, `components_deployed`, `config_vars`, `integrations`, `runbook_entries`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "and so on", "industry-standard", "best practice", "reasonable", "sufficient")
- [ ] § 13 Open Questions is present (empty with "All questions resolved." or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files beyond the referenced registries (ARCHITECTURE § 2, INTERFACES § 1, DATA datastores, QUALITY § 6, SECURITY § 9 when produced, ERRORS § 3 when produced)
- [ ] All thirteen sections § 1 through § 13 are present with their exact headings
- [ ] § 1 has a block per environment with purpose, compute, region, network, data plane, per-component mapping, scaling behaviour, and a diagram (or external link)
- [ ] Every component in ARCHITECTURE § 2 is placed in at least one environment's per-component mapping in § 1 (or the missing placement is raised in § 13)
- [ ] § 2 has a block per CI/CD pipeline with repo, stages, gates, artifacts, promotion model, rollback mechanism; every environment in § 1 is reachable by exactly one pipeline (or annotated bootstrap-only)
- [ ] § 3 has a config difference matrix citing by `CFG_NAME`, a data strategy per environment, and access control per environment
- [ ] § 4 has a row per config var with `CFG_NAME`, type, default, scope, secret flag, per-env-override flag, owner, used-by component list, purpose; alphabetical by `CFG_NAME`; no blanks
- [ ] Every feature flag (typed `bool`, conventionally `CFG_FEATURE_*` or `CFG_FLAG_*`) in § 4 carries a retirement date or an explicit permanence rationale
- [ ] § 5 has a block per secret class with storage, owner, rotation policy, read principals, rotate principals, audit trail; no production secret has `never` as rotation policy
- [ ] § 6 has a block per external integration with provider, purpose, auth, endpoints, rate limits, SLA, failure behaviour, cost model, data sent, contact; every row from INTERFACES § 1 external boundaries has a matching integration (or a § 13 entry)
- [ ] Every § 6 failure behaviour names a concrete mechanism (fail-fast with `ERR_CODE`, queue with TTL, cached fallback with staleness bound, disable-with-user-message, or `SAGA-{name}` compensation) — never "graceful degradation" alone
- [ ] § 7 has a `RUNBOOK-*` entry for every `ALERT-*` in QUALITY § 6 (one-to-one); each entry has severity, symptoms, quick verification, first response, diagnostic steps, fix patterns, escalation, related alerts/metrics
- [ ] § 8 has concrete numeric RTO and RPO per scenario (minutes or hours), a backup strategy per datastore from DATA, a numbered restore procedure with per-step durations, covered failure scenarios, and DR drill cadence
- [ ] § 9 states retention and access control for log / metric / trace / dashboard platforms
- [ ] § 10 names deployment windows, change notification, concrete rollback criteria, rollback procedure, and post-incident review cadence
- [ ] § 11 initial setup runbook has prerequisites, ordered setup steps with exact commands, smoke tests with expected output, and links to § 7 and § 4
- [ ] § 12 Relationship to Other Artifacts names ARCHITECTURE, INTERFACES, DATA, QUALITY, SECURITY, BEHAVIOR, ERRORS, and SPEC-level artifacts
- [ ] Every `CFG_NAME`, `RUNBOOK-*` is unique and stable; retired IDs retain their row with `(deprecated — use CFG_NEW_NAME)` or `(retired — superseded by RUNBOOK-{new-id})`
- [ ] Every component cited exists in ARCHITECTURE § 2; every `EP-name` cited exists in INTERFACES § 6; every `ALERT-*`, `METRIC-*`, `SLO-*` cited exists in QUALITY; every `SAGA-{name}` cited exists in BEHAVIOR § 2; every `ERR_CODE` cited exists in ERRORS § 3; every `ADR-NN` cited exists in ARCHITECTURE
- [ ] No implementation syntax (grep-check: `try`, `catch`, `async`, `await`, `goroutine`, `thread`, `mutex`) and no wire-schema design (grep-check: `JSON body`, `camelCase`, `snake_case` as field-name design, `column`, `table`)
- [ ] Frontmatter counts match the body: `environments`, `components_deployed`, `config_vars`, `integrations`, `runbook_entries`, `open_questions`
- [ ] `status` is `complete` if § 13 is "All questions resolved." and `has_open_questions` otherwise
- [ ] Document length is within the 500–1500 line target (hard cap 2000); if § 7 pushes near the cap, per-component runbooks are extracted to `runbooks/{component}.md` and § 7 is reduced to an index
