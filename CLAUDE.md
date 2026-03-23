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
description: {Action verb phrase}. Use when asked to {trigger scenario 1}, {trigger scenario 2}, or {produce OUTPUT_NAME.md}.
---

# Task: {What the skill does}

## Objective
## Inputs
## Workflow / Rules
## Output Format
## Scope
## Quality Checklist
```

The `name` field is the output filename the skill produces (e.g., `SPEC.md`, `PLAN.md`, `CODE_REVIEW.md`). Output filenames are always `CAPITALIZED_WITH_UNDERSCORES.md`.

### Description line

The description serves as the trigger for the Claude Code plugin system. Write it for intent-matching, not marketing:
- Start with an action verb phrase describing what the skill does
- Follow with `Use when asked to...` listing 3-5 realistic trigger phrases the user might say
- Always include the output filename as a trigger (`produce a CODE_REVIEW.md`)

### Sections explained

**Objective** — One paragraph. What the skill produces, what "done" looks like, and the gap it fills. Should make it clear what the output document enables the reader to do.

**Inputs** — Numbered list of what the agent receives or discovers. State explicitly whether inputs are required, optional, or auto-discovered. If the skill discovers its own inputs (e.g., from git state), describe the discovery mechanism. If the user can override the default scope, say so here.

**Workflow or Rules** — The core of the skill. Use **Workflow** (phased, imperative) when the skill follows a sequential process with distinct stages. Use **Rules** (declarative, constraint-based) when the skill enforces properties the output must satisfy regardless of how it's produced. Some skills use both (workflow for process, rules for constraints on the output). See "Workflow vs Rules" below for guidance.

**Output Format** — A markdown template showing the exact structure of the output document. Wrap the template in a fenced code block. Every section heading in the template must appear in the output. Use `{placeholders}` for variable content. Include inline comments like `(Repeat for each. If none: "No X found.")` to guide the agent on iteration and empty-state handling.

**Scope** — In scope (what this skill delivers) and Out of scope (what it explicitly does not do). Name adjacent skills or phases that own the excluded items. This prevents the agent from helpfully adding out-of-scope content.

**Quality Checklist** — Checkbox list (`- [ ]`) the agent verifies before considering the output complete. Each item must be mechanically verifiable (not subjective). The checklist enforces the skill's own rules — it's the contract between the skill and the agent.

## Common Rules Across All Skills

When creating or modifying any skill in this repo, apply these shared principles:

**No interactive questions.** Skills never pause to ask the user questions during generation. If a decision has one clearly correct answer given the inputs, resolve it silently. If a decision is genuinely ambiguous — multiple valid approaches exist and the inputs are silent or contradictory — place it in an **Open Questions** section at the end of the output document. Open questions must include options with tradeoffs and a suggestion. They should be rare; most decisions should be resolvable from the inputs.

**Precision over vagueness.** Use exact names — function names, type names, file paths, field names. Never use placeholders like "appropriate," "relevant," "as needed," or "etc." If a value is needed, state it.

**Self-contained output.** Each skill's output document must be readable and actionable without opening any other file. If the output references concepts from input documents, inline the relevant details rather than pointing back to them.

**Quality checklist.** Every skill includes a checklist at the end that the agent verifies before considering the output complete. The checklist enforces the skill's own rules — completeness, precision, no placeholders, no unresolved questions.

**Scope discipline.** Each skill must state what is in scope and out of scope. Skills should not bleed into adjacent workflow phases — e.g., a spec skill does not produce implementation plans, a plan skill does not write code.

**Positive observations.** When a skill evaluates existing work (reviews, analysis), the output must acknowledge what was done well — not just flag problems. Balanced feedback is a requirement, not a nicety.

## Workflow vs Rules

Choose the structure based on what the skill does:

- **Rules** work for skills that define properties the output must satisfy (SPEC, SPLIT_WORK). The agent has freedom in how it produces the output, as long as the constraints hold. Rules are declarative: "every function must have a full signature", "no file shared by two units."
- **Workflows** work for skills that follow a sequential process with distinct stages (PLAN, IMPLEMENT, CODE_REVIEW). Each phase depends on the previous one. Workflows are imperative: "Phase 1: discover scope. Phase 2: launch subagents. Phase 3: aggregate."
- **Both** can coexist. A skill can have a workflow for the process and rules for properties the output must satisfy regardless of process.

## Subagent Orchestration

Skills that involve multiple distinct analytical dimensions or parallel workstreams should use subagents. The pattern:

### When to use subagents

- The skill spans multiple independent areas of analysis (security, bugs, quality — each is a separate lens)
- The skill involves exploration of multiple unrelated parts of a codebase
- The skill's work can be parallelized without sequential dependencies between the parallel branches
- The main agent's context would be overwhelmed by doing everything inline

### How to structure subagent skills

1. **Reference files go in `references/`.** Each subagent gets its own markdown file in `skills/{skill-name}/references/`. The file is a complete prompt — the main agent reads it and passes it to the subagent.

2. **Every reference file follows this structure:**
   - **Title and role** — one paragraph establishing what the subagent is and its mission
   - **"What You Receive"** — identical inputs section across all sibling subagents
   - **"Analysis Process"** — multi-step systematic process (4-5 steps). Give each subagent multiple analytical lenses, not just one checklist. A security auditor doesn't just check for SQL injection — it maps attack surfaces, traces data flows, and assesses exploitability.
   - **"Output Format"** — structured markdown with a severity/priority hierarchy. The format must be consistent enough for the main agent to aggregate findings across subagents. Use the pattern: detailed format for high-severity findings (full blocks with file, category, problem, impact, fix), condensed table format for low-severity findings.
   - **"Guiding Principles"** — 3-5 domain-specific principles that inject judgment and prevent mechanical checklist behavior. These are the subagent's personality. They should emphasize proportionality, pragmatism, and evidence over absolutism.

3. **The main SKILL.md orchestrates:**
   - Describes each subagent in a table (name, reference file, what it reviews)
   - Specifies what context each subagent receives
   - States that subagents are launched in parallel
   - Defines the aggregation rules: deduplication, severity classification, noise filtering
   - Owns the final output format — subagents produce intermediate findings, the main agent produces the document

### Subagent design principles

- **Opinionated and thorough.** Subagent prompts should be extensive — long, detailed, covering every angle. A subagent is a specialist; it should know its domain deeply. Err on the side of too much guidance rather than too little.
- **Each subagent gets multiple analytical lenses.** Don't give a subagent a single checklist. Give it 4-6 distinct categories to examine, each with specific patterns to look for. This prevents tunnel vision.
- **Severity hierarchy in output.** Every subagent should return findings at multiple severity levels. High-severity findings get full detailed blocks; low-severity findings get table rows. This gives the aggregator material to work with.
- **Positive observations required.** Every subagent must include a section acknowledging good practices found. This ensures the aggregated output is balanced.
- **"Read beyond the diff."** Subagents that analyze code should be explicitly told to follow data flows, call chains, and dependencies beyond the immediate changes. The most important findings are often invisible in the diff.

## Output Format Design

Output templates range from strict to flexible depending on the skill's purpose:

- **Strict templates** (every section mandatory, exact heading structure) — use for outputs that will be consumed by other skills or need consistent structure for aggregation. Examples: SPEC.md (consumed by PLAN), CODE_REVIEW.md (aggregated from subagents).
- **Flexible guidelines** ("should communicate", "at a minimum") — use for outputs where the structure should adapt to the problem. Example: PLAN.md adapts its structure to the complexity of the feature.

Within the template, use these conventions:
- `{placeholder}` for variable content
- `(Repeat for each X. If none: "No X found.")` for iteration guidance
- `(Omit this section if Y.)` for conditional sections
- Include the empty-state text inline so the agent knows what to write when a section has no content

## Writing Style

Skills in this repo use a direct, imperative voice. Specific conventions:

- **Address the agent as "you" implicitly** through instructions, not "the agent should." Write "Launch six subagents in parallel" not "The agent should launch six subagents."
- **State constraints as requirements**, not suggestions. "Every finding must have a file path" not "Findings should ideally include a file path."
- **Allow judgment where appropriate.** Not everything needs to be a hard rule. Use "Use judgment" or "This is a guideline, not a hard rule" when the agent should adapt to context. The skill should be prescriptive about structure and flexible about content decisions.
- **Be concrete in examples.** When illustrating a pattern, use realistic examples with actual-looking code, file paths, and names — not abstract placeholders.
