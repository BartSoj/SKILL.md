---
name: WORK_UNITS.md
description: Split architecture into implementation work units — the smallest independently-implementable slices (≤ 400 LOC, ≤ 10 tests, ≤ 6 files, 1 concept) organised into a tiered dependency DAG with per-unit concept, files, tests, dependencies, and interface. Use when asked to split the project into smaller work units, decompose the architecture, produce the work-unit catalogue, plan implementation slices, or produce a WORK_UNITS.md.
---

# Task: Split Architecture Into Implementation Work Units

You are given a project's architecture and design documents. Split the entire
project into the smallest possible **work units** that can each be implemented
independently by a single developer (or AI agent) in one session.

## Work Unit Constraints

Each work unit MUST satisfy ALL of these:

| Constraint        | Limit                                                    |
|-------------------|----------------------------------------------------------|
| Lines of code     | ≤400 LOC (production + test combined)                    |
| Test cases        | ≤10 per unit                                             |
| Files touched     | ≤6 files (created or edited)                             |
| Concept count     | Exactly 1 — one behavior, one capability, one boundary   |
| Code overlap      | Zero — no two units modify the same file                 |

## Splitting Rules

1. **Maximize the number of units.** The goal is the smallest scope per unit
   where each unit still compiles, passes its tests, and makes sense alone.
   If a unit could be split further without creating cross-unit file edits, split it.

2. **High cohesion within, low coupling between.** Group code that calls each
   other into one unit. Separate code that communicates only through interfaces,
   types, or HTTP boundaries.

3. **No file shared by two units.** If two units both need to touch `schema.ts`,
   either merge them or extract the shared part into its own unit that runs first.
   When a shared file (e.g., a types file) is used by multiple units, it becomes
   its own unit that all others depend on.

4. **Each unit is a vertical slice of one concept.** A unit includes the type
   definitions, implementation, and tests for one capability. Do NOT create
   units that are "all types" or "all tests" — each unit should be independently
   testable and demoable.
   **Unit tests ship with the code.** Every unit that adds testable logic MUST
   include its unit tests — bugs should be caught immediately, not deferred.
   **Exception — integration/e2e tests:** Integration tests span multiple units
   and cannot be bundled without violating the file-overlap rule. Place these as
   dedicated `(test-only)` units in a later tier, after assembly.

5. **Dependency ordering.** Units form a DAG. For each unit, list which other
   units must be completed first. Maximize the number of units that can run in
   parallel (no dependency between them).

6. **One concept ≠ one file.** Assembly units (app bootstrap, router wiring,
   main entry point) should contain ONLY the wiring — not the route handler
   logic or business logic they compose. If a single file (e.g., `routes.ts`)
   contains too much logic to fit in the assembly unit, split the route handlers
   into their own unit first and have the assembly unit import them.

7. **Anchor every unit to the design suite by stable ID.** Anchor-less units
   drift from design. Specifically:
   - **UI-implementing units** name the exact IA entry they realize — e.g.,
     `WEB_IA.md § pages.profile`, `CLI_IA.md § commands.login`,
     `MOBILE_IA.md § screens.settings`. Use the exact name from the IA, never
     a paraphrase.
   - **Units on an HTTP boundary** name the exact `EP-name` from
     `INTERFACES.md § 6` they implement on one side or consume on the other.
   - **Stateful / multi-step units** name the exact
     `SM-{entity}: {from} → {to}` transition or `SAGA-{name} step {N}` from
     `BEHAVIOR.md §§ 1–2` they implement.
   - **Data-persistence units** name the `{table_or_collection}` from
     `DATA.md § 3` they create or modify.
   - **Foundation / shared-type units** that have no surface declare
     `Realizes: Foundation — no surface` explicitly rather than leaving the
     field blank.

8. **Declare the SPEC read-set per unit.** Each unit lists the design artifacts
   its `/SPEC.md` agent must read end-to-end. This pre-computes the context
   budget for `/SPEC.md` so the orchestrator does not have to infer it. Typical
   read sets:
   - Every unit: `WORK_UNITS`, `DOMAIN`.
   - HTTP unit: add `INTERFACES`, `ERRORS`.
   - Persistence unit: add `DATA`.
   - Stateful / multi-step unit: add `BEHAVIOR`.
   - UI unit: add the relevant `{SURFACE}_IA`.
   - Performance-sensitive unit: add `QUALITY`.
   - Security-sensitive unit: add `SECURITY`.
   - Tier 1+ units: add the `SPEC.md` of every dependency unit.

   The read set is conservative — a unit that *might* touch a layer lists that
   layer. A unit that truly does not touch a layer omits it. The total read
   count per unit should stay at or under eight artifacts; if a single unit
   needs more than eight, it is over-scoped — re-split.

