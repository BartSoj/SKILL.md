# SKILL.md

A Claude Code plugin for **Agent-Driven Development (ADD)** — a novel software development process where every step is an AI-agent invocation that produces one structured markdown artifact. You operate on the document level, not the message level. The trail of documents *is* the project history.

## Installation

```bash
claude plugin marketplace add bartsoj/SKILL.md
claude plugin install SKILL.md@SKILL.md
```

`/VERIFICATION.md` and `/SYSTEM_VERIFICATION.md` require [agent-browser](https://github.com/vercel-labs/agent-browser) for web application testing.

## What is Agent-Driven Development?

ADD treats every step of the software development process as **a stateless AI-agent invocation** whose only memory is the files it reads. Consequences:

- **Files are the only communication channel.** If it isn't in an artifact, it doesn't exist for the next step.
- **Every skill produces exactly one artifact.** Scope is bounded by what one agent can consume and produce.
- **Context budget is real.** Each step reads ≤ 10 artifacts; each artifact is ≤ 2000 lines (typical 500–1500).
- **Stable IDs cross-link artifacts.** `UC-NN`, `INV-NN`, `EP-name`, `ERR_CODE`, `SM-*`, `SAGA-*`, `THREAT-NN`, `D-NNN`, `u<NN>` — assigned once, never renumbered.
- **Two operating modes.** *Bootstrap* (phases A–F) produces the design suite from initial intent, run once. *Continuous* (phase G, on demand) drives ongoing implementation per trigger; *system verification* (phase H) runs on demand.
- **Open questions are surfaced, not hidden.** Every artifact has an Open Questions section that the orchestrator resolves before handoff via `/DECISION.md`.
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
| `/ARCHITECTURE.md` | `ARCHITECTURE.md` | Structural blueprint — components, cross-component flows, technology choices, and foundational `D-NNN` decisions |

### Phase C — Surfaces (one per human-facing channel, if any)

| Skill | Produces |
|-------|----------|
| `/WEB_IA.md` | `WEB_IA.md` — web information architecture |
| `/CLI_IA.md` | `CLI_IA.md` — CLI command tree, grammar, exit codes |
| `/MOBILE_IA.md` | `MOBILE_IA.md` — mobile screen inventory, navigation, deep-link strategy |
| `/TUI_IA.md` | `TUI_IA.md` — terminal UI layout, mode, keybinding matrix |
| `/VOICE_IA.md` | `VOICE_IA.md` — voice intents, slots, dialog model |

**Optional Tier 2 — per-item deep specs.** The IAs above carry a lightweight blueprint for each item. For pages/screens/views/intents/commands whose composition is complex, novel, or high-stakes, opt into a per-item deep spec. Skip entirely otherwise — most items don't need this.

| Skill | Produces (per item, when warranted) |
|-------|--------------------------------------|
| `/PAGE_SPEC.md` | `web-pages/<page-id>/PAGE_SPEC.md` — deep web-page composition: layout, action placement, state-specific layouts, motion |
| `/SCREEN_SPEC.md` | `mobile-screens/<screen-id>/SCREEN_SPEC.md` — deep mobile-screen composition: gestures, orientation, platform variations |
| `/VIEW_SPEC.md` | `tui-views/<view-id>/VIEW_SPEC.md` — deep TUI-view composition: panels, modes, keybindings, capability degradation |
| `/INTENT_SPEC.md` | `voice-intents/<intent-id>/INTENT_SPEC.md` — deep voice-intent composition: utterance library, dialog flow, multimodal expansion |
| `/COMMAND_SPEC.md` | `cli-commands/<cmd-id>/COMMAND_SPEC.md` — deep CLI-command composition: output modes, signal handling, capability degradation (rarest of the family) |

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
| `/DESIGN_REVIEW.md` | `DESIGN_REVIEW.md` | Gate review — cross-artifact consistency, completeness, internal quality. Emits `verdict: pass / fix-required`. Re-runs whenever `/RECONCILIATION.md` applies major top-level edits during continuous work |

### Phase G — Per-trigger implementation

Phase G runs continuously, once per trigger. The orchestrator picks up a roadmap item or issue, allocates a unit in `units/<area>/u<NN>/`, and runs G1–G8.

| Skill | Produces | What it does |
|-------|----------|--------------|
| `/SPEC.md` | `units/<area>/u<NN>/SPEC.md` | Implementation-complete specification of one unit, citing domain, interfaces, behavior, errors, data; frontmatter declares `area`, `files`, `concepts`, `trigger`, `supersedes`, `depends_on`, `related` |
| `/SPEC_REVIEW.md` | `units/<area>/u<NN>/SPEC_REVIEW.md` | Validates the SPEC against trigger requirement and frontmatter scope; emits `verdict: pass / fix-required / prototype-needed` |
| `/PROTOTYPE.md` | `units/<area>/u<NN>/PROTOTYPE.md` *(conditional)* | Empirically resolves SPEC_REVIEW risks via scratch code in `units/<area>/u<NN>/prototype/` |
| `/PLAN.md` | `units/<area>/u<NN>/PLAN.md` | Step-by-step build guide with codebase context |
| `/IMPLEMENTATION.md` | `units/<area>/u<NN>/IMPLEMENTATION.md` | Execute the plan via TDD, commit, document deviations |
| `/CODE_REVIEW.md` | `units/<area>/u<NN>/CODE_REVIEW.md` | Multi-subagent review — security, bugs, quality, contract conformance, test coverage, historical context |
| `/VERIFICATION.md` | `units/<area>/u<NN>/VERIFICATION.md` | QA-test the running feature; capture evidence; verify mock fidelity against INTERFACES and trigger acceptance criteria |
| `/RECONCILIATION.md` | `units/<area>/u<NN>/RECONCILIATION.md` | After VERIFICATION passes, applies edits to the unit's SPEC, selected top-level design artifacts, the trigger artifact's frontmatter (status, `promoted_to_units`), and any peer units' `superseded_by` reciprocity |

### Phase H — System verification (on demand)

| Skill | Produces | What it does |
|-------|----------|--------------|
| `/SYSTEM_VERIFICATION.md` | `SYSTEM_VERIFICATION.md` | Bootstrap the full stack; run cross-cutting end-to-end scenarios |
| `/TRIAGE.md` | `TRIAGE.md` | Trace system failures to originating artifacts; emit ordered fix batches with pipeline re-entry plan; file new `issues/` for newly discovered defects |

### Cross-cutting (any phase, any time)

| Skill | Produces | What it does |
|-------|----------|--------------|
| `/DECISION.md` | `decisions/D-NNN-<slug>/DECISION.md` | Resolves an Open Question from any artifact; reads the source artifact + 1–4 cited artifacts; one decision per directory |
| `/ROADMAP.md` | `roadmap/<NNN>-<slug>/ROADMAP.md` | Files a planned item (feature / refinement / refactor / chore) — becomes a trigger for phase G |
| `/ISSUE.md` | `issues/<NNN>-<slug>/ISSUE.md` | Files a defect (bug / regression) — becomes a trigger for phase G |

### Meta-skills

| Skill | Produces |
|-------|----------|
| `/SKILL_CREATOR.md` | `skills/<name>/SKILL.md` — designs a new skill following project conventions |
| `/ORCHESTRATOR_CREATOR.md` | `agents/<name>.md` — designs a new orchestration agent for a multi-skill workflow |
| `/WORKFLOW.md` | `WORKFLOW.md` — captures a conversational workflow into a generalizable document |

## Workflow

### Bootstrap mode (one-time)

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
                                       verdict == pass        → bootstrap complete
                                       verdict == fix-required →
                                         apply required changes,
                                         regenerate affected skills,
                                         re-run F.  Stop if not converging.
                                   │
                                   ▼
                            (bootstrap complete — continuous mode begins)
```

### Continuous mode (per trigger)

```
Trigger: /ROADMAP.md  →  roadmap/<NNN>-<slug>/ROADMAP.md  (planned: feature/refinement/refactor/chore)
         /ISSUE.md    →  issues/<NNN>-<slug>/ISSUE.md      (defect: bug/regression)
                                   │
                                   ▼
                       Orchestrator picks up trigger:
                       - read trigger; resolve open questions via /DECISION.md
                       - determine area (DOMAIN bounded context)
                       - allocate u<NN> from units/_NEXT_UNIT_ID
                       - create units/<area>/u<NN>/
                                   │
                                   ▼
 G — Per-unit pipeline (sequential per unit; multiple triggers / units may parallelize when independent)

       ┌─ /SPEC.md ◄────────────────────────────┐
       ▼                                        │ spec-level issue
       /SPEC_REVIEW.md ── prototype-needed ─────┘
       ▼                                          ┌─ /PROTOTYPE.md (regenerate SPEC)
       (verdict == pass)                          │
       ▼                                          │
       /PLAN.md      ◄──────────────────┐         │
       ▼                                │ plan    │
       /IMPLEMENTATION.md               │ issue   │
       ▼                                │         │
       /CODE_REVIEW.md ─── issues ──────┤  bug   → /IMPLEMENTATION.md
       ▼                                │  design → /PLAN.md
       /VERIFICATION.md ── failures ────┘
       ▼
       /RECONCILIATION.md
       (applies edits: own SPEC + selected top-level docs +
        trigger frontmatter + superseded peer reciprocity)
       ▼
       ✓ unit done — orchestrator updates trigger status
```

### System verification (on demand)

```
On demand: after a milestone trigger closes; before a release; on user request
                                   │
                                   ▼
 H — System                        /SYSTEM_VERIFICATION.md
                                       verdict == pass → verified
                                       else → /TRIAGE.md
                                                │
                                                ▼
                                       apply artifact updates,
                                       re-enter from the
                                       appropriate earlier phase;
                                       file new issues/ for newly discovered defects.
                                       Stop if not converging.
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

All artifacts live at the project root:

```
<project-root>/
  PROPOSAL.md             USE_CASES.md
  DOMAIN.md               ARCHITECTURE.md
  WEB_IA.md               CLI_IA.md            (surfaces that apply)
  INTERFACES.md           DATA.md              ERRORS.md
  BEHAVIOR.md             QUALITY.md           SECURITY.md         OPERATIONS.md
  DESIGN_REVIEW.md
  SYSTEM_VERIFICATION.md  TRIAGE.md            (when produced)

  units/                                       (per-trigger implementation)
    _NEXT_UNIT_ID                              counter file
    <area>/                                    one folder per bounded context
      u<NN>/
        SPEC.md  SPEC_REVIEW.md  (PROTOTYPE.md)
        PLAN.md  IMPLEMENTATION.md
        CODE_REVIEW.md  VERIFICATION.md  RECONCILIATION.md

  decisions/<D-NNN>-<slug>/DECISION.md         (cross-cutting)
  roadmap/<NNN>-<slug>/ROADMAP.md              (triggers — planned)
  issues/<NNN>-<slug>/ISSUE.md                 (triggers — defects)
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
| `add-orchestrator` | Executes the ADD pipeline autonomously across both modes. **Bootstrap** (A–F): discovery → foundation → surfaces → contracts → behavior & NFR → design review gate. **Continuous** (G): picks up triggers from `roadmap/` and `issues/`, allocates units in `units/<area>/u<NN>/`, runs G1–G8 per unit, applies frontmatter back-links on triggers and superseded peers. **System verification** (H, on demand): runs cross-cutting scenarios; triages failures back to the originating phase. Supports resuming from any phase by inspecting which artifacts already exist. Manages feedback loops within G (SPEC ↔ PLAN ↔ IMPL ↔ REVIEW ↔ VERIFY), at the F gate, and across H stabilization. |

Agents live in `agents/{agent-name}.md`. They author dynamic workflows that spawn a subagent per skill, read output files, resolve open questions, and manage feedback loops. Create new orchestration agents with `/ORCHESTRATOR_CREATOR.md`.

## Meta-Skills

`/SKILL_CREATOR.md` and `/ORCHESTRATOR_CREATOR.md` are meta-skills — they create new skills and orchestration agents that follow project conventions. Use them to extend the workflow with new capabilities.
