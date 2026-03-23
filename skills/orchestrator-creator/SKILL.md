---
name: AGENT.md
description: Design and create an orchestration agent that autonomously executes a multi-skill workflow. Use when asked to create an orchestrator, build a pipeline agent, design a workflow agent, automate a skill sequence, or produce an AGENT.md.
---

# Task: Design and Create an Orchestration Agent

## Objective

Given a set of skills and a workflow describing how they connect, produce a complete orchestration agent — an `AGENT.md` file and supporting reference files — that can autonomously execute the workflow end-to-end by spawning Claude Code instances, reading their output files, making decisions, and managing feedback loops. The agent never does the skill's job. It invokes, inspects, decides, and proceeds.

---

## Inputs

1. **Skills** (required) — the set of skills the agent will invoke. Each skill must produce a named output file and be non-interactive (no pausing for input, ambiguities go into Open Questions). If skills require human interaction during execution, they are not compatible with orchestration — fix them first.
2. **Workflow** (required) — a defined sequence or graph showing which skills run in what order, what feeds into what, and where feedback loops exist. Can be a diagram, a list, or prose.
3. **Decision criteria** (optional) — for each skill output, what constitutes success, failure, and what triggers a feedback loop. If not provided, design these based on the skill outputs (look for verdicts, issue counts, Open Questions patterns).
4. **Project context** (optional) — architecture docs, conventions, or constraints that inform how the agent should operate.

---

## Workflow

Agent creation follows four phases: map, design, write, and validate.

### Phase 1: Map the Workflow

Analyze the skills and workflow to build a complete picture. For each skill, identify:

| Property | Question to answer |
|---|---|
| **Inputs** | What files does this skill read? User-provided or from earlier skills? |
| **Output** | What file does it produce? Exact filename? |
| **Dependencies** | Which skills must complete first? |
| **Parallelism** | Can multiple instances run concurrently? Under what conditions? |
| **Success criteria** | How do you know the output is good? Verdicts, section checks, test results? |
| **Failure modes** | What goes wrong? What does bad output look like? |
| **Feedback target** | If problems surface, which earlier skill gets re-run? |

Draw the workflow as a directed graph (ASCII art). Mark feedback loops explicitly. This becomes the skeleton of the agent's decision logic.

**Example:**
```
ANALYZE.md → DESIGN.md → BUILD.md → TEST.md → DEPLOY.md
                 ^            ^
                 |            |
              TEST.md      TEST.md
             (design       (build
              flaws)        bugs)
```

Divide the workflow into phases:
- **Parallel phases** — multiple independent invocations that can run concurrently. Use Python with `ThreadPoolExecutor`. Common pattern: generating the same document type for multiple independent items.
- **Sequential phases** — invocations that depend on the previous step's output. Use Bash, one at a time.

Rules of thumb:
- Two invocations reading/writing different files with no overlap → parallel is safe.
- A skill reads another skill's output → sequential is required.
- Multiple invocations modify the same codebase → sequential to avoid conflicts.

### Phase 2: Design Decisions

Before writing, make four key design decisions:

**1. File organization.** Design a directory structure for all output:
- Isolate each work item in its own subdirectory
- Use predictable paths: `{output_dir}/{item_id}/{SKILL_OUTPUT.md}`
- Separate orchestration artifacts from application code

```
{output_dir}/
  {phase_1_output}.md                    # Top-level outputs
  {item_id}/
    {skill_1_output}.md                  # Per-item outputs
    {skill_2_output}.md
    BLOCKED.md                           # Created if item hits retry limit
  {item_id}/
    ...
```

**2. Decision tree.** For every skill output, write a decision table:

```
After SKILL_X produces OUTPUT_X.md:
  1. Read OUTPUT_X.md
  2. Check: file exists and is non-empty?
     - No → retry once, then mark failed
  3. Check: Open Questions with unresolved items?
     - Yes → resolve by editing, then proceed or re-run
  4. Check: skill-specific success criteria?
     - Success → proceed to SKILL_Y
     - Failure type A → go back to SKILL_W with context
     - Failure type B → go back to SKILL_V with context
```

Be explicit about what success looks like (a verdict, an empty issues section, all tests passing), what failure looks like, and where to route each failure type.

**3. Feedback loop bounds.** Every feedback loop needs a limit:
- Maximum attempts per skill per item — recommended: 3
- What context to carry backward (failure information from the later skill)
- Escalation path when blocked (skip and continue, stop workflow, or aggregate for human review)

