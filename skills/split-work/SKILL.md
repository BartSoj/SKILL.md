---
name: SPLIT_WORK.md
description: Split architecture into implementation work units. Use when asked to split the project into smaller work unit, module, tasks that can be independently implemented.
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

## Output Format

Use the following structure:

```markdown
# Work Units

> Auto-generated from architecture documents.
> Each unit is one independent implementation session: ≤400 LOC, ≤10 tests, ≤6 files, 1 concept.

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
