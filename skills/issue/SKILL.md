---
name: ISSUE.md
description: File a defect-in-shipped-code trigger — a bug or regression — by reproducing the failure, comparing observed behavior against the source-of-truth artifact (INTERFACES, IA, SPEC), classifying severity, identifying root cause when known, and recording the audit trail in a complete `issues/<NNN>-<slug>/ISSUE.md` with frontmatter, summary, reproduction, expected/observed behavior, root cause, scope, and suggested fix. Use when asked to file an issue, file a bug, file a regression, report a defect, log a defect, document a contract mismatch, or produce an `issues/<NNN>-<slug>/ISSUE.md`.
---

# Task: Generate `issues/<NNN>-<slug>/ISSUE.md` — Defect-in-Shipped-Code Trigger

## Objective

Produce one `issues/<NNN>-<slug>/ISSUE.md` file per invocation that files a single defect in shipped code as a durable, citable trigger artifact. The skill is one of two trigger-creation skills (the other is `/ROADMAP.md` for planned work); the `/ROADMAP.md` and `/ISSUE.md` artifacts are the only way work enters phase G of the Agent-Driven Development pipeline. The output carries enough to scope a fix unit at G1, validate scope at G2, verify the reproduction at G7, and feed orchestrator/reconcile lifecycle transitions at G8. Frontmatter is mutable — the orchestrator and `/RECONCILIATION.md` advance `status` (`open → in-progress → fixed`) and populate `promoted_to_units`, `fixed_in`, `fixed_date` over the issue's lifecycle. The body is largely write-once: regenerate only when the user explicitly asks to refine (typically to add a re-confirmation note when an issue resurfaces).