**4. Prompt patterns.** Design the prompt template for each skill invocation. Every prompt needs three things:

```
/{SKILL_NAME}.md

{Input references — file paths to read}

{Additional context — feedback from previous phases, constraints}

Write the output to {output_path}.
```

Keep prompts focused. Reference file paths rather than embedding content — the skill agent has filesystem access. Embed inline only when content is short (<50 lines), is feedback that doesn't exist as a file, or is a specific excerpt from a larger file.

For feedback loops, include failure context inline:
```
The previous implementation was reviewed and these issues were found:

<issues>
{paste from the review output}
</issues>
```

### Phase 3: Write the Agent

Create the `AGENT.md` file with YAML frontmatter and system prompt. Read `references/claude-code-cli.md` and `references/claude-code-python.md` before writing — they contain the exact invocation patterns to include.

**Frontmatter:**

```yaml
---
name: {agent-name}
description: {Action phrase}. Use when asked to "{trigger 1}", "{trigger 2}", "{trigger 3}".
model: opus
tools: Bash, Read, Edit, Write, Glob, Grep, KillShell, WebFetch, WebSearch
---
```

- `name` — lowercase, hyphenated identifier
- `description` — action verb + 3-5 trigger phrases. This is how the plugin matches user intent.
- `model` — `opus` for orchestration agents (strong reasoning for decisions)
- `tools` — at minimum: `Bash`, `Read`, `Edit`, `Write`, `Glob`, `Grep`

**System prompt structure:**

The body follows this structure. Every section is important — omitting one produces an incomplete agent.

```markdown
# {Agent Name}

{One paragraph: who this agent is, what it does, what it never does.}

---

## Workflow Overview

{ASCII diagram of the full workflow with feedback loops.}

---

## File Organization

{Directory structure. Exact paths.}

---

## How to Invoke Skills

{CLI patterns for sequential. Python patterns for parallel.
 Reference the companion docs: claude-code-cli.md and claude-code-python.md
 in the agent's own references/ directory.}

---

## Phase N: {Phase Name}

{For each phase:
 - Goal (one line)
 - Exact invocation command or Python script template
 - Output inspection (what to check, what success/failure looks like)
 - Decision logic (proceed, retry, go back)
 - Open question resolution}

---

## Feedback Loop Rules

{When to go back to which phase. How to include context.
 Retry limits. What happens when blocked.}

---

## Open Question Resolution

{Universal process for resolving open questions.}

---

## Output Inspection Checklist

{Universal checks plus per-skill checks.}

---

## Error Handling

{Process failures, missing output, infinite loop detection.}

---

## Complete Execution Flow

{Full algorithm as pseudocode — the definitive reference.}

---

## What You Never Do

{Hard constraints the orchestrator must never violate.}
```

**Include these universal patterns in every agent:**

CLI invocation (sequential):
```bash
unset CLAUDECODE && claude -p "<prompt>" --permission-mode bypassPermissions
```
- Always `unset CLAUDECODE` — prevents session conflicts
- Always `--permission-mode bypassPermissions` — prevents interactive hangs
- Always specify input and output file paths in the prompt
- Never parse stdout for data

Python invocation (parallel): write a script using `ThreadPoolExecutor`. Each worker receives a prompt and output path, spawns one `claude` subprocess with `CLAUDECODE` stripped from env, returns a status dict (never raises), validates the output file exists.

Output inspection (after every invocation):
1. File exists? No → retry once
2. File non-empty? No → retry once
3. Open Questions present? Yes → resolve by editing
4. Placeholder language? Grep for "appropriate", "relevant", "as needed", "TBD", "TODO", "etc." Found → re-run
5. Skill-specific checks (verdict, sections, test results)

Open question resolution:
1. Read each question's options and recommendation
2. Decide using project context and the recommendation
3. Edit the file: write decision into relevant section, mark resolved
4. All resolved → "All questions resolved."
5. Genuinely unresolvable → leave, note in final report, use recommendation provisionally

Retry limits:
- Maximum 3 attempts per skill per item
- After 3 → create `BLOCKED.md`, skip to next item
- Track attempts per phase, not globally

Error handling:

