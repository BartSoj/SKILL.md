---
name: PROPOSAL.md
description: Crystallize the product vision — problem, solution thesis, principles, non-goals, success criteria, target actors, scope boundaries, risks, and a glossary stub — as the first artifact in an Agent-Driven Development project. Use when asked to draft a proposal, write the proposal, start a new ADD project, capture the product vision, or produce a PROPOSAL.md.
---

# Task: Generate PROPOSAL.md — Vision Layer for a New ADD Project

## Objective

Produce a PROPOSAL.md that anchors an Agent-Driven Development project at the vision layer. The document states the problem in the user's domain language, the thesis of the solution, the principles that will constrain every later decision, the non-goals that prevent scope creep, the testable success criteria, the target actors, the first-milestone scope boundaries, the risks and assumptions, a small glossary stub, and any genuinely unresolved questions. An agent reading this document alone can set the direction for downstream artifacts — `/use-cases` derives its actors and scenarios from it, `/domain` draws its initial glossary from it, `/architecture` inherits its principles and non-goals — without needing to re-ask the user what the project is about or why it matters.

PROPOSAL.md is the first file written in an ADD project and has no prior artifacts to read. It is produce-once and read-many: every later phase cites it for grounding. A weak proposal produces weak downstream artifacts; treat every section as a commitment that propagates.

---

## Inputs

1. **User intent prompt** (required) — the free-form description of what the user wants built. May be a single sentence or a multi-paragraph brief. This is the single primary input; everything in the output traces back to it.
2. **Reference material** (optional) — prior art the user points at (competitor docs, research notes, related projects, internal memos). Read it to extract decision-shaping facts; do not inline large excerpts. Cite by short name where a principle or non-goal is drawn from a specific source.
3. **Existing PROPOSAL.md** (auto-discovered — only if refining) — if a PROPOSAL.md already exists at the expected path, read it fully and treat every decision as authoritative unless the new prompt explicitly contradicts it. Preserve prior principles, non-goals, and success criteria when they remain valid; merge new information; never silently renumber or drop existing commitments.

Read-set size: 1 required + up to 2 optional. The read count at this step is 0 prior ADD artifacts — the proposal is the seed. Read the user prompt word-for-word; subtle hints about users, scale, or constraints are often the deciding input for a section.

---

## Workflow

Proposal generation proceeds in six phases: intent extraction, problem-and-thesis draft, principles-and-non-goals, success-criteria-and-actors, scope-risks-glossary, and validation. Phases are sequential — each phase feeds the next — but revisit earlier phases if a later one reveals a gap or contradiction.

### Phase 1: Intent Extraction

Read the user prompt and any reference material end-to-end. Extract four things into a working list:

- **Nouns the user uses for the thing being built** — the candidate product name and the candidate names for core concepts (e.g., `Repository`, `Push`, `Collaborator`; `Order`, `Cart`, `Line Item`). These seed § 9 Glossary and hint at actors.
- **Pain statements** — every phrase where the user describes something as broken, slow, awkward, missing, or frustrating. These seed § 1 Problem Statement.
- **Belief statements** — every phrase where the user says "we think", "the key insight is", "it should", "always", "never". These seed § 2 Solution Thesis and § 3 Core Principles.
- **Exclusion statements** — every phrase where the user says "not", "won't", "out of scope", "later", "not in v1". These seed § 4 Non-Goals.

If any of the four categories is empty, do not invent content — note the gap as an open question in Phase 6.

Pick the product name: prefer the name the user uses verbatim; if absent, use a bracketed placeholder like `{ProductName}` throughout and list "confirm product name" as an open question.

### Phase 2: Problem Statement & Solution Thesis

Draft § 1 and § 2 in that order. § 1 must precede § 2 so the thesis answers a named problem.

**§ 1 Problem Statement.** Two to five paragraphs in the user's domain language. Answer: what is broken today, for whom, why existing alternatives do not suffice. Lead with a concrete scenario where possible — "today, a team that wants to collaborate on a shared git repository behind their firewall must choose between standing up their own GitLab, paying per-seat for a hosted product, or scripting raw git over SSH" — rather than abstractions ("current solutions have gaps"). Name the existing alternatives explicitly and say what each one gets wrong.

Keep the problem statement solution-free. The moment the prose says "we need X" or "an API that does Y", stop and rephrase in pure problem language. The thesis in § 2 is where solution language begins.

**§ 2 Solution Thesis.** Two to four paragraphs. Start with a single sentence answering "what we will build" in domain terms. Follow with elaboration of the core insight that makes the solution work — what does the solution *know* or *do differently* that the alternatives do not? A thesis is not a feature list. "We will build a content-addressable storage layer that never re-transmits bytes it has seen before" is a thesis. "We will build a CLI with `push`, `pull`, `clone`, and `init` subcommands" is a feature list.

