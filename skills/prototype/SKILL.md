---
name: PROTOTYPE.md
description: Build a scratch prototype to empirically validate the `R-NN` risks surfaced by spec-review — write throwaway code under `units/<area>/u<NN>/prototype/`, run it, capture concrete evidence (output, dumps, timings, errors), and emit a verdict (`go`, `go-with-caveats`, or `abort`) that drives orchestrator branching back to SPEC regeneration. Use when asked to create a prototype, prototype an integration, empirically validate spec assumptions, test library behavior before implementation, prototype the risks flagged by spec review, or produce a PROTOTYPE.md.
---

# Task: Generate PROTOTYPE.md — Empirical Validation of Risky SPEC Assumptions

## Objective

Produce a PROTOTYPE.md that resolves every `R-NN` risk surfaced by G2 SPEC_REVIEW.md before the SPEC regenerates. Each risk is settled by writing scratch code in `units/<area>/u<NN>/prototype/`, running it, capturing concrete evidence, and translating the empirical findings into a constraint or caveat the SPEC must encode. The skill emits a machine-readable verdict (`go`, `go-with-caveats`, or `abort`) that the orchestrator branches on: `go` regenerates the SPEC unchanged in approach, `go-with-caveats` regenerates with the surfaced caveats encoded, `abort` regenerates with a fresh approach (or escalates to human if the prototype budget is exhausted). An agent reading PROTOTYPE.md alone — without re-reading the SPEC, the SPEC_REVIEW, or the prototype scratch code — can tell exactly what was empirically validated, what caveats apply, what new risks surfaced, and what specific edits the SPEC must apply on regeneration.

PROTOTYPE.md exists because real-world testing of the ADD pipeline showed SPECs frequently encoding **silent assumptions about library behavior, runtime semantics, and integration friction** — assumptions the SPEC author drew from documentation but never verified by running code. Implementations would discover the assumptions wrong only after a complete G5 implementation cycle, forcing expensive re-work. The defining discipline — and the commonest violation — is **evidence over reasoning**: every finding cites concrete output (a token-tree dump, a timing measurement, an error verbatim, a screenshot, a test runner pass/fail), not the agent's reasoning about what should happen. The commonest failure is declaring a risk "resolved" with no captured evidence — typically because the experiment was inconclusive but the agent moved on. If a finding cannot cite evidence, the experiment was not conclusive and must be iterated.

---

## Inputs

1. **SPEC_REVIEW.md** (required, primary driver) — for the unit. Read § 5 Risk Surface (the `R-NN` entries) and § 8 Prototype Brief (the consolidated question-driven brief) end-to-end. These are the questions to answer; every `R-NN` becomes a § 2 entry in the output. Path: `units/<area>/u<NN>/SPEC_REVIEW.md`.
2. **SPEC.md** (required) — for the unit. Read for context — what the unit is meant to do, what the proposed approach is, what the risks relate to. Do not modify. Path: `units/<area>/u<NN>/SPEC.md`.
3. **Design artifacts cited by `R-NN` entries** (required, scoped per risk — typically 1–2) — usually a subset of `INTERFACES.md` (for wire-format risks), `BEHAVIOR.md` (for state/saga risks), `DATA.md` (for schema risks), or one of the surface IAs (for UI risks). Read only the artifacts named by the `R-NN` entries' reference materials; do not pre-load the full design suite.
4. **Existing codebase** (required, accessed via Grep, Glob, and Read) — for reading production code that the prototype will integrate with or mimic. **Read-only.** Do not modify production code under any circumstances.
5. **Web** (required, accessed via WebSearch and WebFetch) — for library documentation, GitHub source, npm/crates/PyPI registries, RFCs, and the literal source files of installed dependencies when documentation diverges from observed behavior.

Read-set size: 1 SPEC_REVIEW + 1 SPEC + 1–2 design artifacts + tool reads (codebase, web). Within ADD's 10-artifact budget.

