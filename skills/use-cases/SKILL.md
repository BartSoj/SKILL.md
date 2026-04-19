---
name: USE_CASES.md
description: Enumerate every user-perspective capability of the system — actors, use cases with stable IDs, main and alternative flows, priorities, cross-cutting scenarios, and the surface mapping matrix — as the second artifact in an Agent-Driven Development project. Use when asked to create the use case catalogue, enumerate use cases, write actor flows, define user scenarios, or produce a USE_CASES.md.
---

# Task: Generate USE_CASES.md — Authoritative User-Perspective Capability Catalogue

## Objective

Produce a USE_CASES.md that enumerates every use case an actor can perform in the system, grouped by actor and priority, plus the cross-cutting end-to-end scenarios that weave multiple use cases together. The document answers five questions: who uses the system, what each actor needs to do, which stories span multiple capabilities, how use cases map to surfaces (web, CLI, mobile, TUI, voice), and which are in v1 versus deferred. Every downstream surface IA skill, `/domain`, `/architecture`, `/WORK_UNITS.md`, and `/system-verify` reads this file to derive its own inventory — orphan use cases and undeclared surfaces cause cascading modelling errors later.

USE_CASES.md is produce-once and read-many. Stable `UC-NN` IDs assigned here are cited everywhere: entities and aggregates in DOMAIN.md trace back to use cases, per-surface IAs filter the surface matrix to generate their pages/commands/screens, and `/system-verify` derives its scenario list from § 3 Cross-Cutting Scenarios. Treat every `UC-NN` ID as permanent — never renumber, never silently drop.

---

## Inputs

1. **PROPOSAL.md** (required) — § 6 Target Users / Actors seeds the actor catalogue; § 1–§ 2 problem and thesis seed use case discovery; § 3 Core Principles constrain what actors can and cannot do; § 4 Non-Goals bound the capability space; § 7 Scope Boundaries anchors the v1 / v2 / deferred split; § 9 Glossary Stub seeds the use-case vocabulary.
2. **Reference material** (optional) — user interviews, competitor features, existing product screenshots, internal memos. Use to ground flows in realistic actor behaviour; cite by short name when a use case is drawn from a specific source.
3. **Existing USE_CASES.md** (auto-discovered — only if refining) — if a USE_CASES.md already exists at the expected path, read it fully. Every assigned `UC-NN` is permanent. New use cases take the next unused number. Deleted use cases leave their ID marked `(retired — superseded by UC-NN)` or `(retired — out of scope)`; never reassign a retired number.

Read set size: 1 required + up to 2 optional. The read count at this step is 1 prior ADD artifact (PROPOSAL.md). Read the proposal end-to-end — every actor, principle, non-goal, and scope boundary materially shapes the catalogue.

---

## Workflow

Use case enumeration proceeds in seven phases: actor catalogue, per-actor use case enumeration, cross-cutting scenarios, priority assignment, surface mapping, glossary additions, and validation. Phases are sequential; revisit earlier phases if a later one reveals a missing actor, a missed capability, or a contradiction.

### Phase 1: Actor Catalogue

Lift the actor list from PROPOSAL.md § 6. Every actor named in the proposal appears in § 1. Inspect the proposal's problem statement, thesis, and scope boundaries for *implicit* actors the enumeration missed — a system that "accepts authenticated pushes" implies at least `Collaborator` (the pusher) and likely `Operator` (who granted access) even if only one was listed explicitly. Add the missing actors with a note explaining why.

For each actor: assign a short stable name (`Operator`, `Collaborator`, `Viewer`, `Anonymous`) used consistently throughout the document. Write a 2–4 sentence role description in the actor's voice — what they care about, what problems they face, what authority they wield. State the scale expectation concretely (`~1 per deployment`, `10–1000 per server`, `unbounded public traffic`). State authority at the capability level (`can create and administer repositories`), not the permission level (`has write access to /api/repos`).