The thesis must be defensible. A reader should be able to ask "why will this work when the alternatives did not?" and the paragraphs must answer. If the answer is missing, the thesis is weak — go back and strengthen it before writing principles.

### Phase 3: Core Principles & Non-Goals

Draft § 3 and § 4 together. They are complementary — principles commit to a shape; non-goals commit to an absence.

**§ 3 Core Principles.** Three to seven bullets. Each principle is a constraint on every later decision, followed by a one-line rationale. Test each candidate principle with the falsification question: "if we deleted this principle, would we build anything differently?" If the answer is no, it is a platitude — drop it. Real principles read like: "server owns all business logic, clients are dumb — clients can be rewritten or replaced without touching invariants"; "content-addressable protocol, never re-transmit known bytes — the protocol scales with change volume, not data volume"; "every write is atomic — partial states are never observable".

Principles are scarce on purpose. Seven is a soft upper bound; more than seven and the list loses force. If the product has a tenth candidate principle, fold similar ones together or demote one to an ADR that comes later in `/architecture`.

**§ 4 Non-Goals.** Bulleted list. Each entry pairs a realistic alternative scope with a one-line reason it is excluded, in the pattern `{capability} — {why not}`. Non-goals must be plausibly expectable — "we will not build a rocket ship" is useless; "we will not provide fine-grained per-branch access control in v1 — the model is repository-level and adding per-branch roles is a separate design effort" is useful. Aim for at least as many non-goals as principles; well-chosen non-goals save more downstream time than any other section.

If the user did not state exclusions explicitly, derive non-goals from the space of features a reader might reasonably expect given § 1 and § 2 — and cut them, with reason.

### Phase 4: Success Criteria & Target Actors

Draft § 5 and § 6.

**§ 5 Success Criteria.** Three to seven bullets, each measurable or testable. A success criterion is one of two shapes:

- **Metric with a target** — "a new user can install the CLI and complete their first push in ≤ 5 minutes on a typical home broadband connection"; "the server sustains 1000 concurrent `fetch` operations on a 4-core VM with p99 latency < 2s"; "adding a new backend plugin requires no changes to `crates/core`".
- **Scenario with a pass/fail outcome** — "a team operator can self-host the service on a single VM with a single binary and a single config file; success = the `getting-started` guide works end-to-end on a clean Ubuntu LTS VM".

Reject vague candidates. "Easy to use" is not a criterion — "a first-time user completes task X in ≤ N minutes without reading docs beyond `--help`" is. If a criterion cannot be mechanically verified, rewrite it or drop it.

Criteria must be at the vision level, not at a feature level. "The `push` command supports `--force`" is a feature, not a success criterion.

**§ 6 Target Users / Actors.** One paragraph per actor class. Name each actor with a short capitalised singular noun (`Operator`, `Collaborator`, `Contributor`, `Viewer`) and a one-line role description. Three to seven actors typical. Full actor modelling — scenarios, permissions, relationships — is owned by `/use-cases`; this section is the enumeration that `/use-cases` will expand.

Every actor here should feel like a stakeholder in § 1 Problem Statement. If a named actor does not map back to someone harmed by the current problem or served by the thesis, re-examine whether they belong.

### Phase 5: Scope Boundaries, Risks & Glossary Stub

Draft § 7, § 8, and § 9.

**§ 7 Scope Boundaries.** What is and is not in the first milestone, phrased as capabilities (not features). "In scope: accepting authenticated pushes, fetching public repositories over HTTPS, enumerating references. Out of scope: branch protection rules, pull requests, issue tracking." This anchors downstream scoping decisions; `/use-cases` will expand the in-scope list into concrete use cases and `/WORK_UNITS.md` will budget them into work units.

Scope boundaries are not non-goals. Non-goals (§ 4) are things we will never build or will deliberately not build. Scope boundaries are things we will build eventually but are out of the first milestone.

**§ 8 Risks & Open Assumptions.** Bulleted. Each risk: one-line description, impact severity (L / M / H), one-line mitigation or reason-to-accept. Each assumption: one-line description of what we are taking for granted, one-line consequence if wrong. Cover technical risk ("the chosen sync algorithm assumes a stable clock across peers — if clock skew exceeds 30s, conflict resolution degrades to last-write-wins"), market risk, and organizational risk where applicable.

