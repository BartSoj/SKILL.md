---
name: IMPLEMENTATION.md
description: Execute an implementation plan task-by-task using test-driven development and delegate detail work to subagents. Produce an IMPLEMENTATION.md that records what was built, every deviation from the plan with rationale, and the final test-suite state. Use when asked to implement a plan, execute implementation steps, turn a plan into working code, or produce an IMPLEMENTATION.md.
---

# Task: Implement a Plan — TDD with Tasks and Subagents

## Objective

Produce working code that satisfies an implementation plan, plus an IMPLEMENTATION.md that serves as the post-implementation record of what actually happened — every task completed, every test run, every deviation from the plan, and every issue encountered. The document must let a downstream agent (reviewer, verifier, next work unit) pick up without re-reading the diff: it states the final test-suite state, the interfaces that were stabilised, and the gotchas discovered along the way.

The defining discipline — and the commonest violation — is **tests first, green before next task, deviations from the plan documented with rationale**: every task starts by writing tests that fail for the right reason, every task ends with the full suite green, and every divergence from the plan is captured in the output with a one-sentence why. The commonest failure is moving on to the next task while a test is still red — or silently making a design change that the plan did not call for and leaving the deviation undocumented. Both of those break the chain that downstream review and verification depend on.

---

## Inputs

1. **Implementation plan** (required) — the source of truth for what to build and in what order. Usually a `PLAN.md` in the repo; may also be a document or direct user input. Task ordering, verification criteria, and interface definitions come from here.
2. **Existing codebase** (required, auto-discovered) — the current state of the repository. Discovered from the working directory. The plan may reference or build upon existing files; read them before writing.
3. **Work-unit identifier** (optional) — the `U-NN` the plan belongs to, if the project is split into work units. Auto-discovered from the plan's frontmatter or from a parent `WORK_UNITS.md`. Used in the output frontmatter's `unit` field.
4. **Upstream artifacts** (optional, auto-discovered) — `DOMAIN.md`, `INTERFACES.md`, `BEHAVIOR.md`, or sibling `SPEC.md`s referenced by the plan. Read on demand when the plan cites them; do not pre-load everything.

---

## Workflow

Implementation proceeds in four phases: task creation, iterative TDD implementation, verification, and documentation.

### Phase 1: Create Tasks

Read the plan and break it into trackable tasks. Use your judgment on how to split them — the plan's structure is a guide, not a mandate. Each task should be a meaningful, completable unit of work.

### Phase 2: TDD Implementation Loop

Work through tasks following the plan's ordering. For each task, the recommended approach is test-driven development:

1. **Write tests first.** Based on the plan's verification criteria, write test cases covering happy path, error paths, and edge cases.
2. **Run tests — confirm they fail.** Verify the new tests fail for the right reasons (missing implementation, not syntax errors).
3. **Write the implementation.** Build the code described in the plan step.
4. **Run tests — confirm they pass.** Run the full test suite to verify correctness and no regressions.
5. **Iterate if needed.** If tests fail, fix and re-run. Do not move to the next task until all tests pass.

Mark each task as completed before moving to the next.

Use subagents to implement tasks. This preserves the main agent's context for orchestration and decision-making while subagents focus on the detail work of editing files and running tests. Give each subagent the relevant plan context and the TDD approach above.

Not every task benefits from writing tests first — configuration files, empty stubs, module wiring, and scaffolding can just be created and verified through compilation. Use judgment.

### Phase 3: End-to-End Verification

After all tasks are complete:

1. Run the full test suite
2. Run any linting or type-checking tools appropriate for the project
3. Walk through the plan's end-to-end verification checklist if one exists

### Phase 4: Produce IMPLEMENTATION.md

After everything passes, write an IMPLEMENTATION.md. This is a post-implementation record — it captures what actually happened, not what was supposed to happen.

---

## IMPLEMENTATION.md Output Format

The output file starts with a single YAML frontmatter block. There is only ever one frontmatter block — do not split fields across multiple YAML blocks. Every count in the frontmatter (`tasks_total`, `tasks_completed`, `tests_total`, `tests_passing`, `tests_failing`, `deviations_from_plan`, `commits`, `open_questions`) must match what the body reports. `status` is `complete` if every task is completed and `tests_failing == 0`; otherwise it is `has_open_questions` (if the body has unresolved questions but work is otherwise finished) or `blocked` (if a task could not be completed).

```markdown
---
skill: IMPLEMENTATION.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
unit: {U-NN}
tasks_total: {N}
tasks_completed: {N}
tests_total: {N}
tests_passing: {N}
tests_failing: {N}
deviations_from_plan: {N}
commits: {N}
open_questions: {N}
---

# IMPLEMENTATION: {Name}

## Summary

One paragraph: what was built, what it delivers, and its current state (compiling, tests passing, etc.).

## What Was Built

For each major component implemented, briefly describe:
- What it does
- Key implementation choices made during coding
- File path(s)

## Deviations from Plan

Any places where the implementation diverged from the plan:
- What changed and why
- Whether the deviation affects downstream work or interfaces

If no deviations: "Implementation followed the plan exactly."

## Test Results

- Total tests: N
- Passing: N
- Failing: N (with brief explanation if any)

List the test commands used and their output summary.

## Issues Encountered

Problems hit during implementation and how they were resolved:
- Build/dependency issues
- Unexpected behavior
- Plan ambiguities that required judgment calls

If none: "No issues encountered."

## Notes for Downstream Work

Anything that subsequent work or developers should know:
- Patterns established that should be followed
- Gotchas discovered
- Interface details that matter for integration

If none: "No special notes."
```

---

## Guidelines

These are recommendations, not rigid rules. Adapt based on the project, language, and complexity.

- **Prefer small, frequent commits** over one large commit at the end. Commit after each task or group of related tasks passes its tests.
- **Read before writing.** When the plan says to modify an existing file, read the current state first. The codebase may have evolved since the plan was written.
- **Trust the plan's ordering.** The plan was written with dependency analysis. If you think a step can be reordered, verify its dependencies first.
- **Keep the test suite green.** Never move to the next task with failing tests unless the failures are in tests for not-yet-implemented steps.
- **Surface blockers early.** If a step cannot be implemented as described, document it in Issues Encountered and resolve it before continuing.

---

## Quality Checklist

Before considering the implementation complete, verify:

- [ ] Every step in the plan has a corresponding completed task
- [ ] All tests pass (full suite, not just new tests)
- [ ] The project compiles/builds without errors or warnings
- [ ] Linting passes (if applicable to the project)
- [ ] IMPLEMENTATION.md is written and accurately reflects what happened
- [ ] Any deviations from the plan are documented with rationale
- [ ] No TODO comments or placeholder code left in the implementation
- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `unit`, `tasks_total`, `tasks_completed`, `tests_total`, `tests_passing`, `tests_failing`, `deviations_from_plan`, `commits`, `open_questions`)
- [ ] Frontmatter counts match the body (tasks completed, tests passing/failing, deviations listed, commits made, open questions raised)
- [ ] `status` is `complete` if every task is completed and `tests_failing == 0`; otherwise `has_open_questions` or `blocked`