---

## Workflow

The skill is a focused, sequential, exploratory task — single-agent, no subagents. Five phases.

### Phase 1: Read Inputs

Read SPEC_REVIEW.md § 5 Risk Surface and § 8 Prototype Brief end-to-end. Read SPEC.md end-to-end for context. Read the 1–2 design artifacts cited by `R-NN` reference materials.

For each `R-NN`, build an in-agent working list:

- The verbatim question from SPEC_REVIEW § 8.
- The verbatim assumption from SPEC_REVIEW § 5.
- The reason the question cannot be answered by reading.
- The suggested prototype scope and reference materials.

If any required input is missing — `units/<area>/u<NN>/SPEC_REVIEW.md` does not exist, has empty Risk Surface, or has no Prototype Brief — stop and emit a PROTOTYPE.md with `status: blocked` describing exactly which input is missing or empty. Do not invent risks to prototype. PROTOTYPE only runs when SPEC_REVIEW emitted `verdict: prototype-needed`.

### Phase 2: Plan Prototypes

Group `R-NN` risks by the kind of experiment they require:

- Risks that share a runtime, library, or integration test → can be answered by one experiment.
- Risks that probe orthogonal assumptions → need separate experiments.
- Risks with measurable performance dimensions → need timing instrumentation.

Sketch a per-risk plan. For each risk:

- The minimum viable code to answer the question (1 file, 30 lines, 1 test, etc.).
- The dependencies that must be installed.
- The toolchain (node + tsx, cargo, uv + pytest, vitest, etc.) appropriate to the technology under test.
- The expected output shape that would settle the assumption.

Do not plan polished test suites. Plan throwaway experiments.

### Phase 3: Set Up Scratch Directory

Create the scratch directory at `units/<area>/u<NN>/prototype/`. Initialize a project appropriate to the technology under test:

| Technology | Initialization | Toolchain |
|------------|----------------|-----------|
| TypeScript / Node | `npm init -y && npm install -D tsx vitest @types/node` | `npx tsx <file>.ts` or `npx vitest` |
| Rust | `cargo init` (in subdir) | `cargo run` or `cargo test` |
| Python | `uv init && uv add <deps>` | `uv run <file>.py` or `uv run pytest` |
| Browser-touching | `npm init -y && npm install -D vitest jsdom @testing-library/dom` | `npx vitest --environment jsdom` |
| Multi-library integration | one of the above, plus the libraries under test pinned at the SPEC's intended versions | matching runner |

Install **only** the dependencies the experiments need. Pin versions explicitly when the SPEC names a target version — divergent versions invalidate the prototype.

**Critical: do not initialize anything outside `units/<area>/u<NN>/prototype/`. Do not modify production code, sibling units' prototype directories, or repository-root configuration files.**

### Phase 4: Execute Experiments Iteratively

For each `R-NN`:

1. **Write the smallest code that would answer the question.** A 30-line script that produces a token-tree dump beats a polished test class. Optimise for getting an answer, not for clean code.
2. **Run it.** Capture the output verbatim — stdout, stderr, exit code, timings, errors.
3. **Inspect the result.** Does it answer the question conclusively? If yes, move on. If no — the result is unclear, the assertion is ambiguous, the output reveals an unexpected shape — iterate:
   - Write a smaller test that isolates the unknown.
   - Add `console.log(JSON.stringify(...))` (or equivalent) to dump intermediate state.
   - Try a variant input.
   - Read library source if behavior diverges from documentation (see Rules § 4).
4. **Capture the evidence.** Save the relevant output to a code block in the in-progress § 2 entry. Save screenshots to `units/<area>/u<NN>/prototype/evidence/` for visual rendering risks. Save raw runner output for test-based risks.
5. **Classify the per-risk verdict:**

