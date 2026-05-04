---
name: DECISION.md
description: Resolve one or more Open Questions from any artifact by gathering evidence (web search, codebase grep, runtime sandbox, prior decisions), reasoning bias-free, and either deciding with an audit trail or handing off to the user via an asynchronous YAML mechanism — writing one `decisions/D-NNN-slug/DECISION.md` file per invocation. Use when asked to decide on an open question, resolve an open question, make a decision, answer a question for the SPEC, produce a DECISION.md, or produce a decisions/D-NNN-slug/DECISION.md.
---

# Task: Generate `decisions/D-NNN-slug/DECISION.md` — Cross-Cutting Open-Question Resolver

## Objective

Produce one `decisions/D-{NNN}-{slug}/DECISION.md` file per invocation that resolves one Open Question (or one batch of dependent questions sharing a single decision) carried by an upstream artifact. The skill is **bias-free**: invoked fresh, with no prior conversation context, it reads the question(s) verbatim, gathers evidence using the broadest tool surface in ADD (web search, codebase grep + targeted read, execution sandbox, prior decision files), reasons from the evidence — not from what an upstream agent might have implied — and either lands on a clearly-evidenced option (`status: accepted`) or hands off to the human via the asynchronous user-question mechanism (`status: awaiting-user`). The output is an audit trail: a future reader, looking only at the file, can reconstruct what was asked, what evidence was gathered, why a particular option won (or why the agent declined to decide), and what artifacts must change as a result.

DECIDE exists because two upstream failure modes proved costly during ADD pipeline testing: silent wrong picks (an upstream skill chose an answer without admitting alternatives existed, blocking only later when the choice failed downstream) and stuck pipelines (the orchestrator could not move forward but had no domain-specific tool to investigate). DECIDE concentrates the investigative tools — web for current docs and standards, codebase grep for existing conventions, runtime sandbox for empirical claims, file-based user-question for genuine product-direction calls — into one purposeful skill. The defining discipline — and the commonest violation — is **no silent picks**: when multiple options are valid and evidence cannot decide, the skill MUST hand off to the user via the YAML mechanism. It MUST NOT pad rationale to disguise a guess as a decision.

---

## Inputs

The orchestrator passes these per invocation. The skill does NOT auto-discover the full design suite; reads are scoped.

1. **Source artifact** (required, primary subject) — the artifact carrying the Open Question(s) being resolved. Typically a path at the project root (e.g., `U03/SPEC.md`, `INTERFACES.md`, `ARCHITECTURE.md`). The skill quotes the question verbatim from this file.
2. **Cited artifacts** (required, 1–4 typically) — artifacts the question text references or that are needed to understand the question. The orchestrator scopes this. Examples:
   - For a wire-format question: `INTERFACES.md`
   - For a domain-invariant question: `DOMAIN.md`
   - For a state-machine question: `BEHAVIOR.md`
   - For a unit-scope question: the unit's `U<NN>/SPEC.md`
3. **Prior dependent decisions** (optional, auto-passed when sequenced) — when this invocation depends on a previously-resolved decision, the prior `decisions/D-NNN-slug/DECISION.md` file is in the input set. The skill cites it under § 4 Prior Decisions.
4. **`decisions/` directory** (auto-discovered) — existing decisions at `decisions/D-*/DECISION.md`. Read for two purposes: (a) idempotency check in Phase 1 (has this question already been resolved?), (b) NNN computation in Phase 5 (next available number).

Read-set size: 1 source + 1–4 cited + 0–2 prior decisions + tool reads (web, codebase, runtime) + glob of `decisions/`. Within ADD's read-set budget (1–5 declared reads + tools).

---

## Tools

DECIDE has the **broadest tool surface in ADD**. Most skills are file-readers; DECIDE is an investigator. Use each tool when the question warrants it, not because it is available (Rule 11).

| Tool | When | How |
|------|------|-----|
| **Web search** | Question involves a library, framework, protocol, format, RFC, standard, registry, or external API. | `WebSearch`, `WebFetch`. Query official docs, GitHub source, RFCs, MDN, registries. Cite URLs in § 4 Web. Multiple queries are normal. |
| **Codebase grep + read** | Question involves an existing convention, naming, pattern, or conflict in this project. | `Grep` (or `rg` / `ast-grep` via `Bash`) for searches; `Read` for targeted file reads. Cite `file:line` in § 4 Codebase with 5–10-line excerpts. |
| **Execution sandbox** | Question is empirical (does X actually do Y?) and other sources are inconclusive. **Use sparingly.** | Create a scratch directory at `decisions/D-{NNN}-{slug}/scratch/`. Initialize an appropriate toolchain (npm/cargo/uv). Run the experiment. Capture stdout/stderr/exit code verbatim. Cite outputs in § 4 Runtime. The scratch dir persists for reproducibility. |
| **User-question via YAML** | Question is user-intent: multiple options are technically valid and the right one depends on product goals or user preferences not derivable from project artifacts. | Write a decision file with `status: awaiting-user`, populate `user_question` (Rule 5 — self-contained brief), set `user_response: null`. Exit. The orchestrator surfaces the file to the human. The human edits it (sets `status: accepted`, adds `user_response`). The orchestrator resumes when status changes. |