If the actor list exceeds seven, consolidate — two "actor classes" with identical authority and scale are the same actor. If it is below three, re-examine whether you have modelled roles separately from scale (an administrative operator and a daily user of the same installation are usually two actors, not one).

### Phase 2: Per-Actor Use Case Enumeration

For each actor in § 1, enumerate every capability they can perform. A use case is a discrete unit of value — one trigger, one actor, one postcondition. Prefer many small use cases to few large ones; atomic use cases compose better into cross-cutting scenarios and surface mappings.

For each use case write the full block: `UC-NN` ID, verb-phrase title, actor, priority placeholder (filled in Phase 4), precondition, trigger, main flow (3–10 numbered steps), alternative flows, postcondition, success criteria, surfaces (filled in Phase 5), notes.

**Main flow rules:**

- Each numbered step is either an actor action (`1. The Operator requests to invite a new Collaborator.`) or a system response (`2. The system records the invitation and notifies the invitee.`). Alternate between the two; consecutive actor steps or system steps usually indicate a missing step in between.
- Use domain language — the glossary in § 6 and in PROPOSAL § 9. Never name a service, endpoint, command, page, or database table. "The Operator pushes a change" — not "the CLI sends `POST /repos/:id/push`" and not "the user clicks the Push button".
- Keep each step atomic. If a step reads "The system validates, stores, notifies, and reports", split it — one verb per step.

**Alternative flows:**

Each alternative flow states `when {condition}, instead {deviation}` — never "sometimes X" or "if things go wrong X". Name the condition precisely (`when the Collaborator is already a member`, `when the repository quota is exhausted`) and describe the deviation concretely. Error paths belong here; the error *taxonomy* (codes, categories) is owned by `/errors` and will reference the condition names set here.

**Success criteria:**

One to three testable assertions. Pass/fail must be mechanically decidable by a human observing the system or by an automated test. "The invitee appears in the repository's Collaborator list" passes. "The invitee is happy" fails. "The operation completes quickly" fails — quantify or drop.