| Per-`R-NN` verdict | Meaning |
|--------------------|---------|
| `resolved` | The experiment produced conclusive evidence; the SPEC's assumption holds as-stated. |
| `resolved-with-caveats` | The assumption holds only under specific conditions (a config value, a version pin, an order constraint). The SPEC must encode the caveat. |
| `unresolved` | The experiment was inconclusive after iteration, OR the experiment ran but the result is ambiguous and a deeper investigation is needed. |
| `new-risk-discovered` | A different empirical question surfaced during the experiment that fundamentally challenges the SPEC's approach. |

When the answer is conclusive, move on to the next risk. Do not pad the prototype with experiments not driven by an `R-NN`.

The skill emphasizes **speed of discovery over code quality** (Rules § 2). The prototype is throwaway. A 30-line script that answers the question in 5 minutes is worth more than a polished test suite that takes an hour.

### Phase 5: Aggregate and Write PROTOTYPE.md

Compute the verdict mechanically (see Rules § 8):

```
all R-NN verdicts in {resolved}                                          → verdict: go
at least one resolved-with-caveats; none unresolved or new-risk          → verdict: go-with-caveats
at least one unresolved (material) OR
  at least one new-risk-discovered (material)                            → verdict: abort
```

A risk's implication is **material** if its consequence affects the SPEC's stated approach — wire shape, behavioral path, library choice, performance budget, error code. An unresolved risk on a non-load-bearing detail (a logging field name, a cosmetic edge case) is non-material and may be surfaced as a § 6 New Risk for spec-review to triage.

Write PROTOTYPE.md to `units/<area>/u<NN>/PROTOTYPE.md`. Frontmatter counts must match the body exactly. Self-check: every `R-NN` from SPEC_REVIEW § 5 appears in § 2 with a per-risk verdict and cited evidence.

The scratch directory at `units/<area>/u<NN>/prototype/` persists after PROTOTYPE.md is written. It is part of the artifact for reproducibility. Do not delete or prune.

---

## Rules

These rules govern the prototype and the output document. Violations are detected by the Quality Checklist.

### 1. Scratch directory discipline

The scratch directory `units/<area>/u<NN>/prototype/` is the prototype's sandbox. The agent has full freedom there:

- Initialize any project structure (`npm init`, `cargo init`, `uv init`, `vite scaffold`, `pnpm create`, etc.).
- Install any dependencies — they are local to the prototype dir; no impact on production.
- Write any files (`.ts`, `.tsx`, `.rs`, `.py`, `.js`, `.html`, `.test.*`, `.json`, scratch shell scripts, fixtures).
- Run scripts and tests repeatedly. Use `vitest`, `cargo test`, `pytest`, `tsx`, `node`, or any runner.
- Save screenshots, captured output, and fixture files inside the scratch dir (under `evidence/` for screenshots).

The agent has **no freedom outside the scratch directory.** Production code is read-only. Other units' prototype directories are off-limits. Repository-root configuration files (`package.json`, `Cargo.toml`, `pyproject.toml`, `tsconfig.json` at the repository root) are read-only.

The scratch directory persists after PROTOTYPE.md is written — it is part of the artifact for reproducibility. Do not delete it. Do not prune intermediate experiments.

### 2. Speed of discovery over code quality

> The prototype is throwaway. Code quality is irrelevant. Speed of discovery matters. Optimise for getting answers, not for clean code. A 30-line script that answers the question in 5 minutes is worth more than a polished test suite that takes an hour.

This is an explicit principle, not a fallback. Do not apologise for messy prototype code. Do not add abstractions, helper functions, type definitions, or test scaffolds beyond what the experiment requires. Do not rewrite working scratch code to be cleaner. The metric is `prototype_runtime_min` — wall-clock minutes between the start of Phase 4 (first experiment run) and the end of Phase 4 (last experiment run). Record the start timestamp before running the first experiment and the end timestamp after the last; the difference, rounded up to the nearest integer minute, is the value. Excludes Phase 1–3 (reading inputs, planning, scratch-dir setup) and Phase 5 (writing PROTOTYPE.md). The goal is to keep this number low while still capturing conclusive evidence.