**No tool pauses the agent to ask the user during execution.** The skill runs headless from start to finish (ADD P9). The user-question mechanism is asynchronous — the skill writes the question into the file and exits; the user answers later by editing the file.

---

## Workflow

Five phases, single-agent (no subagents — the agent is investigator and decision-maker, both roles in one). Phases are sequential; Phase 4 may loop back to Phase 3 if evidence is insufficient.

### Phase 1: Read the Question(s) and Check for Duplicates

Read the source artifact, especially its Open Questions section. Read the cited artifacts. If sequenced, read the prior decision files passed as input.

**Quote the question(s) verbatim** into an in-agent working note. When multiple questions are batched into this invocation, identify whether they truly share a single decision (one answer resolves them all) or whether they are really separate decisions that the orchestrator batched incorrectly.

- If **one decision resolves all batched questions**: `question_origin` will be a YAML list in frontmatter; § 1 Question quotes every batched question with its source citation.
- If **the batch is incorrect** (the questions are independent and need separate decisions): do NOT silently split them within this file. Surface in § 9 Open Questions: "Orchestrator batched these questions but they are independent decisions. Recommend orchestrator dispatch them as separate `decide` invocations." Decline to decide; emit `status: awaiting-user` with `user_question` describing the batching error.

**Duplicate check (Rule 7).** Glob `decisions/D-*/DECISION.md`. For each existing decision, read the frontmatter. If any existing file has `question_origin` matching the current input artifact§section (or any element of the current list when batched):

| Existing status | Action |
|-----------------|--------|
| `accepted` | **Do not write a new file.** Exit with stdout pointer: `duplicate — see decisions/D-NNN-slug/DECISION.md`. The orchestrator will notice. |
| `awaiting-user` | **Do not write a new file.** Exit with stdout pointer: `still awaiting user — see decisions/D-NNN-slug/DECISION.md`. |
| `proposed` | **Do not write a new file.** Exit with stdout pointer: `decision proposed but not yet applied — see decisions/D-NNN-slug/DECISION.md`. |
| `superseded` | Reason afresh (the supersession indicates a deliberate replacement is in progress). Continue to Phase 2; the new file references the superseded one in § 4 Prior Decisions. |
| `abandoned` | Reason afresh. Continue to Phase 2; the new file may but need not reference the abandoned one. |

This check happens BEFORE evidence gathering. Wasting an investigation on a duplicate is the most costly failure mode for this skill.

If any required input is missing (source artifact does not exist; cited artifacts do not exist), exit with `status: blocked` describing exactly which input is missing. Do not invent the question.

### Phase 2: Determine Sources to Consult

Based on the question, decide which tools to use. Most questions need 1–2 sources; some need all four. Map the question to its evidence axes:

| Question shape | Primary source | Secondary source |
|----------------|----------------|------------------|
| "Which version / API / behavior of library X?" | **Web** (official docs, GitHub, registry) | Codebase (current usage in this project) |
| "What naming / casing / pattern do we use here?" | **Codebase** (existing conventions) | Prior decisions (if a convention was previously chosen) |
| "Does library Y actually do Z when called with W?" | **Runtime** (sandbox the call) | Web (library source / GitHub issues for confirmation) |
| "Should we adopt approach A or B?" (product direction) | **User-question** | Web / codebase as background only |
| "Is this consistent with our prior architectural choice?" | **Prior decisions** (`decisions/D-*/DECISION.md`) | Codebase |

The mapping is a guide, not a rule. Use judgment: a question may need web AND codebase AND a prior decision. A question may need only one. Do not consult a tool because it is available — consult it because the question warrants it (Rule 11).

Record the chosen sources; they populate frontmatter `sources_used`. The enum is `web | codebase | runtime | user | prior-decisions` — the `prior-decisions` value is a deliberate extension covering the case where prior `D-MMM` files contributed material evidence to this decision (treated as a distinct source axis from the four primary tools because prior decisions are binding by default unless this decision proposes superseding them).

### Phase 3: Gather Evidence

Apply the chosen tools. Record what was queried and what was found in working notes that will become § 4 Evidence Gathered.

**Web search discipline.**
- Search authoritative sources first: official docs, GitHub source repos, RFCs, MDN, registries (npm, crates.io, PyPI), library issue trackers.
- Avoid stale blog posts unless they are the only source — and then explicitly note the date and the staleness risk.
- Multiple queries are normal. A single search result is rarely enough evidence to decide. (Rule 2 — aggressive research.)
- Cite URLs verbatim. The audit trail must be reproducible: a future reader must be able to re-fetch the same URL and verify the finding.

