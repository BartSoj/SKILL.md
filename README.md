# SKILL.md

A Claude Code plugin for **Agent-Driven Development (ADD)** — a novel software development process where every step is an AI-agent invocation that produces one structured markdown artifact. You operate on the document level, not the message level. The trail of documents *is* the project history.

## Installation

```bash
claude plugin marketplace add bartsoj/SKILL.md
claude plugin install SKILL.md@SKILL.md
```

`/VERIFICATION.md` and `/SYSTEM_VERIFICATION.md` require [agent-browser](https://github.com/vercel-labs/agent-browser) for web application testing.

## What is Agent-Driven Development?

ADD is SDD pushed to its logical limit. Where SDD assumes a human threads context between steps, ADD assumes **every step is a stateless AI-agent invocation** whose only memory is the files it reads. Consequences:

- **Files are the only communication channel.** If it isn't in an artifact, it doesn't exist for the next step.
- **Every skill produces exactly one artifact.** Scope is bounded by what one agent can consume and produce.
- **Context budget is real.** Each step reads ≤ 10 artifacts; each artifact is ≤ 2000 lines (typical 500–1500).
- **Stable IDs cross-link artifacts.** `UC-NN`, `INV-NN`, `EP-name`, `ERR_CODE`, `SM-*`, `SAGA-*`, `THREAT-NN`, `U-NN` — assigned once, never renumbered.
- **Open questions are surfaced, not hidden.** Every artifact has an Open Questions section that the orchestrator resolves before handoff.
- **Feedback by regeneration.** Problems send you back to an earlier skill with extra context, not into edit loops.

## Skills by Phase

### Phase A — Discovery

| Skill | Produces | What it does |
|-------|----------|--------------|
| `/PROPOSAL.md` | `PROPOSAL.md` | Crystallises a one-paragraph idea into vision, principles, non-goals, and success criteria |
| `/USE_CASES.md` | `USE_CASES.md` | Enumerates actors and use cases with stable `UC-NN` IDs, priorities, and a surface-mapping matrix |

### Phase B — Foundation

| Skill | Produces | What it does |
|-------|----------|--------------|
| `/DOMAIN.md` | `DOMAIN.md` | Consolidated domain layer — glossary, bounded contexts, entities, aggregates, value objects, domain events, and `INV-NN` invariants |
| `/ARCHITECTURE.md` | `ARCHITECTURE.md` | Structural blueprint — components, cross-component flows, technology choices, and foundational `ADR-NN`s |

### Phase C — Surfaces (one per human-facing channel, if any)

| Skill | Produces |
|-------|----------|
| `/WEB_IA.md` | `WEB_IA.md` — web information architecture |
| `/CLI_IA.md` | `CLI_IA.md` — CLI command tree, grammar, exit codes |
| `/MOBILE_IA.md` | `MOBILE_IA.md` — mobile screen inventory, navigation, deep-link strategy |
| `/TUI_IA.md` | `TUI_IA.md` — terminal UI layout, mode, keybinding matrix |
| `/VOICE_IA.md` | `VOICE_IA.md` — voice intents, slots, dialog model |

### Phase D — Contracts & Data

| Skill | Produces | What it does |
|-------|----------|--------------|
| `/INTERFACES.md` | `INTERFACES.md` | Consolidated machine-interface contract — wire conventions, `EP-name` endpoints, `EVT-name` events, evolution policy |
| `/DATA.md` | `DATA.md` | Schema, indexes, access-patterns, migrations, sensitive-data inventory |
| `/ERRORS.md` | `ERRORS.md` | Unified error registry — `ERR_CODE` with HTTP status, CLI exit, UI message, user action |

### Phase E — Behavior & Non-functional

| Skill | Produces | What it does |
|-------|----------|--------------|
| `/BEHAVIOR.md` | `BEHAVIOR.md` | State machines (`SM-*`), sagas (`SAGA-*`), idempotency keys, concurrency rules |
| `/QUALITY.md` | `QUALITY.md` | Observability + performance — metrics, SLOs, alerts, budgets, capacity envelope |
| `/SECURITY.md` | `SECURITY.md` | Threat model + controls — STRIDE threats, trust boundaries, auth, abuse prevention |
| `/OPERATIONS.md` | `OPERATIONS.md` | Deployment topology, CI/CD, config catalogue, secrets, runbooks, DR |

### Phase F — Design Gate

| Skill | Produces | What it does |
|-------|----------|--------------|
| `/DESIGN_REVIEW.md` | `DESIGN_REVIEW.md` | Gate review — cross-artifact consistency, completeness, internal quality. Emits `verdict: pass / fix-required` |

### Phase G — Decomposition

| Skill | Produces | What it does |
|-------|----------|--------------|
| `/WORK_UNITS.md` | `WORK_UNITS.md` | Splits the design into the smallest independently-implementable units (≤ 400 LOC, ≤ 10 tests, ≤ 6 files, 1 concept) organised into a tiered dependency DAG |

### Phase H — Per-unit pipeline

| Skill | Produces | What it does |
|-------|----------|--------------|
| `/SPEC.md` | `sdd/U<NN>/SPEC.md` | Implementation-complete specification of one work unit, citing domain, interfaces, behavior, errors, data |
| `/PLAN.md` | `sdd/U<NN>/PLAN.md` | Step-by-step build guide with codebase context |
| `/IMPLEMENTATION.md` | `sdd/U<NN>/IMPLEMENTATION.md` | Execute the plan via TDD, commit, document deviations |
| `/CODE_REVIEW.md` | `sdd/U<NN>/CODE_REVIEW.md` | Multi-subagent review — security, bugs, quality, contract conformance, test coverage, historical context |
| `/VERIFICATION.md` | `sdd/U<NN>/VERIFICATION.md` | QA-test the running feature; capture evidence; verify mock fidelity against INTERFACES |

### Phase I — System verification & stabilization

| Skill | Produces | What it does |
|-------|----------|--------------|
| `/SYSTEM_VERIFICATION.md` | `SYSTEM_VERIFICATION.md` | Bootstrap the full stack; run cross-cutting end-to-end scenarios |
| `/TRIAGE.md` | `TRIAGE.md` | Trace system failures to originating artifacts; emit ordered fix batches with pipeline re-entry plan |

### Meta-skills

| Skill | Produces |
|-------|----------|
| `/SKILL_CREATOR.md` | `skills/<name>/SKILL.md` — designs a new skill following project conventions |
| `/ORCHESTRATOR_CREATOR.md` | `agents/<name>.md` — designs a new orchestration agent for a multi-skill workflow |
| `/WORKFLOW.md` | `WORKFLOW.md` — captures a conversational workflow into a generalizable document |

## Workflow

```
Input: user intent (one paragraph) — or resume from any existing artifact state
                                   │
                                   ▼
 A — Discovery                     /PROPOSAL.md  →  /USE_CASES.md
                                   │
                                   ▼
 B — Foundation                    /DOMAIN.md    →  /ARCHITECTURE.md
                                   │
                                   ▼
 C — Surfaces (parallel per surface, only produced when the project has them)
       /WEB_IA.md   /CLI_IA.md   /MOBILE_IA.md   /TUI_IA.md   /VOICE_IA.md
                                   │
                                   ▼
 D — Contracts & Data              /INTERFACES.md   ‖   /DATA.md
   (D1 ‖ D2, then D3)              (only if multi-component)
                                   ▼
                                   /ERRORS.md
                                   │
                                   ▼
 E — Behavior & NFR                /BEHAVIOR.md
   (E1 first, then E2–E4           ▼
    in parallel)                   /QUALITY.md   ‖   /SECURITY.md   ‖   /OPERATIONS.md
                                   │
                                   ▼
 F — Design Review Gate            /DESIGN_REVIEW.md
                                       verdict == pass        → proceed
                                       verdict == fix-required →
                                         apply required changes,
                                         regenerate affected skills,
                                         re-run F.  (Max 3 gate cycles.)
                                   │
                                   ▼
 G — Decomposition                 /WORK_UNITS.md
                                   │
                                   ▼
 H — Per-unit pipeline (sequential per unit; SPEC parallel within a tier)
     For each unit, in dependency order:

       ┌─ /SPEC.md ──┐ (parallel per tier)
       ▼             
       /PLAN.md      ◄──────────────────┐
       ▼                                │
       /IMPLEMENTATION.md               │  plan-level issue
       ▼                                │
       /CODE_REVIEW.md ─── issues ──────┤  bug   → /IMPLEMENTATION.md
       ▼                                │  design → /PLAN.md
       /VERIFICATION.md ── failures ────┘
       ▼
       ✓ unit done — next unit
                                   │ (all units complete)
                                   ▼
 I — System                        /SYSTEM_VERIFICATION.md
                                       verdict == pass → done
                                       else → /TRIAGE.md
                                                │
                                                ▼
                                       apply artifact updates,
                                       re-enter from the
                                       appropriate earlier phase.
                                       (Max 3 stabilization cycles.)
```

### Feedback loops

| Step fails | Where to go back | Why |
|---|---|---|
| `/IMPLEMENTATION.md` | `/PLAN.md` | Plan had gaps or wrong assumptions |
| `/CODE_REVIEW.md` — implementation bug | `/IMPLEMENTATION.md` | Code has bugs or quality issues |
| `/CODE_REVIEW.md` — contract mismatch | `/IMPLEMENTATION.md` | Align field names, casing, mocks with `INTERFACES.md` and `ERRORS.md` |
| `/CODE_REVIEW.md` — design problem | `/PLAN.md` | Approach is structurally wrong |
| `/VERIFICATION.md` — bug | `/IMPLEMENTATION.md` | Fix the code |
| `/VERIFICATION.md` — approach wrong | `/PLAN.md` | Revise the plan |
| `/DESIGN_REVIEW.md: fix-required` | Affected design skill (B–E) | Cross-artifact contradiction or gap |
| `/SYSTEM_VERIFICATION.md: fail` | `/TRIAGE.md`, then affected phase | Integration failure traced to a design artifact |

## File Organization

All artifacts live under `sdd/` at the project root:

```
sdd/
  PROPOSAL.md             USE_CASES.md
  DOMAIN.md               ARCHITECTURE.md
  WEB_IA.md               CLI_IA.md            (surfaces that apply)
  INTERFACES.md           DATA.md              ERRORS.md
  BEHAVIOR.md             QUALITY.md           SECURITY.md         OPERATIONS.md
  DESIGN_REVIEW.md
  WORK_UNITS.md
  U01/   SPEC.md  PLAN.md  IMPLEMENTATION.md  CODE_REVIEW.md  VERIFICATION.md
  U02/   …
  SYSTEM_VERIFICATION.md  TRIAGE.md
```

## Philosophy

**Documents, not conversations.** Every skill produces a file. You start a skill, walk away, and read the result. No monitoring, no back-and-forth.

**No interaction required.** Skills never pause to ask questions. Ambiguities go into an Open Questions section in the output — with options, tradeoffs, and a recommendation. You resolve them by editing the document, then re-run.

**Autonomous-agent compatible.** Skills need no human input during execution. They work headless, CI-triggered, or manual. Start the agent, get the file.

**Produce → read → adjust → feed forward.** You never read agent logs or chat history. Each output file is self-contained and becomes input to the next skill. The trail of documents is the project history.

**Co-locate by read pattern, not conceptual purity.** Artifacts are sized and partitioned so each downstream step reads the minimum set that keeps it under the 10-artifact / 2000-line-per-artifact budget. This is why DOMAIN merges glossary with domain model, INTERFACES merges wire formats with style and evolution policy, QUALITY merges observability with performance, SECURITY merges threats with controls, OPERATIONS merges deployment with runbooks and config.

## Agents

| Agent | Description |
|-------|-------------|
| `sdd-orchestrator` | Executes the full 9-phase ADD pipeline autonomously — discovery → foundation → surfaces → contracts → behavior & NFR → design review gate → decomposition → per-unit pipeline → system verification. Supports resuming from any phase by inspecting which artifacts already exist. Manages three feedback loops: per-unit (H), design review (F, max 3 cycles), and stabilization (I, max 3 cycles). |

Agents live in `agents/{agent-name}.md`. They spawn Claude Code instances via CLI to invoke skills, read output files, resolve open questions, and manage retries. Create new orchestration agents with `/ORCHESTRATOR_CREATOR.md`.

## Meta-Skills

`/SKILL_CREATOR.md` and `/ORCHESTRATOR_CREATOR.md` are meta-skills — they create new skills and orchestration agents that follow project conventions. Use them to extend the workflow with new capabilities.