### 3. Evidence over reasoning

Every finding in PROTOTYPE.md § 2 must cite concrete evidence — actual output, not the agent's reasoning about what should happen. Acceptable evidence patterns:

- Code blocks containing terminal output (stdout, stderr, exit codes).
- Token-tree or AST dumps from `console.log(JSON.stringify(...))` or equivalent.
- Timing measurements from `console.time`, `performance.now()`, `criterion`, `pytest-benchmark`.
- Error messages quoted verbatim with their stack trace if relevant.
- Screenshot file paths under `units/<area>/u<NN>/prototype/evidence/` for visual rendering risks.
- Test runner output (vitest, cargo test, pytest) with assertion details.
- Library source excerpts with a precise file path and line range (e.g., `node_modules/marked/src/Tokenizer.ts:142–156`).

If a finding cannot cite evidence, the experiment was not conclusive — iterate until it is, or mark the per-risk verdict `unresolved`. Do not paper over inconclusive evidence with reasoning.

### 4. Read library source when documentation disagrees with reality

When the running code diverges from what the library documentation claims, the library source is the ground truth. Read it directly. Source path conventions:

| Ecosystem | Source location |
|-----------|-----------------|
| npm / pnpm / yarn | `node_modules/<package>/dist/...` (compiled) or `node_modules/<package>/src/...` (when `src/` is published) |
| cargo | `~/.cargo/registry/src/index.crates.io-*/<crate>-<version>/src/...` — find with `cargo metadata --format-version 1 \| jq -r '.packages[] \| select(.name=="<crate>") \| .manifest_path'` |
| Python (uv / pip) | `<project>/.venv/lib/python<X.Y>/site-packages/<package>/...` |
| Go modules | `~/go/pkg/mod/<module>@<version>/...` |

Cite the exact file and line range when a finding is grounded in source code. Add the citation to the corresponding § 2 entry. The library source is the authority that overrides documentation when they disagree.

### 5. Surface caveats explicitly

If a risk is `resolved-with-caveats`, the constraint must appear in § 5 Caveats and Constraints with the exact wording the SPEC must encode. Categories of caveats:

- **Configuration values that must be set** (e.g., `library.configure({ strict: true })` before first use).
- **Version pins that must be respected** (e.g., `library@1.4.x`; 1.5.0 changed the default of option X).
- **Error paths that must be handled specifically** (e.g., library throws `TypeError` rather than the documented `ValidationError` when input is undefined).
- **Order-of-operations constraints** (e.g., renderer must be initialised before any tokeniser is attached).
- **Limitations requiring graceful degradation** (e.g., feature unavailable for inputs > 10MB; fall back to chunked path).

Caveats not surfaced in § 5 are caveats the SPEC author will not encode, which means caveats the implementation will violate.

### 6. No production code modifications

The prototype must not touch production code. If understanding production behavior requires running production code:

- Run it through the production code's existing test harness (read-only, no test additions).
- Or reproduce the relevant fragment in `units/<area>/u<NN>/prototype/` as a copy and modify the copy.

Direct modification of production code violates the prototype's read-only contract and corrupts the SPEC's regeneration baseline. Verifiable: every file edit during this skill happens inside `units/<area>/u<NN>/prototype/`.

### 7. New risks may surface

If a new empirical question emerges during prototyping that was not on the original `R-NN` list, surface it in § 6 New Risks Discovered with the same shape as a SPEC_REVIEW `R-NN` (assumption, why-not-readable, suggested prototype, materiality). Do not silently expand prototype scope to chase the new question; the new risk becomes input to the next SPEC_REVIEW round (or escalates to human if the prototype budget is exhausted).

### 8. Verdict discipline