**Codebase grep discipline.**
- Use `Grep` for plain text searches; use `ast-grep` (via `Bash`) for structural code searches when regex is not expressive enough.
- Cite `file:line` with a 5–10-line excerpt. The excerpt must contain enough context that the finding stands without re-opening the file.
- Note conflicts when present: if multiple patterns are in use, surface the conflict explicitly in § 4 Codebase. Conflicts are common evidence that the project has not yet resolved this question — this often justifies escalation to user-intent (Rule 3 — codebase grounding).

**Runtime sandbox discipline.**
- Only when the question is empirical and other sources are inconclusive.
- Scratch dir at `decisions/D-{NNN}-{slug}/scratch/` (NNN computed in Phase 5; create the directory at the time of running the experiment with the chosen slug — the slug can be picked early).
- Capture output verbatim into § 4 Runtime: stdout, stderr, exit code, and the exact command run.
- Pin dependency versions explicitly. Divergent versions invalidate the runtime evidence.
- Keep experiments minimal: 30 lines beats 300. The goal is empirical proof of one claim, not a test suite.

**Prior decisions discipline.**
- When a prior decision file is passed as input (sequential dependency), quote the relevant section and cite the `D-NNN`.
- When this skill discovers a prior decision while gathering evidence (Phase 3 reading reveals a related decision), cite it in § 4 Prior Decisions and treat its conclusions as binding unless a stronger signal in current evidence overrides it (in which case the new decision must propose superseding the prior one — § 7 Implications must include "supersedes D-MMM").

**User-intent recognition.**
- A question is user-intent when: (a) multiple options are technically valid, AND (b) the right choice depends on product goals or user preferences not derivable from project artifacts.
- It is NOT user-intent when the artifacts contain an answer the agent missed — that is a research failure. Re-read the cited artifacts before concluding the question is user-intent. (Rule 1 — bias-free reasoning includes resisting the temptation to punt.)
- When in genuine doubt, frame for the user (Rule 5 — user-question framing) rather than picking. No silent picks (Rule 4).

### Phase 4: Decide (or Hand Off)

Synthesize the evidence into a decision. There are exactly four exit paths:

| Evidence pattern | Decision | Status |
|------------------|----------|--------|
| Evidence overwhelmingly favors one option | Choose it. | `accepted` |
| Evidence supports one option as the strongest leaning, but residual uncertainty exists | Choose the leaning option; record residual uncertainty in § 6 Rationale. | `accepted` |
| Multiple options are valid and the choice is product-direction-dependent | Hand off via user-question. | `awaiting-user` |
| Reasoning reveals dependence on an unresolved sub-question (sequential dependency) | Hand off via user-question with sub-question framing. | `awaiting-user` |

**The two `awaiting-user` exit paths are distinct** — encode them differently:

1. **User-intent hand-off.** § 5 Decision says: "Awaiting user input — multiple valid options; choice depends on product direction." § 6 Rationale explains what evidence was gathered, why each option is technically valid, and which factor the user must weigh. `user_question` field follows Rule 5 framing with options + recommendation + how-to-respond.

2. **Sequential-dependency hand-off (Rule 8).** § 5 Decision says: "Awaiting user input — this decision depends on D-{NNN} which has not been written. Recommend orchestrator dispatch decide on the prerequisite first, then re-invoke decide on this question with the prerequisite's resolution as input." § 6 Rationale explains the dependency chain. `user_question` field describes the dependency and asks the user (or orchestrator-via-user) to confirm the dispatch order.

**No subjective override.** A reviewer who believes a question is decidable downgrades it from user-intent to a chosen option only by surfacing concrete evidence that selects between options — not by overriding the verdict with reasoning.

If evidence gathered in Phase 3 is insufficient (the agent reaches Phase 4 without enough material to decide): loop back to Phase 3, gather more. Only after a second pass and still-insufficient evidence does the agent conclude `awaiting-user` for the "evidence cannot resolve" reason — and the framing must explain what evidence was sought, what was found, and why it was inconclusive.

**Status nuance — `proposed` vs `accepted`.** Use `accepted` when the agent has confidently decided and the rationale is strong. Use `proposed` when the agent has decided but the orchestrator should validate by applying the decision and checking that downstream regeneration succeeds. Default to `accepted` for confident decisions; use `proposed` when the implementation impact is high-reversal-cost and an orchestrator-level checkpoint is wise.

### Phase 5: Compute Path, Write the File

**NNN computation.**

1. Glob `decisions/D-*/DECISION.md`. If no matches (the `decisions/` directory may not yet exist), the next NNN is `001`.
2. For each match, parse the leading three-digit number after `D-` in the directory name (e.g., `decisions/D-007-token-format/DECISION.md` → 7).
3. Take the maximum and add 1. Zero-pad to three digits. (`max=12` → `013`; `max=99` → `100`; `max=999` → `1000` — three-digit pad becomes natural width when overflowing.)

