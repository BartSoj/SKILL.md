---
name: WORKFLOW.md
description: Capture a conversational workflow into a structured, generalizable workflow document. Use when asked to capture the workflow, document what we just did, create a workflow from this conversation, summarize the workflow steps, or produce a WORKFLOW.md.
---

# Task: Capture Conversation Workflow into Structured Document

## Objective

Given a conversation where the user guided an agent step-by-step through a workflow — invoking skills, running ad-hoc commands, making decisions, discovering edge cases — produce a `WORKFLOW.md` that captures the complete workflow in a structured, generalizable format. The result is a document that anyone (human or orchestrator-creator skill) can read to understand every step, every decision point, every feedback loop, and every input/output relationship in the workflow, without needing to replay the conversation. When an existing `WORKFLOW.md` is present, merge new findings with the existing document rather than replacing it.

---

## Inputs

1. **Conversation history** (auto-discovered) — the full conversation in the current context window. This is the primary input. The agent reviews everything: user messages, agent actions, tool calls, skill invocations, outputs, failures, corrections, and explanations.
2. **Existing WORKFLOW.md** (auto-discovered) — if a `WORKFLOW.md` already exists in the working directory, read it before starting. This triggers refinement mode: merge new conversation findings with the existing document instead of writing from scratch.
3. **User's final guidance** (optional) — any instructions the user provides alongside the skill invocation, such as "focus on the deployment steps" or "this was a retry of the analysis phase."

---

## Workflow

Workflow extraction follows four phases: inventory, extract, generalize, and write.

### Phase 1: Inventory

Scan the full conversation from beginning to end. Build a chronological list of every meaningful action. An action is meaningful if it changed state, produced output, consumed input, or represented a decision.

For each action, note:
- **What happened** — the command, skill invocation, file edit, or explanation
- **Who initiated it** — user instruction or agent initiative
- **What it consumed** — files read, data referenced, prior outputs used
- **What it produced** — files written, state changes, information discovered
- **Whether it succeeded or failed** — and what happened next

Also inventory:
- **Decision points** — moments where the user chose between alternatives. Record the options considered, the choice made, and the reasoning given.
- **Corrections** — moments where the user said "no, not that" or "go back and do X instead." These reveal important constraints.
- **Edge cases** — unexpected situations discovered during execution. How were they handled? Were they resolved or left open?
- **Project context** — any information the user shared about the tech stack, architecture, conventions, team practices, or constraints.

If an existing `WORKFLOW.md` is present, read it now. Note which steps it already covers and which are new, changed, or contradicted by the current conversation.

### Phase 2: Extract

Transform the chronological action list into workflow steps. Consecutive actions that serve the same purpose become a single step. Independent actions that always happen together become a single step.

For each step, extract:

| Property | What to capture |
|---|---|
| **Name** | Short, descriptive label (e.g., "Analyze codebase for relevant files") |
| **Type** | `skill` (with exact skill name and plugin if applicable) or `ad-hoc` (with exact commands/actions) |
| **Invocation** | For skills: the exact invocation pattern used (e.g., `/spec Read the architecture from ...`). For ad-hoc: the commands or actions performed. |
| **Inputs** | What the step reads or receives — file paths, prior step outputs, user-provided data |
| **Outputs** | What the step produces — files written, state changes, data extracted |
| **Purpose** | Why this step exists — what it accomplishes in the workflow |
| **Success criteria** | How to know the step succeeded — specific checks on the output |
| **Failure modes** | What can go wrong — observed failures from the conversation plus reasonably anticipated ones |
| **Recovery** | What to do on failure — retry, go back to an earlier step, skip, or escalate |
| **Dependencies** | Which steps must complete before this one can start |

Pay attention to the user's explanations. When the user says "we do this because..." or "the reason for this step is...", that reasoning is critical context — capture it in the Purpose field.

### Phase 3: Generalize

The conversation walked through the workflow with specific inputs (a particular file, a specific bug, a concrete feature). The WORKFLOW.md must describe the general pattern that works for any valid input.

**Generalization rules:**

1. **Replace specifics with parameters.** If the conversation analyzed `UserController.php`, the workflow step should say "the target file" or `{target_file}`, not `UserController.php`. Use `{parameter_name}` syntax for variable inputs.

2. **Keep specifics as examples.** After the generalized description, include the concrete example from the conversation: "For example, in the conversation this was `UserController.php`." Examples anchor the abstract description in reality.

3. **Identify what varies vs what's constant.** Some parts of the workflow are always the same regardless of input (e.g., "always run tests after implementation"). Other parts depend on the input (e.g., "if the file is a controller, also check route bindings"). Distinguish these clearly.