ISSUE exists because every defect must originate from a durable artifact — not a chat message, not a Slack thread, not an inline TODO. Without it, fixes are filed against premises that disappear once the conversation that surfaced them ends, and `/VERIFICATION.md` (G7) has nothing to verify against. The defining discipline — and the commonest violation — is **issues describe shipped behavior that contradicts a documented expectation**: every issue must cite the source-of-truth artifact (`INTERFACES.md`, an IA, `BEHAVIOR.md`, a unit's `SPEC.md`, …) by `ARTIFACT§section`, and the skill MUST NOT fabricate expected behavior when the source-of-truth is silent. Silent source-of-truth is itself a finding — it gets surfaced and the issue is filed `status: open-for-discussion`.

---

## Inputs

1. **Defect intent** (required, primary subject) — free-text description of what's wrong. Includes (or implies) the symptom, the context where it was observed, the suspected component (when known), severity hint (when known), and reproduction steps (when known). The skill extracts these from the prose; missing fields surface as `Open Questions` or as conservative defaults.
2. **Source-of-truth artifacts** (auto-discovered, selective) — the artifact(s) that document the *expected* behavior the defect violates. Pick 1–3 based on the defect's surface: HTTP / wire-format → `INTERFACES.md`; UI → `WEB_IA.md` / `MOBILE_IA.md` / `TUI_IA.md`; CLI → `CLI_IA.md`; voice → `VOICE_IA.md`; state-machine / saga → `BEHAVIOR.md`; data integrity / persistence → `DATA.md`; error-message / error-code → `ERRORS.md`; domain-invariant → `DOMAIN.md`; security → `SECURITY.md`; performance / observability → `QUALITY.md`; ops / config / deployment → `OPERATIONS.md`; unit-level → the unit's `units/<area>/u<NN>/SPEC.md`.
3. **Existing issues** (auto-discovered) — `issues/*/ISSUE.md`. Frontmatter scan only. Used for `<NNN>` allocation, duplicate detection, and related-issue cross-referencing.
4. **Existing units** (auto-discovered) — `units/*/u*/SPEC.md`. Frontmatter scan only. Used to identify the owning unit (via `files:` overlap with the suspected component) and populate `component:` and `References` accurately.
5. **Existing roadmap items** (auto-discovered) — `roadmap/*/ROADMAP.md`. Frontmatter scan only. Used to detect overlap with planned-but-unshipped work, which would make the report a roadmap item, not an issue.
6. **Codebase access** (optional) — for grepping the suspected file/function to confirm root cause when the defect intent points to a symptom but not a root cause.
7. **Existing `issues/<NNN>-<slug>/ISSUE.md`** (auto-discovered — only if refining) — when the user asks to refine an existing issue, read it fully and **preserve orchestrator-managed frontmatter fields** (`id`, `status` transitions, `promoted_to_units`, `fixed_in`, `fixed_date`).

Read budget: 1–3 source-of-truth artifacts + 2–4 frontmatter scans (cheap — YAML headers only) + 0–2 selective unit SPEC reads + codebase grep results. Within ADD's bounded read-set principle (≤ 10 reads).

---

## Workflow

Issue filing proceeds in six phases: symptom extraction, source-of-truth verification, discovery, body authoring, ID allocation and frontmatter, and validation-and-write. Phases are sequential; revisit earlier phases if a later one reveals a missing source-of-truth, an undiscovered duplicate, or a contradiction.

### Phase 1: Symptom Extraction

Read the defect intent end-to-end. Extract seven things into a working note:

- **Symptom** — what is observed. The user-facing or system-facing failure, stated tightly. "POST /repos returns 500", "CLI exit code 0 but no repo created", "web modal shows raw JSON instead of formatted error." Becomes the seed for `title:` and `Summary`.
- **Context** — where it was observed. "While running `syns init` in a fresh checkout", "during code review of u115", "in the system verification pass on 2026-05-08." Becomes the `discovered_in:` field.
- **Severity hint** — `low` | `medium` | `medium-high` | `high`. Inferred from impact:
  - `high`: corrupts user data, breaks the public contract, leaks sensitive information, complete inability to use a primary feature.
  - `medium-high`: visible UX failure on a primary path with no workaround.
  - `medium`: visible failure with a workaround, OR a public-contract drift that nobody is currently relying on.
  - `low`: polish, missed affordance, internal-only inconsistency.
  When uncertain, pick conservatively (lower) and call out the uncertainty in `severity_note`.
- **Component hint** — the code path or module suspected. May be precise (`apps/server/src/modules/repos/create.ts`) or fuzzy (`server, repo creation`). Resolve fuzzy hints by codebase grep in Phase 3 to produce a concrete path for `component:`.
- **Kind** — `bug` (something is broken that has never worked correctly) or `regression` (something that previously worked is now broken). When the prose names a recent change ("after we landed u062"), prefer `regression`. When unstated and indeterminate, default to `bug` and surface the uncertainty in `Open Questions`.
- **Reproduction steps** — when present, extract verbatim. When absent, surface in `Open Questions` ("Reproduction steps required — please add when next observed") and proceed.
- **Expected behavior** — when stated by the user, capture. Otherwise the skill derives it from the source-of-truth artifact in Phase 2 (and only from there — never from imagination).

### Phase 2: Source-of-Truth Verification

This is the load-bearing phase. The skill must verify that the defect actually contradicts a documented expectation. If the source-of-truth is silent on the defect's behavior, this is *not* a routine issue — it is either a missing-spec gap (file as roadmap refinement) or a defect in the spec itself. Either way, the skill does NOT invent expected behavior.

Procedure:

1. Map the symptom to the source-of-truth artifact using the table in Inputs §2.
2. Read the relevant section of that artifact. Confirm it specifies the expected behavior.
3. Capture the citation by stable ID — `ARTIFACT§section` or `ARTIFACT/ID` (per ADD §7.2; never quote inline prose, quotes go stale on regeneration).
4. If the source-of-truth says nothing about this behavior:
   - Surface in `Open Questions`: "Source-of-truth `<ARTIFACT>` does not specify behavior for this scenario. May be a missing-spec gap (file as a roadmap refinement?) rather than a defect."
   - Set `status: open-for-discussion` (Rule 1) — the framing is unsettled and the orchestrator should not pick this up for fix work without direction.
   - Pick severity conservatively.

The skill MUST NEVER invent expected behavior. If the source-of-truth is silent, surface that fact rather than write expected behavior from imagination.

### Phase 3: Discovery — Duplicates, Related Issues, Owning Unit

Phase 3 is mandatory. Concrete file operations:

1. **Duplicate check.** Glob `issues/*/ISSUE.md`. Read frontmatter (only) for each. If any has overlapping `component:` AND overlapping symptom (judged from `title`):
   - If the duplicate is `status: open` or `in-progress`: surface in `Open Questions` ("This may be a duplicate of `issues/<id>-<slug>`. Recommend orchestrator disambiguates.") and DO NOT silently file.
   - If the duplicate is `status: fixed` and the defect is observed in current code: this is a **regression**. Set `kind: regression`, add the closed duplicate to `related:`, and proceed.
2. **Related-issue discovery.** Among non-duplicate hits with overlapping `component:`, list the most relevant 1–3 in `related:` and in the body's `References` section. Cite by `issues/<id>-<slug>`.
3. **Owning-unit discovery.** Read frontmatter (only) of every `units/*/u*/SPEC.md`; find units whose `files:` overlaps with the suspected component. The most likely owning unit goes in `References`. If its files are clearly the root cause, name it in the body's `Root cause` section. Cite by `u<NN>`.
4. **Roadmap-overlap check.** Read frontmatter (only) of every `roadmap/*/ROADMAP.md`. If the defect's symptom corresponds to behavior in a roadmap item with `status: idea | planned | in-design | in-progress | partial`, the work hasn't shipped — this is not an issue, it is "the feature isn't built yet." Surface in `Open Questions` ("Symptom describes behavior in `roadmap/<id>-<slug>` (status: planned); may not yet qualify as a defect.") and recommend the orchestrator file as roadmap (or close as duplicate of the existing roadmap item).

Discovery output is **summarized, not exhaustive** — a tight bulleted list of relevant hits with stable-ID citations, each with a one-sentence relevance note. If discovery finds nothing: state that explicitly in `References` rather than omit the section.

### Phase 4: Body Authoring

Author the body in fixed-order required sections (`Summary` → `Reproduction` → `Expected` → `Observed` → `Root cause` → `Scope / impact` → `Suggested fix` → `Alternatives considered` → `Open Questions` → `References`) plus conditional optional sections (`Workaround in place`, `Verification`, `Re-confirmation`, `Notes`, `Tests`) only when warranted (Rule 10). Section headings are mandatory and exact — downstream skills (`/SPEC.md` G1, `/VERIFICATION.md` G7) `grep` for them. The full template with placeholders is in Output Format below.

While authoring, hold the line on Rule 1 (cite source-of-truth, do not invent expected behavior) and Rule 8 (suggested fix is a sketch, not a SPEC). The two largest temptations: writing `Expected` from imagination when the source-of-truth has no entry, and turning `Suggested fix` into a step-by-step plan. Both are downstream-skill territory paying rent in the wrong section.

### Phase 5: ID Allocation and Frontmatter

**`<NNN>` allocation.** Glob `issues/*/ISSUE.md`. Read frontmatter `id` for each. Take `max(id) + 1`. Zero-pad to 3 digits. Never reuse a retired id. Never go backwards.

**Slug computation.** Kebab-case form of the `title` field — short, descriptive, ASCII letters/digits/hyphens only, lowercase. Filler words (`a`, `an`, `the`, `for`) stripped. Examples:

- "POST /repos/{name}/push returns 200 but commit not persisted" → `push-200-commit-lost`
- "syns clone exit code 0 even when clone fails" → `clone-exit-code-zero-on-fail`
- "Role remove endpoint expects user_id but CLI sends username" → `role-remove-expects-user-id`

The folder name `<NNN>-<slug>`, the frontmatter `id`, and the slug used in citations must agree exactly.

**Frontmatter — required fields at filing time:**

```yaml
---
skill: ISSUE.md
date: <YYYY-MM-DD>
status: open | open-for-discussion

id: "<NNN>"
title: <one-line statement of what is wrong>
kind: bug | regression
severity: low | medium | medium-high | high
severity_note: <one-sentence justification of the chosen severity>
component: <code-path-or-paths>
discovered: <YYYY-MM-DD>
discovered_by: <handle>
discovered_in: <short context — session, fix-batch, code review of u<NN>, system verification pass>

promoted_to_units: []
related: []
---
```

Notes: `status` enum is **deliberately overridden** to the lifecycle values `open | in-progress | fixed | deferred | wontfix | open-for-discussion` (per ADD §7.4); the skill emits only `open` or `open-for-discussion` at filing time, and the orchestrator advances `status` thereafter via the lifecycle bridge rules (ADD §7.7.1). `id` is **quoted** to preserve zero-padding (`"044"` ≠ `44`). `severity_note` is **required** (Rule 4). `promoted_to_units` is an **empty list at filing time**. `related` is populated from Phase 3 discovery.

**Conditional fields** (`fixed_in`, `fixed_date`, `last_reconfirmed`, `last_reconfirmed_in`, `deferred_reason`, `deferred_date`, `wontfix_reason`) are either initialized as `null` at filing time or omitted entirely — be consistent within the file. The orchestrator and reconcile populate them later as the issue's lifecycle advances.

### Phase 6: Validation and Write

Before writing the file, verify:

- All required body sections (`Summary`, `Reproduction`, `Expected`, `Observed`, `Root cause`, `Scope / impact`, `References`, `Open Questions`) are present with their exact headings, in the fixed order.
- `Reproduction` is concrete (numbered steps or copy-pasteable command sequence) — not "follow these steps", "sometimes fails", "occasionally produces", "randomly" (Rule 3).
- `Expected` cites the source-of-truth by `ARTIFACT§section` or `ARTIFACT/ID` — never fabricates expected behavior (Rule 1).
- `Observed` includes verbatim evidence — error message, HTTP response body, exit code, or path to a screenshot saved under `verification-evidence/`.
- `severity_note` is present and is a sentence that justifies the chosen severity (Rule 4).
- `component:` is a concrete code path, not a feature name (Rule 5).
- No forbidden content per Rules 1, 3, 8, 10, 12 leaked anywhere.
- Frontmatter `id` and the folder name `<NNN>-<slug>` agree exactly.
- The skill ran headless — no interactive prompts surfaced during execution.

**Write.** Create the directory `issues/<NNN>-<slug>/` (and the parent `issues/` if needed). Write `ISSUE.md` inside it. If the user supplied evidence artifacts (screenshots, captured HTTP traces, log files), save them under `issues/<NNN>-<slug>/verification-evidence/` — keep evidence inside the issue's folder.

Output to stdout a one-line confirmation with the file path and key fields:

```
Filed issues/057-push-200-commit-lost/ISSUE.md (kind=bug, severity=high, component=apps/server/src/modules/repos/push.ts).
```

---

## Rules

These rules govern the output document. Violations are detected by the Quality Checklist.

### 1. Issues describe shipped behavior that contradicts a source-of-truth

Every issue must contradict a documented expectation. The `Expected` section MUST cite the source-of-truth artifact by `ARTIFACT§section` or `ARTIFACT/ID` — never fabricate expected behavior, never paraphrase prose, never guess.

If the source-of-truth is silent on the defect's behavior, this is *not* a routine issue. It is either:

- A missing-spec gap → surface in `Open Questions` and recommend filing a roadmap refinement instead.
- A bug in the spec itself → surface and recommend updating the source-of-truth (which would be a different unit's work).

Issues filed against silent source-of-truth must be flagged with `status: open-for-discussion` rather than `open`, indicating the framing is unsettled. The orchestrator does not pick up `open-for-discussion` issues for routine fix work without explicit direction.

### 2. One defect per file

If two symptoms share a root cause, file ONE issue and describe both symptoms in `Summary`. If two root causes share a symptom, file TWO issues with reciprocal `related:` entries. Splitting genuinely-coupled symptoms into separate issues fragments the audit trail; combining genuinely-distinct root causes into one issue muddles tracking and verification.

### 3. Reproduction is concrete and copy-pasteable

`Reproduction` is the test G7 runs against the fix. It must be unambiguous:

- HTTP defects → exact `curl` command (or HTTP request method + path + headers + body).
- CLI defects → exact invocation (`syns init --foo bar`).
- UI defects → numbered steps a human follows ("1. Navigate to /repos. 2. Click 'New repository'. 3. Type 'foo' in the name field. 4. Click 'Create'. 5. Observe the modal does not close.").
- State-machine defects → exact trigger sequence and the state transitions observed.

Reject vague reproduction language: "sometimes fails", "occasionally produces", "randomly", "intermittently". If the defect is genuinely intermittent, describe the conditions that affect frequency (load, time-of-day, concurrent users, dataset size) and surface the residual unknowns in `Open Questions`. Frequency vagueness is the single most common reason a fix lands and the issue silently re-opens.

### 4. Severity has a justification

`severity_note` (one sentence) is REQUIRED. It states *why* this severity. Without it, the rating is uninterrogable and triage cannot rank.

Examples:

- `severity: high`, `severity_note: "Corrupts user data: a successful POST returns 200 but the data is lost on next read."`
- `severity: medium-high`, `severity_note: "Login flow is broken with no workaround on mobile Safari; iOS users cannot sign in."`
- `severity: medium`, `severity_note: "Public-contract drift — response includes an extra field not documented; no current consumer reads it."`
- `severity: low`, `severity_note: "Cosmetic — error message wording differs from the design system's error-message library."`

When uncertainty about severity exists (e.g., possible data-corruption risk that has not been confirmed), pick conservatively (lower) and flag the uncertainty in `severity_note` and `Open Questions` simultaneously.

### 5. Component is a code path, not a feature name

`component:` names files or directories where the defect lives, not features. `apps/server/src/modules/teams` is good. `team management` is not. Multi-path is fine when a defect spans paths: `"apps/server/src/modules/teams, apps/web/src/pages/teams"`.

When the defect intent gives a fuzzy hint ("server, repo creation"), the skill grep-resolves it to a concrete path during Phase 3 before writing the field. If grep cannot resolve the hint, surface in `Open Questions` and write the closest concrete approximation (e.g., the module directory).

### 6. Discovery must be performed and reflected

Phase 3 discovery is mandatory. The skill must:

- Glob existing issues for duplicates and related entries.
- Frontmatter-scan `units/*/u*/SPEC.md` to identify the owning unit.
- Frontmatter-scan `roadmap/*/ROADMAP.md` to detect overlap with planned work.

If discovery surfaces a likely duplicate, the new issue does NOT silently file — the skill flags it in `Open Questions` with the duplicate's id and recommends the orchestrator disambiguate. Discovery findings populate `related:` and the body's `References` section; if discovery finds nothing relevant, `References` says so explicitly rather than being omitted.

### 7. Issues are filed against shipped code, not against planned work

If the defect's symptom corresponds to behavior that's still on the roadmap (not yet implemented), this is **not** an issue. The "defect" is "the feature isn't built yet" — which a roadmap item already covers, or a new roadmap item should.

Concrete check: if the suspected component is named in a roadmap item with `status: idea | planned | in-design | in-progress | partial`, the work hasn't shipped. Flag in `Open Questions` and recommend the orchestrator file as roadmap (or close as duplicate of the existing roadmap item).

### 8. Suggested fix is a sketch, not a SPEC

`Suggested fix`, when included, names specific files or contract changes — at the level of "in `apps/server/src/modules/repos/create.ts:34`, the validation regex must accept uppercase letters". It is **not** a full plan; that is `PLAN.md` territory if the fix is promoted to a unit. Keep it tight: a paragraph or a bulleted list of three-to-five concrete changes.

If the fix has multiple plausible approaches, document the rejected ones in `Alternatives considered` with one-line rationale per alternative. If only one obvious approach exists, omit the `Alternatives considered` section entirely.

If no fix is known, omit `Suggested fix` entirely. An empty section is worse than a missing one.

### 9. Citations by stable ID

Cite the source-of-truth and related artifacts by stable ID:

- `ARTIFACT§section` or `ARTIFACT/ID` for top-level design artifacts.
- `issues/<NNN>-<slug>` for related issues.
- `roadmap/<NNN>-<slug>` for related roadmap items.
- `u<NN>` for related units.
- `EP-name` for endpoints, `EVT-name` for events, `ERR_CODE` for error codes, `SM-...` / `SAGA-...` for state machines and sagas, `THREAT-NN` / `MIT-NN` for security, `INV-NN` for invariants, `UC-NN` for use cases.

Never quote artifact prose inline — quotes go stale on regeneration. Reference, do not paraphrase.

### 10. Optional sections are included only when warranted

Required sections (`Summary`, `Reproduction`, `Expected`, `Observed`, `Root cause`, `Scope / impact`, `References`, `Open Questions`) are always present. If genuinely empty (`Open Questions` resolved at filing time, for instance), state "All questions resolved." or the standard empty-state phrase.

`Root cause` is required when known; when unknown, the heading IS present with the body "Root cause undetermined — see Open Questions for investigation needed."

`Suggested fix` and `Alternatives considered` are conditional — if not applicable, omit the entire section heading.

Optional sections (`Workaround in place`, `Verification`, `Re-confirmation`, `Notes`, `Tests`) are included only when warranted. Do NOT stub them empty. The standard empty-state ("None.") is reserved for required sections.

### 11. Single YAML frontmatter block

One YAML frontmatter block at the top of the file. Never emit a second YAML block anywhere. Conditional fields (`fixed_*`, `deferred_*`, `wontfix_*`, `last_reconfirmed_*`) are either initialized as `null` at filing time or omitted entirely; the orchestrator populates them later. Be consistent within the file — do not mix the two styles.

### 12. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "some", "a few", "best practice", "industry-standard", "sometimes", "occasionally", "randomly", "intermittently" (without conditions). Use exact paths, exact codes, exact identifiers, exact response bodies. If exact is impossible because the cause is genuinely uncertain, surface it in `Open Questions` rather than hide behind a placeholder word.

### 13. No interactive questions

The skill runs headless from `claude -p`. Never pause to ask the user. If a decision has one clearly correct answer, resolve silently. If genuinely ambiguous, surface in `Open Questions` with options, tradeoffs, and a recommendation. Common ambiguities for issues:

- Suspected duplicate of an existing issue.
- Source-of-truth silent on the defect's behavior.
- Severity uncertain (write `severity_note` flagging the uncertainty and pick conservatively).
- Reproduction not yet captured ("Reproduction steps required — please add when next observed").
- Bug vs regression unclear (default to `bug`, surface in `Open Questions`).

---

## Output Format

The output is `issues/<NNN>-<slug>/ISSUE.md`. The frontmatter is identical regardless of issue kind; the body uses required sections in fixed order plus conditional and optional sections.

For two complete worked examples (a contract-mismatch bug and a CLI exit-code regression), see [`references/examples.md`](references/examples.md).

### Frontmatter

```yaml
---
skill: ISSUE.md
date: {YYYY-MM-DD}
status: {open | open-for-discussion}

id: "{NNN}"
title: {one-line statement of what is wrong}
kind: {bug | regression}
severity: {low | medium | medium-high | high}
severity_note: {one-sentence justification of why this severity}
component: {code-path-or-paths}
discovered: {YYYY-MM-DD}
discovered_by: {handle}
discovered_in: {short context — session, fix-batch, code review of u<NN>, system verification pass}

promoted_to_units: []
related: [{issues/<id>-<slug> | u<NN> | roadmap/<id>-<slug>}, ...]

# Conditional fields — initialized null at filing time; orchestrator/reconcile populates later
fixed_in: null
fixed_date: null
last_reconfirmed: null
last_reconfirmed_in: null
deferred_reason: null
deferred_date: null
wontfix_reason: null
---
```

### Body

The body template below is wrapped in a 4-backtick fence so that the inner 3-backtick block (the verbatim-evidence block in `## Observed`) renders correctly. The issue file itself uses standard 3-backtick fences.

````markdown
# ISSUE — {title}

## Summary

{One paragraph: what's wrong, what's the user-facing or system-facing effect,
 where it's rooted. Often the title plus 2–3 sentences of elaboration.}

## Reproduction

{Numbered steps or a copy-pasteable command sequence. Include exact inputs.
 For HTTP: include the curl command or the HTTP request. For CLI: include the
 exact invocation. For UI: numbered human-followable steps. For state machines:
 the exact trigger sequence and observed transitions.}

1. {Step}
2. {Step}
3. {Step}

(Never "sometimes", "occasionally", "randomly". If frequency varies, describe
 the conditions that affect it.)

## Expected

{What the source-of-truth document says should happen. Cite by stable ID; do
 not paraphrase or quote inline prose.}

Per `{ARTIFACT§section}` or `{ARTIFACT/ID}`, {one-line statement of expected
behavior}.

## Observed

{What actually happens. Verbatim error messages, HTTP response bodies, exit
 codes, or paths to screenshots. Trim large blobs to the relevant excerpts;
 large artifacts go in a sibling `verification-evidence/` directory.}

```
{verbatim error or HTTP response or stack trace}
```

## Root cause

{When known: the line, function, or design contradiction responsible. Cite
 specific code paths. Example: "In `apps/server/src/modules/repos/create.ts:34`,
 the validation regex `/^[a-z0-9-]+$/` rejects uppercase letters, but the
 rename path at `apps/server/src/modules/repos/rename.ts:18` accepts them —
 the contract is not enforced consistently."

When the root cause is a contradiction between two artifacts, name both. When
unknown, write: "Root cause undetermined — see Open Questions for investigation
needed."}

## Scope / impact

- **Affected:** {who — actor names, deployments, regions}
- **Frequency:** {one-line — "every request", "100% under sustained load", "only when X"}
- **Workaround:** {one-line — "none", "users can ...", "can be avoided by ..."}

## Suggested fix

{Conditional — Rule 8. When known: specific file paths, function names, or
 contract changes. Not a full plan — that lives in PLAN.md if the fix is
 promoted to a unit. Paragraph or tight bulleted list. Skip this entire
 section if no plausible fix is known.}

## Alternatives considered

{Conditional — when the suggested fix has plausible competitors, document the
 rejected ones and why. Skip this entire section when there is only one
 obvious approach or no suggested fix is offered.}

- **{Alternative A}** — {description, why rejected}.

## Open Questions

- [ ] {Question — e.g., "Is this a duplicate of `issues/044-...`?", "Source-of-truth silent on this scenario; should we update INTERFACES first?", "Severity uncertain — possibly higher if data-corruption risk confirmed."}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")

## References

- `{ARTIFACT§section}` — source-of-truth for the expected behavior.
- `{u<NN>}` — owning unit (likely shipped the defective code).
- `{issues/<id>-<slug>}` — related issue.
- `{URL}` — external reference (vendor docs, CVE, RFC).
````

### Optional sections (include only when warranted — Rule 10)

```markdown
## Workaround in place

{When a workaround exists in code or operations — describe it. Often added
 later, after a workaround is deployed while the root fix is being designed.}

## Verification

{Added after the issue is fixed — evidence that the fix works. Reproduction
 steps now produce expected behavior. Cite the unit that fixed it: `u<NN>`,
 date `YYYY-MM-DD`. Typically written by the orchestrator at G7 verification,
 not at filing time.}

## Re-confirmation — {event} {date}

{Added when an existing issue is reconfirmed without being fixed (the issue
 re-surfaced after a separate change, or an old issue is observed again in
 testing). The body grows; the file is not regenerated. The frontmatter
 fields `last_reconfirmed` and `last_reconfirmed_in` are also updated.}

## Notes

{Free-form context that does not fit elsewhere — historical commentary,
 related-but-tangential observations.}

## Tests

{Links to existing failing tests, or notes about tests to add as part of the
 fix. Often empty at filing; populated when the fix is implemented.}
```

---

## Scope

### In scope

- Extracting symptom, context, severity, component, kind, and reproduction from defect intent.
- Verifying the defect against the source-of-truth artifact and citing it by stable ID — never fabricating expected behavior.
- Discovery: globbing existing issues for duplicates, frontmatter-scanning units for the owning unit, frontmatter-scanning roadmap for planned-but-unshipped overlap.
- Allocating the next `<NNN>` from existing issues; computing the slug from the title.
- Creating the directory `issues/<NNN>-<slug>/` and writing `ISSUE.md` inside it.
- Authoring required body sections in the fixed order; including optional sections only when warranted.
- Populating frontmatter completely with `severity_note` justifying severity (Rule 4).
- Surfacing unresolved questions in `Open Questions` rather than guessing silently.
- Saving user-supplied evidence (screenshots, HTTP traces, log files) under `issues/<NNN>-<slug>/verification-evidence/`.
- Preserving orchestrator-managed frontmatter (`id`, `status`, `promoted_to_units`, `fixed_*`, `last_reconfirmed_*`) when refining an existing issue.

### Out of scope

- Resolving the defect — that is a unit running phase G (G1–G8) under `units/<area>/u<NN>/`.
- Filing planned work — owned by `/ROADMAP.md` (sibling skill; `roadmap/<NNN>-<slug>/ROADMAP.md`).
- Resolving open questions — owned by `/DECISION.md` (cross-cutting; `decisions/D-NNN-slug/DECISION.md`).
- Picking up the issue to fix it — owned by the orchestrator (it allocates a unit and runs G1–G8).
- Producing the SPEC for the fix — owned by `/SPEC.md` (G1; reads this issue as a trigger via `trigger.issues`).
- Producing the PLAN, IMPLEMENTATION, CODE_REVIEW, VERIFICATION, RECONCILIATION for the fix — owned by `/PLAN.md`, `/IMPLEMENTATION.md`, `/CODE_REVIEW.md`, `/VERIFICATION.md`, `/RECONCILIATION.md` respectively.
- Editing top-level design artifacts (DOMAIN, INTERFACES, DATA, BEHAVIOR, etc.) — owned by their own skills (`/DOMAIN.md`, `/INTERFACES.md`, etc.) and applied by `/RECONCILIATION.md` (G8).
- Status transitions on the issue after filing (`open` → `in-progress` → `fixed`) — performed by the orchestrator and `/RECONCILIATION.md` per ADD §7.7.1 trigger lifecycle bridge rules.
- Wire formats, database schemas, function signatures, code, step-by-step plans — owned by SPEC, INTERFACES (via reconcile), DATA (via reconcile), IMPLEMENTATION, and PLAN respectively.

---

## Quality Checklist

Before considering the issue file complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill: ISSUE.md`, `date`, `status`, `id`, `title`, `kind`, `severity`, `severity_note`, `component`, `discovered`, `discovered_by`, `discovered_in`, `promoted_to_units`, `related`)
- [ ] Single YAML frontmatter block at the top — never two
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "some", "best practice", "industry-standard", "sometimes", "occasionally", "randomly")
- [ ] Frontmatter `id` and the folder name `<NNN>-<slug>` agree exactly
- [ ] `id` is quoted in frontmatter (preserves zero-padding: `"044"` not `44`)
- [ ] `kind` is one of `bug | regression`
- [ ] `severity` is one of `low | medium | medium-high | high`
- [ ] `severity_note` is present and is a sentence justifying the chosen severity (Rule 4)
- [ ] `component` is a concrete code path (or comma-separated paths), not a feature name (Rule 5)
- [ ] `status` is `open` for newly-filed issues, or `open-for-discussion` if framing is unsettled (source-of-truth silent — Rule 1)
- [ ] `promoted_to_units` is an empty list (`[]`) at filing time — orchestrator populates later
- [ ] All required body sections (`Summary`, `Reproduction`, `Expected`, `Observed`, `Root cause`, `Scope / impact`, `References`, `Open Questions`) are present with their exact headings, in the fixed order
- [ ] `Reproduction` is concrete (numbered steps or copy-pasteable command), not vague language (Rule 3)
- [ ] `Expected` cites the source-of-truth by `ARTIFACT§section` or `ARTIFACT/ID` — never fabricates expected behavior (Rule 1)
- [ ] `Observed` includes verbatim evidence (error message, HTTP response, exit code, or screenshot path)
- [ ] `Root cause` is present with either the actual root cause (citing code paths) or the explicit "undetermined — see Open Questions" (Rule 10)
- [ ] `Scope / impact` lists affected actors, frequency, and workaround presence
- [ ] `Suggested fix` is included when a fix is known, omitted otherwise (no empty section — Rule 10)
- [ ] `Alternatives considered` is included when the suggested fix has competitors, omitted otherwise (Rule 10)
- [ ] Optional sections (`Workaround in place`, `Verification`, `Re-confirmation`, `Notes`, `Tests`) are included only when warranted; not stubbed empty
- [ ] If discovery surfaced a likely duplicate, it is flagged in `Open Questions` (not filed silently — Rule 6)
- [ ] If the source-of-truth is silent on the defect's behavior, it is flagged in `Open Questions` AND `status` is `open-for-discussion` (Rule 1)
- [ ] If the symptom describes still-planned roadmap work, it is flagged in `Open Questions` (Rule 7)
- [ ] Citations use stable IDs (`ARTIFACT§section`, `EP-name`, `EVT-name`, `ERR_CODE`, `SM-name`, `SAGA-name`, `INV-NN`, `UC-NN`, `u<NN>`, `issues/<id>-<slug>`, `roadmap/<id>-<slug>`) — never quoted artifact prose (Rule 9)
- [ ] No interactive questions surfaced during execution — the skill ran headless from start to finish
- [ ] User-supplied evidence (screenshots, HTTP traces, log files) is saved under `issues/<NNN>-<slug>/verification-evidence/`
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] Document length is 80–400 lines; overflow indicates two-issues-in-one (split per Rule 2) or speculation creeping in (trim)