**§ 9 Glossary Stub.** 5–15 project-specific terms with one-line definitions. Seed the vocabulary a reader needs to understand § 1–§ 7 without guessing. Prefer domain terms over technology terms — `Push` and `Repository`, not `HTTP endpoint` and `JSON body`. Mark the section with the preview note: *(Authoritative glossary: DOMAIN.md. The entries here are a preview.)* so readers know this is not the final source of truth.

### Phase 6: Validation & Finalization

Verify the document holds together before finalising:

- Every term used in § 1–§ 8 that is not everyday English appears in § 9 Glossary Stub.
- Every principle in § 3 has a one-line rationale and materially constrains at least one decision a reader can imagine being made later.
- Every non-goal in § 4 names a realistic alternative someone might expect, not a strawman. The count of non-goals is at least the count of principles.
- Every success criterion in § 5 is measurable (metric + target) or testable (scenario + pass/fail). No "easy to use", "fast", "delightful", "robust" left un-quantified.
- Every actor in § 6 has a role description and is recognisable as a stakeholder in § 1.
- § 1 Problem Statement contains no solution language. § 2 onward may.
- No component, technology, or wire-format decisions leaked in. "We will use PostgreSQL" belongs in `/architecture`, not here. "Responses are JSON" belongs in `/interfaces`, not here. If one leaked, demote it to an open question or drop it.
- § 9 Glossary Stub has between 5 and 15 entries and carries the *(Authoritative glossary: DOMAIN.md. The entries here are a preview.)* preview note.
- Every decision where multiple defensible answers existed and the inputs were silent is listed in § 10 Open Questions with options, tradeoffs, and a recommendation. Every other decision was resolved silently.

Update frontmatter counts (`principles_count`, `non_goals_count`, `success_criteria_count`, `open_questions`) to reflect the final document. Set `status` to `complete` if § 10 is "All questions resolved." and `has_open_questions` otherwise.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Domain language in the problem statement

§ 1 Problem Statement must describe the problem in the user's domain language only. Forbidden vocabulary in § 1: `API`, `endpoint`, `database`, `service`, `microservice`, `frontend`, `backend`, `queue`, `container`, `cluster`, `JSON`, `HTTP`, `gRPC`, `SDK`, `library`. The problem is "a team cannot collaborate on a shared repository", not "there is no REST API for git". Solution-shaped vocabulary is allowed starting in § 2 Solution Thesis.

### 2. Principles must be falsifiable constraints

Every bullet in § 3 must pass the falsification test: "if we deleted this principle, would we build anything differently?" If the honest answer is no, the bullet is a platitude — cut it. "Ship quality code" is a platitude; "server owns all business logic, clients are dumb" is a principle because it forbids client-side business rules.

### 3. Non-goals are realistic alternatives, not strawmen

Every bullet in § 4 must name a capability someone might reasonably expect given § 1 and § 2. Strawman non-goals add no value. "We will not build a rocket ship" when building a git server is a strawman. "We will not build fine-grained per-branch access control in v1" is realistic. The strawman test: would removing this non-goal cause a plausible reader to ask "wait, will it do X?" — if no, drop it.

### 4. Success criteria are measurable or testable

Every bullet in § 5 is either a metric with a target (duration, throughput, latency, count, ratio, time-to-first-X) or a scenario with a pass/fail outcome (a named journey a reader can run through and report complete / incomplete). Reject qualitative adjectives without numbers: "fast", "simple", "easy", "robust", "scalable" — rewrite with a target or drop. A rule of thumb: a criterion you cannot write a unit test or a stopwatch test for is not a criterion.

### 5. No architecture, no components, no technology

The proposal commits to *what* and *why*, not *how*. Forbidden anywhere outside § 9 Glossary Stub:

- Component names (`auth-service`, `worker`, `gateway`)
- Technology choices (PostgreSQL, Kafka, React, Redis, Kubernetes, a specific cloud provider)
- Wire formats (JSON, Protobuf, GraphQL, REST, WebSocket)
- Storage shapes (tables, columns, indexes, document schemas)
- Deployment shapes (containers, pods, VMs, serverless)

If the user's prompt specifies a component or technology, record the *intent* behind it in § 3 Core Principles or § 7 Scope Boundaries and leave the component to `/architecture`. Example: the user says "we'll use Postgres for this" — the proposal records "the system needs transactional multi-row writes across {entity} and {entity}" as a principle and lets `/architecture` decide Postgres or alternatives.

### 6. No feature list masquerading as a thesis

§ 2 Solution Thesis must name the *insight* that makes the solution work, not the features that will ship. "We will build a CLI with `push`, `pull`, `clone`, and `init` subcommands" is a feature list. "We will build a content-addressable protocol that never re-transmits known bytes" is a thesis. Features belong in `/use-cases` and per-unit SPECs.