The verdict is computed from per-`R-NN` verdicts mechanically:

```
all in {resolved}                                                 → go
at least one resolved-with-caveats; none unresolved or new-risk   → go-with-caveats
at least one unresolved (material) OR
  at least one new-risk-discovered (material)                     → abort
```

Materiality test: a per-risk verdict is **material** if its consequence affects the SPEC's stated approach (wire shape, behavioral path, library choice, performance budget, error code). Non-material `unresolved` or `new-risk-discovered` items are surfaced in § 6 and the overall verdict may still be `go-with-caveats`.

The verdict drives orchestrator behavior:

- `go` → SPEC regenerates with PROTOTYPE.md added to its inputs; the approach stands.
- `go-with-caveats` → SPEC regenerates with PROTOTYPE.md as input; § 5 Caveats are encoded into the SPEC.
- `abort` → SPEC must regenerate with a fresh approach (the current approach cannot work as-described). If the prototype round count has reached 2 (per ADD convergence cap), the orchestrator escalates to human instead.

There is no subjective override. A reviewer who believes a risk is acceptable downgrades it to `resolved-with-caveats` with an explicit caveat — not by overriding the verdict.

### 9. Risk-driven scope

The prototype tests only what the `R-NN` entries demand. Do not pad with generic edge-case tests, bench suites, or coverage exploration. Specifically:

- § 3 Edge Cases Tested includes only edge cases the prototype actually exercised in service of an `R-NN`, not a generic checklist.
- § 4 Performance Findings is included only when at least one `R-NN` has a performance dimension; otherwise it carries the omission stub.
- § 8 Recommendations is grounded in evidence captured during § 2 experiments, not general advice.

### 10. Single YAML frontmatter block

Exactly one YAML frontmatter block at the top of PROTOTYPE.md, containing the universal fields (`skill`, `date`, `status`) plus the prototype-specific fields. Never two blocks. Counts match the body exactly.

### 11. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "various", "reasonable", "best practice", "industry-standard". Use exact library names with versions, exact file paths, exact line numbers, exact field names, exact configuration values. Unresolvable ambiguity surfaces in § 9 Open Questions with options, tradeoffs, and a recommendation.

---

## Output Format

The template below is wrapped in a 4-backtick fence so that the inner 3-backtick blocks (the Reproduce shell block under § 1 and the evidence code blocks under § 2) render correctly. PROTOTYPE.md itself uses standard 3-backtick fences for those inner blocks.

````markdown
---
skill: PROTOTYPE.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
unit: {U-NN}
verdict: {go | go-with-caveats | abort}
risks_addressed: {N}
risks_resolved: {N}
risks_unresolved: {N}
caveats_surfaced: {N}
new_risks_discovered: {N}
prototype_runtime_min: {N}
open_questions: {N}
---

# PROTOTYPE: {U-NN} — {Unit Name}

> Empirical validation of the `R-NN` risks surfaced by `add/{U-NN}/SPEC_REVIEW.md`.
> Scratch code retained at `add/{U-NN}/prototype/`. Verdict (`go`, `go-with-caveats`,
> `abort`) drives orchestrator branching back to `/SPEC.md` regeneration.

## § 1. Setup

- **Scratch directory:** `add/{U-NN}/prototype/`
- **Toolchain:**
  - `{tool 1 name and version — e.g., "Node.js v22.9.0"}`
  - `{tool 2 name and version — e.g., "tsx 4.19.0"}`
  - `{...}`
- **Dependencies installed:**

| Package | Version (pinned) | Purpose |
|---------|------------------|---------|
| `{name}` | `{semver}` | `{which R-NN this supports}` |

(Repeat per dependency.)

- **Reproduce:**

```bash
cd add/{U-NN}/prototype/
{install command — e.g., "npm install"}
{run command 1 — e.g., "npx tsx experiment-r01.ts"}
{run command 2 — ...}
```

---

