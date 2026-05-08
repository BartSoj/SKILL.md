---
name: ROADMAP.md
description: File a planned-work trigger — a feature, refinement, refactor, or chore — by performing concept-overlap discovery against the design suite and existing roadmap, then synthesizing user intent into a complete `roadmap/<NNN>-<slug>/ROADMAP.md` with frontmatter, context, sketch, acceptance criteria, current state, impact summary, out-of-scope boundaries, and open questions. Use when asked to file a roadmap item, plan a feature, propose a new capability, plan a refactor, schedule a chore, propose a refinement, add to the backlog, or produce a `roadmap/<NNN>-<slug>/ROADMAP.md`.
---

# Task: Generate `roadmap/<NNN>-<slug>/ROADMAP.md` — Planned-Work Trigger

## Objective

Produce one `roadmap/<NNN>-<slug>/ROADMAP.md` file per invocation that files a single planned-work trigger — a feature, refinement, refactor, or chore — into the project's roadmap. The skill is one of two trigger-creation skills (the other is `/ISSUE.md` for defects in shipped code); the `/ROADMAP.md` and `/ISSUE.md` artifacts are the only way work enters phase G of the Agent-Driven Development pipeline. The output carries enough verifiable acceptance criteria for the downstream `/SPEC.md` (G1) to scope a unit, for `/SPEC_REVIEW.md` (G2) to validate scope alignment, and for `/VERIFICATION.md` (G7) to confirm the unit fulfilled the trigger. Frontmatter is mutable — orchestrator and `/RECONCILIATION.md` (G8) advance `status` and populate `promoted_to_units` over the trigger's lifecycle. The body is largely write-once: regenerate only when the user explicitly asks to refine.

ROADMAP exists because every code change in ADD must originate from a durable, citable artifact — not a chat message, not an inline TODO. Without it, units are created on premises that disappear once the conversation that produced them ends, and `/VERIFICATION.md` has nothing to verify against. The defining discipline — and the commonest violation — is **why / what / when-it's-done, never how**: roadmap items must not contain wire formats, schemas, function signatures, code, or step-by-step plans. Those are the territory of SPEC, INTERFACES, DATA, IMPLEMENTATION, and PLAN respectively.

---

## Inputs

1. **User intent** (required, primary subject) — free-text description of what should be filed. May include the kind hint ("feature", "refactor", "chore", "refinement"), the area hint, the priority hint, and the rough description. Extract these from the prose if not stated explicitly.
2. **`PROPOSAL.md`** (required, auto-discovered at project root) — for principles, non-goals, scope boundaries, and target actors. The roadmap item must not contradict any.
3. **`ARCHITECTURE.md`** (required, auto-discovered at project root) — for component topology and to identify which component owns the work.
4. **`DOMAIN.md`** (required, auto-discovered at project root) — for the bounded-context catalog, which is the source of valid `area:` values.
5. **Existing roadmap items** (auto-discovered) — `roadmap/*/ROADMAP.md`. Used for `<NNN>` allocation and for detecting overlapping items.
6. **Existing issues** (auto-discovered) — `issues/*/ISSUE.md`. Frontmatter scan only. Used for cross-references when the new roadmap item closes or supersedes existing issues.
7. **Existing units** (auto-discovered) — `units/*/u*/SPEC.md`. Frontmatter scan plus a one-line concept summary. Used for detecting overlap with shipped or in-flight work and for populating `Current state`.
8. **Surface IAs and contract artifacts** (optional, selectively read) — `WEB_IA.md`, `CLI_IA.md`, `INTERFACES.md`, `DATA.md`, `BEHAVIOR.md`, `ERRORS.md`, `QUALITY.md`, `SECURITY.md`, `OPERATIONS.md`. Read selectively when discovery suggests the feature touches a particular surface or contract layer.
9. **Existing `roadmap/<NNN>-<slug>/ROADMAP.md`** (auto-discovered — only if refining) — when the user asks to refine an existing roadmap item, read it fully and **preserve the orchestrator-managed frontmatter fields** (`id`, `slug`, `status`, `promoted_to_units`).