### 7. One canonical product name

Pick one name for the product and use it consistently. If the user has not chosen a name, use `{ProductName}` as a bracketed placeholder throughout and add "confirm product name" to § 10 Open Questions. Do not invent a creative name; naming is the user's call.

### 8. Glossary stub is a preview, not the authoritative source

§ 9 must carry the preview note *(Authoritative glossary: DOMAIN.md. The entries here are a preview.)* so readers know this is not the final source of truth. Keep it small (5–15 entries) and focused on terms that appear in § 1–§ 7. The full glossary is `/domain`'s deliverable.

### 9. Non-goals ≥ principles in count

Non-goals protect the project from scope drift; principles constrain shape. A proposal with three principles and one non-goal is under-constrained on the "what we will not build" axis. Target at least as many non-goals as principles, and usually more.

### 10. Every section mandatory, no silent omissions

Every numbered section § 1 through § 10 must appear in the output. If a section genuinely has no content (rare — typically only § 10 Open Questions), state "None." or the standard empty-state phrase under the heading. Never omit a heading.

### 11. Single YAML frontmatter block

One YAML frontmatter block at the top of the file, containing common fields (`skill`, `date`, `status`) and proposal-specific count fields. Never emit a second YAML block anywhere in the document. Counts must match the body — `principles_count` equals the number of bullets in § 3, etc.

### 12. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "and so on", "various", "many". Use exact nouns, exact actor names, exact targets. If exact is impossible because the decision is genuinely open, surface it in § 10 Open Questions rather than hiding behind a placeholder word.

---

## Output Format