4. **Preserve conditional branches.** If the user said "if X happens, do Y; otherwise do Z", that conditional logic is part of the workflow. Do not flatten it into a single path.

5. **Generalize decision criteria, not just steps.** If the user said "this output was good because it had no critical issues", generalize to "success when no critical issues are found in the output" — not just "check the output."

### Phase 4: Write (or Merge)

**If no existing WORKFLOW.md:** Write the document from scratch following the output format below. Structure the steps into logical phases. Draw the workflow graph. Document all decision criteria, feedback loops, and project context.

**If existing WORKFLOW.md found (refinement mode):**

Read the existing document and compare it with the extracted workflow. Apply these merge rules:

- **New steps** not in the existing document → add them in the correct position
- **Existing steps confirmed** by the current conversation → preserve and enrich with any new detail
- **Existing steps contradicted** by the current conversation → update to reflect the new understanding, noting what changed and why
- **Existing steps not encountered** in the current conversation → preserve as-is (absence from one conversation does not mean the step is invalid)
- **New edge cases or failure modes** → add to the relevant step's failure modes / recovery section
- **New decision criteria** → merge with existing criteria
- **Contradictions that cannot be resolved** → place in Open Questions with both versions and a recommendation

Increment the `revision` field in frontmatter. Update the `date` field.

---

## Rules

### As-Is Capture

Every step must be documented in the form it was actually performed:
- If a skill was invoked, record it as a skill invocation with the exact skill name
- If ad-hoc commands were run, record them as ad-hoc commands with the exact commands
- If the user manually edited a file, record it as a manual edit
- Do not rewrite ad-hoc steps as skill invocations or vice versa
- Do not suggest that ad-hoc steps should become skills — that decision belongs to a different phase

**Violation:** A step says "use the code-review skill" when the conversation actually used `grep` and manual file reading to review code.

### Generalization Without Loss

The workflow must be generic enough to apply to different inputs, but specific enough that an agent can execute it without guessing.

- Every parameter must have a name, a description, and an example value from the conversation
- Conditional branches must have concrete conditions, not vague "if needed" or "as appropriate"
- Success criteria must be mechanically checkable, not subjective

**Violation:** A step says "check if the output looks good" without defining what "good" means. A step says "analyze the relevant files" without specifying how to identify which files are relevant.

### Completeness for Orchestration

The document must contain enough information for the orchestrator-creator skill to build an autonomous agent without asking questions. At minimum:

- Every step has defined inputs and outputs
- Every step has success criteria and at least one failure mode
- Dependencies between steps are explicit
- Parallel execution opportunities are identified
- Feedback loops specify the trigger, the target step, and what context to carry back
- Project context covers any domain knowledge the agent needs that isn't derivable from the steps themselves

### Conversation Fidelity

Prefer information directly observed in the conversation over inferred information. When inferring (e.g., "this step could probably run in parallel with that one"), mark the inference explicitly: "Inferred: these steps have no data dependency and could run in parallel."

---

## Output Format

The output uses flexible structure — adapt section organization to the workflow being described. The following sections are required; additional sections may be added when the workflow warrants them.

```yaml
---
skill: WORKFLOW.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions}
revision: {1 for new, increment on refinement}
steps_count: {total number of workflow steps}
feedback_loops: {number of feedback loops identified}
open_questions: {number of unresolved questions}
---
```

```markdown
# Workflow: {Descriptive Name}

{One paragraph: what this workflow accomplishes, what triggers it, and what the end state looks like. Written generically — not referencing the specific conversation.}

---

## Parameters

{Workflow-level inputs that vary between runs. Each parameter has a name, type, description, and the example value from the conversation.}

| Parameter | Type | Description | Example |
|---|---|---|---|
| `{name}` | {type} | {what it is} | {value from conversation} |

(Repeat for each parameter.)

---

## Steps

(For each step, use this structure. Steps are numbered for reference in the workflow graph and decision criteria.)

### Step {N}: {Name}

**Type:** {`skill: /skill-name` | `skill: plugin-name:skill-name` | `ad-hoc`}

**Purpose:** {Why this step exists — what it accomplishes.}

**Invocation:**
{For skills: the exact invocation pattern with `{parameter}` placeholders.}
{For ad-hoc: the commands or actions, generalized with parameters.}

**Inputs:**
- {What this step reads or receives. Use `{parameter}` for workflow parameters, `Step N output` for dependencies.}

**Outputs:**
- {What this step produces. Exact filenames or descriptions of state changes.}

**Success criteria:**
- {Mechanically checkable condition — e.g., "output file exists and contains no critical issues"}

**Failure modes and recovery:**
- {Failure}: {What to do — retry, go back to Step M, skip, escalate}

(Repeat for each step. If a step has substeps, use a nested list within the step.)

---

## Workflow Graph

{ASCII diagram showing the execution order, parallel opportunities, and feedback loops. Use arrows for flow, labels for conditions.}

```
{Step 1} → {Step 2} → {Step 3}
                          ↓
                       {Step 4} ──(failure)──→ {Step 2}