Read budget: 3 required full reads (PROPOSAL, ARCHITECTURE, DOMAIN) + 2–4 frontmatter-only scans (cheap — YAML headers only) + 1–3 selective full reads of design artifacts the discovery surfaces. Within ADD's bounded read-set principle (≤ 10 reads).

---

## Workflow

Roadmap filing proceeds in six phases: intent extraction, discovery, body authoring, ID allocation and frontmatter, validation, and write. Phases are sequential; revisit earlier phases if a later one reveals a missing concept, an overlap, or a contradiction.

### Phase 1: Intent Extraction

Read the user intent end-to-end. Extract five things into a working note:

- **Kind**: `feature` | `refinement` | `refactor` | `chore`. If unstated, infer from the prose. A "new capability for users" is a feature. "Improve the simplification we made in u012" is a refinement. "Restructure module X internally" is a refactor. "Upgrade dependency Y" is a chore. When the prose mixes signals (e.g., "let's clean up the validation and add per-field rules"), treat it as two distinct triggers — file the dominant one and surface the second in `Open Questions` for follow-up.
- **Area**: must be one of the bounded contexts in `DOMAIN.md` § 2. If the user named it, use that. Otherwise infer from the concepts and components implied by the intent. If the work straddles two areas equally, the work is over-scoped — see Rule 5; surface in Open Questions.
- **Priority**: `low` | `medium` | `high`. If unstated, default to `medium`.
- **Title**: a short descriptive phrase. Becomes the `title:` field and seeds the slug.
- **Concept keywords**: 3–8 nouns and verbs that name the feature concept. These drive Phase 2 discovery — be generous; over-keyword is cheaper than under-keyword.

If the intent is too vague to extract these (e.g., "add some webhooks somewhere"), proceed with best-effort extraction and surface clarifying questions in `Open Questions` of the output. Do not block on them; the skill still produces a complete file.

If the prose signals the work is "speculative" / "future" / "maybe one day", set `status: idea`. If it signals "let's plan this", "next quarter", "committed", set `status: planned`. Default to `planned`.

### Phase 2: Discovery

This is the load-bearing phase. The skill must perform concrete file operations to surface what already exists in the project relevant to the proposed roadmap item. The discovery output populates the `Current state` section of the body.

Sources to scan, in order:

1. **Concept-overlap in design docs.** `grep -i` across `DOMAIN.md`, `ARCHITECTURE.md`, surface IAs (`WEB_IA.md`, `CLI_IA.md`, etc.), `INTERFACES.md`, `DATA.md`, `BEHAVIOR.md`, `ERRORS.md`, `QUALITY.md`, `SECURITY.md`, `OPERATIONS.md` for the concept keywords from Phase 1. Cite hits by `ARTIFACT§section`. Filter to semantically relevant hits — a hit on the word "webhooks" in a generic security paragraph is not relevant; a hit in `INTERFACES § Events` is.
2. **Use-case overlap.** Scan `USE_CASES.md` for use cases touching the same actors or capabilities. Cite by `UC-NN`.
3. **Unit-frontmatter overlap.** Read frontmatter (only — not bodies) of every `units/*/u*/SPEC.md`. Filter by:
   - `concepts:` overlap with Phase 1 keywords
   - `files:` overlap with files the new feature would likely touch
   - `area:` matching the chosen area
   Cite hits by unit id (`u<NN>`).
4. **Roadmap overlap.** Read frontmatter (only) of every `roadmap/*/ROADMAP.md`. Flag siblings with overlapping concepts, the same `area:`, or overlapping titles. Cite by `roadmap/<id>-<slug>`.
5. **Issue overlap.** Read frontmatter (only) of every `issues/*/ISSUE.md`. Flag issues whose `component:` or `area:` overlaps with the feature's likely files, or whose `kind: regression` indicates the new roadmap item could close them.

