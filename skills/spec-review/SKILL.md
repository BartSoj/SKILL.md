---
name: SPEC_REVIEW.md
description: Gate the unit SPEC before PLAN — read the WORK_UNITS unit entry, the SPEC end-to-end, the unit's declared design read-set, every dependency unit's SPEC, and the codebase, then dispatch four parallel subagents (Scope & Risk Auditor, Compatibility Checker, OSS Library Scout, Internal Quality Reviewer) to surface scope drift, premature deferral, reinvention, convention drift, and empirical gambles, and emit a verdict (`pass`, `fix-required`, or `prototype-needed`) that drives orchestrator branching to PLAN, SPEC regeneration, or PROTOTYPE. Use when asked to review a spec, run a spec review, validate the spec before plan, audit a SPEC.md before implementation, gate the unit before plan, or produce a SPEC_REVIEW.md.
---

# Task: Generate SPEC_REVIEW.md — Per-Unit Specification Gate

## Objective

Produce a SPEC_REVIEW.md that serves as the mandatory gate between H1 (`/SPEC.md`) and H2 (`/PLAN.md`) of the Agent-Driven Development per-unit pipeline. It validates a single unit's SPEC against the contract declared in WORK_UNITS, against the design suite the unit cites, against every dependency unit's SPEC, and against the existing codebase, then catalogues every finding under stable `SF-NN` / `CF-NN` / `QF-NN` / `R-NN` IDs with severity, evidence, and a proposed actionable edit, and emits a machine-readable verdict (`pass`, `fix-required`, or `prototype-needed`) that the orchestrator branches on. An agent reading this document alone can — without re-opening the SPEC, the design read-set, or any dependency SPEC — tell whether the pipeline may proceed to PLAN, must regenerate the SPEC with this review as additional input, or must first run the conditional H1.6 PROTOTYPE step to empirically validate risky assumptions.

SPEC_REVIEW.md exists because every other ADD review gate catches a different category of failure — DESIGN_REVIEW catches cross-artifact contradictions in the design suite, CODE_REVIEW catches issues in implementation — but the SPEC itself, the one consequential intermediate artifact between design and code, had no review gate. Real-world testing of the pipeline surfaced five recurring failure modes the SPEC author silently propagates downstream: **scope drift** (the SPEC implements less than the unit definition calls for), **premature deferral** (the SPEC defers work to "future units" or to PLAN/IMPLEMENT that has no actual home elsewhere), **reinvention** (the SPEC prescribes custom logic where a well-known open-source library already solves the problem), **convention drift** (the SPEC prescribes a filename, package shape, or naming pattern that contradicts framework documentation or existing monorepo packages), and **empirical gambles** (the SPEC makes implementation decisions that depend on unverified assumptions about library behavior, runtime semantics, or external system shapes). The defining discipline — and the commonest violation — is that **the verdict field follows `blocking_findings` and `risk_items` mechanically with no subjective override**. A reviewer who believes a blocking finding is acceptable downgrades the finding's severity, with justification, rather than overrides the verdict. A reviewer who is uncertain whether an assumption is empirical or document-verifiable resolves the question in Phase 1 — not by inflating R-NN.

---

## Inputs

1. **WORK_UNITS.md unit entry** (required) — the entry for this unit (declared scope in/out, dependencies, files touched, declared SPEC read-set). This is the contract the SPEC must satisfy. Every `SF-NN` finding anchors to a verbatim quote of an "in scope" or "out of scope" bullet from this entry.
2. **SPEC.md** (required, primary subject) — the document being reviewed. Path is typically `add/U<NN>/SPEC.md`. Read end-to-end.
3. **The unit's declared design read-set** (required) — the design artifacts the SPEC was supposed to cite, as declared in WORK_UNITS for this unit. Subset of DOMAIN, ARCHITECTURE, INTERFACES, DATA, BEHAVIOR, ERRORS, QUALITY, SECURITY, USE_CASES, surface IAs. Used both for citation validation (every `INV-NN`, `EP-name`, `EVT-name`, `ERR_CODE`, `SM-*`, `SAGA-*`, `METRIC-*`, `SLO-*`, `THREAT-NN`, `MIT-NN`, `UC-NN`, `ADR-NN` cited by the SPEC must resolve in the corresponding registry) and for cross-checking that the SPEC's claims are consistent with the design.
4. **Dependency unit SPEC.md files** (required, auto-discovered) — the SPECs of every unit this unit depends on. Discovery: walk the dependency list in the WORK_UNITS entry for this unit and load each dependency's SPEC.md from `add/U<NN>/SPEC.md`. Used to verify that the SPEC consumes the dependencies correctly — exact exported names, exact function signatures, exact field names. Read end-to-end; truncated reads on dependencies cause silent signature drift.
5. **Existing codebase** (required, accessed via Grep, Glob, and Read) — for compatibility checks. The skill greps and reads files to verify SPEC claims about filename conventions, package shapes, framework patterns, existing types, and existing functions in the monorepo.
6. **Project guideline files** (auto-discovered) — `CLAUDE.md`, `README.md`, `CONTRIBUTING.md`, `.editorconfig`, and any linter configuration files at the repository root. These are the convention authorities that override default framework conventions when they conflict.
7. **Web** (accessed via WebSearch and WebFetch) — for library and framework documentation, OSS alternatives, RFCs, package registries (NPM, crates.io, PyPI, etc.), and breaking-change history.
8. **Issues and roadmap** (required, auto-discovered) — `issues/*/ISSUE.md` and `roadmap/*.md` files at the repository root. Used to verify whether items the SPEC defers to "future work" genuinely have a home elsewhere or are phantom deferrals.

