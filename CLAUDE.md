# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A Claude Code plugin (`SKILL.md`) providing skills for spec-driven development (SDD). Each skill produces a structured markdown document that guides an AI agent through a specific phase of the development workflow.

## Repository Structure

```
skills/
  {skill-name}/
    SKILL.md          # The skill definition (required)
    references/       # Supporting reference files (optional)
```

The plugin manifest lives in `.claude-plugin/plugin.json`. Skills are auto-discovered from `skills/*/SKILL.md`.

## Skill File Format

Every `SKILL.md` uses this structure:

```markdown
---
name: CAPITALIZED_NAME.md
description: One-line description. Use when asked to...
---

# Task: {What the skill does}

## Objective
## Inputs
## Rules / Workflow
## Output Format
## Quality Checklist
```

The `name` field is the output filename the skill produces (e.g., `SPEC.md`, `PLAN.md`, `SPLIT_WORK.md`). Output filenames are always `CAPITALIZED_WITH_UNDERSCORES.md`.

## Common Rules Across All Skills

When creating or modifying any skill in this repo, apply these shared principles:

**No interactive questions.** Skills never pause to ask the user questions during generation. If a decision has one clearly correct answer given the inputs, resolve it silently. If a decision is genuinely ambiguous — multiple valid approaches exist and the inputs are silent or contradictory — place it in an **Open Questions** section at the end of the output document. Open questions must include options with tradeoffs and a suggestion. They should be rare; most decisions should be resolvable from the inputs.

**Precision over vagueness.** Use exact names — function names, type names, file paths, field names. Never use placeholders like "appropriate," "relevant," "as needed," or "etc." If a value is needed, state it.

**Self-contained output.** Each skill's output document must be readable and actionable without opening any other file. If the output references concepts from input documents, inline the relevant details rather than pointing back to them.

**Quality checklist.** Every skill includes a checklist at the end that the agent verifies before considering the output complete. The checklist enforces the skill's own rules — completeness, precision, no placeholders, no unresolved questions.

**Scope discipline.** Each skill must state what is in scope and out of scope. Skills should not bleed into adjacent workflow phases — e.g., a spec skill does not produce implementation plans, a plan skill does not write code.