The discovery output is **summarized, not exhaustive** — a tight bulleted list of relevant hits with stable-ID citations. Each hit gets a sentence on relevance. If discovery finds *no* existing references, that is fine and notable: `Current state` becomes "No prior references in design docs, units, or other roadmap items."

If discovery surfaces a unit whose frontmatter `concepts:` overlaps strongly with the new item, that is a signal of duplicate work or of a unit that should be extended rather than a fresh one filed. Note it in `Current state` and in `Related`; the orchestrator decides downstream whether to widen the existing unit or split.

### Phase 3: Body Authoring

The body sections vary by `kind`. Author them in the order below; each section's full template is in §8 Output Format.

For `kind: feature`: Context → Sketch → Acceptance → Current state → Impact → Out of scope → Open Questions → Related.

For `kind: refinement`: Current behavior → When to revisit → What to build → Acceptance → Current state → Impact → Open Questions → Related.

For `kind: refactor`: Context → Approach → Acceptance → Current state → Impact → Out of scope → Open Questions → Related.

For `kind: chore`: Context → Sketch → Acceptance → Open Questions → Related.

The order is **fixed** — downstream skills `grep` for section headings to extract specific subsections (e.g., `/VERIFICATION.md` reads `## Acceptance`). Reordering breaks the contract.

While authoring, hold the line on Rule 1 (no how). The temptation is strong — most roadmap items are filed by someone who knows a lot about the design they want — but every wire format, schema fragment, or function signature that leaks in is downstream-skill territory paying rent in the wrong section.

### Phase 4: ID Allocation and Frontmatter

**`<NNN>` allocation.** Glob `roadmap/*/ROADMAP.md`. Read frontmatter `id` for each. Take `max(id) + 1`. Zero-pad to 3 digits (`044`, `100`, `1000` once natural overflow happens). Never reuse a retired id; never go backwards.

**Slug computation.** Kebab-case form of `title`: lowercase, ASCII alphanumerics and hyphens only, filler words (`a`, `an`, `the`, `for`) stripped. Examples:
- "Webhooks and event system" → `webhooks-event-system`
- "Per-branch access control (replacing per-repo)" → `per-branch-access-control`
- "Extract repo-name validation into shared module" → `extract-validation-shared-module`

The folder name `<NNN>-<slug>` matches `id` and `slug` in frontmatter exactly. All three must agree.

**Frontmatter (write all fields):**

```yaml
---
skill: ROADMAP.md
date: <YYYY-MM-DD>
status: idea | planned | in-design | in-progress | partial | implemented | deferred | abandoned

id: "<NNN>"
slug: <slug>
title: <one-line title>
kind: feature | refinement | refactor | chore
area: <bounded-context-from-DOMAIN>
priority: low | medium | high
proposed: <YYYY-MM>

promoted_to_units: []
depends_on: []
related: []

acceptance_criteria_count: <N>
open_questions: <N>
---
```

Notes:
- `status` enum is **deliberately overridden** for ROADMAP to the lifecycle values listed (per ADD spec §7.4). Do not "fix" it to the canonical `complete | has_open_questions | blocked`.
- `id` is **quoted** because YAML would otherwise drop the leading zero (`"044"` ≠ `44`).
- `proposed` is **month-precision** (`YYYY-MM`).
- `promoted_to_units` is an **empty list at filing time**; orchestrator and `/RECONCILIATION.md` populate it later.
- `depends_on` (ids that block this) and `related` (informational links) are populated from Phase 2 discovery.
- `acceptance_criteria_count` and `open_questions` are skill-computed counts; they must match the body exactly.

### Phase 5: Validation

Before writing the file, verify:

- All required body sections for the chosen `kind` are present, with exact headings, in the fixed order (per Output Format).
- `## Acceptance` has at least one verifiable scenario; 3–7 total. Approaching 15 indicates over-scope (Rule 5); split.
- `## Current state` reflects Phase 2 findings — not absent, not generic prose. If discovery found nothing, the section explicitly says so.
- `## Impact` (when present) is high-level only — no wire formats, schemas, code, function signatures, or step-by-step plans (Rule 4).
- No forbidden content per Rule 1 leaked anywhere in the body.
- Frontmatter `acceptance_criteria_count` matches the `## Acceptance` bullet count; `open_questions` matches the `- [ ]` checkbox count under `## Open Questions`.
- `area` is one of the bounded contexts in `DOMAIN.md` § 2.

### Phase 6: Write the File

Create the directory `roadmap/<NNN>-<slug>/` (and the parent `roadmap/` if it does not exist). Write `ROADMAP.md` inside it. Output to stdout a one-line confirmation with the file path and key counts:

```
Filed roadmap/044-webhooks-event-system/ROADMAP.md (kind=feature, area=server, priority=medium, acceptance=5, open_questions=2).
```

---

## Rules

These rules govern the output document. Violations are detected by the Quality Checklist.

### 1. Triggers carry "why, what, when-it's-done" — never "how"

A roadmap item answers three questions: **why** (Context / Current behavior), **what at user-visible level** (Sketch / What to build / Approach), and **when-it's-done** (Acceptance). It does NOT answer **how** — that is SPEC, PLAN, and IMPLEMENTATION territory.

Forbidden anywhere in the body:

- Wire formats — HTTP method + path + payload shape. Owned by SPEC and reconcile-driven INTERFACES.
- Database schema — `CREATE TABLE`, column types, indexes, constraints. Owned by SPEC and reconcile-driven DATA.
- Function signatures, pseudocode, or any code. Owned by IMPLEMENTATION.
- Test code beyond reproduction. Owned by VERIFICATION.
- Step-by-step implementation steps. Owned by PLAN.
- Architecture diagrams or component layouts. Owned by ARCHITECTURE itself — the roadmap may say "this introduces a new component called X" but must not draw the box.
- Resolved decisions on contested choices. Those go to `/DECISION.md` and are linked by `D-NNN`; the roadmap parks the question in `Open Questions`.
- A second roadmap item — one trigger per file. If the work spans two distinct capabilities, file two roadmap items linked via `depends_on:` or `related:`.

The reason this rule is hard: roadmap items are most often invoked by a human who knows a lot about the design they want. The temptation to dump it all in is strong. The skill must enforce the boundary.

### 2. Acceptance criteria are read by G7

Every roadmap item must have an `## Acceptance` section with verifiable scenarios. `/VERIFICATION.md` (G7) reads this section to confirm the unit fulfilled the trigger. Without it, the unit is unverifiable.

Acceptance criteria are one of three shapes:

- **Scenarios with given/when/then** — "Given a logged-in user with no webhooks configured, when they register a new webhook for `push.completed`, then the registration returns a webhook id and a signing secret."
- **Measurable assertions** — "Latency p95 of `/repos/{name}/push` is under 200ms with 100 concurrent uploads."
- **Pass/fail outcomes** — "Running `repo list` after the push includes the pushed repository."

Reject vague acceptance: "works smoothly", "feels good", "fast", "easy", "well-tested" — rewrite with a concrete scenario or escalate to `Open Questions`.

Aim for 3–7 scenarios. If you find yourself writing 15+, the roadmap item is over-scoped — see Rule 5.

### 3. Discovery must be performed and reflected

Phase 2 discovery is mandatory. Even when the user signals "this is brand-new, nothing exists yet", the skill scans to verify and to populate `Current state`. Skipping discovery produces stale roadmap items that duplicate work or contradict shipped behavior.

`Current state` is summarized — a paragraph or tight bulleted list with stable-ID citations. Each hit gets one sentence on relevance. If genuinely nothing exists: "No prior references in design docs, units, or other roadmap items."

