---
name: PLAN.md
description: Translate a specification into a sequenced implementation plan — ordered steps with inlined context (types, signatures, rules, constants, file paths), codebase citations for reuse and integration points, and per-step plus end-to-end verification. Use when asked to create a PLAN.md, plan implementation steps, turn a spec into actionable work, design a step-by-step build sequence for a feature, or produce a PLAN.md.
---

# Task: Create PLAN.md — An Implementation Plan

## Objective

Given a specification document and any available architecture or decision documents, produce a PLAN.md that an implementing agent can follow start-to-finish. The plan bridges "what to build" (the spec) and "how to build it" (the code) — it provides the sequence, the context, and the decisions needed to turn a spec into working software.

The defining discipline — and the commonest failure — is that **every step inlines the context the implementer needs**: types, signatures, behavioral rules, error variants, constants, and file paths are embedded directly in the step, not referenced by "see the spec" or "see the architecture doc." The implementing agent reads only PLAN.md. The commonest violation is leaving a step as a pointer to another document, or restating the spec at a summary level without concrete file paths and symbol citations — both produce plans that feel complete but force the implementer back into research mode.

---

## Inputs

1. **Specification document** (required) — the source of truth for what to build (scope, interfaces, behavior, errors, tests). Typically `SPEC.md` for the work unit.
2. **Architecture documents** (optional) — `ARCHITECTURE.md` and related design artifacts describing system structure, conventions, and technology choices.
3. **Decision documents** (optional) — recorded architectural decisions with rationale (e.g., ADRs). Used to lock in choices without re-litigating them in the plan.
4. **Existing codebase** (auto-discovered — via Glob and Grep during Phase 1 Codebase Exploration) — read full method bodies, module structures, and test infrastructure; findings are what the plan's steps inline as concrete citations.

---

## Workflow

Plan generation happens in three phases. The first two are research; the third is writing. Do not skip the research — a plan based on assumptions about the codebase instead of actual exploration will produce steps that duplicate existing utilities, violate conventions, or miss integration points.

### Phase 1: Codebase Exploration

Before writing anything, launch subagents to deeply explore the existing codebase. This is not a file-listing exercise — subagents should read full method bodies, understand patterns, and map out how existing code actually works.

What to explore:
- **Existing code related to the feature** — data structures, patterns, conventions actually in use (not just file paths)
- **Reusable utilities and abstractions** — the plan should never propose building something that already exists in the codebase
- **Similar features as implementation templates** — how comparable things were built elsewhere, as a reference for consistency
- **Integration points** — where new code connects to existing code, what interfaces and contracts it must satisfy

Launch multiple exploration subagents in parallel when the scope spans different areas. Give each a focused search area (one explores existing patterns, another explores integration points, another explores test infrastructure). Subagent findings become the inlined context in the plan's steps. Record the commit hash exploration was verified against and include it in the plan overview so a reader can re-verify the citations.

### Phase 2: Approach Analysis

After exploration, analyze the implementation approach before committing to a plan structure. This can be done via subagent or inline, depending on complexity.

Key questions to answer:
- Given the spec requirements and codebase findings, what implementation structure makes the most sense?
- What existing patterns should the implementation follow for consistency?
- What are the risks or tricky parts that need extra care?
- What ordering minimizes risk and allows incremental verification?

This phase produces the plan's step ordering, risk callouts, and pattern choices.

### Phase 3: Writing the Plan

Using the spec and the findings from exploration and analysis, write PLAN.md.

---

## Principles for Good Plans

These are guidelines, not rigid rules. Apply judgment — a two-step plan for a small feature doesn't need the same structure as a twenty-step plan for a complex system.

### Self-containment

Each step should embed everything the implementer needs: types, signatures, behavioral rules, error variants, constants. If a step depends on something from a previous step, restate the relevant details rather than saying "use the type from step 2." The plan is the only document the implementing agent reads.

### Completeness

- Every file that needs to be created or modified should appear somewhere in the plan with its full path
- Dependencies between steps should be explicit — what must exist before each step can begin
- Integration and wiring steps (registrations, route mounting, config changes) are easy to forget — don't
- For modifications to existing files, describe both the current state and the change, not just the delta