## § 2. Risks Addressed

For each `R-NN` from SPEC_REVIEW.md § 5:

### R-{NN}: {short title — copy from SPEC_REVIEW.md § 5}

- **Question (verbatim from SPEC_REVIEW § 8):** `{exact question}`
- **Approach:** `{one paragraph — what files in prototype/ contain the experiment, what they test, what assertions they make}`
- **Findings:** `{concrete factual answer — data, output, error messages, raw token-tree dumps, timing measurements. Use code blocks for evidence.}`

  ```
  {captured terminal output, JSON dump, error trace, timing measurement, etc.}
  ```

- **Implication for SPEC:** `{specific claim or constraint the SPEC must now reflect, with proposed wording for the SPEC section — e.g., "Replace § 6 'shiki returns serializable token trees' with 'shiki returns token trees containing function references for theme application; serialization requires a custom toJSON conversion (see units/<area>/u03/prototype/serializer.ts)'."}`
- **Code link:** `prototype/{filename}` (relative to the scratch dir)
  - `prototype/{filename-1}` — `{what it tests}`
  - `prototype/{filename-2}` — `{what it tests}` (if multi-file)
- **Library source consulted (if any):** `{e.g., "node_modules/marked/src/Tokenizer.ts:142–156"}`
- **Verdict:** `{resolved | resolved-with-caveats | unresolved | new-risk-discovered}`

(Repeat for each `R-NN`. Every `R-NN` from SPEC_REVIEW.md § 5 must appear here. If SPEC_REVIEW had no risk surface, do not run this skill.)

---

## § 3. Edge Cases Tested

Risk-driven only — only edge cases the prototype actually exercised in service of an `R-NN`. Do not include a generic checklist.

| Case | Expected | Actual | Verdict | Driving R-NN |
|------|----------|--------|---------|--------------|
| `{specific input or scenario}` | `{expected output}` | `{actual output}` | `{pass \| fail \| quirk}` | `R-{NN}` |

(Repeat per case. If none: "No edge cases tested beyond the primary R-NN experiments.")

---

## § 4. Performance Findings

(Include this section only if at least one `R-NN` has a performance dimension. Otherwise: "No performance dimension in any R-NN; section omitted.")

- **Baseline (current SPEC approach):** `{timing — e.g., "1240ms per 100-token tree, n=10"}`
- **Alternative tested (if any):** `{timing}`
- **Bottleneck identified (if any):** `{specific operation and its share of total time}`
- **Implication for SPEC:** `{specific budget or strategy the SPEC must encode}`

---

## § 5. Caveats and Constraints

Every constraint the SPEC must encode for the implementation to succeed. Each entry is paired with the `R-NN` that surfaced it.

#### {Caveat title}

- **Driving R-NN:** `R-{NN}`
- **Type:** `{configuration | version pin | error-path | order-of-operations | limitation}`
- **Constraint (exact wording for the SPEC):** `{e.g., "shiki must be initialized with codeToTokens (not codeToHtml) when rendering inside react-diff-view; see prototype/serializer.ts"}`
- **What breaks if missing:** `{specific implementation failure — e.g., "react-diff-view re-tokenises and discards shiki's theme metadata; the production code will render unstyled tokens."}`

(Repeat per caveat. If none: "No caveats — every R-NN resolved without constraints.")

---

## § 6. New Risks Discovered

Empirical questions that surfaced during prototyping but were not on the original `R-NN` list. These become input to the next SPEC_REVIEW round (or escalate if material and the prototype budget is exhausted).

#### {Short title}

- **Surfaced by:** `R-{NN} experiment in prototype/{filename}`
- **Assumption (newly identified):** `{exact statement of what was implicitly assumed}`
- **Why it cannot be answered by reading:** `{specific reason — same shape as SPEC_REVIEW R-NN justification}`
- **Suggested prototype:** `{one-line scope of what to test}`
- **Materiality:** `{material | non-material — and why}`