Cite by stable IDs only: `ARTIFACT§section`, `UC-NN`, `u<NN>`, `roadmap/<id>-<slug>`, `issues/<id>-<slug>`. Never quote artifact prose — quotes go stale on regeneration.

### 4. Impact is high-level only

The `## Impact` section signals which top-level artifacts the work will likely touch — for the orchestrator's planning. It is **a list of pointers, not a design**.

Good Impact entry: "INTERFACES — likely 2–3 new endpoints under `/repos/{name}/webhooks`; one new event type."

Bad Impact entry: "INTERFACES — `POST /repos/{name}/webhooks` with body `{event_types: string[], target_url: string}` returning 201 with `{id, secret}`."

The bad version is SPEC territory. Roadmap forecasts; SPEC commits.

### 5. One trigger per file

If the work is genuinely two distinct capabilities, file two roadmap items and link them via `depends_on:` or `related:`. Signs of over-scope:

- `## Acceptance` has > 7 scenarios.
- `## Impact` names changes to > 4 top-level artifacts.
- `## Sketch` describes two distinct user-visible capabilities ("users can both X and Y").
- `## Current state` reveals > 2 existing units that overlap.
- The work crosses bounded contexts — `area:` would be a compromise.

When over-scope is detected during authoring, split the trigger before writing the file, not after. A bloated roadmap item that ships and then has to be torn into two units mid-pipeline costs more than splitting it at filing.

### 6. Open Questions are allowed and encouraged

The roadmap's `## Open Questions` section parks unresolved questions for later resolution by `/DECISION.md`. Do not try to answer everything before filing. A roadmap with 3 open questions is healthier than one that buried 3 silent guesses.

Format each question as a checkbox with options + tradeoffs + recommendation:

```markdown
- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}
```

If none: "All questions resolved." (rare for non-trivial roadmap items, common for chores).

### 7. ID allocation and slug discipline

`<NNN>` is `max(id) + 1` from existing `roadmap/*/ROADMAP.md` frontmatter, zero-padded to 3 digits. Never reuse a retired id. Never go backwards. The folder name `<NNN>-<slug>`, the frontmatter `id`, and the frontmatter `slug` must agree exactly.

Slug is kebab-case derived from `title`: ASCII letters, digits, and hyphens only; lowercase only; filler words (`a`, `an`, `the`, `for`) stripped. Slugs are stable — once filed, the slug is permanent.

### 8. Status starts at `planned` for committed work, `idea` for speculative

If the user invokes the skill with concrete intent ("file a roadmap item for webhooks"), default `status: planned`. If the user signals it is speculative ("a future thing we might do", "maybe one day"), default `status: idea`. The orchestrator advances the status thereafter via the trigger lifecycle bridge rules — the skill never emits `in-design`, `in-progress`, `partial`, or `implemented` at filing time. Only `idea`, `planned`, `deferred`, or `abandoned` are valid first-write states; of these, `idea` and `planned` are the only common ones.

### 9. Single YAML frontmatter block

One YAML frontmatter block at the top of the file. Never emit a second YAML block anywhere. All counts (`acceptance_criteria_count`, `open_questions`) match the body exactly.

### 10. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "some", "a few", "best practice", "industry-standard". Use exact nouns, exact criteria, exact stable IDs. If exact is impossible because the question is genuinely open, surface it in `Open Questions` rather than hide behind a placeholder word.

### 11. Domain language for descriptive sections; technical language only when warranted

`## Context` and `## Sketch` (and the `kind`-specific equivalents `## Current behavior` / `## What to build` for refinements) speak in the user's domain language. They describe the user-visible experience.

Technical language ("API", "endpoint", "schema", "table", "service") is permitted in `## Impact`, `## Current state`, and `## Related` because those sections cite top-level artifacts and existing units — citations require their actual names. The descriptive sections stay user-facing.

For `kind: refactor` and `kind: chore`, the language can be more technical because the audience for the work is internal:
- Refactor `## Approach` may name internal patterns ("strangler fig", "dependency injection", "ports and adapters").
- Chore `## Sketch` may name versions and packages ("upgrade `pg` from 8.11.5 to 16.4.0").