### Precision

- Use exact names from the specification — function names, type names, field names, paths
- Distinguish between creating a new file and modifying an existing one
- Order steps so that at each point, all dependencies are already implemented
- Avoid vague language ("as needed", "appropriate handler", "relevant types") — if a thing is needed, name it

### Reuse over reinvention

- Prefer existing codebase utilities, patterns, and abstractions over proposing new ones
- When referencing something discovered during exploration, include its file path and signature inline so the implementer doesn't need to search for it

### Verifiability

Each step should indicate how to verify it works — what tests to write, what to check, what command to run. The plan should also include an end-to-end verification approach for after all steps are complete.

### Inline rationale

When a step makes a non-obvious choice — a specific ordering, a particular library option, an unusual pattern, a workaround — include a brief "Why X:" rationale inline. Steps without rationale are likely to be undone during downstream edits.

---

## Output Structure

The plan should be organized so the implementer can work through it sequentially. The exact structure should fit the problem — a simple feature might need just an overview and a handful of steps, while a complex system might need dependency graphs and integration sections.

### Frontmatter

Every PLAN.md begins with a single YAML frontmatter block at the top of the file. The frontmatter provides the structured, machine-readable header for an otherwise adaptive body:

```yaml
---
skill: PLAN.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
unit: {U-NN}
steps_total: {N}
files_touched: {N}
risks_identified: {N}
open_questions: {N}
---
```

**Frontmatter rule:** there is exactly one YAML frontmatter block, at the very top of the file. Never split fields across multiple YAML blocks. The counts (`steps_total`, `files_touched`, `risks_identified`, `open_questions`) must match the body — if the body lists 12 steps, `steps_total: 12`; if it identifies 3 risks, `risks_identified: 3`. Update the counts in a final pass before finalising the document.

### Body

At a minimum, the plan should communicate:

1. **What is being built** — a brief overview derived from the spec, including key constraints and which codebase patterns are being followed
2. **Conventions / Patterns to follow** — a sub-section in the overview naming cross-step rules (naming, error mapping, casing, file-not-found handling, library conventions) once, so individual steps can reference them rather than restating
3. **Implementation order** — how steps relate to each other, what can be parallelized, what the critical path is. When the unit has natural commit boundaries — a multi-commit refactor where each commit must leave the system working — group steps into phases and document the gate criteria between phases
4. **Implementation steps** — the body of the plan. Each step should cover:
   - Which file is being created or modified
   - What other steps it depends on
   - All the context needed to implement it (types, signatures, behavioral rules — inlined, not referenced)
   - What specifically to build, in enough detail that the implementer doesn't need to make design decisions
   - How to verify the step is correct
5. **Integration and wiring** — the final steps that connect everything: route mounting, module registration, config changes, dependency installation
6. **End-to-end verification** — how to confirm the whole thing works once all steps are done

If genuinely ambiguous decisions remain that couldn't be resolved from the spec, architecture docs, and codebase exploration, place them in an **Open Questions** section with options, tradeoffs, and a suggestion. This should be rare — most decisions should be resolvable from the available inputs.

---

## Quality Checklist

Before considering the plan complete, verify:

- [ ] Codebase exploration was actually performed — the plan reflects real codebase state, not assumptions
- [ ] Every file from the spec's file manifest appears in at least one step
- [ ] Each step has enough inlined context to be implemented without referencing other documents
- [ ] No placeholders or vague language ("as needed", "appropriate", "etc.")
- [ ] Step ordering respects all dependency edges — no step references something not yet built
- [ ] Integration and wiring steps are included — nothing is left unconnected
- [ ] Existing codebase utilities are reused where applicable, not reinvented
- [ ] Modifications to existing files describe both current state and the change
- [ ] Open Questions (if any) contain only genuine ambiguities with options and suggestions, not lazy deferrals
- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `unit`, `steps_total`, `files_touched`, `risks_identified`, `open_questions`)
- [ ] Frontmatter counts match the body (`steps_total`, `files_touched`, `risks_identified`, `open_questions` all reflect the actual document)