Read-set size: 1 WORK_UNITS entry + 1 SPEC + 4–7 design artifacts + 1–4 dependency SPECs + tool reads (codebase, web, issues, roadmap). Within ADD's 10-artifact budget. Read every required input end-to-end before launching subagents — truncated reads produce phantom findings (a `QF-NN` claiming "INV-12 is uncited" when INV-12 appears on a line that was cut off).

---

## Workflow

The review proceeds in six phases. Phases 1, 3, 4, 5, 6 are sequential. Phase 2 launches four subagents in parallel and waits for all of them to complete before Phase 3 begins.

### Phase 1: Context & Citation Index

Read every required input end-to-end. As you read the SPEC, build an in-agent working index of:

- **Stable IDs cited:** every `INV-NN`, `EP-name`, `EVT-name`, `ERR_CODE`, `SM-{entity}-{state}` or `SM-{entity}: {from} → {to}`, `SAGA-name`, `METRIC-*`, `SLO-*`, `SPAN-*`, `THREAT-NN`, `MIT-NN`, `UC-NN`, `ADR-NN`, `CFG_NAME`, paired with the SPEC section that cites it.
- **Dependency imports claimed:** every type, function, or symbol the SPEC says it imports from a dependency unit, paired with the dependency unit ID and the SPEC section that imports it.
- **File manifest:** every file path the SPEC declares (under § 9 File Manifest or equivalent), paired with its declared role.
- **Behavioral paths:** every ordered behavioral path the SPEC describes — happy path, error paths, edge cases — paired with the SPEC section.
- **Deferred items:** every phrase in the SPEC that says "deferred to future unit", "out of scope for this unit", "will be addressed in PLAN/IMPLEMENT", or equivalent, paired with the SPEC section and the deferral target if named.
- **Decisions visibly deferred to PLAN/IMPLEMENT:** every place the SPEC defers a decision that should be made at the SPEC level — wire shape, lock granularity, error code, field name, library choice — even when not explicitly labelled "deferred".

The working index does not appear in the output directly; it feeds every Phase 2 subagent and Phase 3 cross-validation.

Also read project guideline files (`CLAUDE.md`, `README.md`, `CONTRIBUTING.md`, linter configs). These are passed to every subagent as convention context.

If any required input is missing — the WORK_UNITS unit entry cannot be located, the SPEC at the expected path does not exist, a declared dependency SPEC is absent — stop and emit a SPEC_REVIEW.md with `status: blocked` and `verdict: fix-required` describing exactly which input is missing. Do not invent missing content.

### Phase 2: Parallel Subagent Pass

Launch **four subagents in parallel**, one per analytical lens. Each subagent receives:

- The full SPEC.md content
- The WORK_UNITS unit entry (verbatim)
- The unit's declared design read-set (full content)
- Every dependency unit SPEC (full content)
- All project guideline file contents
- The Phase 1 citation index
- Its specific review instructions (from the corresponding file in `references/`)
- Tool access: codebase Grep / Glob / Read for compatibility and quality checks; WebSearch / WebFetch for OSS-library research

Read each subagent's instructions from the `references/` directory of this skill before launching:

| Subagent | Instructions file | What it produces |
|----------|-------------------|------------------|
| Scope & Risk Auditor | `references/scope-risk-auditor.md` | `SF-NN` scope findings (§ 2), § 2.1 Deferred Items Recovered, `R-NN` risk items (§ 5) |
| Compatibility Checker | `references/compatibility-checker.md` | `CF-NN` compatibility findings (§ 3) — framework conventions, monorepo patterns, dependency signatures, project-guideline mandates |
| OSS Library Scout | `references/oss-scout.md` | § 3.1 OSS Alternatives entries — library/framework recommendations with trade-offs |
| Internal Quality Reviewer | `references/quality-reviewer.md` | `QF-NN` quality findings (§ 4) — placeholder language, citation validity, malformed signatures, decisions deferred to PLAN/IMPLEMENT |