9. **Track supersession and impact explicitly — bidirectionally.** When a new
   unit renames, reshapes, replaces, or otherwise changes functionality
   delivered by an earlier unit, the new unit names the older unit(s) in its
   `Supersedes / Impacts` field with a one-line reason. When WORK_UNITS.md is
   regenerated or extended with such a unit, every impacted earlier unit
   receives a reciprocal `Impacted by: U{NN}` marker appended to its entry —
   so any agent reading the older unit's SPEC sees immediately that later
   work has changed the world around it and the spec may be out of date.
   Silent replacement is forbidden — an orphan change record is a contract bug.
   Categories (use the most specific that applies):
   - `Supersedes U{NN}` — the new unit fully replaces the older one; the old
     unit's `Files` are deleted or rewritten. The old entry is marked
     `(superseded by U{NN} on {YYYY-MM-DD})` but kept in the document.
   - `Impacts U{NN}` — the new unit changes types, wire formats, or contracts
     the older unit uses; the older unit's code still ships but its SPEC may
     no longer reflect current interfaces.
   - `Extends U{NN}` — the new unit adds to the older unit's surface without
     changing existing behavior (new endpoint alongside old, new flag on an
     existing command). Reciprocal marker is `Extended by: U{NN}`.

## Output Format

Use the following structure:

```markdown
---
skill: WORK_UNITS.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions}
total_units: {N}
tiers: {N}
max_parallel_tier_0: {N}
critical_path_length: {N}
units_superseded: {N}
units_impacted: {N}
open_questions: {N}
---

# Work Units

> Auto-generated from architecture and design documents.
> Each unit is one independent implementation session: ≤400 LOC, ≤10 tests, ≤6 files, 1 concept.
> Every unit anchors to the design suite by stable ID (`WEB_IA § pages.*`, `EP-name`, `SM-*`, `SAGA-*`, `{table}`) and declares the SPEC read-set its `/SPEC.md` agent needs.

## Summary

| Metric                | Value |
|-----------------------|-------|
| Total units           | N     |
| Max parallel (tier 1) | N     |
| Critical path length  | N     |
| Total estimated LOC   | N     |

---

## Dependency Graph

Units at the same tier have no dependencies on each other and CAN be built in parallel.

### Tier 0 — No dependencies
- U01, U02, U03

### Tier 1 — Depends only on Tier 0
- U04 (← U01), U05 (← U02)

### Tier 2 — Depends on Tier 0–1
- U06 (← U04, U05)

...

**Critical path:** U01 → U04 → U06 → ... (N units deep)

---

## Units

### U01: [Name] (≤5 words)

**Concept:** One sentence — the single capability this unit delivers.

**Repo:** repo-name (only needed for multi-repo projects)

**Depends on:** none | U01, U02

**Realizes:** The exact IA entry / endpoint / state transition / saga step / table this unit implements. Examples:
- `WEB_IA.md § pages.profile` (UI unit)
- `CLI_IA.md § commands.push` (CLI command)
- `EP-push-create` (INTERFACES.md § 6)
- `SM-repository: draft → published` (BEHAVIOR.md § 1)
- `SAGA-checkout step 3` (BEHAVIOR.md § 2)
- `accounts` table (DATA.md § 3)
- `Foundation — no surface` (shared types, config scaffolding)

Multiple anchors allowed when the unit spans layers — e.g., a server-side HTTP handler unit realizes both `EP-*` and a `SAGA-*` step.

**SPEC reads:** Comma-separated list of design artifacts the `/SPEC.md` agent must read end-to-end when specifying this unit — e.g., `DOMAIN, INTERFACES, ERRORS, BEHAVIOR, WEB_IA`. At minimum `DOMAIN`. Keep the total at or under eight artifacts; if more are needed, the unit is over-scoped and must be re-split.

**Supersedes / Impacts:** prior unit IDs whose behavior this unit renames, reshapes, or replaces, each with a one-line reason. Use `Supersedes U{NN} — {reason}`, `Impacts U{NN} — {reason}`, or `Extends U{NN} — {reason}` per rule 9. If this unit is purely additive: `None (new capability)`.

**Impacted by:** reciprocal marker — unit IDs added later whose work has changed the interfaces, contracts, or behaviors this unit was specified against. Initial value: `None`. Appended to by later regenerations when this unit appears in a later unit's `Supersedes / Impacts`. Format: `Impacted by: U{NN} ({on YYYY-MM-DD — one-line reason})`.

**Files (creates/edits):**
- `path/to/file1.ext` — what it contains
- `path/to/file2.ext` — what it contains

**Tests (~N):**
1. Brief description of test case
2. Brief description of test case

**Estimated LOC:** N prod + N test = N total

**Interface exposed:** What other units will import or call from this unit.

---

### U02: [Name]
...
```