```

{Below the diagram, list:}
- **Sequential dependencies:** {which steps must wait for which}
- **Parallel opportunities:** {which steps can run concurrently — state why (no shared inputs/outputs)}
- **Feedback loops:** {which steps loop back, under what condition, carrying what context}

---

## Decision Criteria

(For each step that has non-trivial success/failure evaluation:)

### After Step {N}: {Name}

1. {Check}: {what to look for}
   - {Outcome A} → {action: proceed to Step M}
   - {Outcome B} → {action: retry / go back to Step K with context}
   - {Outcome C} → {action: escalate / skip}

(Repeat for each step with decision logic. Omit steps with trivial pass/fail.)

---

## Feedback Loops

(For each feedback loop identified:)

### {Loop Name}

- **Trigger:** {what condition activates this loop — e.g., "code review finds critical issues"}
- **Source step:** Step {N} ({name})
- **Target step:** Step {M} ({name})
- **Context to carry back:** {what information from the source step the target step needs — e.g., "the list of issues found"}
- **Maximum iterations:** {recommended limit — e.g., 3}
- **Blocked behavior:** {what happens after max iterations — e.g., "mark as blocked, continue to next item"}

(Repeat for each loop. If none: "No feedback loops identified.")

---

## Project Context

{Information about the project that an agent needs to execute this workflow correctly but cannot derive from the steps themselves. Include only context that was mentioned or demonstrated in the conversation.}

- **Tech stack:** {languages, frameworks, tools observed}
- **Conventions:** {naming patterns, directory structure, coding standards mentioned}
- **Constraints:** {limitations, requirements, team practices that affect the workflow}
- **Domain knowledge:** {any domain-specific information the user explained}

(Omit subsections that have no content. If no project context was shared: "No project-specific context was provided during the conversation.")

---

## Edge Cases and Observations

{Situations discovered during the conversation that affect how the workflow should handle unusual inputs or unexpected states.}

- **{Edge case}:** {what happened, how it was handled, what the general rule should be}

(Repeat for each. If none: "No edge cases were encountered.")

---

## Open Questions

- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(Repeat for each. If none: "All questions resolved.")
```

---

## Scope

### In scope

- Extracting workflow steps from the full conversation history
- Documenting skill invocations and ad-hoc steps exactly as performed
- Generalizing specific conversation actions into a reusable workflow pattern
- Identifying decision points, failure modes, feedback loops, and parallelism
- Capturing project context, edge cases, and unresolved questions
- Merging new findings with an existing WORKFLOW.md when one is present
- Producing a document complete enough for the orchestrator-creator skill to consume

### Out of scope

- Creating new skills for ad-hoc steps — owned by the skill-creator skill
- Building orchestration agents from the workflow — owned by the orchestrator-creator skill
- Executing the workflow — WORKFLOW.md is a document, not an executable
- Modifying the conversation or re-running steps — the skill only observes and documents
- Evaluating whether the workflow is optimal — document what was done, not what should have been done

---

## Quality Checklist

Before the output is complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `revision`, `steps_count`, `feedback_loops`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without replaying the conversation
- [ ] Every step has: name, type, purpose, invocation, inputs, outputs, success criteria, and at least one failure mode
- [ ] Steps use exact skill names for skill invocations and exact commands for ad-hoc steps
- [ ] Workflow is generalized — uses `{parameter}` placeholders instead of conversation-specific values
- [ ] Every parameter has a name, description, and example value from the conversation
- [ ] Workflow graph is present and shows all steps, dependencies, parallel opportunities, and feedback loops
- [ ] Decision criteria cover every step with non-trivial success/failure evaluation
- [ ] Every feedback loop has: trigger, source, target, context to carry, max iterations, and blocked behavior
- [ ] Sequential dependencies and parallel opportunities are explicitly identified
- [ ] If refinement: existing valid content is preserved, new content is merged, contradictions are in Open Questions
- [ ] No step rewrites ad-hoc actions as skill invocations or vice versa (as-is capture rule)
- [ ] Project context section captures tech stack, conventions, and constraints mentioned in the conversation
