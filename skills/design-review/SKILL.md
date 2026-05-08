---
name: DESIGN_REVIEW.md
description: Gate the design suite before continuous implementation begins — read DOMAIN, ARCHITECTURE, INTERFACES, DATA, BEHAVIOR, QUALITY, SECURITY, ERRORS end-to-end and catalogue every cross-artifact contradiction, every completeness gap that blocks downstream steps, every internal-quality defect, and every dangling cross-reference, then emit a verdict (`pass` or `fix-required`) that drives orchestrator branching. Use when asked to review the design, run a design review, check design artifacts for consistency, gate the bootstrap pipeline before continuous workflow begins, audit the design suite, or produce a DESIGN_REVIEW.md.
---

# Task: Generate DESIGN_REVIEW.md — Design-Suite Gate Review

## Objective

Produce a DESIGN_REVIEW.md that serves as the mandatory gate at the end of the bootstrap design phases (A–E) of the Agent-Driven Development workflow, and re-runs whenever G8 reconcile applies major top-level edits during continuous implementation. It reviews the eight-artifact design suite — DOMAIN, ARCHITECTURE, INTERFACES, DATA, BEHAVIOR, QUALITY, SECURITY, ERRORS — along three orthogonal axes (cross-artifact consistency, completeness against downstream needs, internal quality of each artifact), catalogues every finding under stable `CF-NN` / `MF-NN` / `QF-NN` IDs with severity, evidence, and a proposed actionable edit, validates every stable-ID citation via a cross-reference matrix, and emits a machine-readable verdict (`pass` or `fix-required`) that the orchestrator branches on. An agent reading this document alone can — without opening any of the eight reviewed artifacts — tell which artifacts need regeneration, exactly what content must change, and whether the pipeline may proceed to continuous per-trigger work or must loop back through design.

DESIGN_REVIEW.md is the type-checker of the design phase. It catches the errors that do not manifest within any single artifact but emerge from relationships between them: an invariant declared in DOMAIN but unenforced by a DATA constraint; a wire event declared in INTERFACES but un-emitted by any BEHAVIOR saga; an error code referenced from an endpoint but missing from the ERRORS registry; a use case present in USE_CASES but un-realised by any endpoint or saga. Without this gate these contradictions surface during `/spec` or `/implement`, forcing cascade regenerations of downstream artifacts; with it, the fix is localised to the offending design artifact and the pipeline re-enters only the minimum necessary scope. The defining discipline — and the commonest violation — is **every finding is actionable, every finding cites evidence, and the `verdict` field matches the blocking-finding count exactly**. Subjective complaints, vague "this is inconsistent" findings, and verdicts that override the blocking-count rule turn the gate into noise.

---

## Inputs

1. **DOMAIN.md** (required) — ubiquitous vocabulary, bounded contexts, entities, invariants (`INV-NN`). Reviewed for internal quality; used as the authoritative source for every entity-name and `INV-NN` citation in other artifacts.
2. **ARCHITECTURE.md** (required) — components, cross-component flows, architecture decisions (`ADR-NN`). Used as the authoritative source for every component name referenced elsewhere; reviewed for completeness against DOMAIN contexts.
3. **INTERFACES.md** (required) — boundaries, endpoints (`EP-name`), events (`EVT-name`), authentication, idempotency. Used as the authoritative source for every endpoint and event reference; reviewed for consistency with DOMAIN vocabulary and BEHAVIOR emission timing.
4. **DATA.md** (required) — storage model, schema, constraints, indexes, sensitive-column inventory. Reviewed for coverage of DOMAIN entities and enforcement of DOMAIN invariants at the schema level.
5. **BEHAVIOR.md** (required) — state machines (`SM-entity-state`), sagas (`SAGA-name`), idempotency keys, event emission timing. Reviewed for coverage of DOMAIN entity lifecycles and consistency with INTERFACES endpoints and events.
6. **QUALITY.md** (required) — logging, metrics (`METRIC-*`), SLOs (`SLO-*`), alerts, performance budgets. Reviewed for coverage of INTERFACES endpoints and BEHAVIOR sagas.
7. **SECURITY.md** (required) — assets, actors, trust boundaries, threats (`THREAT-NN`), mitigations (`MIT-NN`), auth and authorization. Reviewed for coverage of INTERFACES boundaries, USE_CASES actors, and internal threat-to-mitigation closure.
8. **ERRORS.md** (required) — error code registry (`ERR_CODE`), HTTP mapping, user-facing messages. Reviewed for coverage of the error codes cited across INTERFACES, BEHAVIOR, QUALITY, and SECURITY.
9. **USE_CASES.md** (optional — read when use-case-coverage analysis is requested) — the `UC-NN` registry. Enables § 6 Use-Case Coverage to trace every use case to an endpoint or saga that realises it. Read it when requested; skip the section if not read.
10. **PROPOSAL.md** (optional — read when cited by QUALITY or SECURITY for numeric targets) — used only to verify that citations in downstream artifacts are faithful (e.g., `SLO-push-latency-p95` target cites PROPOSAL verbatim).
11. **Surface IAs** (optional — read one at a time if requested) — `{WEB,CLI,MOBILE,TUI,VOICE}_IA.md`. Read at most one per review; reading more than one exhausts the context budget. When not read, § 6 silently skips surface-coverage checks.