**Slug computation (Rule 9).**

- Short (≤ 5 words), kebab-case, derived from the question's noun phrases.
- Examples: "What format should tokens use?" → `token-format`; "Should errors wrap or replace?" → `error-wrap-vs-replace`; "Which idempotency key for the push saga?" → `push-saga-idempotency-key`.
- Uniqueness check: glob `decisions/D-*-{slug}/` (against all NNN). If a collision exists, append a disambiguator drawn from the question (e.g., `token-format` collides → `token-format-jwt-vs-paseto`; never append a numeric counter).
- Only ASCII letters, digits, and hyphens. Lowercase only.

**Write path.** `decisions/D-{NNN}-{slug}/DECISION.md`. Create the `decisions/D-{NNN}-{slug}/` directory (and the parent `decisions/` if needed) before writing.

**Frontmatter completeness.**
- `id` and the filename's NNN agree.
- `kind`: `adr` for architectural decisions (foundational, structural, technology choice) or `resolution` for open-question answers (the more common case for this skill — questions surfaced by upstream skills). Use judgment: a decision that constitutes a foundational architectural commitment is `adr`; a decision that resolves a question raised in a SPEC's Open Questions section is `resolution`.
- `status`: one of the lifecycle states — `proposed | accepted | superseded | awaiting-user | abandoned`. **This is an intentional deviation from the standard `complete | has_open_questions | blocked` enum used in most artifacts** because decisions have a lifecycle (the orchestrator and user mutate this state over time). Encode the lifecycle states; do not "fix" to the standard enum.
- `awaiting_user`: boolean; true iff `status: awaiting-user`. (Redundant with status; the brief specifies both — encode both.)
- `user_question`: the populated brief when `awaiting_user: true`; empty/null otherwise. Multi-line YAML scalar (`|` block).
- `user_response`: `null` until the user fills it.
- `resolved_date`: today's date (`YYYY-MM-DD`) when the skill emits `status: accepted` directly. `null` when emitting `status: proposed | awaiting-user | abandoned`. The orchestrator updates this from `null` to today's date on the `awaiting-user` → `accepted` transition (when the user edits the file).
- `implications`: YAML list of artifact IDs (e.g., `[INTERFACES, DOMAIN, U03/SPEC]`) that must be updated as a result.
- `reversal_cost`: `low | medium | high`.
- `open_questions`: count of unresolved checkbox items in § 9.

**Self-check before exit.**

- Frontmatter complete: every required field populated, types correct.
- § 1 Question is verbatim from source artifact (or all batched questions verbatim).
- § 4 Evidence has at least one entry per source listed in `sources_used`.
- § 5 Decision is explicit ("Decision: Option X" or "Awaiting user input — {reason}").
- § 7 Implications lists every artifact change required (concrete artifact citations).
- If `awaiting_user: true`: `user_question` field is populated per Rule 5; § 5 and § 6 explain the hand-off reason.
- If `status: accepted` AND a prior decision is being superseded: § 7 Implications includes "supersedes D-MMM" and the path of the superseded file.

If runtime sandbox was used, the scratch directory `decisions/D-{NNN}-{slug}/scratch/` persists. Do not delete it.

---

## Rules

These rules govern the decision file. Violations are detected by the Quality Checklist.

### 1. Bias-free reasoning

You are invoked fresh, with no prior conversation context. Do not assume any prior bias toward an answer. Reason from the question and the evidence you gather, not from what an upstream agent might have implied. If you find yourself drawn to an answer because "it seems like what was meant," check whether the artifacts and tools support that answer — if not, surface in § 9 Open Questions or hand off to the user. The audit trail must reflect evidence, not aesthetic preference.

### 2. Aggressive research

Decisions are only as good as the evidence that supports them. When the question relates to external knowledge — libraries, APIs, standards, RFCs, registries — be aggressive about web search. Query official docs, GitHub source, registries, RFCs. Multiple queries are normal. Do not decide based on a single search result when more sources are available. A finding from a stale blog post is weaker than a finding from a current GitHub commit; cite the strongest sources you can find and call out staleness when it is the only signal.

### 3. Codebase grounding

When the question relates to existing project conventions, the codebase is the primary authority. Find the convention; do not invent. If multiple conflicting conventions exist, surface the conflict in § 4 Codebase and treat it as a § 9 Open Question or as a candidate for user hand-off — do not silently pick the convention that "feels more idiomatic." A pattern present at five sites is stronger evidence than a pattern present at one.

### 4. No silent picks

