---
name: DOMAIN.md
description: Model the domain — ubiquitous language, bounded contexts, entities, aggregates, value objects, domain events, services, and invariants with stable IDs. Use when asked to model the domain, create the domain model, define the glossary and entities, establish the ubiquitous language, or produce a DOMAIN.md.
---

# Task: Generate DOMAIN.md — Authoritative Domain Layer

## Objective

Produce a DOMAIN.md that serves as the single source of truth for the system's domain layer at the concept level, independent of technology and UI. It merges what older guidance split as GLOSSARY + DOMAIN_MODEL: in Agent-Driven Development those two are always read together, so they live in one file. DOMAIN.md fixes the ubiquitous language (every term, one canonical definition), enumerates bounded contexts and their integration patterns, models entities/aggregates/value objects, names domain events (meaning only — wire schemas live in INTERFACES.md), lists domain services, and indexes every invariant with a stable `INV-NN` ID. An agent reading this document alone can speak the domain precisely — using the right terms, knowing which aggregate owns which invariant, and knowing which events exist and what they mean — without guessing or drifting from the vocabulary.

DOMAIN.md is the most-read artifact downstream in the ADD workflow (`/architecture`, every IA skill, `/interfaces`, `/data`, `/behavior`, `/security`, every per-unit `/SPEC.md` reads it). The bounded contexts enumerated in this document also define the area catalog used by `units/<area>/u<NN>/`. Treat every decision in this document as a commitment that propagates everywhere.

---

## Inputs

1. **PROPOSAL.md** (required) — principles, scope, initial glossary. Seeds the ubiquitous language and bounds the domain.
2. **USE_CASES.md** (required) — actors and use cases with stable `UC-NN` IDs. Every entity and aggregate in this document must trace back to at least one use case. Use-case-specific terminology feeds the glossary.
3. **Existing DOMAIN.md** (auto-discovered — only if refining) — read fully and preserve every `INV-NN` and `EVT-name` already assigned. Never renumber stable IDs.
4. **Reference material on the domain** (optional) — industry vocabulary, regulations, standard models (e.g., the `Push` and `Fetch` terms in Git; the `Order` / `Cart` / `Line Item` vocabulary in e-commerce). Use to anchor terms to familiar meanings when available; override only with explicit rationale.

Read set size: 2 required inputs + up to 2 optional. Read them fully — truncated reads cause silent term drift.

---

## Workflow

Domain modelling proceeds in seven phases: term harvest, glossary, bounded contexts and map, structural model (entities → aggregates → value objects), events and services, invariants index, and validation. Phases are sequential — later phases depend on earlier decisions — but revisit earlier phases if a later one reveals a missing term or a contradiction.

### Phase 1: Term Harvest

Read PROPOSAL.md and USE_CASES.md end-to-end. Collect every domain noun, verb, and adjective into a working list. Separate three kinds of terms:

- **Domain terms** — `Repository`, `Push`, `Commit`, `Access Control List`, `Tenant`, `Order`, `Cart`. These belong in the glossary.
- **Technical terms** — `HTTP request`, `JSON payload`, `SQL table`, `gRPC stub`. These do NOT belong here. Flag them for architecture / interfaces / data.
- **UI terms** — `dashboard`, `settings page`, `login button`. These do NOT belong here. Flag them for IA skills.

If PROPOSAL.md ships with an initial glossary, treat it as an authoritative starting point — but expand, refine, and add missing terms from USE_CASES.md. The output's glossary must cover every domain term used anywhere in the system's artifacts, not just those the proposal listed.

For each harvested term, note every synonym encountered across the inputs (e.g., "repo" vs "repository", "user" vs "account" vs "principal"). You will pick one canonical form per term in Phase 2 and list the others as forbidden synonyms.

### Phase 2: Glossary (Ubiquitous Language)

Pick one canonical form per term and define it in § 1 Glossary. Every entry has term, definition (one–two sentences), category (entity / value-object / aggregate / event / service / external / relationship), the document section where the term is formally modelled, and — critically — a `(not: X)` list of forbidden synonyms.

Rules:

- **One canonical form per concept.** If the codebase, proposal, or use cases use two words for the same thing, pick one. Record the rejected form as `(not: X)`.
- **Definitions are domain-level.** Never define a term in terms of technology ("a JSON object representing…") or UI ("what shows up on the profile page"). Define in terms of the business concept.
- **Definitions are testable.** "An `Order` is a finalised request by a `Customer` to purchase one or more `Line Items` at stated prices" — you can point to an instance and verify it is or is not an Order. Vague definitions are modelling failures.
- **No tautologies.** Avoid "A `Repository` is a repository." Define using other glossary terms plus plain English.
- **Deprecated terms stay until drained.** If renaming `Collection` → `Folder`, keep `Collection` in the glossary with `(deprecated — use Folder)` until every reference in every artifact has been updated. Do not silently delete.

