---
name: IMPLEMENTATION.md
description: Implement code from a plan using TDD and subagents. Use when asked to implement a plan, execute implementation steps, or turn a plan into working code.
---

# Task: Implement a Plan — TDD with Tasks and Subagents

## Objective

Given an implementation plan, execute it by breaking it into trackable tasks, implementing each using TDD, and delegating work to subagents to keep the main agent focused on orchestration. After all tasks pass, produce an IMPLEMENTATION.md documenting what was built, decisions made, and anything relevant going forward.

---

## Inputs

1. **Implementation plan** — the source of truth for what to build and in what order. May be a file, a document, or direct user input.
2. **Existing codebase** — the current state of the code, which the plan may reference or build upon.

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

```markdown
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