Read set size: 8 required artifacts. The eighth is the heaviest step in ADD by design — the review exists because cross-artifact analysis cannot be delegated downward. Optional inputs (9–11) are read only when they add a concrete check; each one consumes context that could otherwise be spent reading the required eight more carefully.

Read all required inputs **end-to-end**. The single most common failure mode of this skill is truncated reads producing phantom findings — a CF-5 that claims "DOMAIN § 9 has no `INV-12`" when INV-12 appears on line 1280 of a file that was cut off at line 900. Before writing any finding that asserts absence of an ID, re-scan the referenced artifact's full length.

---

## Workflow

The review proceeds in six phases: context and cross-reference extraction, per-artifact quality pass, pairwise consistency pass, completeness pass, use-case coverage pass (optional), verdict and required-changes synthesis. Phases are sequential — later phases consume the cross-reference index built in Phase 1 — but revisit earlier phases if a later one surfaces a citation that Phase 1 missed.

Before Phase 2, read `references/review-checklist.md` for the systematic probes applied to each axis. The checklist organises:

- **Per-artifact quality probes** — eight sections, one per artifact, each listing the internal-quality defects to look for (placeholder language, unresolved open questions, malformed IDs, orphan entries).
- **Pairwise consistency probes** — ordered pairs (e.g., DOMAIN ↔ DATA, INTERFACES ↔ BEHAVIOR, SECURITY ↔ INTERFACES) with the specific contradictions that arise at each pair.
- **Completeness probes** — grouped by downstream consumer (per-unit `/SPEC.md`, `/system-verification`), each listing the information those steps depend on.
- **Cross-reference index recipe** — the exact prefixes to extract and match (`INV-`, `EP-`, `EVT-`, `ERR_`, `SM-`, `SAGA-`, `METRIC-`, `SLO-`, `THREAT-`, `MIT-`, `UC-`, `ADR-`, `CFG_`).

The checklist owns the systematic enumeration; the workflow below describes how to apply its output.

### Phase 1: Context & Cross-Reference Extraction

Read all eight required artifacts end-to-end. As each artifact is read, extract into an in-agent working index:

- **Definitions:** every stable-ID entry (`INV-NN` in DOMAIN § 9, `EP-name` in INTERFACES § 6, `EVT-name` in INTERFACES § 7, `ERR_CODE` in ERRORS § 3, `SM-{entity}: {from} → {to}` in BEHAVIOR § 1, `SAGA-{name}` in BEHAVIOR § 2, `METRIC-*` and `SLO-*` in QUALITY § 3–4, `THREAT-NN` and `MIT-NN` in SECURITY § 5–6).
- **Citations:** every stable-ID referenced from outside its defining artifact, paired with the referencing location (`{artifact} § {N}`).
- **Frontmatter counts and statuses:** per artifact, the declared counts (`entities: 12`, `endpoints: 8`, …) and `status` fields.

The working index does not appear in the output directly; it feeds § 5 Cross-Reference Validation Matrix and every consistency finding.

### Phase 2: Per-Artifact Internal-Quality Pass

For each of the eight required artifacts, apply the artifact-specific probes from `references/review-checklist.md` § "Per-artifact quality probes". Typical defects surfaced:

- Placeholder language (`appropriate`, `relevant`, `as needed`, `etc.`, `various`, `industry-standard`, `best practice`, `reasonable`) — cite the exact section and propose the specific replacement.
- Unresolved open questions when the artifact's frontmatter claims `status: complete`.
- Malformed IDs (inconsistent casing, missing prefix, gaps in the sequence without a `(retired — …)` row).
- Orphan entries (e.g., a `THREAT-NN` with no `MIT-NN` and no residual-risk row; an `SLO-*` with no burn-rate alert; an `EP-name` whose possible_errors list is empty).
- Frontmatter counts that contradict the body (e.g., `threats: 14` but § 5 lists 13 entries).

Each defect becomes a `QF-NN` entry in § 4 Quality Findings.

### Phase 3: Pairwise Cross-Artifact Consistency Pass

For each ordered pair of required artifacts (DOMAIN ↔ ARCHITECTURE, DOMAIN ↔ DATA, DOMAIN ↔ BEHAVIOR, INTERFACES ↔ DOMAIN, INTERFACES ↔ BEHAVIOR, INTERFACES ↔ ERRORS, INTERFACES ↔ SECURITY, BEHAVIOR ↔ QUALITY, SECURITY ↔ INTERFACES, SECURITY ↔ USE_CASES when read, QUALITY ↔ BEHAVIOR), apply the pair-specific probes from `references/review-checklist.md` § "Pairwise consistency probes". Typical findings:

- Naming disagreements: INTERFACES § 6 uses `parent_sha`; INTERFACES § 2 declares camelCase convention. DOMAIN § 1 spells `Repository`; ARCHITECTURE § 2 spells `Repo`.
- Semantic disagreements: DOMAIN `INV-7` says "repo names are unique per owner"; DATA § 3 has no unique constraint on `(owner_id, name)`.
- Ordering disagreements: BEHAVIOR `SAGA-push` step 3 emits `EVT-push-accepted`; QUALITY `SPAN-push.2.accept` is tagged to that step — step numbering contradicts.
- Missing cross-references: INTERFACES `EP-push` lists `ERR_NAME_TAKEN` in `possible_errors`; ERRORS § 3 has no such entry.
- Coverage gaps exposed by pairs: every `EVT-name` in INTERFACES § 7 must be emitted by at least one BEHAVIOR saga step; every `SAGA-name` in BEHAVIOR § 2 must have an originating `EP-name` in INTERFACES § 6 or an explicit "triggered by scheduler" annotation.

Each contradiction becomes a `CF-NN` entry in § 2 Consistency Findings.

### Phase 4: Completeness Pass

For each downstream skill that consumes the design suite — the per-unit `/SPEC.md`, `/system-verification` — apply the completeness probes from `references/review-checklist.md` § "Completeness probes". A finding is a completeness defect when downstream execution cannot succeed without the missing content. Typical findings:

- BEHAVIOR has `SAGA-push` but no idempotency key — `/spec` for the push handler will invent one, violating the idempotency-key authority rule.
- QUALITY § 4 has `SLO-push-availability` but no burn-rate alert in § 6 — `/operations` runbook has no routing.
- DOMAIN has a `Repository` aggregate but no lifecycle state machine — `/spec` for mutation endpoints has no state invariants to assert.
- ERRORS § 3 has no entry for a code cited from SECURITY § 11 rate-limit text — the abuse-prevention behaviour has no user-facing contract.
- ARCHITECTURE § 2 lacks a component that DATA § 3 references as the owner of a table.

Each gap becomes an `MF-NN` entry in § 3 Completeness Findings, annotated with the downstream step that fails without it.

### Phase 5: Cross-Reference Validation Matrix

Using the working index from Phase 1, build § 5 as a table. For every citation extracted in Phase 1, emit one row: `{reference-id}` | `{referencing artifact and section}` | `{defining artifact and section}` | `OK` or `MISSING` or `AMBIGUOUS`.

- **OK** — the referenced ID is defined in the expected authoritative artifact.
- **MISSING** — the referenced ID has no definition in any reviewed artifact. Every MISSING row also generates a `CF-NN` entry in § 2 with severity typically `blocking` (wrong code would be written).
- **AMBIGUOUS** — the referenced ID is defined in more than one artifact under conflicting schemas (e.g., an `EVT-name` defined differently in INTERFACES § 7 and QUALITY § 2). Every AMBIGUOUS row also generates a `CF-NN` with severity `blocking` or `high`.

The matrix must cover **every** prefix pattern — see the checklist's "Cross-reference index recipe" section for the complete list.

### Phase 6: Use-Case Coverage (optional), Verdict, Required Changes

If USE_CASES.md was read, apply § 6. For every `UC-NN` in USE_CASES § 2, verify three things:

1. An entity or aggregate in DOMAIN represents the use case's subject.
2. At least one endpoint in INTERFACES § 6 or one saga in BEHAVIOR § 2 realises it.
3. A permissions row in SECURITY § 8 covers it (or it is explicitly `public`).

Orphan use cases — present in USE_CASES, absent from one of (2) or (3) — are completeness findings in § 3.

§ 7 Verdict: count blocking findings across `CF-*`, `MF-*`, `QF-*`. If `blocking_findings == 0`, verdict is `pass`. Otherwise verdict is `fix-required`. The Rules section enforces this as a hard contract — no subjective override.

§ 8 Required Changes (emitted only if verdict is `fix-required`): group the blocking and high-severity findings by target artifact, and within each group list the specific edits the originating skill must make on regeneration. This is the exact instruction set the orchestrator hands to the re-run of `/domain`, `/interfaces`, `/behavior`, etc.

§ 9 Open Questions: review-level ambiguities only — cases where a finding could be a bug or a deliberate design decision and the reviewer cannot tell from the artifact. Not a general-purpose TODO list.

Before finalising, verify:

- Every `CF-NN`, `MF-NN`, `QF-NN` ID is unique and sequential within its prefix.
- Every blocking finding cites specific evidence (quote, section, or stable ID).
- Every blocking finding has a proposed-resolution bullet that is an actionable edit (`"Rename parent_sha → parentSha in INTERFACES § 6 EP-push"`, not `"fix naming"`).
- Verdict matches blocking count.
- Frontmatter counts (`consistency_findings`, `completeness_findings`, `quality_findings`, `blocking_findings`) match the body exactly.
- `status` is `complete` if § 9 Open Questions reads "All questions resolved.", `has_open_questions` otherwise.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Every finding is actionable

A finding is a specific contradiction, gap, or defect with a specific proposed edit. `"DOMAIN and DATA disagree about uniqueness"` is not a finding. `"DOMAIN INV-7 says 'repo names are unique per owner' (DOMAIN § 9); DATA § 3 table `repositories` has no unique index on `(owner_id, name)`; add `UNIQUE (owner_id, name)` to DATA § 3 `repositories` schema."` is a finding.

### 2. Every finding cites evidence

Every `CF-NN`, `MF-NN`, and `QF-NN` entry cites the exact section (`DOMAIN § 9`, `INTERFACES § 6 EP-push`) or stable ID (`INV-7`, `EVT-push-accepted`, `ERR_NAME_TAKEN`) that supports the claim. Findings with no citation are rejected by the quality checklist.

### 3. Severity is defined by downstream impact, not reviewer mood

- **blocking** — ignoring the finding will cause wrong code to be written. Uses `fix-required` verdict.
- **high** — the finding will confuse downstream readers and force a discretionary fix, but the wrong code can be unwound without cascade.
- **medium** — cosmetic or consistency issue that degrades readability without blocking execution.
- **low** — minor wording or style inconsistency.

Blocking severity must be used sparingly. Inflating every finding to blocking is the surest way to make the gate useless — it degenerates into "always fix-required".

### 4. Verdict follows blockers, mechanically

```
blocking_findings == 0  →  verdict: pass
blocking_findings > 0   →  verdict: fix-required
```

There is no subjective override. A reviewer who believes the design is acceptable despite one blocking finding must downgrade that finding's severity (with justification in the finding body) rather than override the verdict.

### 5. Stable IDs for findings

`CF-NN`, `MF-NN`, `QF-NN` are assigned in order of first mention within their section. IDs are permanent across regenerations of DESIGN_REVIEW.md on the same design suite — if Phase F re-enters, the orchestrator may compare finding IDs across runs to confirm that a specific finding was addressed. Retired findings (fixed by a regeneration and re-run) retain their row marked `(resolved by {regenerated-artifact} {YYYY-MM-DD})` when this is a re-run; a first run has no retired findings.

### 6. No new design

The review proposes edits that update existing artifacts; it does not itself introduce new design content. A proposed resolution that says `"Add a new event EVT-repo-renamed to INTERFACES § 7 with payload {...}"` names the event and the referenced emission point but does not specify the payload schema — that belongs to `/interfaces` on regeneration. The review's job is to surface the gap, not fill it.

### 7. Missing-ID findings are blocking