When multiple options are valid and evidence cannot decide between them, the skill MUST hand off via the user-question YAML mechanism. It MUST NOT pad rationale to disguise a guess as a decision. The first-order test: if you find yourself writing § 6 Rationale that names a tradeoff without citing § 4 evidence that selects across the tradeoff, you are silently picking. Stop, hand off via `awaiting-user`.

### 5. User-question framing

When `status: awaiting-user`, the `user_question` frontmatter field MUST be a self-contained brief that the user can answer without re-reading the source artifact or any other file. Format (multi-line YAML scalar, `|` block):

```
user_question: |
  Question: <restate the question concisely, one or two sentences>

  Context: <one paragraph — why this needs to be decided, what downstream
  artifacts and unit pipelines depend on the answer, what is blocked
  until the user responds>

  Options:

  1. {Option A name}
     Pros: ...
     Cons: ...

  2. {Option B name}
     Pros: ...
     Cons: ...

  3. {... if more}

  Recommendation (advisory only): {agent's lean if any, with one-line
  reasoning, OR "no lean — pure user intent"}

  How to respond: edit this file. Set `status: accepted`, add a
  `user_response` field with the chosen option (e.g.,
  `user_response: "Option B — chose for X reason"`), and the orchestrator
  will resume.
```

The `user_response` field starts as `null`. The orchestrator detects the status flip from `awaiting-user` to `accepted` and resumes the affected pipeline branch.

### 6. Implications are concrete

§ 7 Implications must list **specific** artifact changes that follow from the decision. "Update INTERFACES.md" is not concrete; "Add EP-foo to INTERFACES § 6 with payload `{bar: string, baz: int}`" is concrete. The orchestrator uses § 7 to drive subsequent regenerations; vague implications waste the next agent's work. Each implication entry names: the artifact, the section, and the specific change (add / modify / remove + the content).

### 7. Idempotency

If the same question is invoked twice (orchestrator dispatches a duplicate), the second invocation MUST detect the existing decision file in Phase 1 BEFORE gathering evidence. Behavior by existing status:

- `accepted` → no new file; stdout reports `duplicate — see decisions/D-NNN-slug/DECISION.md`.
- `awaiting-user` → no new file; stdout reports `still awaiting user — see decisions/D-NNN-slug/DECISION.md`.
- `proposed` → no new file; stdout reports `decision proposed but not yet applied — see decisions/D-NNN-slug/DECISION.md`.
- `superseded` or `abandoned` → reason afresh and write a new D-NNN.

This is a soft safety net — the orchestrator should not normally invoke duplicates. Failing to check before investigating wastes the entire run.

### 8. Sequential dependency handling

If during Phase 3 or Phase 4 reasoning the agent realizes the question depends on a sub-question that has NOT been resolved (and is not in any prior decision file passed as input), the agent does NOT decide the sub-question silently. It writes the decision file for the original question with `status: awaiting-user` (because the prerequisite is unresolved), and § 6 Rationale explains the dependency: "this decision depends on the resolution of {sub-question} which has not been written; recommend orchestrator dispatch `decide` on the sub-question first, then re-invoke `decide` on this question with the sub-question's resolution as input." `user_question` describes the dependency and asks the user (or orchestrator-via-user) to confirm the dispatch order.

### 9. Slug discipline

Slug is short (≤ 5 words), kebab-case, derived from the question's noun phrases. Lowercase ASCII letters, digits, and hyphens only. Examples:
- "What format should tokens use?" → `D-007-token-format`
- "Should errors wrap or replace?" → `D-012-error-wrap-vs-replace`
- "Which idempotency key for the push saga?" → `D-018-push-saga-idempotency-key`

Slugs must be unique within `decisions/`. On collision, append a disambiguator drawn from the question (never a numeric counter — `token-format` + `jwt-vs-paseto` is good; `token-format-2` is bad).

### 10. Reversal cost honest

§ 8 Reversal Cost must be honest. Decisions with high reversal cost are extra-important to get right. Test: if reversal would require changes across multiple top-level artifacts (DOMAIN, INTERFACES, BEHAVIOR, plus several unit SPECs and their implementations), flag as `high` even if the decision seems simple at first glance. A decision that affects only one unit's internal helper function is `low`. A decision that changes a wire format that has been deployed and is being read by external clients is `high`. A decision in between is `medium`.

### 11. Don't pre-bias toward any source

The agent has many tools but uses them based on what the question warrants, not because they are available. A question with a clear codebase answer doesn't need web research. A question about library docs doesn't need runtime testing. A question that is purely user-intent doesn't need extensive evidence-gathering — frame it for the user and exit. Use judgment.

### 12. Status uses lifecycle enum, not artifact-state enum

The `status` field uses the decision lifecycle: `proposed | accepted | superseded | awaiting-user | abandoned`. This is an intentional deviation from the canonical artifact-state enum (`complete | has_open_questions | blocked`) used in most other ADD artifacts, because decisions have a lifecycle that the orchestrator and user mutate over time:

- `proposed` — agent has decided but the orchestrator should validate by applying.
- `accepted` — agent has confidently decided OR user has confirmed an awaiting-user hand-off.
- `awaiting-user` — agent has declined to decide; user must edit the file to set status `accepted` and add `user_response`.
- `superseded` — never emitted at write time; only when a later decision references this one as superseded.
- `abandoned` — decision became irrelevant before resolution (the question itself was rendered moot by an upstream change).

The `awaiting_user: bool` field is true iff `status: awaiting-user`; encode both per the brief.

### 13. Single YAML frontmatter block

One YAML frontmatter block at the top, containing all fields together. Never emit a second YAML block. The `user_question` field uses a multi-line YAML scalar (`|` block) when populated — ensure the file is valid YAML by indenting the block correctly.

### 14. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "some", "a few", "best practice", "industry-standard". Use exact library names with versions, exact file paths, exact line numbers, exact field names, exact configuration values, exact `D-NNN` references. Unresolvable ambiguity surfaces in § 9 Open Questions or escalates to user-intent — it does not hide behind vague prose.

---

## Output Format

The template below is wrapped in a 4-backtick fence so that the inner 3-backtick blocks (codebase excerpts and runtime output in § 4 Evidence) render correctly. The decision file itself uses standard 3-backtick fences for those inner blocks.

````markdown
---
skill: DECISION.md
date: {YYYY-MM-DD}
status: {proposed | accepted | superseded | awaiting-user | abandoned}
id: D-{NNN}
kind: {adr | resolution}
title: {short-kebab-case-title}
question_origin: {ARTIFACT§section}  # OR list when batched: [ARTIFACT§5.2, ARTIFACT§5.3]
sources_used: [{web | codebase | runtime | user | prior-decisions}, ...]
awaiting_user: {true | false}
user_question: |
  {Multi-line YAML scalar. Populated only when awaiting_user is true.
  Format: Question / Context / Options / Recommendation / How to respond.
  See Rule 5. Empty string or null when awaiting_user is false.}
user_response: null
resolved_date: {YYYY-MM-DD when status is accepted; null otherwise}
implications: [{ARTIFACT_OR_UNIT}, ...]
reversal_cost: {low | medium | high}
open_questions: {N}
---

# DECISION D-{NNN}: {Title}

> Cross-cutting decision file produced by the `decide` skill. Audit trail
> for one Open Question (or one batch of dependent questions sharing a
> single decision). Source artifact: `{path}`. Status lifecycle:
> proposed → accepted (or awaiting-user → accepted via user edit).

## § 1. Question

{Verbatim quote of the open question from the source artifact. When multiple
 questions are batched into this single decision, quote each one with its
 source citation. Format:}

- **From `{ARTIFACT§section}`:** "{verbatim question text}"
- (Repeat for each batched question.)

## § 2. Context

{What forces are at play. Why this question matters. What downstream skills
 and artifacts depend on the answer. Be specific — name the artifacts and
 sections (`ARTIFACT§section`) and units (`U-NN`) that will use the decision.
 2–4 paragraphs.}

## § 3. Options Considered

{At least two options. The null option — "defer / do nothing" — counts when
 deferral is genuinely a choice with consequences.}

### Option A: {short name}

- **Description:** {what this option means concretely — exact mechanism, exact API, exact convention, exact field shape}
- **Pros:** {specific advantages, each tied to a concrete property}
- **Cons:** {specific disadvantages, each tied to a concrete property}
- **Trade-off summary:** {one sentence}

### Option B: {short name}

- **Description:** {...}
- **Pros:** {...}
- **Cons:** {...}
- **Trade-off summary:** {...}

(Repeat per option. At least two.)

## § 4. Evidence Gathered

Per source used, what was queried and what was found. Be concrete. If a source
category was not used, omit the subsection. Each used source must have at
least one entry.

### Web

- **Query:** `{exact query string}`
- **Result summary:** {what was found, in 1–3 sentences}
- **Source URL:** `{full URL}`
- (Repeat per query.)

### Codebase

- **Search:** `{exact rg / ast-grep command or pattern}`
- **Finding location:** `{file:line-range}`
- **Excerpt:**

  ```
  {5–10 lines of code, verbatim}
  ```

- **Implication:** {what this excerpt says about the question, in 1–2 sentences}
- (Repeat per search.)

### Runtime

(Omit subsection if no runtime use.)

- **Scratch dir:** `decisions/D-{NNN}-{slug}/scratch/`
- **Toolchain:** `{e.g., "Node.js v22.9.0, tsx 4.19.0"}`
- **What was run:**

  ```bash
  {exact command(s)}
  ```

- **Output (verbatim):**

  ```
  {stdout, stderr, exit code}
  ```

- **Implication:** {what this output says about the question, in 1–2 sentences}
- (Repeat per experiment.)

### Prior Decisions