## Guidelines for the output

- **Number units sequentially by tier.** All Tier 0 units come first (U01, U02, ...),
  then Tier 1, then Tier 2, etc. Never renumber units after assigning them — if
  you need to add a unit, append it at the end of its tier's range. The reader
  should be able to scan from U01 to the last unit and see a natural build order.
- **Multi-repo projects: tag each unit with its repo.** When the architecture
  spans multiple repositories or deployable services, add a `**Repo:**` field
  to each unit (e.g., `syns-git`, `syns-cli`, `syns/server`, `syns/web`). In
  the dependency graph, group tiers by repo where possible and show a
  per-component critical path alongside the overall critical path.
- Group related units under a comment or heading if helpful (e.g., `<!-- Git Engine -->`)
  but every unit must still have its own `### UXX` heading.
- In the dependency graph, show the arrow notation `(← dependency)` so the
  reader sees at a glance what must exist before building a unit.
- Keep test descriptions to one line each. They should communicate *what* is
  tested, not *how*.
- The `Interface exposed` field is critical — it tells the implementer of
  downstream units what contract to code against before the upstream unit exists
  (stubs, mocks, interface files).
- If a unit creates a shared abstraction (types, interfaces, error hierarchy),
  say so explicitly in the concept. These units tend to be small and appear in
  Tier 0.
- **Validate dependencies are complete.** If unit A creates a file that unit B
  imports or uses, B MUST list A in its `Depends on` field — even if both are
  test-only units. A missing dependency edge means two units could be built in
  parallel when one would actually fail.
- **Zero-test units are acceptable** for configuration, scaffolding, and CI/CD
  units where correctness is validated by downstream units compiling and passing.
  Mark these with `_Configuration only — validated by downstream units._`

---

## Quality Checklist

Before considering WORK_UNITS.md complete, verify each item:

- [ ] Output has valid YAML frontmatter with all fields (`skill`, `date`, `status`, `total_units`, `tiers`, `max_parallel_tier_0`, `critical_path_length`, `units_superseded`, `units_impacted`, `open_questions`); counts match the body
- [ ] At least one work unit is defined
- [ ] Units are numbered sequentially by tier (U01, U02, … with no gaps and no renumbering)
- [ ] Every unit fits within the constraints: ≤ 400 LOC combined, ≤ 10 tests, ≤ 6 files, exactly 1 concept
- [ ] No two units modify the same file (zero code overlap)
- [ ] Every unit has a `Concept` field stating its single capability in one sentence
- [ ] Every unit has a `Depends on` field listing upstream unit IDs (or `none` for tier 0)
- [ ] Every unit has a `Realizes` field citing at least one stable-ID anchor — `WEB_IA § pages.*` / `CLI_IA § commands.*` / `MOBILE_IA § screens.*` / `EP-name` / `SM-{entity}: {from} → {to}` / `SAGA-name step N` / `{table_name}` — or the literal `Foundation — no surface`
- [ ] UI-implementing units name an IA entry by the exact name used in the IA document
- [ ] HTTP-boundary units name at least one `EP-name` from INTERFACES.md § 6
- [ ] Stateful / multi-step units name at least one `SM-*` or `SAGA-*` from BEHAVIOR.md §§ 1–2
- [ ] Persistence units name at least one table / collection from DATA.md § 3
- [ ] Every unit has a `SPEC reads` field listing the design artifacts its `/SPEC.md` agent must read; `DOMAIN` is always present; total count is ≤ 8 artifacts
- [ ] Every unit has a `Supersedes / Impacts` field — either listing prior unit IDs with one-line reasons, or the literal `None (new capability)`
- [ ] Every unit has an `Impacted by` field — initially `None`; appended-to on regeneration when a later unit names this unit in its `Supersedes / Impacts`
- [ ] When this is a regeneration and a new unit supersedes / impacts / extends an older one, the older unit's `Impacted by` field has been updated with the reciprocal marker (bidirectional linkage — no silent replacement)
- [ ] Every unit has `Files`, `Tests` (or `_Configuration only — …_`), `Estimated LOC`, and `Interface exposed` fields populated
- [ ] Dependency graph is a DAG (no cycles); tier N depends only on tiers 0..N-1
- [ ] Critical path is named explicitly in the Dependency Graph section
- [ ] No placeholders or vague language (`appropriate`, `relevant`, `as needed`, `etc.`, `various`) anywhere in the document
- [ ] Superseded units retain their entry in the document with a `(superseded by U{NN} on {YYYY-MM-DD})` marker — never silently deleted
- [ ] Frontmatter `units_superseded` equals the count of units whose entry carries a `(superseded by …)` marker; `units_impacted` equals the count of units whose `Impacted by` field is non-empty