### 12. No interactive questions

The skill runs headless from `claude -p`. Never pause to ask the user. If a decision has one clearly correct answer, resolve silently. If genuinely ambiguous, surface in `## Open Questions` with options, tradeoffs, and a recommendation.

---

## Output Format

The body section set varies by `kind`. Frontmatter is identical across kinds. Section headings are **mandatory and exact** — downstream skills `grep` for them.

### Common frontmatter (all kinds)

```yaml
---
skill: ROADMAP.md
date: {YYYY-MM-DD}
status: {idea | planned | in-design | in-progress | partial | implemented | deferred | abandoned}

id: "{NNN}"
slug: {slug}
title: {one-line title}
kind: {feature | refinement | refactor | chore}
area: {bounded-context-from-DOMAIN}
priority: {low | medium | high}
proposed: {YYYY-MM}

promoted_to_units: []
depends_on: [{roadmap-id}, ...]
related: [{roadmap-or-issue-id}, ...]

acceptance_criteria_count: {N}
open_questions: {N}
---
```

### Body — `kind: feature`

```markdown
# ROADMAP — {title}

## Context

{Two to four paragraphs in the user's domain language. Why this matters now;
 what problem it solves; what value it provides; who is affected. What
 triggered the filing — observed limitation, recurring user request, strategic
 decision, recurring issue category. A reader joining the project a year later
 should understand from this section alone why we are doing this.}

## Sketch

{What we will build, at the user-visible level. Capability description in user
 terms. "Users can X by Y; the system responds with Z." Sample interactions
 described in plain language (a CLI command sequence or a web flow). Not a
 design — no wire formats, no schemas, no function signatures.}

## Acceptance

- {Scenario 1 — given/when/then or measurable assertion or pass/fail outcome}
- {Scenario 2 — ...}

(Aim for 3–7 scenarios. If approaching 15, split this roadmap item.)

## Current state

- **Design references:** {ARTIFACT§section} — {one-line on relevance}.
- **Use cases:** {UC-NN} — {one-line}.
- **Existing units:** {u<NN>} — {what it covers, what it does not}.
- **Related roadmap:** {roadmap/<id>-<slug>} — {relationship}.
- **Related issues:** {issues/<id>-<slug>} — {relationship}.
- **Code state:** {placeholder types, stubs, TODOs found via codebase glob, if any}.

(If genuinely nothing exists: "No prior references in design docs, units, or
 other roadmap items.")

## Impact

- USE_CASES — {sentence on what is likely added}.
- DOMAIN — {sentence}.
- INTERFACES — {sentence}.
- DATA — {sentence}.
- (Continue for each likely-touched artifact. For artifacts not touched:
  "No expected changes." Or omit them entirely — the goal is signal, not
  exhaustive enumeration.)

## Out of scope

- {Capability someone might reasonably expect from this work but that we are
   explicitly NOT doing} — {one-line reason; often "follow-up roadmap item
   if/when needed"}.

(Repeat for each excluded capability. The budget guardrail.)

## Open Questions

- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")

## Related

- {roadmap/<id>-<slug>} — {one-line on relationship}.
- {issues/<id>-<slug>} — {one-line}.
- {u<NN>} — {one-line}.
- {ARTIFACT§section} — {one-line; for source-of-truth references}.
```

### Body — `kind: refinement`

The `Acceptance`, `Current state`, `Open Questions`, and `Related` sections share their format with `kind: feature`. The opening sections differ:

```markdown
# ROADMAP — {title}

## Current behavior

{What the system does today — the simplification we accepted at first ship.
 The "what we shipped that we plan to revisit." Cite the unit(s) that
 implemented the simplified version: {u<NN>}.}

## When to revisit

{The trigger condition for picking this up. Concrete: "When user count > 1000",
 "When the hardcoded list reaches 10 entries", "When a real customer asks for
 it", "After refactor X completes". Not a date — a condition.}

## What to build

{The fuller version we will do when we revisit. User-visible level, not
 designed.}

## Acceptance
{3–7 scenarios; same shape as feature.}

## Current state
{Discovery findings; typically shorter than feature — often "the simplified
 version is u<NN>" plus a few related references.}

## Impact
{Often smaller than feature. Often just "INTERFACES — small change to
 {EP-name}; DATA — possibly one new column."}

## Open Questions
{Same format as feature.}

## Related
{Same format as feature.}
```

### Body — `kind: refactor`

```markdown
# ROADMAP — {title}

## Context

{What is wrong with the current code. The smell, the friction, the prediction
 that this will hurt later. Cite specific code paths or units. "Module X is
 doing two things; module Y has reached 800 lines and three responsibilities;
 the client has duplicated the validation logic across three call sites."}

## Approach

{High-level approach. Pattern, structure, migration strategy. "Extract the
 validation into a shared module"; "Apply strangler fig: route new calls
 through the new path while the old path is deprecated"; "Split unit X along
 the Y / Z axis." NOT a step-by-step plan — that is PLAN.md.}

## Acceptance

- {Regression scope — typical: "All existing scenarios from impacted unit
   SPECs continue to verify; no observable behavior change; all existing
   tests pass."}
- {Positive criteria — "Module X is now reusable from module Y"; "Module Z
   no longer depends on module W."}

## Current state
{Cite affected units (`u<NN>`) and file paths; cite issues whose root cause
 is the smell this refactor addresses.}

## Impact
{Often minimal. Refactors usually do not change top-level artifacts. "No
 top-level changes expected" is a valid Impact for many refactors.}

## Out of scope

- {Capability — "Renaming X is out of scope; we will do that in a follow-up."}

(Especially important for refactors. Scope creep on refactors kills weeks.)

## Open Questions
{Same format.}

## Related
{Same format.}
```

### Body — `kind: chore`

```markdown
# ROADMAP — {title}

## Context

{What triggered this — dependency upgrade window, security advisory, vendor
 deprecation, license change, build infra. Cite the trigger source: a CVE
 id, a vendor deprecation notice URL, an internal policy, a tooling change.}

## Sketch

{What needs to happen. Often very mechanical: "Upgrade {package} from X.Y to
 Z.W"; "Replace {vendor A} with {vendor B}"; "Pin Node version in CI to 22 LTS".}

## Acceptance

- {What proves done — "Version pinned in package.json"; "Security alert
   dismissed"; "Build green on CI"; "All references to {old vendor} removed."}

## Open Questions
{Often "All questions resolved." for chores.}

## Related
{Same format.}
```

---

## Scope

### In scope

- Extracting kind, area, priority, title, and concept keywords from user prose
- Performing concept-overlap discovery across design docs, units, existing roadmap, and existing issues
- Allocating the next `<NNN>` from existing roadmap items; computing the slug from the title
- Creating the directory `roadmap/<NNN>-<slug>/` and writing `ROADMAP.md` inside it
- Authoring body sections appropriate to the chosen `kind`, in the fixed order
- Populating frontmatter completely, including computed counts (`acceptance_criteria_count`, `open_questions`)
- Citing stable IDs (`UC-NN`, `EVT-name`, `EP-name`, `u<NN>`, `roadmap/<id>-<slug>`, `issues/<id>-<slug>`, `ARTIFACT§section`) — never quoted artifact prose
- Surfacing unresolved questions in `## Open Questions` rather than guessing silently
- Preserving orchestrator-managed frontmatter fields (`id`, `slug`, `status`, `promoted_to_units`) when refining an existing roadmap item

### Out of scope