(Omit subsection if no prior decisions consulted.)

- **`D-{MMM}` — {title}:** {one-line relevance — why this prior decision matters here}
- **Quote:** "{verbatim excerpt from D-MMM § 5 or § 6 that bears on this question}"
- (Repeat per prior decision.)

## § 5. Decision

{The chosen option, stated explicitly: **"Decision: Option X — {short name}"**.}

{When `status: awaiting-user`, this section is shorter: **"Decision: Awaiting
 user input."** followed by one sentence naming the hand-off reason — either
 "multiple valid options; choice depends on product direction" (user-intent
 hand-off) or "this decision depends on D-{MMM} which has not been written;
 recommend dispatching `decide` on the prerequisite first" (sequential-dependency
 hand-off). The user_question frontmatter field carries the full brief.}

## § 6. Rationale

{Why this option won. Map specific evidence (§ 4 entries) to specific advantages.
 The rationale is the audit trail — a future reader should understand the
 reasoning without needing to re-do the research. 2–6 paragraphs.}

{When `status: awaiting-user`, this section explains why the agent declined
 to decide — what evidence was gathered, what each option's technical validity
 looks like, and which factor the user must weigh. It does NOT name a winner.}

{When residual uncertainty exists despite a confident decision (the agent
 chose a leaning option but evidence is not overwhelming), name the residual
 uncertainty here in one paragraph: what could still go wrong, what would
 trigger a reversal.}

## § 7. Implications

{What artifacts must change as a result. The orchestrator uses this to drive
 subsequent regenerations. Each entry names the artifact, the section, and
 the specific change.}

- **`{ARTIFACT}` § {section}:** {add / modify / remove + the specific content — e.g., "Add EP-push-token to INTERFACES § 6 with payload `{token: string, expires_at: ISO-8601}`."}
- **`{U-NN}/SPEC.md` § {section}:** {specific change — e.g., "Modify § 4 Interface Contract to cite EP-push-token instead of the previously assumed inline format."}
- (Repeat per implication.)

(If supersedes a prior decision: include an implication entry of the form
 **`decisions/D-{MMM}-{slug}.md`:** mark `status: superseded` with `superseded-by: D-{NNN}`.)

(If no implications: "No artifact updates required — the decision aligns with
 current artifacts as written." This is rare; question this conclusion before
 emitting.)

## § 8. Reversal Cost

- **Cost:** {low | medium | high}
- **Reasoning:** {what would have to change if this decision is reversed —
  specific artifacts, specific code, specific user-facing changes. 1–2 paragraphs.}

## § 9. Open Questions

Decision-level ambiguities only. Cases where applying the decision surfaces
new sub-questions, or where the agent's reasoning reached a sub-point that
needs its own decision file.

- [ ] {Question — e.g., "Once we adopt JWT for tokens, do we need a refresh-token strategy in v1?"}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggested resolution or "dispatch a separate `decide` invocation"}

(If none: "All questions resolved within this decision.")
````

---

## Scope

### In scope

- Reading the source artifact and quoting the Open Question(s) verbatim
- Reading 1–4 cited artifacts named by the orchestrator's invocation
- Reading prior decision files when passed as input (sequential dependency)
- Tool-driven evidence gathering: web search, codebase grep + targeted read, execution sandbox, prior-decisions traversal
- Reasoning bias-free from evidence (Rule 1) — not from what upstream agents implied
- Aggressive web research when the question involves external knowledge (Rule 2)
- Codebase grounding when the question involves existing project conventions (Rule 3)
- Empirical verification via the runtime sandbox at `decisions/D-{NNN}-{slug}/scratch/` when other sources are inconclusive
- Producing exactly one `decisions/D-{NNN}-{slug}/DECISION.md` file per invocation
- Computing the next NNN by globbing existing files; choosing a unique slug
- Handling batched questions (multiple Open Questions sharing a single decision) with `question_origin` as a list and § 1 quoting all batched questions
- Handing off to the user via `status: awaiting-user` with a self-contained `user_question` brief when the question is user-intent or depends on an unresolved sub-question (Rules 4, 5, 8)
- Idempotency check at Phase 1 — detecting existing decisions for the same question and exiting with a stdout pointer (Rule 7)
- Surfacing batching errors (orchestrator batched truly-independent questions) in § 9 with `status: awaiting-user` rather than silently splitting

### Out of scope

- Modifying the source artifact directly — owned by the orchestrator after `status: accepted` (the orchestrator regenerates or edits the source, flips the Open Questions checkbox to `[x]`)
- Generating any artifact other than the decision file (and its scratch dir if runtime used)
- Resolving Open Questions that have unambiguous answers — those should be resolved silently by the upstream skill, not surfaced. (DECIDE handles only genuine ambiguity that escaped the upstream skill.)
- Pre-loading the full design suite — reads are scoped to what the orchestrator passes plus what tools warrant
- Pausing during execution to ask the user — user-question is asynchronous (file-based). The skill never blocks on human interaction during a run (ADD P9)
- Writing code, tests, or production implementation — owned by `/IMPLEMENTATION.md` (H5) and other implementation-style skills
- Architectural design beyond what the question's options entail — owned by `/ARCHITECTURE.md` for foundational structure
- Updating prior decision files (other than the new file linking to a superseded prior via § 7 Implications) — the orchestrator applies supersession edits

