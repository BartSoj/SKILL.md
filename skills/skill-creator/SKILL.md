---
name: SKILL_CREATOR.md
description: Design and create a new skill following SDD conventions. Use when asked to create a skill, design a new skill, write a skill definition, build a SKILL.md, or scaffold a skill from requirements.
---

# Task: Design and Create a New Skill

## Objective

Given a description of what a skill should do, produce a complete skill directory — a `SKILL.md` file and any supporting reference files — that follows project conventions, works with autonomous agents, produces self-contained document output with trackable frontmatter, and never requires human interaction during execution. The result is a skill that an agent can invoke to produce a structured output file from start to finish.

---

## Inputs

1. **Skill requirements** (required) — what the skill should do, what it produces, what triggers it. Can be a description, user stories, or a reference to an existing workflow phase.
2. **Workflow context** (optional) — where this skill fits in a larger pipeline. What skills run before it (providing input), what skills run after it (consuming output). This informs the input/output contracts.
3. **Existing skills** (auto-discovered) — read existing skills in `skills/*/SKILL.md` to understand conventions, voice, and patterns in use. Match their style.

---

## Workflow

Skill creation follows three phases: design, write, and validate.

### Phase 1: Design

Before writing anything, make these design decisions:

**1. Name and output file.** Choose the output filename — always `CAPITALIZED_WITH_UNDERSCORES.md`. The name should clearly identify what the document is (e.g., `SPEC.md`, `CODE_REVIEW.md`, `DEPLOYMENT_PLAN.md`). This becomes the `name` field in frontmatter.

**2. Folder name.** Choose the skill's directory name under `skills/` — lowercase, hyphenated (e.g., `code-review`, `deployment-plan`). This is how the skill is referenced in the plugin system.

**3. Workflow or Rules (or both).** Decide the core structure:
- **Workflow** — if the skill follows a sequential process with distinct phases (explore → analyze → write). Use when the order of operations matters.
- **Rules** — if the skill enforces properties the output must satisfy, regardless of how it gets there. Use when the constraints matter more than the process.
- **Both** — a workflow for the process and rules for output constraints. Common for skills that have a defined process but also need to guarantee output properties.

This is a judgment call. Look at the skill's nature: does it feel more like "follow these steps" or "satisfy these properties"? When in doubt, lean toward guidelines with clear principles rather than rigid step-by-step instructions.

**4. Subagent decomposition.** Decide if the skill needs subagents:
- Does it span multiple independent analysis dimensions? → subagents
- Can work be parallelized across independent items? → subagents
- Would doing everything inline overwhelm the agent's context? → subagents
- Is it a focused, single-concern skill? → no subagents, keep it simple

If using subagents, design each one as a specialist with its own reference file in `references/`.

**5. Output strictness.** Decide how strict the output template should be:
- **Strict** — when the output is consumed by another skill or aggregated from subagents. Every section mandatory, exact heading structure.
- **Flexible** — when the output should adapt to the problem. Define what must be communicated, not the exact structure.

**6. Frontmatter fields.** Design the YAML frontmatter for the output file. Include common tracking fields plus skill-specific fields (see Output Frontmatter section below).

### Phase 2: Write

Create the skill files following the conventions below. Read the official skill writing guide at `references/official-skill-guide.md` for structural patterns and progressive disclosure rules.

Write the `SKILL.md` following this structure:

```markdown
---
name: OUTPUT_NAME.md
description: {Action verb phrase}. Use when asked to {trigger 1}, {trigger 2}, or {produce OUTPUT_NAME.md}.
---

# Task: {What the skill does}

## Objective
## Inputs
## Workflow / Rules
## Output Format
## Scope
## Quality Checklist
```

If the skill uses subagents, also create reference files in `references/` — one per subagent.

### Phase 3: Validate

After writing, verify the skill against the quality checklist at the end of this document. Also:
- Read the skill aloud (mentally) — does it flow as clear instructions?
- Check that an agent reading only this file could produce the output without asking questions
- Verify the output template is complete — try mentally generating output for a realistic scenario

---

## Rules

These rules govern how to write each section of a skill. They are the core of this skill — internalize them.

### Skill Frontmatter

The YAML frontmatter has two required fields:

```yaml
---
name: OUTPUT_NAME.md
description: {Action verb phrase}. Use when asked to {trigger}, {trigger}, or {produce OUTPUT_NAME.md}.
---
```

**`name`** — The output filename this skill produces. Always `CAPITALIZED_WITH_UNDERSCORES.md`. This is what the agent writes to disk.

**`description`** — The trigger line. This is how the plugin system matches user intent to the skill. Write it for intent-matching:
- Start with an action verb phrase: "Review code changes", "Create an implementation plan", "Split architecture into work units"
- Follow with `Use when asked to...` and list 3-5 realistic phrases a user might say
- Always include the output filename as one trigger: "produce a CODE_REVIEW.md"
- Keep it under 100 words — this is always loaded in context

**Example:**
```yaml
description: Review code changes using parallel subagents for security, bugs, quality, contracts, test coverage, and historical context. Use when asked to review code, review changes, do a code review, check code before merge, or produce a CODE_REVIEW.md.
```