**Subagent role split — read carefully.** Citation validity (does `INV-7` resolve in DOMAIN § 9?) belongs to the Internal Quality Reviewer; signature consistency (does the dependency unit's SPEC export the exact field names this SPEC imports?) belongs to the Compatibility Checker. OSS Library Scout produces § 3.1 *recommendations* — not findings, no severity, no `CF-NN` ID — for cases where a well-known library could replace SPEC-prescribed custom logic. The Compatibility Checker produces `CF-NN` findings for actual mismatches against an authoritative source; if a project guideline (`CLAUDE.md`, `CONTRIBUTING.md`) explicitly mandates a specific library and the SPEC prescribes a different one, that *is* a `CF-NN` finding (mandate violation), not an OSS recommendation.

Wait for ALL four subagents to complete before proceeding to Phase 3.

### Phase 3: Cross-Validation Pass

Aggregate the subagents' outputs and resolve overlap:

1. **Deduplicate findings.** If two subagents flag the same SPEC location for overlapping reasons (e.g., Scope Auditor flags a deferred storage decision as recovered scope and Quality Reviewer flags the same line as a decision deferred to PLAN), merge them into one finding under the more authoritative subagent's prefix. Keep the more severe classification.

2. **Resolve OSS / risk overlap.** A library recommendation in § 3.1 that, if adopted, would settle a risk item in § 5 (e.g., "use `tokio-retry` instead of building retry logic from scratch" addresses the risk "we assume our hand-rolled exponential backoff is correct under contention") must appear in both, with one as primary and the other as a cross-reference. Choose primary by which framing is more actionable: if the risk is "does our custom logic behave correctly?", the OSS replacement is the primary fix and the risk is the cross-reference; if the risk is "does the library integrate cleanly with our setup?", the risk is primary and the OSS suggestion is supporting.

3. **Resolve scope / risk overlap.** A scope-recovery item in § 2.1 (e.g., "deferred timeout configuration must be added back to scope") that would resolve via reading docs is a scope finding, not a risk; verify the Scope Auditor classified it correctly.

4. **Verify severity calibration.** Every `blocking` finding must point to a specific, named harm: "wrong code will be written" (scope drift, signature mismatch, missing error code) or "critical scope is silently dropped". A finding marked blocking with only "this should be addressed" justification is downgraded to `high` with a note.

### Phase 4: Verdict + Required Changes / Prototype Brief

Apply the verdict mechanic strictly (see Rules § 1):

```
blocking_findings > 0                              →  verdict: fix-required
blocking_findings == 0  AND  risk_items > 0        →  verdict: prototype-needed
blocking_findings == 0  AND  risk_items == 0       →  verdict: pass
```

Then:

- If `verdict: fix-required`, write § 7 Required Changes — group every blocking and high-severity finding by the SPEC section it targets, and list the specific edit the SPEC must make on regeneration. This is the exact instruction set the orchestrator hands to the H1 re-run.
- If `verdict: prototype-needed`, write § 8 Prototype Brief — consolidate every `R-NN` into a question-driven brief that the H1.6 PROTOTYPE skill consumes. For each: question to answer, what to test, acceptable outcomes, reference materials.
- If `verdict: pass`, both § 7 and § 8 read as their omission stubs ("No changes required — pipeline may proceed to /PLAN.md." and "No prototype required — risk surface is empty.").

### Phase 5: Self-Check

Before writing, verify:

- Every finding has severity, a specific SPEC section or quote, an authority or evidence reference, and a proposed-resolution that is an actionable edit (not a vague suggestion).
- Every `SF-NN` cites a verbatim WORK_UNITS quote and a specific SPEC section.
- Every § 2.1 entry shows a documented search of `issues/` and `roadmap/` confirming no home for the deferred item.
- Every `CF-NN` cites a specific authority — a framework docs URL, a file path with relevant content, a `package.json` path, or a dependency unit SPEC section.
- Every § 3.1 entry has a real library name, a link, and an explicit trade-off note.
- Every `QF-NN` names the exact phrase or stable ID that is the defect — no "vague language" findings without naming the phrase.
- Every `R-NN` states why the question cannot be answered by reading docs alone (specific reason, not generic).
- Frontmatter counts match the body exactly.
- § 6 Verdict paragraph contains at least one positive observation.
- § 7 is present iff `verdict: fix-required`; § 8 is present iff `verdict: prototype-needed`.

### Phase 6: Frontmatter + Write

Compute counts. Write SPEC_REVIEW.md to `add/U<NN>/SPEC_REVIEW.md`. Set `status: complete` if § 9 reads "All questions resolved.", `has_open_questions` otherwise, `blocked` only when a missing input prevented review (Phase 1 abort).

---

## Rules

These rules govern the output document. Violations are detected by the Quality Checklist.

### 1. Verdict follows counts, mechanically

```
blocking_findings > 0                              →  verdict: fix-required
blocking_findings == 0  AND  risk_items > 0        →  verdict: prototype-needed
blocking_findings == 0  AND  risk_items == 0       →  verdict: pass
```

`blocking_findings` always wins, regardless of `risk_items` count. There is no subjective override. A reviewer who believes a blocking finding is acceptable must downgrade that finding's severity (with justification in the finding body) — not override the verdict. A reviewer who believes a SPEC should ship despite risks must demote the relevant `R-NN` to a `CF-NN` (if reading docs settles the question) or absorb it into Open Questions (if it is a review-level uncertainty), not override.

### 2. Severity is defined by downstream impact, not reviewer mood

- **blocking** — wrong code will be written, or critical scope is silently dropped, if the finding is not fixed. Drives `verdict: fix-required`.
- **high** — the finding will confuse the implementing agent and force a discretionary fix, but wrong code can be unwound without cascading regenerations.
- **medium** — cosmetic or convention issue that degrades readability without blocking execution.
- **low** — minor wording or style inconsistency.

`blocking` is rare. Inflating findings to blocking is the surest way to make the gate useless ("always fix-required"). Across a typical review, expect zero or one blocking finding; two or more is unusual and warrants self-scrutiny in Phase 3.

### 3. Stable IDs for findings

`SF-NN`, `CF-NN`, `QF-NN`, `R-NN` are assigned in order of first mention within their section, starting at `01`. IDs are permanent across regenerations of SPEC_REVIEW.md on the same SPEC — if H1.5 re-enters after a SPEC regeneration, the orchestrator may compare finding IDs across runs to confirm that a specific finding was addressed. § 3.1 OSS Alternatives entries are *not* findings and have no ID prefix; they count toward `oss_alternatives_suggested` only.

### 4. Risk surface is for empirical-only questions

`R-NN` items are reserved for assumptions that **cannot be verified by reading**. If the question "does the framework support X?" is answered by reading framework docs, that is a `CF-NN` (compatibility) or `QF-NN` (citation), not a risk. If the question requires running code to settle — measuring an actual timing, observing an actual response shape, integrating two libraries to see whether their types compose, calling an external API to see what it returns — it is a risk.

This discipline matters: PROTOTYPE is expensive. Inflating the risk surface makes prototyping a routine bottleneck. Most units have zero `R-NN` entries; one is common; two or more is rare. If you are unsure whether a question is empirical or document-verifiable, attempt to answer it from the docs first; if you succeed, it was never a risk.

### 5. No new design

The review proposes edits the SPEC must make on regeneration; it does not introduce new design content. A proposed addition that says "add a new endpoint `EP-foo` to handle Z" names the endpoint and the gap, but defers the wire format design to `/INTERFACES.md` — that's a finding for the design phase, not for SPEC-level resolution. The review's job is to surface the gap, not fill it.

### 6. If it can be done here, it must be done here

For scope analysis, the default disposition is: if work falls within the unit's natural scope and is not explicitly assigned to another unit's WORK_UNITS entry, an existing `issues/NNN-slug/ISSUE.md`, a `roadmap/slug.md`, or PROPOSAL non-goals, it should be done in this unit. Deferral is the exception, not the default. Items deferred without a documented home become recovered scope (§ 2.1 entries with corresponding `SF-NN` findings).

This rule is the encoding of the most effective scope-drift prompt observed in real-world testing: *"Did you discover anything that should be included in the scope of this issue and spec — either because it was deferred or because it was not included previously but actually more could be done? I want to do as much as possible to have it working as well as possible and not defer anything."*

### 7. Aggressive web research for OSS alternatives

For every block of custom logic the SPEC prescribes — parsers, validators, schedulers, retry/backoff, rate limiters, error wrappers, serialization, config loading, observability instrumentation, auth middleware, content addressing — search authoritative sources (official docs, GitHub repos, package registries, RFCs, MDN) for established libraries. Surface candidates with explicit trade-offs; do not unconditionally recommend replacement. The goal is not to flag every divergence but to surface knowledge the SPEC author may have missed.

### 8. Every finding cites evidence

Every `SF-NN` quotes the WORK_UNITS unit-entry bullet verbatim. Every `CF-NN` cites a specific authority (framework docs URL, repo file path, dependency SPEC section, `package.json` path, project guideline section). Every `QF-NN` cites the exact SPEC section and the exact phrase or stable ID that is the defect. Every `R-NN` quotes the SPEC location and states why the assumption cannot be verified by reading. Findings with no citation are rejected by the Quality Checklist.

### 9. Every finding's resolution is actionable

A proposed resolution is an exact edit: "Add a § 5 Error Surface row for `ERR_NAME_TAKEN` referencing ERRORS § 3 with the user-facing message and HTTP 409 mapping." It is not "fix the error handling" or "address the missing case". The implementing agent must be able to apply the edit without making any further design decisions.

### 10. Positive observations required

§ 6 Verdict paragraph names at least one concrete strength of the SPEC observed during the review — even when verdict is `fix-required`. Patterns to note: clean error catalog binding, full design citations, exhaustive test specifications, well-bounded scope, precise file manifest, careful dependency-signature alignment. Balanced reporting prevents the gate from reading as pure criticism and lets the orchestrator recognise when a re-run has improved the SPEC vs. when regressions have appeared.

### 11. Open Questions are review-level only

§ 9 Open Questions captures genuine cases where the reviewer cannot tell whether something is a defect or a deliberate choice and needs an authoritative answer. It is *distinct* from the SPEC's own open questions (which belong in the SPEC and would be resolved by the cross-cutting `/decide` skill before this review runs). Do not duplicate the SPEC's open-question list; surface only review-level ambiguities.

### 12. Single YAML frontmatter block

Exactly one YAML frontmatter block at the top, containing the universal fields (`skill`, `date`, `status`) plus the review-specific fields (`unit`, `verdict`, `scope_findings`, `compatibility_findings`, `quality_findings`, `risk_items`, `blocking_findings`, `deferred_items_recovered`, `oss_alternatives_suggested`, `open_questions`). Never emit two blocks. Counts match the body exactly.

### 13. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "various", "reasonable", "industry-standard", "best practice". Use exact IDs, exact section numbers, exact quotes, exact edit instructions. Unresolvable ambiguity surfaces in § 9 Open Questions with options, tradeoffs, and a recommendation.

---

## Output Format

```markdown
---
skill: SPEC_REVIEW.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
unit: {U-NN}
verdict: {pass | fix-required | prototype-needed}
scope_findings: {N}
compatibility_findings: {N}
quality_findings: {N}
risk_items: {N}
blocking_findings: {N}
deferred_items_recovered: {N}
oss_alternatives_suggested: {N}
open_questions: {N}
---

# SPEC_REVIEW: {U-NN} — {Unit Name}

> Per-unit gate review of `add/{U-NN}/SPEC.md` before `/PLAN.md`. Consumes the
> WORK_UNITS unit entry, the SPEC end-to-end, the unit's declared design
> read-set, every dependency unit SPEC, project guidelines, the codebase, web
> sources, and `issues/` + `roadmap/`. Produces a machine-readable verdict
> (`pass`, `fix-required`, `prototype-needed`) that the orchestrator branches on.

## § 1. Review Scope

**WORK_UNITS unit entry (verbatim):**

> {Quote the entire entry for {U-NN} from WORK_UNITS.md — title, tier, scope in,
> scope out, dependencies, files touched, declared SPEC read-set, interface
> summary. This anchors every SF-NN finding to a verifiable source.}

**Inputs reviewed:**

| Artifact | Path | Date | Status (frontmatter) | Read end-to-end? |
|----------|------|------|----------------------|------------------|
| SPEC.md (subject) | `add/{U-NN}/SPEC.md` | `{YYYY-MM-DD}` | `{status}` | yes |
| WORK_UNITS.md | `WORK_UNITS.md` | `{YYYY-MM-DD}` | `{status}` | (unit entry only) |
| `{design artifact 1}` | `{path}` | `{YYYY-MM-DD}` | `{status}` | yes |
| `{design artifact N}` | `{path}` | `{YYYY-MM-DD}` | `{status}` | yes |
| `{dependency SPEC 1}` | `add/{U-NN}/SPEC.md` | `{YYYY-MM-DD}` | `{status}` | yes |
| `{dependency SPEC N}` | `add/{U-NN}/SPEC.md` | `{YYYY-MM-DD}` | `{status}` | yes |
| Project guidelines | `CLAUDE.md`, `README.md`, `CONTRIBUTING.md` | — | — | yes |
| Codebase | (Grep / Glob / Read) | — | — | as needed |
| Web | (WebSearch / WebFetch) | — | — | as needed |
| `issues/` + `roadmap/` | (Glob / Read) | — | — | yes |

(If an input's frontmatter `status` is `has_open_questions` or `blocked`, the orchestrator should not have dispatched this review — note the anomaly as a `QF-NN` finding.)

---

## § 2. Scope Findings

Items the SPEC fails to cover from the WORK_UNITS unit entry. Numbered `SF-NN` in the order first surfaced.

#### SF-{NN}: {short title}

- **Severity:** `{blocking | high | medium | low}`
- **WORK_UNITS bullet (verbatim):** `{exact quote of the in-scope or out-of-scope bullet from the unit entry}`
- **What's missing in SPEC:** `{specific file / function / behavior / test absent — name the SPEC section that should contain it}`
- **Proposed addition:** `{exact edit — e.g., "Add a § 9 file entry for src/auth/middleware.ts with the verifyToken function signature `verifyToken(token: string): Promise<UserClaims>` per the auth dependency SPEC § 6."}`
- **Blocks:** `{downstream step affected — e.g., "/PLAN.md will produce no plan step for token verification; /implement will skip the middleware; /verify will fail UC-04."}`

(Repeat per finding. If none: "No scope findings — every WORK_UNITS bullet is covered by a SPEC section.")

### § 2.1 Deferred Items Recovered

Items the SPEC defers to "future work", "a later unit", or "PLAN/IMPLEMENT" that have no documented home. Each entry is paired with a corresponding `SF-NN` finding above.

#### {Quoted SPEC deferral text}

- **SPEC location:** `{section number and quote}`
- **Search of `issues/`:** `{result — e.g., "No issue under `issues/` mentions {topic}; searched for {keywords}."}`
- **Search of `roadmap/`:** `{result — e.g., "No roadmap entry mentions {topic}."}`
- **Search of WORK_UNITS:** `{result — e.g., "No other unit's scope-in bullet covers {topic}; checked U-01 through U-N."}`
- **Recommendation:** `{exact section and content to add to the SPEC on regeneration}`
- **Linked finding:** `SF-{NN}`

(Repeat per recovered item. If none: "No deferred items recovered — every deferral has a documented home.")

---

## § 3. Compatibility Findings

Mismatches between SPEC claims and an authoritative source — framework docs, monorepo patterns, dependency unit SPECs, or project guideline mandates. Numbered `CF-NN`.

#### CF-{NN}: {short title}

- **Severity:** `{blocking | high | medium | low}`
- **Type:** `{framework convention | monorepo pattern | filename | package shape | dependency signature | API contract | project-guideline mandate}`
- **SPEC claim (quote):** `{exact quote with section number — e.g., "§ 9 says 'config file at src/config/server.json'"}`
- **Authority:** `{specific source — e.g., "Next.js docs at https://nextjs.org/docs/... | existing file packages/auth/package.json | dependency SPEC add/U-03/SPEC.md § 6 | CLAUDE.md § 'Configuration conventions'"}`
- **Mismatch:** `{exact contradiction — e.g., "Next.js convention places config at next.config.js (or next.config.ts) at the package root; src/config/server.json is not a Next.js convention."}`
- **Proposed correction:** `{exact edit — e.g., "Replace § 9 'config file at src/config/server.json' with 'config file at next.config.ts at the package root, exporting a typed NextConfig object per Next.js docs.'"}`

(Repeat per finding. If none: "No compatibility findings — every SPEC claim is consistent with framework, monorepo, dependency, and project-guideline authorities.")

### § 3.1 OSS Alternatives

Recommendations for blocks of custom logic in the SPEC where a well-known open-source library could replace, augment, or complement the SPEC-prescribed approach. These are **not findings** — they have no severity and no `CF-NN` ID. Counted in `oss_alternatives_suggested`.

#### {Library or framework name}

- **SPEC's prescribed approach (quote):** `{exact quote with SPEC section number}`
- **Suggested library:** `{name and link — e.g., "tokio-retry — https://docs.rs/tokio-retry"}`
- **Why it fits:** `{specific reason grounded in the SPEC's stated requirements}`
- **Trade-offs:** `{what is gained — battle-tested behavior, less code, observability hooks, etc.; what is lost — added dependency, less control, version-pinning concern}`
- **Recommendation:** `{replace | augment | keep custom — and the reasoning}`

(Repeat per alternative. If none: "No OSS alternatives identified — the SPEC's custom logic blocks have no obvious library replacements within the project's existing dependency posture.")

---

## § 4. Quality Findings

Internal SPEC defects unrelated to the WORK_UNITS contract or external authorities. Numbered `QF-NN`.

#### QF-{NN}: {short title}

- **Severity:** `{blocking | high | medium | low}`
- **SPEC location:** `§ {N}` (with quote where the issue is at sub-section granularity)
- **Issue type:** `{placeholder language | missing citation | invented stable ID | unresolved citation | malformed signature | undefined term | decision deferred to PLAN/IMPLEMENT | open question that should have been resolved | frontmatter mismatch}`
- **The exact defect:** `{name the phrase, the stable ID, the signature — e.g., "§ 5 Error Surface row 3 references ERR_NAME_TAKEN; this code has no entry in ERRORS.md § 3 (only ERR_NAME_CONFLICT exists)."}`
- **Proposed correction:** `{exact replacement — e.g., "Replace ERR_NAME_TAKEN with ERR_NAME_CONFLICT in § 5 row 3 to match the registry; if the registry's name is wrong, surface it as an open question for /errors regeneration."}`

(Repeat per finding. If none: "No internal-quality findings — the SPEC passes its own rules.")

---

## § 5. Risk Surface

Assumptions in the SPEC that depend on empirical questions — library behavior under load, runtime semantics, integration friction between libraries, performance characteristics, external system shapes — that cannot be answered by reading docs alone. Each `R-NN` is a candidate for the H1.6 PROTOTYPE step. Numbered `R-NN`.

#### R-{NN}: {short title}

- **SPEC location:** `§ {N}` (with quote)
- **Assumption:** `{exact statement of what the SPEC implicitly or explicitly assumes}`
- **Why it cannot be verified by reading:** `{specific reason — e.g., "The Tokio runtime's `spawn_blocking` documents the default thread-pool size but not its behavior when 64+ blocking tasks are queued simultaneously, which is the SPEC's §-7 worst case. We need to run the load to observe queue-eviction behavior."}`
- **Suggested prototype:** `{one-line scope of what to test — e.g., "Spawn 100 concurrent `spawn_blocking` tasks each sleeping 100ms; measure queue depth and task-completion order over 5 runs."}`
- **Consequence if assumption is wrong:** `{which downstream step fails and how — e.g., "/implement will produce a handler that drops requests under load; /verify will catch the issue but only after a complete H3 cycle."}`

(Repeat per risk. If none: "No risks requiring empirical validation — every SPEC assumption is verifiable from documentation, codebase, or dependency SPECs. PROTOTYPE step not needed.")

---

## § 6. Verdict

**verdict: `{pass | fix-required | prototype-needed}`**

{One paragraph. Always begin with at least one positive observation grounded in the SPEC: name a specific strength such as "every error in § 5 binds to a registered ERR_CODE", "every dependency import in § 2 matches the dependency SPEC's exported signature exactly", "the test-specification triples in § 8 cover both happy path and every documented error path", or "the file manifest in § 9 names every file with a precise size hint". Then, if `verdict: fix-required`, name the single most significant blocking finding with its ID and proposed action. If `verdict: prototype-needed`, name the single most significant `R-NN` with its assumption and proposed prototype. If `verdict: pass`, conclude that the pipeline may proceed to /PLAN.md.}

---

## § 7. Required Changes

(Include this section only if `verdict: fix-required`. If `pass` or `prototype-needed`: write "No changes required — see § 6 verdict." and omit the rest.)

Grouped by the SPEC section the edit targets. Every item cites the finding ID(s) that drove it.

#### § {N} of SPEC.md — {section name}

- {specific edit} — per `SF-{NN}` / `CF-{NN}` / `QF-{NN}`
- {specific edit} — per `{finding-id}`

(Repeat per affected SPEC section. The orchestrator hands this list to the H1 SPEC re-run as additional input.)

---

## § 8. Prototype Brief

(Include this section only if `verdict: prototype-needed`. If `pass` or `fix-required`: write "No prototype required — see § 6 verdict." and omit the rest.)

Consolidated brief for the H1.6 PROTOTYPE skill. Each entry maps one `R-NN` to a self-contained prototype task.

#### R-{NN}: {short title}

- **Question to answer:** `{single-sentence question — e.g., "How does Tokio's spawn_blocking behave when 100+ blocking tasks are queued simultaneously?"}`
- **What to test:** `{exact test to run — code shape, inputs, what to measure}`
- **Acceptable outcomes:** `{the answers that, if observed, settle the assumption — and the answers that invalidate it}`
- **Reference materials:** `{paths and URLs — e.g., "tokio docs at https://docs.rs/tokio/...; SPEC § 7 worst-case scenario; dependency SPEC add/U-02/SPEC.md § 4 lock-acquisition flow"}`

(Repeat per `R-NN`. The orchestrator hands this section to the H1.6 PROTOTYPE skill as primary input.)

---

## § 9. Open Questions

Review-level ambiguities only — cases where the reviewer cannot tell whether a SPEC choice is a defect or a deliberate decision and needs an authoritative answer before the verdict can be finalised. Distinct from the SPEC's own open questions.

- [ ] `{Question — e.g., "SPEC § 6 prescribes BLAKE3 for content addressing; ARCHITECTURE § 4 names SHA-256 as the project standard. Is the SPEC's choice a deliberate departure (faster on the hot path) or convention drift?"}`
  - **Option A:** `{interpretation 1 with downstream consequence}` — `{tradeoff}`
  - **Option B:** `{interpretation 2 with downstream consequence}` — `{tradeoff}`
  - **Recommendation:** `{suggested resolution grounded in the artifacts read}`

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- End-to-end reading of WORK_UNITS unit entry, SPEC.md, declared design read-set, and dependency unit SPECs
- Tool-driven access to the codebase, project guidelines, the web, `issues/`, and `roadmap/`
- Scope findings under stable `SF-NN` IDs anchored to verbatim WORK_UNITS bullets
- Deferred-items recovery in § 2.1 with documented searches of `issues/`, `roadmap/`, and other units' WORK_UNITS scope
- Compatibility findings under stable `CF-NN` IDs against framework docs, monorepo patterns, dependency SPECs, and project-guideline mandates
- OSS-alternative recommendations in § 3.1 with library names, links, and trade-offs
- Quality findings under stable `QF-NN` IDs covering placeholder language, citation validity, malformed signatures, and decisions deferred to PLAN/IMPLEMENT
- Risk surface under stable `R-NN` IDs for assumptions requiring empirical validation
- Verdict (`pass | fix-required | prototype-needed`) computed mechanically from `blocking_findings` and `risk_items`
- Required-changes list grouped by SPEC section (only when `fix-required`)
- Prototype brief consolidating `R-NN` entries (only when `prototype-needed`)
- Positive observations about the SPEC's strengths
- Genuinely review-level ambiguities surfaced in § 9 Open Questions

### Out of scope

- Authoring or modifying SPEC content — owned by `/SPEC.md` (H1) on regeneration. This review proposes edits; the SPEC skill applies them.
- Running code to validate empirical assumptions — owned by `/PROTOTYPE.md` (H1.6). This review surfaces the risks; PROTOTYPE settles them.
- Cross-unit consistency checks within a tier — units in the same tier are independent (ADD invariant); each unit gets its own SPEC_REVIEW.
- Whole-system design consistency — owned by `/DESIGN_REVIEW.md` (F).
- Implementation correctness — owned by `/CODE_REVIEW.md` (H4).
- Acceptance verification against the SPEC scenarios — owned by `/VERIFICATION.md` (H5).
- Resolving the SPEC's own open questions — owned by the cross-cutting `/decide` skill before this review runs. Review-level ambiguities surface in § 9 instead.
- Bulk web research beyond the OSS-alternative scout — the OSS Library Scout is the only subagent tasked with broad web search; the others use the web only for citation verification.

---

## Quality Checklist

Before considering SPEC_REVIEW.md complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `unit`, `verdict`, `scope_findings`, `compatibility_findings`, `quality_findings`, `risk_items`, `blocking_findings`, `deferred_items_recovered`, `oss_alternatives_suggested`, `open_questions`)
- [ ] Single YAML frontmatter block at the top — never two
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "reasonable", "industry-standard", "best practice")
- [ ] § 9 Open Questions is present (empty with "All questions resolved." or with genuine review-level ambiguities only — distinct from the SPEC's own open questions)
- [ ] Output is self-contained — readable and actionable without re-opening the SPEC, the design read-set, or any dependency SPEC
- [ ] All nine sections § 1 through § 9 are present with their exact headings (§ 7 contains its omission stub when verdict is `pass` or `prototype-needed`; § 8 contains its omission stub when verdict is `pass` or `fix-required`; § 5 contains its empty-state line when no risks)
- [ ] § 1 Review Scope quotes the WORK_UNITS unit entry verbatim and lists every artifact read with paths, dates, and status
- [ ] All four subagents (Scope & Risk Auditor, Compatibility Checker, OSS Library Scout, Internal Quality Reviewer) were launched in parallel and all four completed
- [ ] Phase 3 cross-validation deduplicated overlapping findings and resolved OSS / risk and scope / risk overlap
- [ ] Every `SF-NN` cites a verbatim WORK_UNITS unit-entry bullet and names a specific SPEC section that should contain the missing content
- [ ] Every § 2.1 entry shows documented searches of `issues/`, `roadmap/`, and WORK_UNITS confirming no home for the deferred item
- [ ] Every `CF-NN` cites a specific authority (framework docs URL, file path, dependency SPEC section, project guideline section) and names the exact mismatch
- [ ] Every § 3.1 entry has a real library name, a link, an explicit trade-off note, and a `replace | augment | keep custom` recommendation — no severity, no `CF-NN` ID
- [ ] Every `QF-NN` names the exact phrase, stable ID, or signature that is the defect (no "vague language" findings without naming the phrase)
- [ ] Every `R-NN` states a specific reason why the question cannot be answered by reading docs alone — generic "we should test this" justifications are rejected
- [ ] Every finding's proposed resolution is an actionable edit that names the section to change, the exact new wording or stable ID to add, or the exact deletion to make
- [ ] Severity distribution is not inflated — `blocking` is reserved for findings whose unfixed form will cause wrong code to be written or critical scope to be silently dropped; medium and low make up the bulk of typical reviews
- [ ] `verdict` follows the mechanic strictly: `fix-required` iff `blocking_findings > 0`; `prototype-needed` iff `blocking_findings == 0 AND risk_items > 0`; `pass` iff both are zero
- [ ] § 6 Verdict paragraph contains at least one concrete positive observation about the SPEC's strengths
- [ ] § 7 Required Changes is present iff `verdict: fix-required`, grouped by SPEC section, and every edit cites the finding ID(s) that drove it
- [ ] § 8 Prototype Brief is present iff `verdict: prototype-needed`, with one entry per `R-NN`, each providing question, what to test, acceptable outcomes, reference materials
- [ ] Frontmatter counts match the body exactly: `scope_findings` equals `SF-NN` count; `compatibility_findings` equals `CF-NN` count; `quality_findings` equals `QF-NN` count; `risk_items` equals `R-NN` count; `blocking_findings` equals the count of findings with `severity: blocking` across `SF`, `CF`, `QF`; `deferred_items_recovered` equals the entry count in § 2.1; `oss_alternatives_suggested` equals the entry count in § 3.1; `open_questions` equals the unresolved checkbox count in § 9
- [ ] `status` is `complete` if § 9 reads "All questions resolved.", `has_open_questions` if any unresolved checkbox remains, `blocked` only when a missing input prevented review and the gap is named in § 1
