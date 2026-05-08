---
name: RECONCILIATION.md
description: Reconcile a unit's SPEC and the top-level design suite with what implementation actually built — surface discrepancies, classify them, filter out implementation details, decide which discoveries are general truths about the system, and apply direct edits to SPEC.md and selected top-level artifacts. Use when asked to reconcile spec with implementation, update spec after implementation, run post-implementation spec update, do tier reconciliation, reconcile after a unit is done, or produce a RECONCILIATION.md.
---

# Task: Reconcile SPEC and Top-Level Design with Implementation

## Objective

Produce a RECONCILIATION.md that records every discrepancy between a unit's SPEC.md and what was actually built — and apply the resulting edits directly. The output document is an audit log; the side outputs are surgical edits to `add/U<NN>/SPEC.md` and, where discoveries qualify as general truth about the system, edits to top-level design artifacts at `add/` (DOMAIN.md, ARCHITECTURE.md, INTERFACES.md, DATA.md, BEHAVIOR.md, ERRORS.md, QUALITY.md, SECURITY.md, OPERATIONS.md, USE_CASES.md, surface IAs). The defining test of "done" is the user's stated criterion: **if the implementation were deleted and the pipeline re-run from the updated SPEC, the implementation would be reproducible without surprises.** Updated SPEC + updated top-level docs = sufficient design context. If after reconcile a fresh implementation would still hit the same surprises, the reconcile was incomplete.

This skill is the most destructive in the pipeline. It rewrites design documents that every future unit will read. Get it wrong and the design suite drifts into an implementation snapshot — the opposite of design. Get it right and the design suite stays trustworthy after every implementation cycle.

---

## Inputs