### Objective Section

One paragraph. Three things:
1. What the skill produces (the output document)
2. What "done" looks like (the state after successful execution)
3. What gap it fills (what the reader of the output can do that they couldn't before)

Keep it focused. Do not describe the process — that belongs in Workflow.

### Inputs Section

Numbered list. For each input:
- What it is (name and description)
- Whether it is **required**, **optional**, or **auto-discovered**
- If auto-discovered: describe the discovery mechanism (e.g., "from git state", "from package.json")
- If the user can override the default scope, say so

The first input is typically the primary input (what the skill operates on). Subsequent inputs provide context.

### Workflow and Rules Sections

This is the core of the skill — where you teach the agent how to do the work. The approach depends on the skill's nature (decided in Phase 1).

**Guidelines for writing workflows:**
- Break into named phases (Phase 1: Discovery, Phase 2: Analysis, Phase 3: Writing)
- Each phase should have a clear goal stated at the top
- Be specific about what to do, but allow judgment in how to do it
- If a phase involves subagents, specify what each subagent receives and what it returns
- Include decision points: "If X, then Y. If Z, then W."

**Guidelines for writing rules:**
- State each rule as a requirement, not a suggestion
- Group related rules under descriptive subheadings
- For each rule, make it clear what a violation looks like — this helps the agent self-check
- Distinguish hard constraints ("must", "never") from guidelines ("prefer", "use judgment")

**The flexible-over-rigid principle:** Prefer teaching principles and patterns over prescribing exact steps. A skill that says "analyze the code for these 5 categories of issues" produces better results than one that says "run grep for X, then check file Y, then count Z." Give the agent the domain knowledge and let it apply judgment. Use strict rules only for properties that must hold in every case (naming conventions, required sections, output contracts).

### Output Format Section

Define the structure of the output document. This is the contract between the skill and whoever reads the output.

**Output frontmatter (required in all skill outputs):**

Every output file must start with YAML frontmatter containing tracking fields. Define two categories:

1. **Common fields** (present in every skill's output):

```yaml
---
skill: OUTPUT_NAME.md          # Which skill produced this
date: YYYY-MM-DD               # When it was produced
status: complete               # complete | has_open_questions | blocked
---
```

2. **Skill-specific fields** (vary by skill, designed for the domain):

Choose fields that enable monitoring, benchmarking, or pipeline decisions. Good skill-specific fields answer: "What would an orchestrator or dashboard want to know about this output at a glance?"

**Examples of skill-specific frontmatter fields:**

For a code review skill:
```yaml
verdict: pass                  # pass | concerns | fail
critical_issues: 0
high_issues: 2
files_reviewed: 12
```

For a specification skill:
```yaml
unit: U01
functions_specified: 8
open_questions: 0
estimated_loc: 350
```

For an implementation skill:
```yaml
unit: U01
tests_total: 14
tests_passing: 14
deviations_from_plan: 1
```

For a verification skill:
```yaml
verdict: pass
scenarios_pass: 8
scenarios_fail: 0
scenarios_blocked: 1
```

Design frontmatter fields that are:
- **Machine-readable** — numbers, enums, dates. Not prose.
- **Decision-useful** — an orchestrator could read these to decide the next step without parsing the full document.
- **Comparable** — the same field across multiple runs enables benchmarking.

**Output template:**

After frontmatter, define the document structure. Wrap the template in a fenced code block. Use these conventions:
- `{placeholder}` for variable content
- `(Repeat for each X. If none: "No X found.")` for iteration
- `(Omit this section if Y.)` for conditional sections
- Include empty-state text inline

**Strict vs flexible templates:**
- Use **strict** (every section mandatory) when the output is consumed by another skill or needs consistent structure
- Use **flexible** ("should communicate", "at a minimum") when structure should adapt to the problem
- This is a spectrum, not a binary — most skills land somewhere in between

### Open Questions Section (in output)

Every skill output must include an Open Questions section, typically at the end. This is the escape valve for genuine ambiguity.

Rules for the skill definition:
- Instruct the agent to resolve questions silently when one answer is clearly correct
- Instruct the agent to surface questions only when multiple valid approaches exist and inputs are ambiguous
- Define the question format: checkbox, options with tradeoffs, recommendation
- State that Open Questions should be rare — most decisions are resolvable from the inputs

Template for questions in the output:
```markdown
## Open Questions

- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")
```

### Scope Section

Two subsections: **In scope** and **Out of scope**.

In scope: bulleted list of what this skill delivers. Be specific.

Out of scope: bulleted list of what this skill does NOT do. Name the adjacent skill or phase that owns each excluded item. This is critical — without it, eager agents will helpfully add out-of-scope content.

Good out-of-scope entries follow the pattern: "{thing} — owned by {adjacent skill/phase}."

### Quality Checklist

Checkbox list (`- [ ]`) that the agent verifies before the output is complete. Each item must be:
- **Mechanically verifiable** — not subjective. "Every function has a return type" is verifiable. "The code is clean" is not.
- **Enforceable from the skill's own rules** — each checkbox should trace back to a rule or convention defined in the skill.
- **Concrete** — "No placeholders, TODOs, or vague language" is concrete. "Output is high quality" is not.

Include these universal items in every skill's checklist:
- `[ ] Output file has valid YAML frontmatter with all required fields`
- `[ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")`
- `[ ] Open Questions section is present (empty or with genuine ambiguities only)`
- `[ ] Output is self-contained — readable and actionable without opening other files`

Add skill-specific items after the universal ones.

### Writing Style

Match the voice used across all skills in this project:

- **Imperative, direct.** "Launch six subagents in parallel" not "The agent should launch six subagents."
- **"You" implicitly.** Address the agent through instructions, not third person.
- **Constraints as requirements.** "Every finding must have a file path" not "Findings should ideally include a file path."
- **Judgment where appropriate.** Not everything needs a hard rule. "Use judgment" and "This is a guideline, not a hard rule" are valid when the agent should adapt to context.
- **Concrete examples.** When illustrating a pattern, use realistic names, paths, and code — not abstract placeholders.
- **No marketing language.** Skills are technical instructions, not product descriptions.

### No-Interaction Rule

Skills never pause to ask the user questions during execution. This is a hard constraint for autonomous agent compatibility.

- If a decision has one clearly correct answer → resolve silently
- If genuinely ambiguous → place in Open Questions section with options, tradeoffs, and a recommendation
- Open questions should be rare — most decisions are resolvable from the inputs

The skill must be executable from a headless CLI invocation (`claude -p "/SKILL_NAME.md ..." --permission-mode bypassPermissions`) without any interactive prompts.

### Progressive Disclosure

Follow the three-level loading system:

1. **Metadata** (frontmatter) — always in context (~100 words). Keep the description concise and trigger-focused.
2. **SKILL.md body** — loaded when the skill triggers. Target under 500 lines. If approaching this limit, move detailed guidance into reference files.
3. **Reference files** — loaded as needed. Use for subagent instructions, domain-specific guides, large examples. Reference them clearly from SKILL.md with guidance on when to read each one.

For large reference files (>300 lines), include a table of contents at the top.

### Subagent Reference Files

When the skill uses subagents, each gets its own markdown file in `references/`. Follow this structure:

```markdown
# {Role Name} — Subagent Instructions

{One paragraph: who this subagent is, its mission, its expertise.}

## What You Receive

{Identical inputs section across all sibling subagents.}

## Analysis Process

{4-5 step systematic process. Give multiple analytical lenses, not a single checklist.}

## Output Format

{Structured markdown with severity/priority hierarchy.
 Detailed format for high-severity findings.
 Condensed table format for low-severity findings.}

## Guiding Principles

{3-5 domain-specific principles that inject judgment.
 Emphasize proportionality, pragmatism, evidence over absolutism.}
```

Design principles for subagent prompts:
- **Opinionated and thorough.** Err on the side of too much guidance.
- **Multiple analytical lenses.** 4-6 distinct categories per subagent, not a single checklist.
- **Severity hierarchy.** High-severity gets full detail blocks; low-severity gets table rows.
- **Positive observations.** Every subagent includes a section for good practices found.

---

## Scope

### In scope

- Designing and writing a complete skill definition (`SKILL.md`)
- Creating subagent reference files if the skill uses subagents
- Defining output frontmatter with tracking fields
- Applying project conventions, writing style, and structural patterns
- Validating the skill against the quality checklist

### Out of scope

- Running or testing the created skill — that is a separate step after creation
- Modifying existing skills — use this skill for new skills only; modify existing skills by reading and editing them directly
- Creating orchestration agents — see `agents/ORCHESTRATION_GUIDELINES.md` for that
- Writing the application code or content that the skill will eventually produce

---

## Quality Checklist

Before the skill is ready, verify:

- [ ] `SKILL.md` has valid YAML frontmatter with `name` (CAPITALIZED_WITH_UNDERSCORES.md) and `description` (action verb + 3-5 trigger phrases)
- [ ] Description starts with an action verb phrase and includes the output filename as a trigger
- [ ] Objective is one paragraph covering: what it produces, what "done" looks like, what gap it fills
- [ ] Inputs are numbered with required/optional/auto-discovered status for each
- [ ] Core section is Workflow, Rules, or both — chosen appropriately for the skill's nature
- [ ] Output format defines YAML frontmatter with common fields (`skill`, `date`, `status`) and skill-specific trackable fields
- [ ] Output template uses `{placeholders}`, iteration guidance, conditional sections, and empty-state text
- [ ] Open Questions pattern is defined in the output format
- [ ] In-scope and out-of-scope sections are present, with out-of-scope naming adjacent skills/phases
- [ ] Quality checklist includes universal items plus skill-specific verifiable items
- [ ] No interactive questions — the skill runs headless from start to finish
- [ ] Writing style is imperative, direct, with concrete examples
- [ ] SKILL.md is under 500 lines (or uses reference files for overflow)
- [ ] If subagents are used: each has a reference file with role, inputs, process, output format, and guiding principles
- [ ] The skill is self-contained — an agent reading only SKILL.md (and referenced files) can produce the output
