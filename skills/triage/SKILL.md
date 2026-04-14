---
name: TRIAGE.md
description: Analyze system verification failures, trace each to its originating design artifact, group related issues into fix batches, and produce an actionable fix plan. Use when asked to triage failures, diagnose integration bugs, analyze why the system broke, trace bugs to root causes, or produce a TRIAGE.md.
---

# Task: Triage System Verification Failures

## Objective

Given a SYSTEM_VERIFICATION.md (or equivalent failure report) documenting integration failures in a composed application, produce a TRIAGE.md that traces every failure to the specific design artifact that, if correct, would have prevented the bug — then groups related issues into ordered fix batches and provides specific artifact update instructions. The output enables a human to update the right documents (architecture, contract registry, specs) so the pipeline can re-run and produce correct implementations.

---

## Inputs

1. **System verification report** (required) — SYSTEM_VERIFICATION.md or equivalent document listing failures with symptoms, evidence, categories, and affected components.
2. **Architecture documents** (auto-discovered) — system structure, endpoint tables, component descriptions, conventions. Used to trace failures to architectural gaps.
3. **CONTRACT_REGISTRY.md** (auto-discovered) — wire-format contracts. Used to identify missing or incorrect contract entries.
4. **Unit SPECs** (auto-discovered) — per-unit specifications. Used to trace whether a spec was wrong, incomplete, or correctly specified but incorrectly implemented.
5. **Unit IMPLEMENTATION.md files** (optional) — implementation records. Used to check whether implementations deviated from their specs.
6. **Decision documents** (optional) — architectural decisions. Used to understand the rationale behind choices that may have contributed to failures.
7. **SPLIT_WORK.md** (auto-discovered) — work unit decomposition. Used to identify gaps where no unit covers a needed capability.

---

## Workflow

Triage proceeds in four phases: failure analysis, artifact tracing, batch grouping, and fix plan generation.

### Phase 1: Failure Analysis

Read the system verification report and build a complete picture of every failure.

For each failure:

1. **Understand the symptom.** What specifically went wrong? (HTTP 400, wrong field names, missing endpoint, service won't start, empty response, type error at runtime)
2. **Understand the evidence.** What was the expected behavior vs actual behavior? What HTTP responses, error messages, or screenshots were captured?
3. **Identify the boundary.** Which components are involved? Which side of the boundary is wrong — or are both wrong because they independently derived an unspecified contract?
4. **Classify the root cause category:**

| Category | What went wrong | Typical artifact gap |
|----------|----------------|---------------------|
| **Contract mismatch** | Client and server disagree on wire format — field names, casing, structure, types | CONTRACT_REGISTRY.md missing or incorrect entry; SPEC derived wire format from prose instead of registry |
| **Missing configuration** | Env var, migration, library setup, dev script not present | SPEC for infrastructure/scaffold unit incomplete; ARCHITECTURE.md missing config section |
| **Logic bug** | Code does the wrong thing despite correct contracts | SPEC was wrong (spec bug) or IMPLEMENTATION deviated from spec (implementation bug) |
| **Missing feature** | Something needs to exist but was never specified | SPLIT_WORK.md gap — no unit covers this capability; ARCHITECTURE.md missing component |
| **Third-party library** | Library requires setup not documented in architecture | ARCHITECTURE.md or SPEC missing library-specific requirements |

### Phase 2: Artifact Tracing

For each failure, identify THE specific artifact that is the root cause — the document that, if it had been correct, would have prevented the bug. This is the most important phase. Be precise — "the architecture is incomplete" is not actionable; "ARCHITECTURE.md is missing the response shape for `GET /repos/{id}/tree`, which caused CLI unit U21 and server unit U15 to independently derive incompatible types" is actionable.

**Tracing rules:**

1. **Contract mismatches trace to the contract source of truth.** If CONTRACT_REGISTRY.md exists but is missing the endpoint → the registry is the root cause. If no contract registry exists → the absence of a shared contract source is the root cause. If the registry has the correct entry but a SPEC ignored it → the SPEC is the root cause.

2. **Missing configuration traces to the owning unit's SPEC.** Every piece of infrastructure (env loading, migration scripts, docker-compose config) should be owned by a unit. If the SPEC doesn't mention it, the SPEC is incomplete. If no unit owns it, SPLIT_WORK.md has a gap.

3. **Logic bugs trace to either SPEC or IMPLEMENTATION.** Read the SPEC for the affected unit. If the SPEC correctly describes the expected behavior but the implementation does something different, the implementation is the root cause. If the SPEC itself describes wrong behavior, the SPEC is the root cause.

4. **Missing features trace to SPLIT_WORK.md or ARCHITECTURE.md.** If the feature should exist but no unit covers it, SPLIT_WORK.md has a decomposition gap. If the feature isn't mentioned anywhere in the architecture, ARCHITECTURE.md has a gap.

5. **Third-party library issues trace to ARCHITECTURE.md or the unit SPEC.** If the architecture documents the library but omits a critical configuration requirement, ARCHITECTURE.md is the root cause. If the architecture mentions the requirement but the SPEC doesn't include it, the SPEC is the root cause.

**For each failure, produce an artifact trace:**

```
Failure: {symptom}
Root artifact: {document path and section}
Gap: {what is missing or wrong in that artifact}
Fix: {what the artifact should say instead}
```

### Phase 3: Batch Grouping

Group related failures into fix batches ordered by dependency. Earlier batches must be fixed before later ones — both because they may block testing and because their fixes may cascade to resolve later issues.

**Standard batch ordering:**

| Batch | Focus | Why first |
|-------|-------|-----------|
| **Batch 1: Foundation** | System glue — env loading, migrations, service startup, database tables | Nothing can be tested if the system cannot start |
| **Batch 2: Contracts** | Wire-format alignment — field names, casing, request/response shapes | Contract fixes affect many features simultaneously; fixing them first prevents re-fixing individual features |
| **Batch 3: Feature-specific** | Individual endpoint, command, or component fixes that are independent of contracts | These are isolated fixes that do not cascade |
| **Batch 4: Ergonomics** | Dev tooling, docker-compose improvements, documentation, developer experience | Non-blocking but improve the development workflow |

Within each batch, group failures that share a root artifact — if five failures all trace to "CONTRACT_REGISTRY.md is missing Server API response shapes," they are one fix item, not five.

### Phase 4: Fix Plan Generation

Produce the fix plan with two distinct sections:

**Section A: Artifact Updates (for the human).** For each fix item, provide:
- Which document to update (exact file path)
- Which section within the document
- What to add, change, or remove — specific enough to make the edit without further research
- Why this change prevents the failure(s) it addresses

**Section B: Pipeline Re-entry (informational).** After the human updates artifacts:
- Which units need their SPEC regenerated (because the architecture or contract registry changed)
- Which units need re-implementation (because their SPEC will change)
- What the expected re-entry point is (typically: regenerate SPECs for affected units → re-run per-unit pipeline → re-run system verification)

---

## Rules

### Precision over generality

"Fix the contracts" is not actionable. "Add the response shape for `GET /repos/{id}/tree` to CONTRACT_REGISTRY.md § Server API with fields `entries: Array<{path: string, type: "blob" | "tree", sha: string}>` and `truncated: boolean`" is actionable. Every fix instruction must be specific enough to execute without further research.

### One root cause per failure

Every failure has exactly one root artifact. If a contract mismatch exists because the registry is missing AND the spec derived the wrong format, the root cause is the missing registry entry — had it existed, the spec would have referenced it correctly. Trace to the deepest cause in the chain.

### Do not propose code fixes

Triage identifies which design artifacts to update. It does not write code, modify implementations, or suggest code patches. The pipeline handles code generation — triage feeds corrected inputs back into the pipeline.

### Acknowledge what works

The triage report must include a section noting what worked well. If 15 of 20 scenarios passed, that is significant — the pipeline produced largely correct results and the failures are concentrated in identifiable categories.

---

## Output Format

```markdown
---
skill: TRIAGE.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions}
failures_analyzed: {N}
root_causes_identified: {N}
fix_batches: {N}
artifacts_to_update: {N}
units_to_reprocess: {N}
---

# TRIAGE: {project name}

## Summary

- **Failures analyzed:** {N} from SYSTEM_VERIFICATION.md
- **Root causes identified:** {N} unique artifact gaps
- **Fix batches:** {N} ordered batches
- **Artifacts requiring updates:** {N} documents
- **Units requiring reprocessing:** {N} units after artifact updates

### Failure Distribution

| Category | Count | % of total |
|----------|-------|------------|
| Contract mismatch | {N} | {N}% |
| Missing configuration | {N} | {N}% |
| Logic bug | {N} | {N}% |
| Missing feature | {N} | {N}% |
| Third-party library | {N} | {N}% |

---

## What Worked Well

{Acknowledge successes. How many scenarios passed? Which component boundaries
work correctly? What does this tell us about the pipeline's effectiveness?}

---

## Root Cause Analysis

### Failure {N}: {symptom title}

**Scenario:** {which system verification scenario}
**Symptom:** {what went wrong — expected vs actual}
**Category:** {contract_mismatch / missing_config / logic_bug / missing_feature / third_party_library}
**Root artifact:** `{document path}` § {section name}
**Gap:** {what is missing or wrong in the artifact}
**Fix:** {what the artifact should say — specific and complete}
**Related failures:** {list other failure numbers that share this root cause, if any}

(Repeat for each failure.)

---

## Fix Batches

### Batch 1: Foundation

{Description of what this batch addresses and why it must be done first.}

#### Fix 1.1: {title}

**Artifact:** `{file path}` § {section}
**Change:** {exact description of what to add/modify/remove}
**Resolves:** Failure(s) #{N}, #{N}
**Rationale:** {why this change prevents the failures}

(Repeat for each fix in this batch.)

### Batch 2: Contracts

{Description}

#### Fix 2.1: {title}

**Artifact:** `{file path}` § {section}
**Change:** {exact description}
**Resolves:** Failure(s) #{N}, #{N}
**Rationale:** {why}

(Repeat for each fix. Repeat for each batch.)

---

## Pipeline Re-entry Plan

After the artifact updates above are applied:

### Units Requiring SPEC Regeneration

| Unit | Reason | Affected artifact |
|------|--------|-------------------|
| {unit ID} | {why the spec needs regeneration — e.g., "CONTRACT_REGISTRY entry added for this endpoint"} | {which fix item} |

### Units Requiring Re-implementation

| Unit | Reason | Depends on |
|------|--------|------------|
| {unit ID} | {why — e.g., "SPEC will change to use correct wire-format field names"} | SPEC regen for {unit ID} |

### Re-entry Procedure

1. Apply all artifact updates from the fix batches above.
2. Re-enter the pipeline from Phase 2 (SPEC regeneration) for the units listed above.
3. Run the per-unit pipeline (PLAN → IMPLEMENTATION → CODE_REVIEW → VERIFICATION) for each affected unit.
4. Re-run SYSTEM_VERIFICATION to confirm the fixes.

---

## Open Questions

- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Analyzing every failure in the system verification report
- Tracing each failure to the specific originating design artifact
- Categorizing failures (contract mismatch, missing config, logic bug, missing feature, third-party library)
- Grouping related failures into ordered fix batches
- Providing specific artifact update instructions for the human
- Identifying which units need reprocessing after artifact updates

### Out of scope

- Fixing code or modifying implementations — triage identifies what to fix in design artifacts, the pipeline handles code changes
- Running the system or performing verification — SYSTEM_VERIFICATION handles that
- Updating the artifacts directly — the human applies the triage recommendations
- Creating new work units — if SPLIT_WORK.md has gaps, triage reports the gap; the human updates the work unit decomposition

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `failures_analyzed`, `root_causes_identified`, `fix_batches`, `artifacts_to_update`, `units_to_reprocess`)
- [ ] Every failure from the system verification report has a corresponding root cause analysis entry
- [ ] Every root cause identifies a specific artifact (file path and section), not a vague reference
- [ ] Every fix instruction is specific enough to execute without further research — exact field names, types, and values
- [ ] Failures sharing a root cause are cross-referenced and consolidated into single fix items
- [ ] Fix batches are ordered by dependency — foundation before contracts before feature-specific before ergonomics
- [ ] Pipeline re-entry plan lists every unit that needs reprocessing and why
- [ ] "What Worked Well" section acknowledges successes and pipeline effectiveness
- [ ] No code fixes or implementation patches — only design artifact update instructions
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