**Single-actor rule:** exactly one actor per use case. If two actors interact, model it as two use cases (one for each actor's perspective) plus a cross-cutting scenario in § 3 that composes them. Multi-actor use cases conflate responsibilities and break surface mapping.

### Phase 3: Cross-Cutting Scenarios

Some stories span multiple use cases — they are the system's end-to-end value narratives and the primary source for `/system-verify`'s scenario list. For each: `SCN-NN` ID, title, participating actors (often multiple), the ordered list of composed `UC-NN`s, a one-to-two paragraph narrative walking through the story, and one line on *why* it is cross-cutting (which integration property it exercises).

Scenarios are not a second use case catalogue. Do not invent per-scenario flows or per-scenario success criteria — those are owned by the composed use cases. The scenario narrative integrates them into a single arc. Three to ten scenarios typical; more than fifteen is a sign you are restating use cases instead of weaving them.

Every scenario must reference at least two composed `UC-NN`s. A "scenario" that composes one use case is just that use case.

### Phase 4: Priority Assignment

For every use case in § 2, assign exactly one priority: `v1`, `v2`, or `deferred`. Derive from PROPOSAL § 7 Scope Boundaries — capabilities listed "In first milestone" land as `v1`; capabilities "Not in first milestone" land as `v2` or `deferred` depending on whether they have a planned later milestone. Use case priorities must be consistent with the proposal's scope boundaries — a use case marked `v1` whose underlying capability is listed "Not in first milestone" is a contradiction to flag.

"TBD" is not a priority. "Maybe v1" is not a priority. If the inputs are genuinely silent on priority for a use case, place it in § 7 Open Questions with options and a recommendation — do not leave it unset.

Aggregate into § 4 Priority & Milestones: three bulleted lists (`v1 scope`, `v2 scope`, `Deferred / won't-do`) citing each `UC-NN` by ID. Deferred entries carry a one-line reason.

### Phase 5: Surface Mapping Matrix

For each use case, decide on which surfaces it is realised. The surface universe is fixed: `WEB`, `CLI`, `MOBILE`, `TUI`, `VOICE`. Include only the surfaces the project actually builds — if PROPOSAL scoped only WEB and CLI, the matrix has two columns.

Cell values:

- `primary` — the main way the use case is realised on this surface. At least one surface must be `primary` for every use case.
- `secondary` — an alternative realisation available on this surface.
- `—` — not realised on this surface.

Every cell must be one of those three values; blanks are forbidden. Every use case has exactly one `primary` or more (some use cases are genuinely realised equivalently on multiple surfaces; that is permitted). A use case with zero `primary` surfaces is an orphan — either the use case is not needed, or a surface is missing from scope.

Place the matrix in § 5 as a single markdown table. Row labels are `UC-NN` plus the use case title; column labels are the surface names.

### Phase 6: Glossary of Use-Case-Specific Terms

§ 6 carries any term used in §§ 1–5 that is not already in PROPOSAL § 9 Glossary Stub. These are preview entries — the authoritative glossary is DOMAIN.md. Typical entries: collaboration terms that emerge during use case enumeration (`Invitation`, `Pending Collaborator`), lifecycle states (`Active`, `Suspended`, `Pending`), and relationship nouns (`Fork`, `Upstream Tracking`). Each entry: term, one-line definition, category (actor-role, capability, state, relationship, external).

Mark § 6 with the preview note: *(Authoritative glossary: DOMAIN.md. The entries here are a preview of use-case-specific vocabulary.)* Terms already in PROPOSAL § 9 are not repeated here.

### Phase 7: Validation & Finalization

Verify the document holds together before finalising:

- Every actor in § 1 has at least one use case in § 2. An actor with zero use cases is either a modelling error or an observer role that belongs in § 1 with a note rather than a catalogued actor.
- Every `UC-NN` referenced from § 3 Cross-Cutting Scenarios, § 4 Priority & Milestones, or § 5 Surface Mapping Matrix appears in § 2 with a full block.
- Every use case in § 2 has exactly one priority in § 4 and at least one `primary` surface in § 5.
- Every use case main flow has 3–10 numbered steps. Fewer than three indicates an under-specified flow; more than ten indicates a use case that should be split.
- No architecture leaks: grep the document for `API`, `endpoint`, `HTTP`, `JSON`, `database`, `table`, `column`, `service`, `queue`, `SDK`. Rephrase or flag violations.
- No UI leaks: grep for `button`, `click`, `tap`, `page`, `screen`, `tab`, `modal`. Use cases describe actor *intent*, not interaction mechanics — those are IA skills' job.
- Every cross-cutting scenario references ≥ 2 composed `UC-NN`s.
- Every term used in §§ 1–5 that is not everyday English appears in § 6 or in PROPOSAL § 9.
- Every open question has options, tradeoffs, and a recommendation.

Update frontmatter counts (`use_cases_total`, `use_cases_v1`, `use_cases_deferred`, `actors`, `cross_cutting_scenarios`, `open_questions`) to match the body exactly. Set `status` to `complete` if § 7 is "All questions resolved." and `has_open_questions` otherwise.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Stable `UC-NN` IDs, forever

`UC-NN` IDs are assigned once at creation and never renumbered, never reused, never deleted-without-trace. A retired use case leaves its ID in § 2 marked `(retired — superseded by UC-NN)` or `(retired — out of scope)` so downstream references still resolve.

### 2. User perspective, not implementation

Flows describe what an actor does and what the system responds, in domain terms. Forbidden in flow steps: service names (`auth-service`, `api-gateway`), endpoint names (`POST /api/v1/...`), wire formats (`JSON response`, `gRPC call`), storage shapes (`row in the users table`, `Redis cache lookup`), UI mechanics (`click the submit button`, `the modal appears`). Domain verbs only: "the Collaborator pushes", "the system accepts", "the invitation expires".

### 3. Numbered main flow, atomic steps

Every main flow uses numbered steps. Each step is either one actor action or one system response — never a composite ("validates, stores, and notifies"). Count is 3–10 per use case; fewer indicates under-specification, more indicates a use case that should be split.

### 4. One actor per use case

Exactly one actor per use case. Multi-actor stories live in § 3 Cross-Cutting Scenarios as compositions of single-actor use cases. A use case with two actors in the **Actor** field is a modelling error.

### 5. Alternative flows describe condition → deviation

Each alternative flow is phrased `when {precise condition}, instead {concrete deviation}`. Forbidden: "sometimes", "if things go wrong", "in case of an error". Condition names established here will be referenced by `/errors` — precision propagates.

### 6. Testable success criteria

Every success criterion is either a mechanically observable outcome ("the invitee appears in the repository's Collaborator list") or a pass/fail scenario ("running `repo list` after the push includes the pushed repository"). Reject adjectives: "fast", "easy", "smooth", "intuitive" — rewrite with a target or drop.

### 7. Explicit priority — no `TBD`

Every use case has exactly one priority: `v1`, `v2`, or `deferred`. `TBD`, `maybe`, and blanks are forbidden. Priority decisions silent in the inputs surface in § 7 Open Questions — the body of § 2 and § 4 never carries a placeholder priority.

### 8. No orphan use cases

Every use case must be mapped to at least one `primary` surface in § 5. An orphan use case — zero `primary` — is either out of scope (move to `deferred` in § 4 and mark the matrix row with `—` across) or indicates a missing surface in the project's scope. Either resolution is fine; silent orphans are not.

### 9. Surface matrix completeness

Every cell in § 5's matrix has a value — `primary`, `secondary`, or `—`. Blanks are forbidden. The matrix is a contract read by every IA skill; ambiguity cascades.

### 10. Cross-cutting scenarios compose ≥ 2 use cases

Every `SCN-NN` references at least two composed `UC-NN`s in sequence. A scenario composing one use case is just that use case — promote it back to § 2 or drop it.

### 11. Glossary is preview, not authoritative

§ 6 carries the preview note: *(Authoritative glossary: DOMAIN.md. The entries here are a preview of use-case-specific vocabulary.)* Terms already in PROPOSAL § 9 are not repeated. The full glossary is `/domain`'s deliverable.

### 12. Single YAML frontmatter block

One YAML frontmatter block at the top containing common fields (`skill`, `date`, `status`) and use-case-specific count fields. Never emit a second YAML block anywhere. Counts must match the body — `use_cases_total` equals the number of `UC-NN` entries in § 2, `use_cases_v1` equals the count of `v1` entries in § 4, etc.

### 13. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "and so on", "various", "many". Use exact actor names, exact use case titles, exact surface labels. Unresolvable ambiguity surfaces in § 7 Open Questions.

---

## Output Format

```markdown
---
skill: USE_CASES.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
use_cases_total: {N}
use_cases_v1: {N}
use_cases_v2: {N}
use_cases_deferred: {N}
actors: {N}
cross_cutting_scenarios: {N}
open_questions: {N}
---

# USE_CASES — {ProductName}

> Authoritative catalogue of user-perspective capabilities. Every `UC-NN` ID is
> stable and cited downstream: DOMAIN.md traces entities back to use cases,
> surface IA skills derive their inventories from § 5, and `/system-verify`
> derives its scenario list from § 3.

## § 1. Actors

### `{ActorName}`

- **Role:** {2–4 sentences — who this actor is, what they care about, what authority they wield in domain terms}
- **Scale:** {concrete expectation — "~1 per deployment", "10–1000 per server", "unbounded public traffic"}
- **Authority:** {capability-level statement — "can create and administer repositories, manage Collaborators, and configure TLS"; never permission-level}

(Repeat for each actor. Three to seven typical. Every actor here must have
 at least one use case in § 2.)

---

## § 2. Use Case Catalogue

### `{ActorName}`

#### UC-{NN}: {short verb-phrase title}

- **Actor:** `{ActorName}`
- **Priority:** {v1 | v2 | deferred}
- **Precondition:** {what must be true for the flow to be attempted — "the Operator has an authenticated session and owns the target repository"}
- **Trigger:** {what initiates the flow — "the Operator initiates an invitation for a named email address"}
- **Main flow:**
  1. {actor action — "The Operator requests to invite a new Collaborator by email."}
  2. {system response — "The system records the pending invitation and sends a notification to the invitee."}
  3. {actor action or system response}
  (3–10 numbered steps total.)
- **Alternative flows:**
  - {when `{precise condition}`, instead `{concrete deviation}` — "when the invitee is already a Collaborator, instead the system rejects with a duplicate-member signal and no pending invitation is recorded"}
- **Postcondition:** {what is true after successful completion — "a pending Invitation exists for the invitee, linked to the repository"}
- **Success criteria:**
  - {testable assertion — "the invitee appears in the repository's Invitation list with status `pending`"}
- **Surfaces:** {WEB | CLI | MOBILE | TUI | VOICE, subset — filled consistent with § 5}
- **Notes:** {optional — constraints, cross-references to `INV-NN` once DOMAIN is written, references to prior art}

(Repeat for every use case, grouped by owning actor. Retired use cases keep
 their ID with the marker `(retired — superseded by UC-NN)` or `(retired —
 out of scope)`.)

---

## § 3. Cross-Cutting Scenarios

### SCN-{NN}: {short title}

- **Participating actors:** `{Actor1}`, `{Actor2}`
- **Composed use cases:** `UC-{NN}` → `UC-{NN}` → `UC-{NN}`
- **Narrative:** {1–2 paragraphs walking through the story in domain terms,
  referencing composed use cases at each beat}
- **Why it's cross-cutting:** {one line — which integration property this
  exercises, e.g., "tests that invitation acceptance survives an Operator-
  initiated repository rename mid-flow"}

(Repeat for each scenario. Three to ten typical. Every scenario composes
 ≥ 2 use cases. If none: "No cross-cutting scenarios identified at this stage.")

---

## § 4. Priority & Milestones

### v1 scope

- `UC-{NN}` — {short title}

(Repeat for every v1 use case.)

### v2 scope

- `UC-{NN}` — {short title}

(Repeat for every v2 use case. If none: "No v2 use cases planned.")

### Deferred / won't-do

- `UC-{NN}` — {short title} — {one-line reason}

(Repeat for every deferred use case. If none: "Nothing deferred.")

---

## § 5. Surface Mapping Matrix

| Use case | {SURFACE_1} | {SURFACE_2} | ... |
|----------|-------------|-------------|-----|
| `UC-01` — {short title} | primary | secondary | — |

(Every row is a use case; every column is a surface in project scope. Every
 cell is one of `primary`, `secondary`, `—`. Every use case has ≥ 1 `primary`.
 Column set is fixed by PROPOSAL's surface scope.)

---

## § 6. Glossary of Use-Case-Specific Terms

*(Authoritative glossary: DOMAIN.md. The entries here are a preview of use-case-specific vocabulary.)*

| Term | Definition | Category |
|------|-----------|----------|
| `{Term}` | {one-line, domain-level definition} | actor-role / capability / state / relationship / external |

(Terms used in §§ 1–5 that are not already in PROPOSAL § 9. If none: "No
 additional terms — all vocabulary is covered by PROPOSAL § 9 Glossary Stub.")

---

## § 7. Open Questions

- [ ] {Question — e.g., "Can a Collaborator invite further Collaborators, or is invitation restricted to the Operator? UC-07 and UC-12 both touch invitation but the proposal is silent."}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Actor catalogue with stable names, role descriptions, scale, and authority
- Use case enumeration with stable `UC-NN` IDs, numbered main flows, alternative flows, pre/postconditions, and testable success criteria
- Cross-cutting scenarios composing multiple use cases, with narrative and participating actors
- Priority assignment (`v1` / `v2` / `deferred`) consistent with PROPOSAL § 7
- Surface mapping matrix declaring `primary` / `secondary` / `—` per (use case, surface) cell
- Use-case-specific glossary preview (authoritative glossary lives in DOMAIN.md)
- Genuinely ambiguous decisions surfaced in § 7 Open Questions

### Out of scope

- Full glossary, entities, aggregates, value objects, invariants, bounded contexts, domain events — owned by `/domain`
- System architecture, components, technology choices, ADRs — owned by `/architecture`
- Wire formats, endpoint shapes, event schemas — owned by `/interfaces`
- Error taxonomy and codes — owned by `/errors` (use cases name *conditions*; codes are assigned in `/errors`)
- Persistence schema, indexes, access patterns — owned by `/data`
- State machines, sagas, lifecycles — owned by `/behavior`
- Observability, SLOs, performance budgets — owned by `/quality`
- Threat modelling, security controls — owned by `/security`
- Deployment, runbooks, config catalogue — owned by `/operations`
- Per-surface inventories (pages, commands, screens, views, intents) — owned by IA skills (`/web-ia`, `/cli-ia`, `/mobile-ia`, `/tui-ia`, `/voice-ia`), derived by filtering § 5's surface column
- Per-unit specs, plans, implementations — owned by `/WORK_UNITS.md` and the per-unit pipeline

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `use_cases_total`, `use_cases_v1`, `use_cases_v2`, `use_cases_deferred`, `actors`, `cross_cutting_scenarios`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "and so on", "TBD", "maybe")
- [ ] § 7 Open Questions is present (empty with "All questions resolved." or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] All seven sections § 1 through § 7 are present with their exact headings
- [ ] Every use case has a unique `UC-NN` ID; no renumbering; retired IDs retained with `(retired — ...)` marker
- [ ] Every actor in § 1 has at least one use case in § 2
- [ ] Every use case has exactly one actor named in the `Actor` field (multi-actor stories are in § 3)
- [ ] Every main flow uses numbered steps, 3–10 per use case
- [ ] Every alternative flow is phrased `when {condition}, instead {deviation}` with a concrete, named condition
- [ ] Every success criterion is mechanically observable or a pass/fail scenario — no un-quantified adjectives
- [ ] Every use case has exactly one priority: `v1`, `v2`, or `deferred` (no `TBD`)
- [ ] Every use case has ≥ 1 `primary` surface in § 5 (no orphan use cases)
- [ ] § 5 surface mapping matrix has a value in every cell (`primary`, `secondary`, or `—`) — no blanks
- [ ] Every cross-cutting scenario in § 3 references ≥ 2 composed `UC-NN`s in sequence
- [ ] Every `UC-NN` referenced in §§ 3–5 appears as a full block in § 2
- [ ] No architecture leaks in use case bodies (grep-check: `API`, `endpoint`, `HTTP`, `JSON`, `database`, `table`, `column`, `service`, `queue`, `SDK`)
- [ ] No UI leaks in use case bodies (grep-check: `button`, `click`, `tap`, `page`, `screen`, `tab`, `modal`, `navigate`)
- [ ] § 6 Glossary carries the preview note *(Authoritative glossary: DOMAIN.md. The entries here are a preview of use-case-specific vocabulary.)*; terms already in PROPOSAL § 9 are not repeated
- [ ] Priority assignment in § 4 is consistent with PROPOSAL § 7 Scope Boundaries (v1 use cases map to in-milestone capabilities; deferred use cases map to excluded or later-milestone capabilities)
- [ ] Frontmatter counts (`use_cases_total`, `use_cases_v1`, `use_cases_v2`, `use_cases_deferred`, `actors`, `cross_cutting_scenarios`, `open_questions`) match the body exactly
- [ ] `status` is `complete` if § 7 is "All questions resolved." and `has_open_questions` otherwise
- [ ] Document length is within the 500–1500 line target (hard cap 2000); overflow indicates use cases that should be split or flows that drifted into implementation detail