Any row in § 5 Cross-Reference Validation Matrix marked `MISSING` generates a `CF-NN` with severity `blocking`: a referenced ID that does not resolve will cause `/spec` or `/implement` to either guess (producing wrong code) or halt (producing a cascade regeneration). There is no non-blocking interpretation of an unresolved citation.

### 8. Internal-quality findings name the exact defect

A `QF-NN` says `"INTERFACES § 6 EP-login response has field 'appropriate fields' — replace with the explicit field list required by UC-04 login flow"`, not `"INTERFACES § 6 uses vague language"`. The defect and the edit both appear in the finding body.

### 9. Cross-reference matrix is exhaustive

Every stable-ID prefix pattern listed in the checklist's "Cross-reference index recipe" is scanned. If the design suite contains zero instances of a prefix (e.g., no `SAGA-*` exists because BEHAVIOR has no sagas yet), the matrix states `"No SAGA-* citations in the suite."` inline — silent omission is indistinguishable from failed extraction.

### 10. Single YAML frontmatter block

One YAML block at the top containing common fields (`skill`, `date`, `status`, `verdict`) and review-specific counts (`artifacts_reviewed`, `consistency_findings`, `completeness_findings`, `quality_findings`, `blocking_findings`, `open_questions`). Never emit a second block. Counts match the body exactly.

### 11. Precision over vagueness

No `"appropriate"`, `"relevant"`, `"as needed"`, `"etc."`, `"various"`, `"reasonable"`, `"industry-standard"`, `"best practice"`. Use exact IDs, exact section numbers, exact edit instructions. Unresolvable ambiguity surfaces in § 9 Open Questions with options, tradeoffs, and a recommendation.

### 12. Positive observations are required

§ 7 Verdict (and, when verdict is `pass`, the preceding paragraph) acknowledges what the design did well: invariants are enforced by DATA constraints; every endpoint has an error contract; BEHAVIOR sagas carry idempotency keys; SECURITY threats all have mitigations. Balanced reporting prevents the gate from reading as pure criticism and lets the orchestrator recognise when a re-run has actually improved the suite.

---

## Output Format