- Resolving open questions — owned by `/DECISION.md` (cross-cutting; `decisions/D-NNN-slug/DECISION.md`)
- Filing defects in shipped code — owned by `/ISSUE.md` (sibling skill; `issues/<NNN>-<slug>/ISSUE.md`)
- Picking up the trigger to begin implementation — owned by the orchestrator (it allocates a unit and runs G1–G8)
- Producing the unit's SPEC — owned by `/SPEC.md` (G1; reads this roadmap as a trigger artifact)
- Editing top-level design artifacts (DOMAIN, INTERFACES, DATA, BEHAVIOR, etc.) — owned by their own skills (`/DOMAIN.md`, `/INTERFACES.md`, etc.) and applied by `/RECONCILIATION.md` (G8)
- Status transitions on the roadmap item after filing (`planned` → `in-design` → `in-progress` → `partial` → `implemented`) — performed by the orchestrator and `/RECONCILIATION.md` per ADD spec §7.7 trigger lifecycle bridge rules
- Wire formats, database schemas, function signatures, code, step-by-step plans — owned by SPEC, INTERFACES (via reconcile), DATA (via reconcile), IMPLEMENTATION, and PLAN respectively

---

## Quality Checklist

Before considering the roadmap file complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill: ROADMAP.md`, `date`, `status`, `id`, `slug`, `title`, `kind`, `area`, `priority`, `proposed`, `promoted_to_units`, `depends_on`, `related`, `acceptance_criteria_count`, `open_questions`)
- [ ] Single YAML frontmatter block at the top — never two
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "some", "best practice", "industry-standard")
- [ ] `## Open Questions` is present (with genuine questions or "All questions resolved.")
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] Frontmatter `id`, frontmatter `slug`, and the folder name `<NNN>-<slug>` agree exactly
- [ ] `id` is quoted in frontmatter (preserves zero-padding: `"044"` not `44`)
- [ ] `kind` is one of `feature | refinement | refactor | chore`
- [ ] `area` is a bounded context defined in `DOMAIN.md` § 2
- [ ] `status` is `idea` (speculative) or `planned` (committed) for a freshly-filed item — not `in-design`, `in-progress`, `partial`, or `implemented` (those are orchestrator-managed transitions)
- [ ] `promoted_to_units` is an empty list (`[]`) at filing time — orchestrator populates later
- [ ] `proposed` is `YYYY-MM` (month precision)
- [ ] All required body sections for the chosen `kind` are present, with exact headings, in the fixed order (per Output Format)
- [ ] `## Acceptance` has at least one verifiable scenario (given/when/then, measurable assertion, or pass/fail outcome)
- [ ] `## Acceptance` has 3–7 scenarios — split the roadmap if approaching 15
- [ ] `## Current state` reflects Phase 2 discovery findings with stable-ID citations (or explicitly states "No prior references" if discovery found none)
- [ ] `## Impact` is high-level — no wire formats, schemas, function signatures, code, or step-by-step plans leaked in
- [ ] No forbidden content per Rule 1 anywhere in the body (no wire formats, no schemas, no code, no plans, no architecture diagrams, no resolved decisions on contested choices)
- [ ] `## Context` / `## Sketch` (or `## Current behavior` / `## What to build` for refinements) speak in domain language — no `endpoint`, `JSON`, `table`, `service` outside `## Impact`, `## Current state`, `## Related`
- [ ] Frontmatter `acceptance_criteria_count` matches the bullet count under `## Acceptance`
- [ ] Frontmatter `open_questions` matches the `- [ ]` checkbox count under `## Open Questions`
- [ ] Citations use stable IDs (`UC-NN`, `EP-name`, `EVT-name`, `u<NN>`, `roadmap/<id>-<slug>`, `issues/<id>-<slug>`, `ARTIFACT§section`) — never quoted artifact prose
- [ ] Slug is kebab-case, lowercase ASCII only, filler words stripped; the slug is unique within `roadmap/`
- [ ] No interactive questions surfaced during execution — the skill ran headless from start to finish
- [ ] Document length is 100–500 lines; overflow indicates over-scope (split per Rule 5) or detail-leak (move to SPEC per Rule 1)