---

## Quality Checklist

Before considering the decision file complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `id`, `kind`, `title`, `question_origin`, `sources_used`, `awaiting_user`, `user_question`, `user_response`, `resolved_date`, `implications`, `reversal_cost`, `open_questions`)
- [ ] Single YAML frontmatter block at the top — never two
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "some", "best practice", "industry-standard")
- [ ] § 9 Open Questions is present (empty with "All questions resolved within this decision." or with genuine decision-level sub-questions only)
- [ ] Output is self-contained — readable and actionable without opening the source artifact, the cited artifacts, or the prior decisions
- [ ] Filename matches `decisions/D-{NNN}-{slug}/DECISION.md` pattern with sequential NNN (computed from `max(existing) + 1`, zero-padded to 3 digits) and a unique slug
- [ ] `id` field equals `D-{NNN}` from the filename
- [ ] `status` is one of the lifecycle values (`proposed | accepted | superseded | awaiting-user | abandoned`) — NOT the standard artifact-state enum
- [ ] `awaiting_user: true` if and only if `status: awaiting-user`
- [ ] `kind` is `adr` (architectural commitment) or `resolution` (open-question answer); use judgment per Rule 12
- [ ] § 1 Question is a verbatim quote from the source artifact (when batched, every question is quoted with its source citation)
- [ ] When questions are batched, `question_origin` is a YAML list with one entry per question; § 1 lists every batched question; the agent has verified they share a single decision (not silently split independent decisions)
- [ ] § 2 Context names the specific downstream artifacts and units that depend on the answer — no vague "various downstream consumers"
- [ ] § 3 Options Considered has at least two options; each option has Description, Pros, Cons, and Trade-off summary
- [ ] § 4 Evidence has at least one entry per source listed in `sources_used`; entries are concrete (URLs for web, `file:line` + excerpt for codebase, command + verbatim output for runtime, `D-MMM` + quote for prior decisions)
- [ ] § 4 sources match `sources_used` frontmatter exactly — no extra subsections, no missing subsections
- [ ] § 5 Decision is explicit — "Decision: Option {X}" when accepted, "Decision: Awaiting user input." with hand-off reason when awaiting-user
- [ ] § 6 Rationale maps specific § 4 evidence entries to specific advantages — not generic prose, no bare reasoning without evidence anchors
- [ ] § 7 Implications lists specific artifact changes — exact section, exact change content (add/modify/remove + the actual content), not "update relevant docs"
- [ ] § 8 Reversal cost is reasoned per Rule 10 (multiple top-level artifacts → high; single unit internal → low; otherwise medium)
- [ ] When `status: awaiting-user`, the `user_question` frontmatter field is populated per Rule 5 framing (Question + Context + Options + Recommendation + How to respond)
- [ ] When `status: awaiting-user` due to sequential dependency (Rule 8), § 5 and § 6 explicitly name the unresolved prerequisite sub-question and recommend the orchestrator's dispatch order; user_question describes the dependency
- [ ] When `status: accepted` and a prior decision is superseded, § 7 includes an implication marking the superseded file (`status: superseded`, `superseded-by: D-{NNN}`)
- [ ] `resolved_date` is set to today's date when `status: accepted` is emitted directly; `resolved_date` is `null` when `status: proposed | awaiting-user | abandoned`
- [ ] No silent picks — multi-option questions either land on a clearly-evidenced option (with § 4 evidence selecting between options) or transition to awaiting-user (Rule 4)
- [ ] Slug is unique within `decisions/`; on collision, a disambiguator from the question is appended (not a numeric counter)
- [ ] Idempotency check ran at Phase 1 before any evidence-gathering — no investigation was wasted on a duplicate question (Rule 7)
- [ ] If runtime sandbox was used, the scratch directory `decisions/D-{NNN}-{slug}/scratch/` exists and is not empty; § 4 Runtime cites files inside it
- [ ] Citations use stable IDs (`D-NNN`, `UC-NN`, `SCN-NN`, `INV-NN`, `EP-name`, `EVT-name`, `ERR_CODE`, `SM-entity-state`, `SAGA-name`, `THREAT-NN`, `MIT-NN`, `METRIC-name`, `SLO-name`, `CFG_NAME`, `U-NN`) and `ARTIFACT§section` references — never line numbers or uncited quoted prose
- [ ] `open_questions` frontmatter count equals the unresolved checkbox count in § 9