```markdown
---
skill: DESIGN_REVIEW.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions}
verdict: {pass | fix-required}
artifacts_reviewed: {N}
consistency_findings: {N}
completeness_findings: {N}
quality_findings: {N}
blocking_findings: {N}
open_questions: {N}
---

# DESIGN_REVIEW — {ProductName}

> Gate review of the design suite before continuous per-trigger work begins. Consumes DOMAIN,
> ARCHITECTURE, INTERFACES, DATA, BEHAVIOR, QUALITY, SECURITY, ERRORS end-to-end
> and (optionally) USE_CASES and one surface IA. Produces a machine-readable
> verdict (`pass` or `fix-required`) that the orchestrator branches on.
> Downstream design-skill re-runs consume § 8 Required Changes as their edit
> list.

## § 1. Review Scope

| Artifact | Path | Date | Status (frontmatter) | Read end-to-end? |
|----------|------|------|----------------------|------------------|
| DOMAIN.md | `{path}` | `{YYYY-MM-DD}` | `{complete / has_open_questions / blocked}` | yes |
| ARCHITECTURE.md | `{path}` | `{YYYY-MM-DD}` | `{status}` | yes |
| INTERFACES.md | `{path}` | `{YYYY-MM-DD}` | `{status}` | yes |
| DATA.md | `{path}` | `{YYYY-MM-DD}` | `{status}` | yes |
| BEHAVIOR.md | `{path}` | `{YYYY-MM-DD}` | `{status}` | yes |
| QUALITY.md | `{path}` | `{YYYY-MM-DD}` | `{status}` | yes |
| SECURITY.md | `{path}` | `{YYYY-MM-DD}` | `{status}` | yes |
| ERRORS.md | `{path}` | `{YYYY-MM-DD}` | `{status}` | yes |
| USE_CASES.md | `{path or "not read"}` | `{YYYY-MM-DD}` | `{status}` | `{yes / skipped}` |
| `{Surface IA}` | `{path or "not read"}` | `{YYYY-MM-DD}` | `{status}` | `{yes / skipped}` |

(If an artifact's `status` in its own frontmatter is `has_open_questions` or `blocked`, this is itself a `QF-NN` finding — a design suite with un-resolved internal open questions cannot pass the gate regardless of cross-artifact consistency.)

---

## § 2. Consistency Findings

Cross-artifact contradictions. Numbered `CF-NN` in the order first surfaced.

#### CF-{NN}: {short title}

- **Severity:** `{blocking / high / medium / low}`
- **Artifacts involved:** `{list — e.g., INTERFACES.md, DOMAIN.md}`
- **Evidence:**
  - `{artifact A} § {N}`: `{quote or stable-ID citation}`
  - `{artifact B} § {N}`: `{quote or stable-ID citation}`
- **Nature of conflict:** `{naming / semantics / ordering / missing cross-ref / schema drift}`
- **Proposed resolution:** `{specific actionable edit — e.g., "Rename `parent_sha` → `parentSha` in INTERFACES § 6 EP-push to match the camelCase convention declared in INTERFACES § 2 naming rules."}`
- **Blocks:** `{downstream step that will produce wrong code without this fix — e.g., "/spec for push handler; SDK client generation"}`

(Repeat per finding. If none: "No consistency findings — all pairwise cross-artifact probes passed.")

---

## § 3. Completeness Findings

Missing information that downstream steps need. Numbered `MF-NN`.

#### MF-{NN}: {short title}

- **Severity:** `{blocking / high / medium / low}`
- **Artifact:** `{which document is incomplete}`
- **What's missing:** `{specific content gap — e.g., "SAGA-push in BEHAVIOR § 2 does not list an idempotency key; INTERFACES § 4 mandates one for every mutating endpoint."}`
- **Who needs this:** `{downstream step — /SPEC.md for {u<NN>}, /system-verification}`
- **Proposed resolution:** `{specific actionable edit — e.g., "Add Idempotency Key field to SAGA-push in BEHAVIOR § 2 citing INTERFACES § 4 header name."}`

(Repeat per finding. If none: "No completeness findings — every downstream consumer has what it needs.")

---

## § 4. Quality Findings

Internal quality issues within single artifacts. Numbered `QF-NN`.

#### QF-{NN}: {short title}

- **Severity:** `{blocking / high / medium / low}`
- **Artifact & location:** `{artifact} § {N}`
- **Issue:** `{placeholder language / missing citation / malformed ID / unresolved open question / frontmatter mismatch — name the specific defect}`
- **Proposed resolution:** `{specific edit — e.g., "Replace 'appropriate rate limit' in SECURITY § 11 with the concrete threshold `100 requests per minute per IP` cited from INTERFACES § 6 EP-auth-login rate-limit field."}`

(Repeat per finding. If none: "No internal-quality findings — every artifact passes its own rules.")

---

## § 5. Cross-Reference Validation Matrix

Every stable-ID prefix is scanned. Every citation is either `OK`, `MISSING` (→ `CF-NN` blocking), or `AMBIGUOUS` (→ `CF-NN` blocking or high).

| Reference | Referenced from | Defined in | Status |
|-----------|----------------|-----------|--------|
| `INV-{NN}` | `{referencing artifact} § {N}` | `DOMAIN § 9` | `OK` |
| `EP-{name}` | `{referencing artifact} § {N}` | `INTERFACES § 6` | `OK` |
| `EVT-{name}` | `{referencing artifact} § {N}` | `INTERFACES § 7` | `MISSING` |
| `ERR_{CODE}` | `{referencing artifact} § {N}` | `ERRORS § 3` | `OK` |
| `SM-{entity}-{state}` | `{referencing artifact} § {N}` | `BEHAVIOR § 1` | `OK` |
| `SAGA-{name}` | `{referencing artifact} § {N}` | `BEHAVIOR § 2` | `OK` |
| `METRIC-{name}` | `{referencing artifact} § {N}` | `QUALITY § 3` | `OK` |
| `SLO-{name}` | `{referencing artifact} § {N}` | `QUALITY § 4` | `OK` |
| `THREAT-{NN}` | `{referencing artifact} § {N}` | `SECURITY § 5` | `OK` |
| `MIT-{NN}` | `{referencing artifact} § {N}` | `SECURITY § 6` | `OK` |
| `UC-{NN}` | `{referencing artifact} § {N}` | `USE_CASES § 2` | `OK` |
| `ADR-{NN}` | `{referencing artifact} § {N}` | `ARCHITECTURE § {N}` | `OK` |
| `CFG_{NAME}` | `{referencing artifact} § {N}` | `OPERATIONS § 4` | `{OK / MISSING — OPERATIONS not in scope}` |

(Include every citation. If a prefix has zero citations across the suite, add a single row: `"No {PREFIX}-* citations in the suite."` to prove the prefix was scanned and is not silently omitted.)

---

## § 6. Use-Case Coverage

(Include this section only if USE_CASES.md was read; otherwise write "USE_CASES.md not read — use-case coverage not verified." and skip the table.)

For each `UC-NN` in USE_CASES § 2, verify traceability to DOMAIN, INTERFACES/BEHAVIOR, and SECURITY.

| Use case | Domain subject | Realised by | SECURITY § 8 permission row | Coverage |
|----------|---------------|-------------|------------------------------|----------|
| `UC-{NN}` | `{EntityName}` (DOMAIN § 4) | `EP-{name}` (INTERFACES § 6) or `SAGA-{name}` (BEHAVIOR § 2) | `{role list / "public"}` | `OK` |
| `UC-{NN}` | `{EntityName}` | — | — | `ORPHAN` (→ `MF-{NN}`) |

(Every `UC-NN` from USE_CASES § 2 appears. ORPHAN rows generate `MF-NN` completeness findings in § 3.)

---

## § 7. Verdict

**verdict: `{pass | fix-required}`**

{One paragraph. If `pass`: acknowledge the strongest design aspects observed (specific — "DOMAIN invariants INV-1 through INV-14 are all enforced by DATA constraints; every EP-name carries an explicit ERR_* contract"). If `fix-required`: name the single most significant blocking finding with its ID and proposed edit.}

{Balanced paragraph — even when `fix-required`, acknowledge what the design did well so the orchestrator can see where re-runs have improved the suite vs. where regressions have appeared.}

---

## § 8. Required Changes

(Include this section only if `verdict: fix-required`. If `pass`: write "No changes required — bootstrap is complete; the continuous per-trigger workflow may begin." and omit the rest.)

Grouped by target artifact. Each item cites the finding ID that drove it.

#### DOMAIN.md

- `{specific edit}` per `CF-{NN}` / `MF-{NN}` / `QF-{NN}`

#### ARCHITECTURE.md

- `{specific edit}` per `{finding-id}`

#### INTERFACES.md

- `{specific edit}` per `{finding-id}`

#### DATA.md

- `{specific edit}` per `{finding-id}`

#### BEHAVIOR.md

- `{specific edit}` per `{finding-id}`

#### QUALITY.md

- `{specific edit}` per `{finding-id}`

#### SECURITY.md

- `{specific edit}` per `{finding-id}`

#### ERRORS.md

- `{specific edit}` per `{finding-id}`

(Omit any artifact subsection that has no required changes. The orchestrator re-runs only the skills whose artifact appears here.)

---

## § 9. Open Questions

Review-level ambiguities only — cases where the reviewer cannot tell whether a finding is a bug or a deliberate design decision, and needs an authoritative answer before the verdict can be finalised.

- [ ] `{Question — e.g., "BEHAVIOR SAGA-push step 5 emits EVT-push-accepted, but the DFD in SECURITY § 4 labels the edge as 'public data classification'. Is the event payload genuinely public, or should SECURITY reclassify the edge?"}`
  - **Option A:** `{interpretation 1 with its downstream consequence}` — `{tradeoff}`
  - **Option B:** `{interpretation 2 with its downstream consequence}` — `{tradeoff}`
  - **Recommendation:** `{suggested resolution with reasoning grounded in the artifacts read}`

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- End-to-end reading of DOMAIN, ARCHITECTURE, INTERFACES, DATA, BEHAVIOR, QUALITY, SECURITY, ERRORS
- Optional reading of USE_CASES (for § 6) and one surface IA (to extend coverage when requested)
- Cross-artifact consistency findings under stable `CF-NN` IDs
- Completeness findings under stable `MF-NN` IDs, annotated with the failing downstream step
- Internal-quality findings under stable `QF-NN` IDs
- Cross-reference validation matrix covering every stable-ID prefix in the suite
- Use-case coverage verification when USE_CASES is read
- Verdict (`pass` or `fix-required`) computed mechanically from `blocking_findings`
- Required-changes list grouped by target artifact (emitted only when `fix-required`)
- Positive observations about the design's strengths
- Genuinely review-level ambiguities surfaced in § 9 Open Questions

### Out of scope

- Authoring new design content — owned by the originating design skills (`/domain`, `/architecture`, `/interfaces`, `/data`, `/behavior`, `/quality`, `/security`, `/errors`). This review proposes edits; the skills apply them on regeneration.
- Per-unit code review — owned by `/review` (post-implementation).
- System-level end-to-end integration testing — owned by `/system-verify`.
- Substantive audit of performance claims, security controls, or capacity targets — owned by the respective design skills and by `/system-verify`. This review verifies internal-quality and cross-artifact consistency of those artifacts, not the correctness of their substantive claims (e.g., we flag an orphan THREAT-NN, not whether the threat model is comprehensive for the domain).
- Operations review — owned by `/operations`; OPERATIONS.md is not in the required read set because this gate runs before decomposition, often before `/operations` has produced.
- Surface-IA review beyond the one optionally read — exceeding one IA exhausts the context budget. Additional surface review is a separate invocation.
- Picking up triggers and creating units — performed by the orchestrator after `pass`. Each trigger (`roadmap/<NNN>-<slug>/ROADMAP.md` or `issues/<NNN>-<slug>/ISSUE.md`) drives a per-unit pipeline run starting at `/SPEC.md`.
- Conflict resolution when a reviewer finds a genuinely ambiguous case — surfaced in § 9; the orchestrator or a human resolves and re-runs the originating skill.

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `verdict`, `artifacts_reviewed`, `consistency_findings`, `completeness_findings`, `quality_findings`, `blocking_findings`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language (`appropriate`, `relevant`, `as needed`, `etc.`, `various`, `reasonable`, `industry-standard`, `best practice`)
- [ ] § 9 Open Questions is present (empty with "All questions resolved." or with genuine review-level ambiguities only)
- [ ] Output is self-contained — readable and actionable without re-opening any of the reviewed artifacts beyond the citations embedded in each finding
- [ ] All nine sections § 1 through § 9 are present with their exact headings (§ 6 contains the "USE_CASES.md not read" statement when USE_CASES was skipped; § 8 contains "No changes required — …" when verdict is `pass`)
- [ ] § 1 Review Scope lists every artifact that was read and names its frontmatter `status`; any artifact with `has_open_questions` or `blocked` generates a `QF-NN` finding
- [ ] Every `CF-NN` entry has severity, artifacts involved, evidence citing exact sections or stable IDs, nature of conflict, proposed resolution, and blocks field
- [ ] Every `MF-NN` entry has severity, artifact, what's missing, who needs this (named downstream step), and proposed resolution
- [ ] Every `QF-NN` entry has severity, artifact and location, specific issue, and proposed resolution — no "vague language" findings that do not name the exact phrase to replace
- [ ] Every finding's proposed resolution is an actionable edit (names the section to change, the exact new wording or ID to add, or the exact deletion to make) — "fix the inconsistency" is rejected
- [ ] Every blocking finding cites specific evidence (direct quote, section number, or stable ID)
- [ ] § 5 Cross-Reference Validation Matrix covers every stable-ID prefix listed in the checklist's "Cross-reference index recipe" section; prefixes with zero citations get an explicit "No {PREFIX}-* citations in the suite." row
- [ ] Every row marked `MISSING` in § 5 has a corresponding `CF-NN` entry with `severity: blocking`
- [ ] Every row marked `AMBIGUOUS` in § 5 has a corresponding `CF-NN` entry with `severity: blocking` or `high`
- [ ] § 6 Use-Case Coverage is present iff USE_CASES.md was read; every `UC-NN` from USE_CASES § 2 appears; every `ORPHAN` row has a matching `MF-NN`
- [ ] `verdict` is `pass` iff `blocking_findings == 0`; `fix-required` iff `blocking_findings > 0` — no subjective override
- [ ] § 7 Verdict paragraph acknowledges at least one concrete strength of the design (positive observation)
- [ ] § 8 Required Changes is present iff verdict is `fix-required`, grouped by target artifact, and every edit cites the finding ID that drove it
- [ ] Severity distribution is not inflated — not every finding is `blocking`; `blocking` is reserved for findings whose unfixed form will cause wrong code to be written
- [ ] Frontmatter counts match the body exactly: `artifacts_reviewed` equals the number of required + optional artifacts read; `consistency_findings` equals the `CF-NN` count; `completeness_findings` equals `MF-NN` count; `quality_findings` equals `QF-NN` count; `blocking_findings` equals the count of findings with `severity: blocking` across all three sections
- [ ] `status` is `complete` if § 9 is "All questions resolved." and `has_open_questions` otherwise
- [ ] Document length is within the 300–800 line target (hard cap 1200)