(Repeat per new risk. If none: "No new risks discovered.")

---

## § 7. Verdict

**verdict: `{go | go-with-caveats | abort}`**

{One paragraph. Begin with what was empirically validated (name the strongest finding — e.g., "shiki's token trees survive round-trip through react-diff-view's diff hunks when codeToTokens is used"). Then summarise: what caveats apply (point at § 5), what new risks surfaced (point at § 6), what the SPEC author must update before the unit can proceed. If `abort`, name the unresolved risk and the reason the current approach cannot work as-described. If `go`, conclude that the SPEC may regenerate with PROTOTYPE.md as input and proceed to /PLAN.md.}

---

## § 8. Recommendations for Production Implementation

Concrete, evidence-grounded advice for the SPEC author. Each recommendation cites the `R-NN` (and § 2 finding) that supports it.

#### {Recommendation title}

- **Driving R-NN(s):** `R-{NN}`
- **Recommendation:** `{exact API shape, exact strategy, exact infrastructure addition — e.g., "Use shiki.codeToTokens() and feed tokens to a self-recursive renderToken function that walks react-diff-view hunks; do not pass through codeToHtml() because react-diff-view re-tokenises HTML."}`
- **Final API/struct shape (if applicable):**

  ```
  {exact type signature or struct definition}
  ```

- **Quirks the production code must handle:** `{specific quirk and how — e.g., "Empty hunks with line:0 do not produce output; guard with `if (hunk.lines.length === 0) return null;`."}`
- **Graceful degradation needed (if any):** `{when and how — e.g., "Inputs > 1MB exceed react-diff-view's default chunking; fall back to plain <pre> rendering."}`

(Repeat per recommendation. If none: "No production recommendations beyond § 5 caveats.")

---

## § 9. Open Questions

Genuinely unresolved review-level questions where empirical evidence was contradictory, insufficient, or would require infrastructure beyond the scratch directory.

- [ ] `{Question — e.g., "Behavior under sustained load (10⁴ token trees / sec) was not measurable in a single-process prototype; do we need a load-test rig before SPEC?"}`
  - **Option A:** `{interpretation}` — `{tradeoff}`
  - **Option B:** `{interpretation}` — `{tradeoff}`
  - **Recommendation:** `{suggested resolution or "escalate to human"}`

(If none: "All questions resolved.")
````

---

## Scope

### In scope

- Reading SPEC_REVIEW.md § 5 Risk Surface and § 8 Prototype Brief end-to-end
- Reading the unit's SPEC.md for context
- Reading the 1–2 design artifacts cited by `R-NN` reference materials
- Tool-driven access to the codebase (read-only) and web sources
- Initialising a scratch project at `units/<area>/u<NN>/prototype/` with appropriate toolchain
- Installing dependencies inside the scratch directory at versions pinned by the SPEC
- Writing throwaway code, fixtures, scripts, and tests inside the scratch directory
- Running experiments iteratively, capturing output, dumps, timings, and errors as evidence
- Reading installed library source when documentation diverges from reality
- Producing PROTOTYPE.md with per-`R-NN` findings, caveats, recommendations, and a mechanically-computed verdict
- Surfacing newly discovered empirical risks in § 6 for the next SPEC_REVIEW round
- Retaining the scratch directory as a permanent artifact for reproducibility

### Out of scope