### Phase 3: Bounded Contexts & Context Map

Partition the domain into bounded contexts — sub-domains with their own models and integration rules. A bounded context is the scope within which a term has one unambiguous meaning. Different contexts may reuse the same word with different meanings (e.g., `User` in the `Identity` context is an authentication subject; `User` in the `Billing` context is a cost centre) — and that is exactly why bounded contexts exist.

For each context: name (short, capitalised, singular noun — `Access`, `Storage`, `Identity`, `Billing`), one-paragraph purpose, the glossary-term subset that lives in this context, and the integration pattern to each neighbour (shared kernel | customer/supplier | conformist | anti-corruption layer | published language) with rationale. Naming is part of the contract — downstream artifacts will cite context names; choose stable ones.

Produce the context map in § 3 as a Mermaid `flowchart LR` plus prose. Nodes are bounded contexts. Edges are integration patterns labelled with the pattern name. Prose explains any non-obvious edge and the dependency direction.

### Phase 4: Structural Model — Entities, Aggregates, Value Objects

Work outward in this order: entity → aggregate → value object. Each has its own section with the specific fields required by the template.

**Entities** are nouns with identity — two entities with identical attributes are still distinct if their identities differ. For each entity: name, owning bounded context, how identity is established (natural key or generated ID), lifecycle (create → … → terminal), *conceptual* key attributes (not typed — types are DATA.md's concern), relationships to other entities, and the `INV-NN` references for invariants that apply at the entity level.

**Aggregates** are clusters of entities and value objects treated as a single consistency boundary — everything inside the boundary is updated atomically together. For each aggregate: root entity, members (other entities / VOs inside the boundary), the consistency-boundary statement, the `INV-NN` references for invariants enforced atomically, and the access rule (only-through-root or direct access to members permitted — and if permitted, under what constraint).

**Value objects** are nouns without identity — equality by value alone. For each VO: name, conceptual composition (fields, untyped), equality rule (always by-value), and `INV-NN` references for validation invariants (e.g., "`Email` must be RFC 5321", "`ContentHash` is 64 lowercase hex characters").

Resolve "is this an entity or a value object?" with the identity test: do two instances with identical attributes refer to the same thing in the business? If yes (two `Money` values of `$5` are the same `Money`) → value object. If no (two `Orders` each for `$5` are distinct) → entity.

### Phase 5: Domain Events & Domain Services

**Domain events** are meaningful state changes in the business. Name them in past tense: `EVT-push-accepted`, `EVT-order-placed`, `EVT-subscription-renewed`. For each event: `EVT-name` (kebab-case, past-tense verb), when emitted (the triggering action in domain terms), conceptual payload (the meaningful information carried — *names and roles* of fields, not wire types), and significance (what downstream logic depends on this event).

Wire schemas for events live in INTERFACES.md — this document owns names and meanings only.

**Domain services** are operations that cross entities or do not naturally belong to a single one. For each: verb-phrase name (`TransferFunds`, `AllocateInventory`, `AcceptPush`), purpose, participating entities / aggregates, and invariants preserved across the operation. A service is a red flag if it could equally well be a method on one of its participating entities — in that case, move it. Services exist precisely when the operation does not fit any single entity.

### Phase 6: Invariants Index

Collect every invariant mentioned in entities, aggregates, value objects, and services into the master `INV-NN` index in § 9. Each entry: stable ID (assigned once, never renumbered), statement (one sentence, precise, testable in principle), enforcer (which aggregate / service / context enforces it), and category (uniqueness | lifecycle | reference-integrity | authorisation | temporal | value-constraint).

When refining an existing DOMAIN.md, new invariants get the next unused `INV-NN`. Deleted invariants leave their ID retired with a `(retired)` marker — do not reassign the number. This preserves downstream references.

Every aggregate must list at least one invariant in § 5, or explicitly state why it is invariant-free (rare; usually means the aggregate boundary is wrong). Every value object with any constraint must list it as an invariant.

### Phase 7: Traceability & Validation

Verify the document holds together:

- Every term used in §§ 2–10 appears in § 1 Glossary.
- Every entity and aggregate traces back to at least one `UC-NN` from USE_CASES.md. Orphans are modelling errors — either the entity is not needed, or a use case is missing.
- Every `INV-NN` referenced from §§ 4–8 appears in § 9 Invariants Index.
- Every `EVT-name` referenced from anywhere in the document appears in § 7 Domain Events.
- No technology vocabulary leaked in (`HTTP`, `SQL`, `JSON`, `REST`, `endpoint`, `table`, `column`, `serialise`, `parse`) — grep the output before finalising.
- No UI vocabulary leaked in (`page`, `button`, `dashboard`, `screen`, `tab`, `modal`) — grep the output before finalising.
- Forbidden synonyms are listed for every canonical term that has known alternatives in the inputs.
- Every aggregate has ≥ 1 invariant, or an explicit invariant-free justification.

Update frontmatter counts to reflect the final document.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Single source of truth for every term

If a term is defined in § 1 Glossary, it is not redefined anywhere else in the document or in any downstream artifact. Entity and aggregate sections may *use* a term and reference its glossary row; they must not rephrase the definition. Duplication is the commonest cause of contradiction bugs.

### 2. Stable IDs are forever

`INV-NN` and `EVT-name` IDs are assigned once and never renumbered, never reused, never deleted-without-trace. A retired invariant leaves its ID marked `(retired — superseded by INV-42)` in § 9 so downstream references still resolve.

### 3. Domain language only — no technology

Forbidden vocabulary in this document: `HTTP`, `REST`, `gRPC`, `JSON`, `XML`, `SQL`, `NoSQL`, `table`, `column`, `row`, `endpoint`, `route`, `URL`, `path`, `header`, `payload`, `schema`, `serialise` / `serialize`, `deserialise` / `deserialize`, `parse`, `encode`, `decode`, `wire`, `protocol`, `socket`, `TCP`, `WebSocket`. Those belong in `/architecture`, `/interfaces`, `/data`.

### 4. Domain language only — no UI

Forbidden vocabulary: `page`, `screen`, `view` (as a UI noun), `button`, `tab`, `modal`, `dialog`, `form` (as a UI noun — domain forms like "tax form" are fine), `click`, `tap`, `swipe`, `navigate`, `dashboard`, `menu`. Those belong in IA skills.

### 5. Every aggregate lists at least one invariant

An aggregate exists precisely to enforce invariants atomically. An invariant-free aggregate is almost always a modelling error — usually the boundary is wrong. If an aggregate genuinely has no invariants, include an explicit one-line justification in the aggregate entry ("Invariant-free: this aggregate exists solely to group {entity A} and {entity B} for cascade deletion; no cross-member consistency is enforced."). This is rare; prefer to re-examine the boundary first.

### 6. Every entity and aggregate traces to a use case

Every entity and aggregate in §§ 4–5 must reference at least one `UC-NN` from USE_CASES.md. The reference can be inline ("Used by UC-04, UC-07, UC-11") or in a traceability table at the end of the section. Orphans indicate either an unneeded concept or a missing use case — surface the discrepancy.

### 7. Canonical form plus forbidden synonyms

For every canonical glossary term, list the forbidden synonyms as `(not: X, Y)`. The forbidden list is non-empty if and only if the inputs contain more than one form for the concept. Silent omission of a forbidden synonym causes term drift in downstream artifacts.

### 8. Context-local terms are explicit

If the same word has different meanings in two bounded contexts (classic `User` case), the glossary has two entries, each qualified by its context: `User (Identity context)` and `User (Billing context)`. An unqualified cross-context term is ambiguous and forbidden.

### 9. Events name state changes, never commands

`EVT-order-placed` is an event. `PlaceOrder` is a command (belongs to `/behavior` or `/interfaces`). Domain events are always past tense and always name something that has already happened. A present-tense or imperative event name is a modelling error.

### 10. Entities vs value objects: the identity test

Resolve the entity-vs-VO distinction by asking whether two instances with identical attributes are the same thing in the business. Two `Money` values of `$5` are the same `$5` → VO. Two `Orders` each totalling `$5` are distinct — each has its own history and identity → entities. Apply this test consistently.

### 11. No wire details in events

Event entries describe the *conceptual* payload — what meaningful information the event carries, by role. They never include JSON field names, protobuf field numbers, casing conventions, or versioning syntax. Those live in INTERFACES.md. An event entry that reads like a JSON schema is a rule-3 violation.

### 12. Forbid "appropriate", "relevant", "as needed", "etc."

Use exact terms, exact context names, exact invariant IDs. If you cannot be exact, flag the ambiguity as an open question rather than hiding behind a placeholder.

---

## Output Format

```markdown
---
skill: DOMAIN.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
bounded_contexts: {N}
entities: {N}
aggregates: {N}
value_objects: {N}
invariants: {N}
domain_events: {N}
glossary_terms: {N}
domain_services: {N}
open_questions: {N}
---

# DOMAIN

> Authoritative domain layer. Every term, entity, aggregate, value object, event,
> service, and invariant is defined here and here only. Downstream artifacts cite
> by name (`Repository`, `Access` context) or by stable ID (`INV-07`, `EVT-push-accepted`).

## § 1. Glossary (Ubiquitous Language)

| Term | Definition | Category | First modelled in | Forbidden synonyms |
|------|-----------|----------|-------------------|--------------------|
| `{CanonicalTerm}` | {one–two sentences, domain-level, testable} | entity / value-object / aggregate / event / service / external / relationship | § {N.M} {section name} | (not: {synonym}, {synonym}) — or (none) |

(Alphabetical. Repeat for every domain term used anywhere in this document or
 expected to be used by downstream artifacts. Deprecated terms retain an entry
 marked `(deprecated — use {replacement})`. Context-local homonyms have two
 rows qualified by context, e.g., `User (Identity context)` and `User (Billing context)`.)

---

## § 2. Bounded Contexts

### `{ContextName}`

- **Purpose:** {one paragraph — what problem this context owns in the business}
- **Ubiquitous-language subset:** {comma-separated list of glossary terms that live in this context}
- **Integration with neighbours:**
  - To `{NeighbourContext}`: {shared kernel | customer/supplier | conformist | anti-corruption layer | published language} — {one-line rationale}

(Repeat for every bounded context.)

---

## § 3. Context Map

```mermaid
flowchart LR
    {Context1}[{Context1}] -- {pattern} --> {Context2}[{Context2}]
    {Context1} -- {pattern} --> {Context3}[{Context3}]
```

{One-paragraph prose walking through the map: the non-obvious edges, the
 dependency directions, and any anti-corruption layer rationale.}

---

## § 4. Entities

### `{EntityName}`

- **Owning context:** `{ContextName}`
- **Identity:** {how instances are distinguished — natural key "{field}", generated `{IdKind}`, composite of {fields}}
- **Lifecycle:** {create → intermediate states → terminal state; cite `SM-{entity}` if stateful, with a note that details live in BEHAVIOR.md}
- **Key attributes (conceptual):** {comma-separated conceptual attributes — "name, owner, visibility, created-at"; no types}
- **Relationships:** {each relationship as "{EntityName} — {cardinality, e.g., 1..*} — via {relationship name from glossary}"}
- **Invariants:** `INV-{NN}`, `INV-{NN}`
- **Use cases:** `UC-{NN}`, `UC-{NN}`

(Repeat for every entity, grouped by owning bounded context.)

---

## § 5. Aggregates

### `{AggregateName}`

- **Root entity:** `{EntityName}`
- **Members:** {entities and value objects inside the boundary}
- **Consistency boundary:** {one sentence — what is updated atomically with the root}
- **Invariants enforced atomically:** `INV-{NN}`, `INV-{NN}`
- **Access rule:** {only-through-root | direct member access permitted for {operation} because {constraint}}
- **Use cases:** `UC-{NN}`, `UC-{NN}`

(Repeat for every aggregate. If an aggregate is invariant-free, include an
 explicit one-line justification — but first re-examine the boundary.)

---

## § 6. Value Objects

### `{VOName}`

- **Composition (conceptual):** {fields without types — "local part, domain part" for an Email; "amount, currency" for Money}
- **Equality:** by-value
- **Invariants:** `INV-{NN}`, `INV-{NN}`

(Repeat for every value object.)

---

## § 7. Domain Events

### `EVT-{name-in-kebab-case-past-tense}`

- **When emitted:** {the triggering action in domain terms — "when a push is accepted by the Access context"}
- **Conceptual payload:** {field roles, no types — "the repository whose state changed, the push identifier, the actor who pushed, the accepted-at instant"}
- **Significance:** {what downstream logic depends on this event — "the Storage context uses it to schedule pack compaction; the Audit context uses it to record write history"}

(Repeat for every domain event. If no domain events: "No domain events modelled.")

---

## § 8. Domain Services

### `{ServiceName}`

- **Purpose:** {one sentence — what the service does, in domain terms}
- **Participating entities / aggregates:** {list}
- **Invariants preserved:** `INV-{NN}`, `INV-{NN}`

(Repeat for every domain service. If none: "No domain services — all operations
 belong to single entities or aggregates.")

---

## § 9. Invariants Index

| ID | Statement | Enforced by | Category |
|----|-----------|-------------|----------|
| `INV-01` | {one sentence, precise, testable in principle} | `{aggregate / service / context}` | uniqueness / lifecycle / reference-integrity / authorisation / temporal / value-constraint |

(Repeat for every invariant. Retired invariants keep their ID with the marker
 `(retired — superseded by INV-{NN})`.)

---

## § 10. Anti-Corruption Layers

### `{ExternalModelName}` → `{InternalContextName}`

- **External model:** {short description of the foreign vocabulary}
- **Internal mapping:** {which internal context receives the translation}
- **Translation rules:** {the field-by-field or concept-by-concept mapping — at domain level, not wire level}
- **Rationale:** {what would go wrong without this ACL}

(Repeat for every ACL. If none: "No anti-corruption layers required.")

---

## § 11. Open Questions

- [ ] {Question — e.g., "Is `Fork` an entity distinct from `Repository`, or a value object that carries a `parent-repository` reference? The inputs are silent."}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Ubiquitous language: glossary with canonical forms, forbidden synonyms, categories, and where each term is modelled
- Bounded contexts, their ubiquitous-language subsets, and integration patterns to neighbours
- Context map (Mermaid diagram plus prose)
- Entities, aggregates, value objects — at the concept level, with identity rules, lifecycles, relationships, and invariant references
- Domain events — names and meanings only
- Domain services — cross-entity operations with invariants preserved
- Invariants index with stable `INV-NN` IDs and category classification
- Anti-corruption layers where external vocabularies meet the domain

### Out of scope

- Technology choices (HTTP, SQL, message brokers, specific frameworks) — owned by `/architecture`
- Wire formats and event schemas on the wire (JSON field names, protobuf numbers, versioning syntax) — owned by `/interfaces`
- Persistence schema (database tables, indexes, column types) — owned by `/data`
- State transitions, sagas, compensating actions (the *how* of lifecycles) — owned by `/behavior`
- UI surfaces (pages, screens, navigation) — owned by IA skills (`/web-ia`, `/cli-ia`, `/mobile-ia`, `/tui-ia`, `/voice-ia`)
- Threat modelling and security controls — owned by `/security`
- Observability signals and SLOs — owned by `/quality`
- Config, deployment, runbooks — owned by `/operations`
- Code — owned by `/spec` and `/implement`

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `bounded_contexts`, `entities`, `aggregates`, `value_objects`, `invariants`, `domain_events`, `glossary_terms`, `domain_services`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] Stable IDs (`INV-NN`, `EVT-name`) are assigned and consistent; no renumbering; retired IDs marked `(retired)`
- [ ] Every term used anywhere in §§ 2–10 appears in § 1 Glossary
- [ ] Every canonical glossary term with known alternatives lists forbidden synonyms via `(not: X, Y)`; terms without known alternatives list `(none)`
- [ ] Every aggregate in § 5 references at least one `INV-NN`, or carries an explicit invariant-free justification
- [ ] Every `INV-NN` referenced in §§ 4–8 appears exactly once in § 9 Invariants Index
- [ ] Every `EVT-name` referenced anywhere appears in § 7 Domain Events
- [ ] Every entity in § 4 and every aggregate in § 5 references at least one `UC-NN`
- [ ] Domain events are named in past tense (`EVT-order-placed`, never `EVT-place-order`)
- [ ] Context map diagram is present in § 3 as a Mermaid `flowchart LR` with integration patterns labelled on edges
- [ ] Context-local homonyms (same term, different meaning per context) have separate qualified glossary rows
- [ ] No technology vocabulary appears in the document (grep-check: `HTTP`, `REST`, `JSON`, `SQL`, `table`, `column`, `endpoint`, `payload`, `serialise`, `parse`)
- [ ] No UI vocabulary appears in the document (grep-check: `page`, `button`, `screen`, `tab`, `modal`, `dashboard`, `click`, `tap`)
- [ ] No wire schema, type syntax, or serialization convention appears in event entries or value-object compositions
- [ ] Frontmatter counts (`bounded_contexts`, `entities`, `aggregates`, `value_objects`, `invariants`, `domain_events`, `glossary_terms`, `domain_services`) match the body
