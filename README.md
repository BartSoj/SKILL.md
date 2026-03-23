# SKILL.md

A Claude Code plugin for **spec-driven development**. Each skill produces a structured markdown document — you operate on the document level, not the message level.

## Installation

```bash
claude plugin marketplace add bartsoj/SKILL.md
claude plugin install SKILL.md@SKILL.md
```

`/VERIFICATION.md` requires [agent-browser](https://github.com/vercel-labs/agent-browser) for web application testing.

## Skills

| Skill | Description |
|-------|-------------|
| `/SPLIT_WORK.md` | Splits project scope into independent work units with dependency ordering |
| `/SPEC.md` | Writes an implementation-complete specification for one work unit |
| `/PLAN.md` | Turns a spec into a step-by-step build guide with codebase context |
| `/IMPLEMENTATION.md` | Executes a plan using TDD, documents what was built |
| `/CODE_REVIEW.md` | Reviews code changes via 6 parallel specialist subagents |
| `/VERIFICATION.md` | QA-tests the running application, captures evidence |
| `/SKILL_CREATOR.md` | Designs and creates a new skill following project conventions |
| `/AGENT.md` | Designs and creates an orchestration agent for a multi-skill workflow |

## Workflow

```
                    ┌─────────────────────────────────────────────────────┐
                    │              SPLIT_WORK.md                          │
                    │  Project requirements / architecture → Work units   │
                    └──────────────────────┬──────────────────────────────┘
                                           │
                            ┌──────────────┼─────────────┐
                            ▼              ▼             ▼
                      ┌──────────┐  ┌──────────┐  ┌──────────┐
                      │ SPEC.md  │  │ SPEC.md  │  │ SPEC.md  │   one per
                      │ (unit 1) │  │ (unit 2) │  │ (unit N) │   work unit
                      └────┬─────┘  └─────┬────┘  └─────┬────┘
                           └──────────────┼─────────────┘
                                          │
                    ┌─────────────────────────────────────────────┐
                    │        For each spec, sequentially:         │
                    │                                             │
                    │  ┌───────────┐                              │
                    │  │  PLAN.md  │◄──────────────────────┐      │
                    │  └─────┬─────┘                       │      │
                    │        │                             │      │
                    │        ▼                             │      │
                    │  ┌─────────────────┐         problems│      │
                    │  │IMPLEMENTATION.md│─────────────────┘      │
                    │  └─────┬───────────┘                        │
                    │        │                                    │
                    │        ▼                                    │
                    │  ┌────────────────┐                         │
                    │  │ CODE_REVIEW.md │                         │
                    │  └─────┬──────────┘                         │
                    │        │                                    │
                    │     issues? ──yes──► back to                │
                    │        │             IMPLEMENTATION.md      │
                    │        no                                   │
                    │        ▼                                    │
                    │  ┌──────────────────┐                       │
                    │  │ VERIFICATION.md  │                       │
                    │  └─────┬────────────┘                       │
                    │        │                                    │
                    │     issues? ──yes──► back to PLAN.md        │
                    │        │             or IMPLEMENTATION.md   │
                    │        no                                   │
                    │        ▼                                    │
                    │     ✓ Done — next spec                      │
                    └─────────────────────────────────────────────┘
```

1. **`SPLIT_WORK.md`** — Given project requirements or architecture docs, breaks the scope into small independent work units (≤400 LOC each) with explicit dependencies and a build order.

2. **`SPEC.md`** — For each work unit, produces an implementation-complete specification: every function signature, error path, constant, and behavioral rule spelled out. Specs for independent units can be written in parallel.

3. **`PLAN.md`** — Explores the existing codebase, then produces a step-by-step build guide with all context inlined. The implementing agent reads only the plan.

4. **`IMPLEMENTATION.md`** — Executes the plan using TDD. Documents what was built, deviations from plan, and test results. If implementation hits problems → back to `PLAN.md`.

5. **`CODE_REVIEW.md`** — Six parallel subagents (security, bugs, quality, contracts, test coverage, historical context) review the changes. If issues found → back to `IMPLEMENTATION.md`.

6. **`VERIFICATION.md`** — Launches the application and interacts with it as an end user. Records pass/fail with evidence. If failures found → back to `PLAN.md` or `IMPLEMENTATION.md`.

### Feedback loops

| Step fails | Go back to | Why |
|---|---|---|
| `IMPLEMENTATION.md` | `PLAN.md` | Plan had gaps or wrong assumptions |
| `CODE_REVIEW.md` | `IMPLEMENTATION.md` | Code has bugs or quality issues |
| `VERIFICATION.md` | `PLAN.md` or `IMPLEMENTATION.md` | Feature doesn't work as intended |

## Philosophy

**Documents, not conversations.** Every skill produces a file. You start a skill, walk away, and read the result. No monitoring, no back-and-forth. The document *is* the deliverable.

**No interaction required.** Skills never pause to ask questions. Ambiguities go into an Open Questions section in the output — with options, tradeoffs, and a recommendation. You resolve them by editing the document, then re-run.

**Autonomous-agent compatible.** Skills need no human input during execution. They work headless, CI-triggered, or manual. Start the agent, get the file.

**Produce → read → adjust → feed forward.** You never read agent logs or chat history. Each output file is self-contained and becomes input to the next skill. The trail of documents is the project history.

## Agents

| Agent | Description |
|-------|-------------|
| `sdd-orchestrator` | Executes the full SDD workflow autonomously — splits work, generates specs in parallel, then runs plan → implement → review → verify for each unit with feedback loops |

Agents live in `agents/{agent-name}/AGENT.md`. They spawn Claude Code instances via CLI to invoke skills, read output files, resolve open questions, and manage retries. Create new orchestration agents with `/AGENT.md`.

## Meta-Skills

`/SKILL_CREATOR.md` and `/AGENT.md` are meta-skills — they create new skills and orchestration agents that follow project conventions. Use them to extend the workflow with new capabilities.