- Authoring or modifying SPEC content — owned by `/SPEC.md` (G1) on regeneration. PROTOTYPE proposes constraints and recommendations; the SPEC skill encodes them.
- Re-running spec-review or recomputing the risk surface — owned by `/SPEC_REVIEW.md` (G2).
- Modifying production code, sibling units' prototype directories, or repository-root configuration files — strict read-only outside the scratch directory.
- Implementation of the unit — owned by `/IMPLEMENTATION.md` (G5) after the SPEC is final.
- Cross-unit risk analysis — every unit's prototype is independent; risks for one unit do not leak across units.
- Long-running experiments (> 1 hour wall-clock) — surface as a § 6 New Risk and escalate; the prototype budget is for fast empirical answers, not extended benchmarking.
- Polishing prototype code, adding test coverage beyond what an `R-NN` demands, or refactoring scratch experiments — the prototype is throwaway.
- Resolving the SPEC's own open questions — owned by the cross-cutting `/decide` skill before SPEC_REVIEW runs.
- Fetching live data from production external systems — use staging endpoints or recorded fixtures inside the scratch directory.

---

## Quality Checklist

Before considering PROTOTYPE.md complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `unit`, `verdict`, `risks_addressed`, `risks_resolved`, `risks_unresolved`, `caveats_surfaced`, `new_risks_discovered`, `prototype_runtime_min`, `open_questions`)
- [ ] Single YAML frontmatter block at the top — never two
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "reasonable", "best practice", "industry-standard")
- [ ] § 9 Open Questions is present (empty with "All questions resolved." or with genuine review-level ambiguities only)
- [ ] Output is self-contained — readable and actionable without re-opening SPEC_REVIEW.md, the SPEC, or the prototype scratch code
- [ ] Every `R-NN` from SPEC_REVIEW.md § 5 appears in § 2 with a per-risk verdict (`resolved`, `resolved-with-caveats`, `unresolved`, or `new-risk-discovered`)
- [ ] Every § 2 finding cites concrete evidence (terminal output, JSON dump, error verbatim, timing measurement, screenshot path, test runner output, library source path) — no findings rest on agent reasoning alone
- [ ] § 1 Setup lists exact toolchain versions and exact pinned dependency versions; the Reproduce block contains commands that recreate the experiments end-to-end
- [ ] Scratch directory exists at `units/<area>/u<NN>/prototype/` and is not empty — every § 2 finding's Code link resolves to a file inside it
- [ ] § 5 Caveats and Constraints names every constraint the SPEC author must encode; each caveat is paired with the `R-NN` that surfaced it and states exactly what breaks if missing
- [ ] Verdict is computed mechanically from per-`R-NN` verdicts: `go` iff every per-risk verdict is `resolved`; `go-with-caveats` iff at least one is `resolved-with-caveats` and none are materially `unresolved` or `new-risk-discovered`; `abort` iff at least one is materially `unresolved` or `new-risk-discovered`
- [ ] § 6 New Risks Discovered is present (empty with "No new risks discovered." or with newly-surfaced empirical questions in the same shape as SPEC_REVIEW `R-NN` entries — assumption, why-not-readable, suggested prototype, materiality)
- [ ] No production-code modifications occurred — every file edit during this skill landed inside `units/<area>/u<NN>/prototype/`
- [ ] § 8 Recommendations are concrete (exact API shapes, exact struct definitions, exact strategies) and grounded in § 2 evidence — no vague "consider X" or "use the right Y" advice
- [ ] § 4 Performance Findings is present iff at least one `R-NN` had a performance dimension; otherwise it carries the omission stub
- [ ] § 7 Verdict paragraph names the strongest empirical validation finding and connects it to § 5 caveats and § 6 new risks
- [ ] Frontmatter counts match the body exactly: `risks_addressed` equals the count of § 2 entries; `risks_resolved` equals the count of per-risk verdicts in `{resolved, resolved-with-caveats}`; `risks_unresolved` equals the count of per-risk verdicts in `{unresolved, new-risk-discovered}`; `caveats_surfaced` equals the § 5 entry count; `new_risks_discovered` equals the § 6 entry count; `open_questions` equals the unresolved checkbox count in § 9
- [ ] `status` is `complete` if § 9 reads "All questions resolved.", `has_open_questions` if any unresolved checkbox remains, `blocked` only when a missing input prevented prototyping (Phase 1 abort)