| Error | Action |
|---|---|
| Non-zero exit code | Read stderr. Rate limit → wait 60s, retry. Context too long → summarize inputs, retry. Otherwise → report and skip. |
| Output file missing | Glob for `*.md` nearby. Found elsewhere → move. Not found → retry with explicit path. |
| Infinite loop (3 cycles same two phases) | Create `BLOCKED.md`, move on. |

### Phase 4: Validate

Place the companion reference docs in the agent's directory:

```
agents/
  {agent-name}/
    AGENT.md
    references/
      claude-code-cli.md
      claude-code-python.md
```

Copy `claude-code-cli.md` and `claude-code-python.md` from this skill's `references/` directory into the new agent's `references/` directory. These are always-present companion documents that every orchestration agent needs.

Then validate the agent against the quality checklist.

**Testing recommendations** (not mandatory, but strongly recommended):
1. **Dry run with one item.** Full workflow, single simple item. Verify files get created at expected paths.
2. **Test the feedback loop.** Bad input → verify correct routing to earlier phase with context.
3. **Test the retry limit.** Persistent failure → verify `BLOCKED.md` after 3 attempts.
4. **Test parallel execution.** At least 3 concurrent items. Verify no file collisions.
5. **Test open question resolution.** Input that generates questions → verify resolution editing.

---

## Output Format

The skill produces an `AGENT.md` file in `agents/{agent-name}/AGENT.md`, plus the two companion reference docs copied to `agents/{agent-name}/references/`.

```markdown
---
name: {agent-name}
description: {Action phrase}. Use when asked to "{trigger 1}", "{trigger 2}", "{trigger 3}".
model: opus
tools: Bash, Read, Edit, Write, Glob, Grep, KillShell, WebFetch, WebSearch
---

# {Agent Name} — Autonomous Skill Execution Agent

{Identity paragraph: who, what it does, what it never does.}

---

## Workflow Overview

{ASCII workflow diagram with feedback loops}

---

## File Organization

{Directory structure with exact paths}

---

## How to Invoke Skills

{CLI and Python patterns, referencing companion docs}

---

## Phase 1: {Name}

{Goal, invocation, inspection, decision logic}

(Repeat for each phase.)

---

## Feedback Loop Rules

{Triggers, targets, context to carry, retry limits, blocked behavior}

---

## Open Question Resolution

{Universal resolution process}

---

## Output Inspection Checklist

{Universal checks + per-skill checks}

---

## Error Handling

{Process failures, missing output, infinite loops}

---

## Complete Execution Flow

{Full pseudocode algorithm}

---

## What You Never Do

{Hard constraints}
```

---

## Scope

### In scope

- Designing and writing a complete orchestration agent definition (`AGENT.md`)
- Mapping workflows into phases with parallel/sequential classification
- Designing decision trees, feedback loops, and retry bounds
- Defining file organization for orchestration output
- Copying companion reference docs (`claude-code-cli.md`, `claude-code-python.md`) into the agent's directory
- Including all universal patterns (CLI invocation, output inspection, error handling)

### Out of scope

- Creating the skills the agent will invoke — use the skill-creator skill (`/SKILL_CREATOR.md`) for that
- Running or testing the created agent — that is a separate step after creation
- Writing application code or business logic
- Modifying existing agents — edit them directly

---

## Quality Checklist

Before the agent is ready, verify:

- [ ] `AGENT.md` has valid YAML frontmatter with `name`, `description` (action verb + 3-5 triggers), `model`, and `tools`
- [ ] Identity paragraph states what the agent does and what it never does
- [ ] Workflow overview has an ASCII diagram showing all phases and feedback loops
- [ ] File organization specifies exact paths for every output file
- [ ] Every skill in the workflow has a corresponding phase section
- [ ] Every phase section has the exact invocation command (bash or Python)
- [ ] Every phase section has output inspection criteria specific to that skill
- [ ] Every feedback loop has a defined trigger, target phase, and context to carry
- [ ] Retry limits are defined (recommended: 3) and blocked-item behavior is specified
- [ ] Complete execution flow covers every branch including retries and blocked items
- [ ] Companion docs (`claude-code-cli.md`, `claude-code-python.md`) are copied to the agent's `references/` directory
- [ ] "How to Invoke Skills" references the companion docs
- [ ] Open question resolution process is defined
- [ ] Error handling covers: process failure, missing output, infinite loops
- [ ] "What You Never Do" prevents the orchestrator from doing skill work itself
- [ ] No placeholders, TODOs, or vague language