1. **SPEC.md** (required, primary subject) — `add/U<NN>/SPEC.md`. The unit's spec, which will be edited in place.
2. **IMPLEMENTATION.md** (required) — `add/U<NN>/IMPLEMENTATION.md`. The build report. Authoritative source for what was actually built and why deviations from the plan occurred.
3. **CODE_REVIEW.md** (required, all rounds) — `add/U<NN>/CODE_REVIEW.md` plus `CODE_REVIEW_R2.md`, `CODE_REVIEW_R3.md` if present. Findings and their resolution status.
4. **VERIFICATION.md** (required) — `add/U<NN>/VERIFICATION.md`. Acceptance evidence. Source for confirmed behavior.
5. **WORK_UNITS.md** (required, current unit's entry only) — `add/WORK_UNITS.md`. Used for scope boundaries and dependency context.
6. **Implementation source code** (required, accessed via codebase read tools) — the actual files listed in the SPEC's File Manifest. The ultimate source of truth for what exists. Read every file the SPEC claims this unit owns; spot-check internals where IMPLEMENTATION.md or CODE_REVIEW.md hint at deviations.
7. **Top-level design artifacts** (required, accessed selectively) — `add/DOMAIN.md`, `add/ARCHITECTURE.md`, `add/INTERFACES.md`, `add/DATA.md`, `add/BEHAVIOR.md`, `add/ERRORS.md`, `add/QUALITY.md`, `add/SECURITY.md`, `add/OPERATIONS.md`, `add/USE_CASES.md`, surface IAs. **Read on demand** based on what the unit touched. The unit's "Design References" subsection in SPEC.md § 1 names the candidate artifacts.
8. **Dependency unit SPECs and dependent unit SPECs** (required, auto-discovered) — `add/U<MM>/SPEC.md` for every unit named in this unit's dependency list and every unit that lists this unit as a dependency. Read to verify scope-leak classifications (so a discovery isn't silently absorbed into this unit's SPEC when it actually belongs to a sibling).

Read-set size: 4–5 per-unit artifacts + 1 WORK_UNITS entry + 0–4 top-level artifacts (only those the unit touched) + dependency SPECs as needed. Within the ADD read-set budget.

---

## Workflow

Reconciliation runs as a single agent in four phases. **Do not dispatch subagents at any point.** Every read, every classification, every edit is performed by this agent directly.

### Phase 1 — Gather and Classify (Discrepancy Discovery)

Read all required inputs end-to-end before classifying anything. The order:

1. SPEC.md (cover to cover — every section, every constant, every signature).
2. IMPLEMENTATION.md (especially "Deviations from Plan" and "Notes for Downstream Work").
3. CODE_REVIEW.md and any later rounds (focus on findings that were resolved by changing the code rather than fixing the spec).
4. VERIFICATION.md (any scenario whose "actual" diverged from "expected" but was deemed acceptable is a discrepancy).
5. WORK_UNITS.md — the unit's entry plus its dependencies and dependents.
6. Source files listed in the SPEC's File Manifest.

Then walk the SPEC section by section and compare against code and reports. For every place where SPEC and implementation/code disagree, record a `D-NN` entry (D-01, D-02, … — local to this RECONCILIATION; not registered in `decisions/`). Capture, verbatim:

- The SPEC's claim, with section reference (e.g., `SPEC § 4: "rejects empty input with INVALID_REPO_NAME"`).
- The implementation's behavior, with file and line (e.g., `src/repo/create.rs:42 returns Ok(()) when name is empty`).
- The source of divergence — quote the IMPLEMENTATION.md deviation entry or CODE_REVIEW.md finding that explains why this happened. If no entry explains it, mark "undocumented divergence."

Then classify each `D-NN` as exactly one of:

- **Spec-level fix** — the original SPEC had a wrong decision, a missing concept, an internal inconsistency, or was out of sync with another unit's SPEC. The fix belongs in this SPEC.
- **Implementation detail** — the implementation chose how to do something (naming of a private symbol, internal data structure, ordering of independent operations) that the SPEC does not need to prescribe. The fix is to **leave the SPEC silent on this**, not to add the detail.
- **Scope leak** — a change made during implementation of a different unit leaked into this unit's code, or vice versa. The fix belongs in the **other unit's** SPEC, not this one. Flag for redirection (see Phase 3 / § 7 Escalations).

For each discrepancy, write the classification and the reasoning explicitly. **If you are unsure whether a decision was correct or which unit a change belongs to, surface the discrepancy in § 9 Open Questions rather than deciding silently.** Reconcile is too high-stakes to guess.

If SPEC and implementation match verbatim throughout, record "No discrepancies — implementation matches SPEC verbatim." and proceed to Phase 4 with verdict `clean`.

### Phase 2 — Filter and Evaluate (Sanity Check)

Take every `D-NN` classified as **spec-level fix** in Phase 1 and apply four filters in order. Drop or reclassify discrepancies that fail any filter, and record the reasoning in the audit log.

1. **Is the implementation's decision actually correct?** Do not assume code is right. Evaluate the decision on its merits against the architecture, the unit's purpose, and the SPEC's intent. If the implementation made a wrong call, the fix belongs in the implementation (escalate via § 7), not the SPEC. The SPEC should describe what the unit *should* do; the implementation is what gets corrected.

2. **Does this belong in this unit's scope?** Check the WORK_UNITS dependency graph and the SPECs of dependency and dependent units. If a change originated from a downstream unit's implementation deviating from its own spec, it belongs in **that** unit's SPEC. Reclassify as a scope leak and redirect via § 7. Do **not** silently absorb it into this unit's SPEC just because the code happens to live here.

3. **Is this a specification concern or an implementation detail?** SPEC defines *what* and *why* — public contracts, behavioral requirements, key architectural decisions, and constraints whose violation would cause cross-unit, cross-phase, or wire-format consequences. SPEC does **not** define *how* — exact internal algorithm, exact internal data structure, exact private naming, exact line-by-line ordering of independent operations. Demote pure implementation details out of the SPEC update plan; they belong in the code, not the spec.

4. **Does the SPEC need to be prescriptive here, or can it leave room?** Apply the heuristic: "Would a competent implementer arrive at this decision without explicit guidance?" If yes, leave it out. The SPEC pins decisions where getting them wrong would cause cross-unit consequences. Implementation freedom below that level is not the SPEC's business.

Items dropped during filtering are recorded in § 3 of the output with the reason. The survivors form the SPEC Update Plan in § 4. Items reclassified as scope leaks move to § 7 Escalations with a redirection recommendation.

### Phase 3 — Top-Level Artifact Decision (Most Delicate)

For every surviving spec-level fix from Phase 2, ask: does this discovery belong in the SPEC alone, or does it imply a change to a top-level design artifact?

A change belongs in a top-level artifact when:

- The discovery is a **general truth about the system** that other units will benefit from.
- The discovery affects the design slice the artifact owns (DOMAIN: ubiquitous language, invariants, events; ARCHITECTURE: component topology, technology choices; INTERFACES: wire-format contracts, endpoints, events; DATA: persistent schema, indexes; BEHAVIOR: state machines, sagas, idempotency; ERRORS: error code registry; QUALITY: metrics, SLOs, observability; SECURITY: threats, mitigations; OPERATIONS: deployment, runbook, config; USE_CASES: actor-perspective catalog; surface IAs: per-surface interaction).
- Without the change, future units reading the artifact will hit the same wrong assumption.

A change does **not** belong in a top-level artifact when:

- It is unit-local — specific to how this unit implements its own concern.
- It references this unit's progress, build status, or implementation state.
- It references the codebase, file paths, or other implementation details below the design layer.

For each proposed top-level edit, record in § 5 of the output: target artifact, target section, current wording (verbatim, or "[section absent — to be added]"), new wording (exact replacement or addition), magnitude (minor or major), and a one-line justification stating *how* the proposed wording satisfies the top-level discipline (general truth, no codebase refs, no progress refs, slice-respecting).

**Self-check every proposed top-level edit before promoting it to § 5:**

- Is this a general truth about the system, or a unit-local detail? (Reject unit-local — keep it in the SPEC.)
- Does the proposed wording reference the codebase, file paths, or any unit folder? (Reject — rephrase as a general statement about the system, or drop.)
- Does the proposed wording reference progress, status, or "what we've built so far"? (Reject — rephrase or drop.)
- Does the change fit the slice the target artifact owns? (Reject if it crosses slices — propose for the right artifact instead.)
- Will future readers of this artifact benefit from the change, or is this an artifact of *this specific unit's path*? (Reject path-specific.)

**If any check fails, do not propose the edit.** Either rephrase until all checks pass, or drop it (and capture the discovery only in the SPEC).

If no top-level edits qualify, § 5 reads "No top-level updates — discoveries were unit-local."

### Phase 4 — Apply

Edits are applied directly by this agent. **No subagents. No proposing-for-orchestrator and waiting.** Apply, log, verify.

**Step 1: Apply SPEC edits.** For each item in the SPEC Update Plan (§ 4):

1. Use `Edit` for surgical replacements. Quote the current wording verbatim in `old_string` and the new wording verbatim in `new_string`. Preserve every unaffected line.
2. After applying, re-read the edited section to verify the change landed cleanly and surrounding context still makes sense.
3. Update SPEC.md frontmatter counts (`files_specified`, `tests_specified`, `errors_specified`, `estimated_loc_prod`, `estimated_loc_test`, `open_questions`) so they match the body after edits.
4. Log the edit in § 6 Edits Applied with a 3-5 line context snippet.

**Step 2: Apply top-level artifact edits.** For each item in the Top-Level Artifact Update Plan (§ 5):

1. Read the entire target artifact end-to-end before editing — top-level docs have voice, section structure, and conventions that must be preserved. Read first, edit second.
2. Identify the smallest change that captures the truth. Do not bloat. Do not add details that aren't general truths about the system.
3. Apply the edit using `Edit` for replacements or `Write` only when adding a brand-new section that has no existing prose to replace. Never rewrite the whole artifact.
4. Re-read the edited section after applying. Verify the change is consistent with the artifact's voice and structure, and that it satisfies all top-level discipline checks (no codebase refs, no progress refs, slice-respecting, general truth).
5. If the target artifact carries frontmatter counts that the edit affects, update them.
6. Log the edit in § 6 Edits Applied with target artifact, section, and a 3-5 line context snippet.

**Step 3: Abandon edits that fail re-read.** If re-reading reveals the edit is incorrect (lands in the wrong section, breaks structure, contains a residual codebase reference, or introduces an inconsistency), revert it and record the abandonment in § 6 Edits Applied with the reason. Do **not** leave a half-applied edit.

**Step 4: Compute the verdict.** From the magnitude of edits actually applied:

- `clean` — no edits applied (no discrepancies survived Phase 2).
- `minor-fix` — SPEC edits applied; top-level docs untouched, OR only minor scope-preserving touches (a missing invariant added; an existing endpoint description corrected; an error code registered).
- `major-fix` — SPEC edits applied AND top-level docs received structural changes (a new domain event added to INTERFACES; a new invariant added to DOMAIN; a new saga step added to BEHAVIOR; a new component added to ARCHITECTURE).

The orchestrator interprets `major-fix` as a signal to re-enter the relevant B/D/E phase and re-run F design-review before the next tier proceeds. The skill **does** apply the major-fix top-level edits directly (per the user's no-subagents/direct-edits invariant); it does not propose-and-wait. Setting the verdict and listing modified artifacts in `top_level_artifacts_modified` is what the orchestrator reads.

**Step 5: Write RECONCILIATION.md.** Populate every section of the Output Format (below). Frontmatter counts must match the body. § 6 Edits Applied must list every edit performed in Steps 1–3 in the order applied (including abandonments).

---

## Rules

These rules are the discipline that makes RECONCILE worth running. Encode every one in your behavior.

### Direct edits, no subagents

You make every edit yourself, using the `Edit` tool (for surgical replacements) or `Write` tool (only for adding new sections — never full-artifact rewrites). **Do not dispatch any subagent for editing or analysis.** Subagents introduce edit-drift — slightly different interpretations of "what should change" applied across separate contexts — and reconcile is too high-stakes to tolerate that drift. The user has stated this as a hard invariant: edits must be made by the agent itself.

This rule applies to *every* tool action in this skill: reads, classifications, evaluations, and writes. If you find yourself reaching for a subagent because the work feels large, you are wrong; partition your own work into smaller passes instead.

### Top-level discipline (the most important rule)

Top-level documents (`add/DOMAIN.md`, `add/ARCHITECTURE.md`, `add/INTERFACES.md`, `add/DATA.md`, `add/BEHAVIOR.md`, `add/ERRORS.md`, `add/QUALITY.md`, `add/SECURITY.md`, `add/OPERATIONS.md`, `add/USE_CASES.md`, surface IAs) are first-class citizens of the design suite. They:

- Do **not** depend on the codebase or any files in unit folders.
- Do **not** reference past work, future plans, or implementation progress.
- Each describes a different *slice* of the architecture. Slices do not bleed into one another. (DOMAIN: ubiquitous language and invariants. ARCHITECTURE: component topology. INTERFACES: machine-to-machine contracts. DATA: persistent schema. BEHAVIOR: state machines and sagas. ERRORS: error taxonomy. QUALITY: observability and performance. SECURITY: threat model and controls. OPERATIONS: deployment and runbook. USE_CASES: actor-perspective catalog. Surface IAs: per-surface interaction.)
- Are general truth without reference to past work or future plans.
- Describe what the system **converges toward**, not the current implementation state.

When you propose a top-level edit, the proposed wording must satisfy these constraints. **If you find yourself writing "we discovered" or "in U-12 we found" or "as of unit U-15" or "the implementation in `src/foo.rs` shows" or "this was added during the W3 tier", the edit is wrong** — rephrase as a general truth, or drop the proposal. Top-level wording should read as if it were authored from scratch by the original design phase agent, with no awareness that any unit has yet been built.

The acid test: would this wording still make sense in a fresh project with no implementation yet, given only the design context? If yes, it is a candidate for the top-level artifact. If no, it belongs in the SPEC (or nowhere).

### Filtering discipline (resist the urge to copy implementation into spec)

The biggest reconcile failure mode is the agent copying every implementation detail into the SPEC, producing a SPEC that's a code-summary rather than a contract. The result is a SPEC that no longer describes *what the unit must do* but *what this particular implementation happens to look like*.

When in doubt about whether to include something in the SPEC, ask: **"Would a competent implementer arrive at this decision without explicit guidance?"** If yes, leave it out. The SPEC pins the decisions where getting it wrong would cause cross-unit, cross-phase, or wire-format consequences. Below that level, the implementation is free.

Specific patterns to demote out of the SPEC update plan:

- Internal helper function names, private struct field names, internal module organization.
- Choice between equivalent algorithms (e.g., `HashMap` vs `BTreeMap` for an internal-only collection).
- Exact error message strings (unless the strings are part of a public wire-format contract).
- Line-by-line ordering of operations that have no observable ordering effect.
- Test implementation details below "what scenario this test verifies."

Pin in the SPEC only what the public contract or cross-unit/cross-phase consumers actually depend on.

### Spec update rules

When applying SPEC edits in Phase 4, encode these as hard rules:

- **Correct wrong decisions rather than mirroring the implementation.** Aim for the best spec, not a copy of what was built. If the implementation is wrong, the SPEC should describe what the unit *should* do, and the implementation is what gets fixed (escalate via § 7).
- **State requirements as capabilities and contracts, not mechanisms.** Say what must be true, not how to make it true. SPEC: "the function rejects empty input and returns `ERR_INVALID_REPO_NAME`." Implementation: how the rejection is structured.
- **Keep error path descriptions at "return an error indicating X" level.** Do not pin exact error message strings unless the strings are part of a public contract (e.g., wire-format error response bodies that clients parse).
- **For test specifications, describe essential scenarios and what they prove, not exhaustive every-test-case lists.** SPEC tests are guidance for what behaviors must be verified, not a copy of the test file.
- **If the spec previously had a deliberate decision that the implementation overrode, evaluate which is better rather than automatically taking the implementation's side.** A reverted implementation is sometimes the right call. If the original SPEC decision was sound and the implementation deviated for convenience, escalate via § 7 — the fix is in the code.
- **Where types or enums are designed to be extended by downstream units, add a note to that effect rather than pre-adding variants that belong to those units' scope.** Don't absorb a sibling unit's vocabulary just because this unit's code currently happens to handle it.
- **Make sure the document is internally consistent after all edits.** Cross-references resolve. Section counts in frontmatter match the body. Design citations (`INV-NN`, `EVT-name`, `SM-*`, `SAGA-*`, `METRIC-*`, `SLO-*`, `THREAT-NN`, `MIT-NN`, `UC-NN`, `ERR_CODE`) still point to live IDs in the live registries. After all edits, a second pass through the SPEC must reveal no contradictions.

### Scope-leak redirection

When a discrepancy is classified as a scope leak, the skill does **not** silently fix it in this unit's SPEC. Surface it in § 7 Escalations with a redirection recommendation:

> "This change belongs in `U-XX`'s SPEC because [reason — cite the dependency graph or the sibling unit's scope statement]. Recommend a separate reconcile run for `U-XX` (or a scope-correction triage entry) to apply the change there. Do not absorb it into this unit's SPEC."

Scope leaks are escalations, not absorptions. The orchestrator owns the decision of whether to schedule another reconcile or to triage the inter-unit boundary.

### Convergence to truth (the success criterion)

Encode the user's success criterion as the gate for considering reconcile done:

> If the implementation were deleted and the pipeline re-run from the updated SPEC, the implementation would be reproducible without surprises. Updated SPEC + updated top-level docs = sufficient design context. If after reconcile a fresh implementation would still hit the same surprises, the reconcile was incomplete.

Before declaring `complete`, mentally run this test on every section that received edits. If you can identify a class of surprise that a fresh re-implementation would still hit, the reconcile is incomplete — there is a missing edit somewhere (in the SPEC, in a top-level artifact, or in § 7 Escalations).

### No contradictions with codebase

After all edits, the SPEC must have no contradictions with the actual codebase state as evidenced by IMPLEMENTATION.md, CODE_REVIEW.md, VERIFICATION.md, and the inspected source files. Internal consistency extends to consistency with the code: every claim in the SPEC must be either true of the current code or true of the code that the implementation owes (and the divergence must be tracked in § 7 Escalations as "implementation must be corrected").

If the SPEC says "function `foo` returns `Result<T, E>`" and the code returns `Option<T>`, then either (a) the SPEC is right and the code must change (escalation), or (b) the code is right and the SPEC must be edited. There is no third option of leaving them inconsistent.

### Edit log integrity

§ 6 Edits Applied is the audit trail. Every edit performed in Phase 4 must appear in § 6, in the order applied. If an intended edit is dropped mid-application (e.g., re-reading reveals it's incorrect), record the abandonment and the reason in § 6 — do not silently omit. The user must be able to reconstruct exactly what reconcile changed by reading § 6 alone.

### Verdict mechanics

The verdict is computed mechanically from edits actually applied, not from intent:

- `clean` — zero edits applied (no discrepancies survived Phase 2).
- `minor-fix` — SPEC edits applied; top-level docs untouched, or top-level edits were minor (a missing invariant added to DOMAIN; an existing endpoint description corrected in INTERFACES; an error code registered in ERRORS — anything that adds detail without changing the artifact's structure).
- `major-fix` — SPEC edits applied and top-level docs received structural changes (a new domain event added to INTERFACES; a new invariant added to DOMAIN; a new saga step added to BEHAVIOR; a new component added to ARCHITECTURE — anything that changes the slice's structure).

`major-fix` triggers orchestrator action (re-enter B/D/E + re-run F design-review). The skill records the verdict and the list of modified artifacts in `top_level_artifacts_modified`; it does not orchestrate the re-entry.

### No-interactive-questions rule

This skill never pauses to ask the user a question during execution. Genuine ambiguities — cases where the skill cannot tell whether a discrepancy is a spec defect, an implementation artifact, or a scope leak — go in § 9 Open Questions with options, tradeoffs, and a recommendation. The skill is invoked headless and must produce its output without interactive prompts.

---

## Output Format

The output file is `add/U<NN>/RECONCILIATION.md`. It begins with a single YAML frontmatter block (never multiple), followed by the audit log. Every count in the frontmatter must match the body.

```markdown
---
skill: RECONCILIATION.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
unit: U-{NN}
verdict: {clean | minor-fix | major-fix}
discrepancies_found: {N}
spec_level_fixes: {N}
implementation_details_excluded: {N}
scope_leaks_redirected: {N}
spec_edits_applied: {N}
top_level_edits_applied: {N}
top_level_artifacts_modified: [{ARTIFACT_NAME}, ...]   # use [] when no top-level edit was applied
escalations: {N}
open_questions: {N}
---

# RECONCILIATION: U-{NN} — {Unit Name}

## § 1. Reconciliation Scope

### Inputs Read

| Artifact | Path | Date / version |
|----------|------|----------------|
| SPEC | `add/U{NN}/SPEC.md` | {date} |
| IMPLEMENTATION | `add/U{NN}/IMPLEMENTATION.md` | {date} |
| CODE_REVIEW | `add/U{NN}/CODE_REVIEW.md` | {date} |
| VERIFICATION | `add/U{NN}/VERIFICATION.md` | {date} |
| WORK_UNITS (this unit's entry) | `add/WORK_UNITS.md` § U-{NN} | {date} |
| {Dependency SPEC, if read} | `add/U{MM}/SPEC.md` | {date} |
| {Top-level artifact, if read} | `add/{ARTIFACT}.md` | {date} |

### Source Files Inspected

- `{path}:{line range}` — {what was checked}
- ...

(If none beyond what is implied by the SPEC's File Manifest: "All files in SPEC § 9 File Manifest read end-to-end.")

### Top-Level Artifacts Considered

| Artifact | Read? | Edited? |
|----------|-------|---------|
| DOMAIN | yes / no | yes / no |
| ARCHITECTURE | yes / no | yes / no |
| INTERFACES | yes / no | yes / no |
| DATA | yes / no | yes / no |
| BEHAVIOR | yes / no | yes / no |
| ERRORS | yes / no | yes / no |
| QUALITY | yes / no | yes / no |
| SECURITY | yes / no | yes / no |
| OPERATIONS | yes / no | yes / no |
| USE_CASES | yes / no | yes / no |
| {surface IA, if relevant} | yes / no | yes / no |

---

## § 2. Discrepancies Surfaced

For each gap between SPEC and implementation/code:

### D-{NN}: {short title}

- **SPEC says:** "{verbatim quote}" — `SPEC.md § {section}`
- **Implementation does:** "{verbatim quote or summary}" — `{file path}:{line}` (or `IMPLEMENTATION.md § {section}` / `CODE_REVIEW.md § {finding}`)
- **Source of divergence:** {why they diverged — quote from IMPLEMENTATION.md deviations or CODE_REVIEW.md, or "undocumented divergence"}
- **Classification:** {spec-level fix | implementation detail | scope leak}
- **Reasoning:** {why this classification — be explicit, not "obvious"}

(Repeat per discrepancy. If none: "No discrepancies — implementation matches SPEC verbatim.")

---

## § 3. Filtering and Evaluation

For each discrepancy classified as **spec-level fix** in § 2, record the result of applying the four filters:

### D-{NN}: {short title}

- **Filter 1 (implementation correct?):** {pass / fail with reason}
- **Filter 2 (in this unit's scope?):** {pass / fail with reason — if fail, reclassify as scope leak}
- **Filter 3 (specification or implementation detail?):** {pass / fail with reason — if fail, demote out of update plan}
- **Filter 4 (does SPEC need to prescribe?):** {pass / fail with reason — if fail, leave SPEC silent}
- **Conclusion:** {keep in SPEC update plan / drop with reason / reclassify}

(Repeat per spec-level-fix candidate. If none: "No spec-level-fix candidates after Phase 1.")

Items dropped during filtering are recorded here with the reason. Items reclassified as scope leaks move to § 7 Escalations.

---

## § 4. SPEC Update Plan

The final list of changes to apply to `add/U{NN}/SPEC.md`. For each:

### Edit S-{NN}

- **SPEC section:** § {section number and name}
- **Current wording (verbatim):**
  > "{exact current text}"
- **New wording (exact replacement):**
  > "{exact new text}"
- **Driving discrepancy ID(s):** D-{NN}, D-{NN}
- **Rationale:** {one line — why this is the right SPEC-level wording}

(Repeat per edit. If none: "No SPEC edits — implementation matched SPEC verbatim or all discrepancies filtered out.")

---

## § 5. Top-Level Artifact Update Plan

(Include only when reconcile identifies that a top-level artifact must change.)

### Edit T-{NN}

- **Target artifact:** {DOMAIN | ARCHITECTURE | INTERFACES | DATA | BEHAVIOR | ERRORS | QUALITY | SECURITY | OPERATIONS | USE_CASES | {surface IA}}
- **Target section:** § {section number and name}
- **Current wording:** "{verbatim quote}" or "[section absent — to be added]"
- **New wording:** "{exact replacement or new section content}"
- **Magnitude:** {minor | major}
- **Driving discrepancy ID(s):** D-{NN}
- **Justification:** {how this proposed wording is general truth about the system, satisfies the no-codebase-refs rule, the no-progress-refs rule, and fits the slice this artifact owns}

(Repeat per top-level edit. If none: "No top-level updates — discoveries were unit-local.")

---

## § 6. Edits Applied

Log of edits actually performed in Phase 4, in order. Every edit attempted (including abandonments) appears here.

### {N}. {file path} — {section}

- **Plan ID:** S-{NN} or T-{NN}
- **Edit summary:** {one line}
- **Diff sketch:**
  ```
  - {3-5 lines of removed context}
  + {3-5 lines of added context}
  ```
- **Status:** {applied | abandoned with reason}

(Repeat per edit, in the order applied. If none: "No edits applied — verdict is `clean`.")

---

## § 7. Escalations

Issues this skill cannot resolve and that must surface to the human or to a deeper design phase. Each entry:

### E-{NN}: {short title}

- **Description:** {what the issue is}
- **Why escalation:** {one of: discrepancy implies architectural rethink | conflict between IMPLEMENTATION and design suite that is neither spec-fix nor implementation-detail | SPEC and a sibling unit's SPEC are now in conflict | implementation is wrong and must be corrected (cite which discrepancy) | scope leak that belongs in another unit (cite the unit)}
- **Recommended next step:** {specific action — e.g., "Run reconcile for U-{MM}", "Re-enter B1 domain phase to add invariant INV-{NN}", "Open issue and re-run H3 implement for this unit", "Trigger triage on the U-{NN}/U-{MM} contract boundary"}
- **Driving discrepancy ID(s):** D-{NN}

(Repeat per escalation. If none: "No escalations.")

---

## § 8. Verdict

**Verdict:** {clean | minor-fix | major-fix}

{One paragraph: summary of what was reconciled, what was preserved, what was elevated to the design layer, and whether the convergence-to-truth criterion is satisfied (i.e., a fresh re-implementation from the updated SPEC + updated top-level docs would not hit the same surprises).}

---

## § 9. Open Questions

Reconciliation-level ambiguities only — cases where the skill cannot tell whether a discrepancy is a spec defect, an implementation artifact, or a scope leak from another unit, and needs an authoritative answer before resolution.

- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Reading the unit's full per-unit artifact set (SPEC, IMPLEMENTATION, CODE_REVIEW, VERIFICATION, WORK_UNITS entry) and the unit's source files.
- Selectively reading top-level design artifacts based on what the unit's SPEC § 1 Design References indicate were touched.
- Surfacing every discrepancy between SPEC and implementation/code as a `D-NN` audit-log entry.
- Classifying discrepancies as spec-level fix, implementation detail, or scope leak.
- Filtering spec-level-fix candidates through the four filters of Phase 2.
- Deciding whether each surviving fix belongs in the SPEC alone or also in a top-level artifact.
- Applying SPEC edits directly via `Edit` (and `Write` for new sections only).
- Applying top-level artifact edits directly, after self-checking each against the top-level discipline.
- Logging every edit (including abandonments) in § 6.
- Escalating scope leaks, implementation errors, and architectural rethinks via § 7 — never silently absorbing them.
- Computing the verdict mechanically from edits applied.
- Producing the RECONCILIATION.md audit log with frontmatter counts that match the body.

### Out of scope

- **Authoring fresh SPEC content beyond what reconcile applies** — owned by `/SPEC.md` skill (H1) when the orchestrator regenerates a SPEC.
- **Authoring fresh top-level design content** — owned by the corresponding design skill (`/DOMAIN.md` (B1), `/ARCHITECTURE.md` (B2), `/INTERFACES.md` (D1), `/DATA.md` (D2), `/ERRORS.md` (D3), `/BEHAVIOR.md` (E1), `/QUALITY.md` (E2), `/SECURITY.md` (E3), `/OPERATIONS.md` (E4), `/USE_CASES.md` (A2), surface IAs (C)) when the orchestrator triggers re-entry on a `major-fix` verdict.
- **Modifying implementation source code** — the implementation has already happened by H6. Reconcile updates docs to match what the system *should* be, not the other way around. If the implementation is wrong, escalate via § 7.
- **Cross-unit consistency repair** — units in the same tier are independent (WORK_UNITS tier-independence invariant). Sibling unit SPECs are not modified by this reconcile invocation. If a discrepancy belongs in a sibling SPEC, escalate the redirect via § 7.
- **System-level failure analysis** — owned by `/TRIAGE.md` (I2) for failures discovered during system verification, not unit-tier reconciliation.
- **Orchestrating re-entry to design phases** — the skill sets the verdict and lists modified artifacts; the orchestrator interprets and schedules re-entry.
- **Dispatching subagents for any reason** — hard rule from the user's invariant.

---

## Quality Checklist

Before considering a RECONCILIATION.md complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `unit`, `verdict`, `discrepancies_found`, `spec_level_fixes`, `implementation_details_excluded`, `scope_leaks_redirected`, `spec_edits_applied`, `top_level_edits_applied`, `top_level_artifacts_modified`, `escalations`, `open_questions`)
- [ ] Frontmatter counts match the body exactly: `discrepancies_found` = `D-NN` entries in § 2; `spec_level_fixes` = items kept after § 3 filtering; `implementation_details_excluded` = items demoted in § 3 as implementation detail; `scope_leaks_redirected` = items escalated as scope leaks in § 7; `spec_edits_applied` = `S-NN` entries in § 6 with status `applied`; `top_level_edits_applied` = `T-NN` entries in § 6 with status `applied`; `escalations` = entries in § 7; `open_questions` = unresolved items in § 9
- [ ] `verdict` computed mechanically: `clean` if zero edits; `minor-fix` if SPEC edits applied with no or minor top-level edits; `major-fix` if any structural top-level edit was applied
- [ ] `top_level_artifacts_modified` lists only artifacts that received an edit with status `applied` in § 6 (not those that were merely read)
- [ ] Every discrepancy in § 2 has all five fields (SPEC says / Implementation does / Source of divergence / Classification / Reasoning)
- [ ] Every spec-level-fix candidate has explicit pass/fail reasoning for all four filters in § 3
- [ ] Every § 4 SPEC Update Plan item has current wording (verbatim quote) and new wording (exact replacement)
- [ ] Every § 5 Top-Level Artifact Update Plan item has a justification that explicitly states how the proposed wording satisfies the top-level discipline (general truth, no codebase refs, no progress refs, slice-respecting)
- [ ] No § 5 entry's `New wording` field contains the words "we discovered", "in U-{NN}", "as of", "the implementation in", "currently", "so far", or any reference to a file path under `add/U<NN>/` — these signal the proposed wording is implementation reportage, not general truth (this check applies only to the proposed top-level wording, not to the structural metadata fields like `Target artifact` or `Driving discrepancy ID(s)`)
- [ ] § 6 Edits Applied logs every edit performed in Phase 4 (including abandonments) with file path, plan ID, edit summary, diff sketch, and status
- [ ] Every § 4 SPEC Update Plan item with no abandonment in § 6 corresponds to an `applied` entry in § 6, and vice versa (no orphan plans, no unplanned edits)
- [ ] Every § 5 Top-Level Update Plan item with no abandonment in § 6 corresponds to an `applied` entry in § 6, and vice versa
- [ ] After SPEC edits, `add/U<NN>/SPEC.md` frontmatter counts (`files_specified`, `tests_specified`, `errors_specified`, `estimated_loc_prod`, `estimated_loc_test`, `open_questions`) match the SPEC body
- [ ] After top-level edits, target artifacts' frontmatter (where applicable) is updated to reflect the edits
- [ ] Every escalation in § 7 names the type (architectural rethink / cross-unit conflict / implementation must be corrected / scope leak) and a specific recommended next step
- [ ] Scope leaks are escalated in § 7 with a redirection recommendation, never silently absorbed into this unit's SPEC
- [ ] No subagent invocations made anywhere in the skill execution (verifiable: no Task tool used)
- [ ] No new design content invented — only updates to truths the implementation revealed; no inventing endpoints, invariants, sagas, or events that the implementation did not actually surface
- [ ] After all edits, the SPEC has no contradictions with the codebase as evidenced by IMPLEMENTATION.md, CODE_REVIEW.md, VERIFICATION.md, and inspected source files
- [ ] After all edits, mentally running the convergence-to-truth test (a fresh re-implementation from the updated SPEC + updated top-level docs would not hit the same surprises) passes; if not, the reconcile is incomplete and missing edits are surfaced in § 7 or § 9
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] § 9 Open Questions section is present (empty or with genuine reconciliation-level ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