```markdown
---
skill: PROPOSAL.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
product_name: {name or "{ProductName}"}
principles_count: {N}
non_goals_count: {N}
success_criteria_count: {N}
open_questions: {N}
---

# PROPOSAL — {ProductName}

> Vision layer for the {ProductName} project. Every principle and non-goal here
> constrains later decisions. Downstream artifacts (USE_CASES, DOMAIN, ARCHITECTURE,
> and every per-unit SPEC) cite this document for grounding.

## § 1. Problem Statement

{Two to five paragraphs in the user's domain language. Lead with a concrete
 scenario where possible. Name the existing alternatives and say what each
 one gets wrong. No solution language — no API, endpoint, database, service,
 JSON, HTTP, frontend, backend, container, cluster, queue.}

---

## § 2. Solution Thesis

{Single opening sentence answering "what we will build" in domain terms.}

{Two to four paragraphs elaborating the core insight that makes the solution
 work — what the solution knows or does differently that the alternatives do
 not. Defensible against the reader's "why will this work when the alternatives
 did not?" — not a feature list.}

---

## § 3. Core Principles

- **{Principle, one short phrase}** — {one-line rationale stating what this
  principle forbids or requires in later decisions}.

(Repeat 3–7 times. Each principle must pass the falsification test: deleting
 it would change what we build.)

---

## § 4. Non-Goals

- **{Capability}** — {one-line reason it is excluded}.

(Repeat — count ≥ principles_count. Each entry is a capability someone might
 reasonably expect, paired with why we are not building it.)

---

## § 5. Success Criteria

- {Metric with a target, OR scenario with a pass/fail outcome — e.g., "a new
  user can install and complete their first push in ≤ 5 minutes on a typical
  home broadband connection" or "the server sustains 1000 concurrent fetch
  operations on a 4-core VM with p99 latency < 2s"}.

(Repeat 3–7 times. Every entry is measurable or testable; no un-quantified
 adjectives.)

---

## § 6. Target Users / Actors

### `{ActorName}`

{One paragraph: who this actor is, what they care about, one-line role.
 Example: "Operator — self-hosts the service for a team. Runs the binary,
 configures TLS, provisions storage, and manages user invites. Cares about
 low operational burden and long-running uptime."}

(Repeat for each actor. Three to seven typical. Full actor modelling is owned
 by USE_CASES.md; this section is the enumeration USE_CASES will expand.)

---

## § 7. Scope Boundaries

### In first milestone

- {Capability phrased at the vision level — not a feature. Example: "Accepting
  authenticated pushes to a repository owned by the authenticated user."}

(Repeat for each in-scope capability.)

### Not in first milestone

- {Capability — one-line reason it is deferred (not excluded forever — that is
  § 4). Example: "Pull requests — modelled as a separate collaboration layer
  on top of the storage substrate; out of v1."}

(Repeat for each deferred capability.)

---

## § 8. Risks & Open Assumptions

### Risks

- **{Risk, one-line description}** — impact: {L | M | H}. Mitigation: {one line}.

(Repeat for each risk. If none: "None identified at this stage.")

### Assumptions

- **{Assumption, one-line description}** — if wrong: {one-line consequence}.

(Repeat for each assumption. If none: "None — scope is self-contained.")

---

## § 9. Glossary Stub

*(Authoritative glossary: DOMAIN.md. The entries here are a preview.)*

| Term | Definition |
|------|-----------|
| `{Term}` | {one-line, domain-level, testable definition} |

(Repeat 5–15 times. Terms that appear in § 1–§ 7 and would be ambiguous to a
 reader unfamiliar with the product's domain.)

---

## § 10. Open Questions

- [ ] {Question — e.g., "Confirm the product name — the user prompt uses both
      'Bitserve' and 'bit-serve' informally."}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Problem statement in the user's domain language
- Solution thesis — the single sentence of what will be built and the supporting insight
- Core principles — 3–7 falsifiable constraints on later decisions, each with a rationale
- Non-goals — realistic alternative scopes excluded with a reason, count ≥ principles
- Success criteria — measurable metrics with targets, or scenarios with pass/fail outcomes
- Target actors — enumeration with one-line roles (full modelling is `/use-cases`)
- Scope boundaries for the first milestone (capabilities, not features)
- Risks and open assumptions
- Glossary stub — small preview of terms, marked as non-authoritative
- Genuinely ambiguous decisions surfaced in § 10 Open Questions

### Out of scope

- Use-case catalogue, actor scenarios, detailed user journeys — owned by `/use-cases`
- Entities, aggregates, value objects, invariants, bounded contexts, full glossary — owned by `/domain`
- System architecture, components, technology choices, ADRs — owned by `/architecture`
- Wire formats, endpoint shapes, event schemas — owned by `/interfaces`
- Persistence schema, indexes, access patterns — owned by `/data`
- Error taxonomy — owned by `/errors`
- State machines, sagas, lifecycles — owned by `/behavior`
- Observability, SLOs, performance budgets — owned by `/quality`
- Threat modelling, security controls — owned by `/security`
- Deployment, runbooks, config catalogue — owned by `/operations`
- UI surfaces — owned by IA skills (`/web-ia`, `/cli-ia`, `/mobile-ia`, `/tui-ia`, `/voice-ia`)
- Per-unit specs, plans, implementations — owned by `/WORK_UNITS.md` and the per-unit pipeline

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `product_name`, `principles_count`, `non_goals_count`, `success_criteria_count`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "and so on")
- [ ] § 10 Open Questions is present (empty with "All questions resolved." or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] All ten sections § 1 through § 10 are present with their exact headings
- [ ] § 1 Problem Statement contains no solution language — no `API`, `endpoint`, `database`, `service`, `frontend`, `backend`, `queue`, `container`, `cluster`, `JSON`, `HTTP`, `gRPC`, `SDK`
- [ ] § 2 Solution Thesis opens with a single sentence stating what will be built, followed by 2–4 paragraphs of supporting insight — not a feature list
- [ ] § 3 Core Principles has 3–7 entries, each with a one-line rationale, each passes the falsification test (deleting it would change what we build)
- [ ] § 4 Non-Goals has ≥ `principles_count` entries, each names a realistic alternative (no strawmen)
- [ ] § 5 Success Criteria has 3–7 entries, each is a metric with a target or a scenario with a pass/fail outcome — no un-quantified adjectives
- [ ] § 6 Target Users / Actors has 3–7 named actors, each with a one-paragraph role description
- [ ] § 7 Scope Boundaries separates "In first milestone" from "Not in first milestone", stated as capabilities (not features)
- [ ] § 8 Risks & Open Assumptions lists each risk with L/M/H impact and a mitigation; each assumption with a one-line consequence-if-wrong
- [ ] § 9 Glossary Stub has 5–15 entries and carries the preview note *(Authoritative glossary: DOMAIN.md. The entries here are a preview.)*
- [ ] No component names, technology choices, wire formats, storage shapes, or deployment shapes leaked into any section (those belong in `/architecture`, `/interfaces`, `/data`, `/operations`)
- [ ] Product name used consistently — either the user's chosen name or the `{ProductName}` placeholder with an open question to confirm
- [ ] Frontmatter counts (`principles_count`, `non_goals_count`, `success_criteria_count`, `open_questions`) match the body exactly
- [ ] `status` is `complete` if § 10 is "All questions resolved." and `has_open_questions` otherwise
- [ ] Document length is 200–1000 lines (target 200–500); overflow indicates feature-list drift or principle inflation
