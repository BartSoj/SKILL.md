---
name: BEHAVIOR.md
description: Model the system's behavioral contract — state machines per stateful aggregate, sagas for multi-step workflows with explicit compensations and timeouts, an idempotency-key registry, concurrency rules, temporal ordering invariants, and event emission timing under stable `SM-*` and `SAGA-*` IDs. Use when asked to model the behavior, design state machines and sagas, specify workflows and idempotency, define concurrency rules, or produce a BEHAVIOR.md.
---

# Task: Generate BEHAVIOR.md — Behavioral Contract

## Objective

Produce a BEHAVIOR.md that serves as the single source of truth for everything the system does that is stateful or multi-step: how every stateful aggregate moves between states; how every multi-step operation crosses components, compensates on partial failure, and bounds itself with timeouts and retries; which idempotency keys exist and with what lifecycle; what concurrency rules govern interleaving and locking; what temporal ordering invariants must always hold; and exactly when in a state-machine or saga each domain / integration event is emitted. This is a *declarative* artifact — it states *what* must happen, never *how* to code it.

Without this document, state transitions get scattered across implementation files, saga compensations become discoverable only by reading code, idempotency semantics drift silently between endpoints, and concurrency rules live in comments the next agent does not read. BEHAVIOR.md sits between INTERFACES.md (which owns per-endpoint contracts in isolation) and per-unit SPECs (which implement individual behaviours) and pins the cross-endpoint sequencing that neither isolates. Downstream readers are `/quality` (metrics and SLOs derive from transition rates and saga timings), `/security` (threat-on-state-transition analysis and idempotency abuse), `/operations` (runbook procedures for compensating a stuck saga), every per-unit `/SPEC.md` for a stateful unit (cites the `SM-*` or `SAGA-*` it implements), and `/system-verification` (end-to-end scenarios exercise saga happy paths and failure modes). The defining discipline — and the commonest violation — is **every citation resolves**: every `INV-NN` exists in DOMAIN § 9, every `ERR_CODE` in ERRORS.md § 3, every `EP-name` in INTERFACES § 6, every `EVT-name` in INTERFACES § 7, every `UC-NN` in USE_CASES.md.

---

## Inputs

1. **DOMAIN.md** (required) — § 5 Aggregates supply the list of candidate state machines; § 9 Invariants Index supplies the `INV-NN`s transitions must preserve; § 7 Domain Events supply event names whose emission timing this document fixes.
2. **ARCHITECTURE.md** (required) — § 2 Components and cross-component flow sections supply the participants of every saga; any `ADR-NN` on consistency and transaction boundaries frames § 4 Concurrency Rules.
3. **INTERFACES.md** (required) — § 6 Endpoints are the entry points of saga triggers and state-machine transitions; § 7 Events supply the integration events whose emission timing § 6 of this document fixes; § 4 Idempotency in INTERFACES pins the `Idempotency-Key` header shape that § 3 here maps to saga-level keys.
4. **ERRORS.md** (required-if-available, otherwise optional) — illegal transitions cite `ERR_CODE`s from § 3; non-retryable failures in saga blocks cite `ERR_CODE`s classified there. If ERRORS.md does not yet exist, cite codes by their intended string and surface the lack of a registry entry as an open question for later reconciliation.
5. **USE_CASES.md** (required) — every `SM-*` and `SAGA-*` cites at least one `UC-NN`; orphans reveal either an unneeded behavioural model or a missing use case.
6. **Existing BEHAVIOR.md** (auto-discovered — only if refining) — read fully; every assigned `SM-*`, `SAGA-*`, and idempotency-key name is permanent. New entries take new unused IDs. Retired entries remain with `(retired — superseded by {new-id})` markers.

Read set size: 3 required artifacts + ERRORS when present + USE_CASES + optional prior BEHAVIOR. Read all required inputs end-to-end. Truncated reads cause two specific failure modes: invariant violations in transitions (DOMAIN not fully read) and saga steps that cite endpoints that do not exist (INTERFACES not fully read).

---

## Workflow

Behavioral-contract construction proceeds in seven phases: aggregate-to-SM mapping, saga identification, idempotency-key registration, concurrency rules, temporal invariants, event emission timing, validation. Phases are sequential — later phases cite IDs introduced by earlier phases — but revisit earlier phases if a later one reveals a missing transition, a mis-scoped key, or an un-enforced invariant.

### Phase 1: State Machine Catalogue

List every aggregate in DOMAIN.md § 5 whose description mentions a lifecycle, states, or a sequence of observable conditions. Every such aggregate gets an `SM-{aggregate-name}` block following `references/saga-template.md` verbatim.

For each SM:

- **Identify states from DOMAIN.md.** The aggregate's `Lifecycle` bullet is the seed. Enumerate every named condition (`pending`, `active`, `archived`, `suspended`) as a state. If DOMAIN uses a short prose description (`create → review → publish → archive`), convert each arrow-separated condition to a state name in lowercase-hyphen form.
- **Identify transitions from USE_CASES and INTERFACES.** Every endpoint in INTERFACES.md § 6 that mutates the aggregate triggers a transition. Every use case that changes the aggregate's condition describes a transition. Enumerate all of them in the transition table — one row per (from-state, event-or-action) pair.
- **Populate invariants.** Every transition's `Invariants preserved` column lists every `INV-NN` from DOMAIN § 9 that applies to the aggregate. Transitions that might violate an invariant name the mitigation explicitly (a pre-transition guard, an atomic effect, or a compensation path if the mitigation is only partial).
- **Enumerate illegal transitions.** For every (state, action) pair NOT in the transition table where a client might plausibly attempt the action, add a row to the illegal-transitions table citing the `ERR_CODE` emitted. Silent rejection is forbidden.
- **Draw the mermaid diagram.** Use `stateDiagram-v2`. Every state in the transition table appears in the diagram; every terminal state ends with `--> [*]`. Diagrams must parse — an SM whose diagram has a dangling edge is a modelling error.

Aggregates explicitly without a lifecycle (invariant-only aggregates per DOMAIN § 5) do not get an SM — instead § 1 of BEHAVIOR opens with a one-line note that explicitly names them as SM-free and why. This prevents silent omission.

### Phase 2: Saga Identification

Every multi-step operation that crosses two or more aggregates, two or more components, or requires compensation on partial failure is a saga. Scan three sources to identify sagas:

- **ARCHITECTURE.md cross-component flows** — every flow describing a sequence of interactions between components is a saga candidate.
- **USE_CASES.md multi-step scenarios** — every use case that touches more than one aggregate or involves external systems is a saga candidate.
- **INTERFACES.md endpoints that emit events consumed by other components** — every such chain (`EP-X → EVT-Y → consumer component → effect`) is a saga.

For each saga produce a `SAGA-{name}` block per the template in `references/saga-template.md`. Rules:

- **Trigger is exactly one of four forms.** User action via a specific `EP-name`; scheduled event on a named cadence; domain event `EVT-name`; state-machine transition `SM-entity: from → to`. Multi-trigger sagas either split into separate `SAGA-*` entries or declare a discriminator trigger.
- **Participants cite ARCHITECTURE and DOMAIN.** Components come from ARCHITECTURE § 2. Aggregates come from DOMAIN § 5. An unlisted component or aggregate is a contract gap — surface as an open question, not as new documentation.
- **Compensations are mandatory per step.** Every happy-path step has a row in the compensations table, even if the compensation is `none (read-only)`. Missing compensations are modelling bugs. A step whose side effects are "inherently irreversible" (an external email sent, a refund wired) has a compensation that records the fact and triggers operator action — not a silent gap.
- **Timeouts are explicit, numeric.** Per-step and overall timeouts name a duration (`5s`, `30s`, `2m`). `(none)` is permitted only with explicit rationale — e.g., `(none — step is bounded by external SLA of downstream EP-name)`.
- **Retries cite ERR_CODE classes.** Retryability is never redefined here; it is referenced from ERRORS.md's `retryable` column. A saga that retries on a code ERRORS.md classifies as `no` is a contract violation.
- **Idempotency-key is named.** Every mutating saga names the key it uses (from § 3 below). The § 3 row reciprocally names the saga in its `Used by` column.

### Phase 3: Idempotency-Key Registration

§ 3 is the registry of every idempotency key the system uses. One row per key, sorted alphabetically by key name. Columns: `Key name`, `Scope` (per-{resource}, global), `Storage` (Postgres table, Redis namespace, in-memory cache), `TTL` (duration or `permanent`), `Generation` (client-provided via header, server-derived from body hash, system-generated ULID), `Collision handling` (return cached result, reject with `ERR_CODE`, serialise behind lock), `Used by` (comma-separated `SAGA-*` / `EP-*` / `EVT-*` references).

Rules:

- Every mutating saga in § 2 names exactly one key in its `Idempotency key` bullet and that key has a row here. Bidirectional citation — the row's `Used by` column names the saga.
- The `Idempotency-Key` HTTP header from INTERFACES.md § 4 is one row in § 3 with scope `global (header)` and `Used by` listing every mutating endpoint that honours it (or `all mutating endpoints per INTERFACES § 4` if the list is the default).
- Keys that the system auto-derives (from a body hash, from a natural key like `repo_id` + branch) state the derivation precisely enough that two implementations would agree on whether two requests are duplicates.
- TTL is a duration (`24h`, `15m`, `7d`) or `permanent`. `(none)` is never permitted — a key without a TTL leaks storage, which is an operational bug.

### Phase 4: Concurrency Rules

§ 4 enumerates the rules that govern interleaving and locking across the system. Organise into four subsections — each either populated or explicitly stating "(none applicable)":

- **Per-aggregate locking.** Which aggregate instances are serialised behind a lock and on what key. Example: "`Push` operations acquire a per-repository advisory lock on `repo_id`; concurrent pushes to the same repo serialise." Name the lock kind (advisory, row-level, distributed-via-Redis-per-ARCHITECTURE.md § X) and the release condition.
- **Optimistic concurrency.** Which aggregates use version fields and how conflicts manifest. Cite `ERR_CODE` from ERRORS.md (e.g., "`Repository` carries an `_etag`; stale-write attempts return `VERSION_CONFLICT`").
- **Serializability expectations.** Pairs or sets of operations that must not interleave. Name the operations by `EP-name` or `SAGA-name` and the invariant their serialisation preserves.
- **Parallel-safe operations.** Explicit list of read patterns that require no coordination. This is the positive list that prevents future agents from "helpfully" adding locks where none are needed.

Every rule in this section cites the aggregate, endpoint, or saga it governs and the `INV-NN` or `ERR_CODE` it preserves or emits.

### Phase 5: Temporal Ordering Invariants

§ 5 is a bulleted list of facts about ordering that must always hold. Examples:

- "A repository's first push creates the repository record; subsequent pushes require the record to exist." (cites `INV-03`)
- "Fork creation waits for source-repo snapshot completion; source repo may be deleted only after fork-creation completes." (cites `INV-09`, `INV-14`)
- "Credential revocation must visibly propagate to every session within 30 seconds." (cites `INV-22`)

Each invariant:

- Names the two or more events / state changes and their required ordering.
- Cites its `INV-NN` in DOMAIN § 9. If DOMAIN does not yet have a matching invariant, propose a new `INV-NN` as an open question for the `/domain` skill to register.
- States how the ordering is enforced — a lock from § 4, an event-consumption waitpoint in a saga from § 2, a compensating action in an SM from § 1. Invariants without enforcement are wishes; enforcement is the contract.

### Phase 6: Event Emission Timing

For every `EVT-name` in INTERFACES.md § 7, state exactly when in the state-machine transition or saga step the event is emitted. The form is one row per event in a table:

| `EVT-name` | Emitted by | Emission point | Transactional? | Durability |
|------------|-----------|----------------|----------------|------------|

- `Emitted by` is the `SM-*` transition (`SM-push: validating → accepted`) or saga step (`SAGA-push step 3`) that emits it.
- `Emission point` is one of three values: `inside DB transaction`, `after commit (outbox)`, `before commit (optimistic)`. State the concrete mechanism.
- `Transactional?` is `yes` (emitted atomically with state change via outbox or transactional topic) or `no` (at-least-once fire-and-forget; consumers deduplicate).
- `Durability` states what survives a crash: `persisted in outbox; retried until ack` / `in-memory only; lost on crash` / `persisted but single-shot; manual replay on loss`.

Every event in INTERFACES § 7 appears exactly once in this table. Events emitted by multiple transitions (rare) get one row per transition with distinct `Emitted by` values.

### Phase 7: Validation

Before finalising verify:

- Every aggregate in DOMAIN § 5 with a lifecycle has either an `SM-*` block in § 1 or an explicit one-line exemption in the § 1 opening note.
- Every `SM-*` block has initial state, terminal states (or an explicit `(none)`), a full transition table, an illegal-transition table with every row citing an `ERR_CODE`, and a mermaid `stateDiagram-v2` that parses.
- Every `SAGA-*` block has a single-form trigger, participating components and aggregates, numbered happy-path steps, a compensation row per step, per-step and overall timeouts, retry policy citing `ERR_CODE` classes, and a failure-modes table with observable signals.
- Every idempotency key in § 3 is cited by at least one saga (`Used by` non-empty) and every mutating saga in § 2 cites a key from § 3 (bidirectional).
- Every `INV-NN` referenced in §§ 1, 4, 5 exists in DOMAIN § 9.
- Every `ERR_CODE` referenced in §§ 1, 2 exists in ERRORS.md § 3. If ERRORS.md is not yet produced, cite by intended string and list each code in § 8 Open Questions for reconciliation.
- Every `EP-name` cited in § 2 saga steps exists in INTERFACES § 6.
- Every `EVT-name` cited in §§ 1, 2, 6 exists in INTERFACES § 7 (or in DOMAIN § 7 if the event is pre-registration pending INTERFACES).
- Every `UC-NN` cited in any block exists in USE_CASES.md.
- § 6 Event Emission Timing has a row for every `EVT-name` in INTERFACES § 7; no event is missing.
- No transition violates a stated `INV-NN` without a documented mitigation.
- No implementation syntax appears (grep-check: `try`, `catch`, `panic`, `async`, `await`, `mutex`, `semaphore`, `goroutine`, `thread`).
- No wire schema appears (grep-check: `camelCase`, `snake_case`, `JSON`, `protobuf`, `HTTP status`).

Update frontmatter counts to reflect the final document. `status` is `complete` if § 8 Open Questions reads `All questions resolved.`, `has_open_questions` otherwise.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Stable IDs forever

`SM-{entity-name}`, `SAGA-{name}`, and idempotency-key names are assigned once and never renumbered, never reused, never silently deleted. Retired entries remain with `(retired — superseded by {new-id})` markers so downstream citations keep resolving.

### 2. Every citation resolves

Every `INV-NN` must exist in DOMAIN § 9. Every `ERR_CODE` must exist in ERRORS.md § 3 (or, if ERRORS.md is not yet produced, the code string is listed in § 8 Open Questions for reconciliation). Every `EP-name` must exist in INTERFACES § 6. Every `EVT-name` must exist in INTERFACES § 7 or DOMAIN § 7. Every `UC-NN` must exist in USE_CASES.md. Every component name must exist in ARCHITECTURE.md § 2. Every aggregate name must exist in DOMAIN.md § 5. An unresolvable citation is a contract bug and surfaces in § 8 Open Questions, never silently inlined.

### 3. Every stateful aggregate has an SM or an exemption

Every aggregate in DOMAIN § 5 that carries a `Lifecycle` bullet with named conditions or a multi-step progression has a matching `SM-{entity-name}` in § 1. Aggregates that are genuinely lifecycle-free (static reference data, write-once ledgers) are listed explicitly in § 1's opening note as SM-free, with a one-line reason. Silent omission is a modelling error.

### 4. Illegal transitions cite ERR_CODE

Every (from-state, action) pair that must be rejected names the `ERR_CODE` the system emits. The code must exist in ERRORS.md § 3. Silent rejection ("the state machine simply does not advance") is forbidden — observers need an observable signal.

### 5. No transition leaks a domain invariant

A transition whose effects could violate any `INV-NN` must state the mitigation in the same row (a pre-transition guard tightening, an atomic effect, a compensation path). A transition that is known to temporarily break an invariant must document the compensating transition that restores it and the window during which the violation is observable.

### 6. Sagas have explicit compensations per step

Every happy-path step has a row in the compensations table. Missing rows are modelling bugs. `none (read-only)` is a legitimate compensation value for steps without side effects. Steps whose side effects are inherently irreversible record the fact and the operator-action path — never silence.

### 7. Every mutating saga names its idempotency key

A saga that mutates state and has no idempotency key is a contract violation — replays will double-apply. Every mutating saga cites a key from § 3; that key's § 3 row reciprocally lists the saga in `Used by`.

### 8. Idempotency-key storage and TTL are concrete

Every key row has a named store (Postgres table, Redis namespace, outbox topic) and a concrete TTL (`24h`, `15m`, `permanent`). `(none)` TTL is never permitted — unbounded storage is an operational bug.

### 9. Concurrency rules cite invariants they preserve

Every locking, optimistic-concurrency, or serializability rule in § 4 names the `INV-NN` it preserves or the `ERR_CODE` it emits on conflict. A concurrency rule without a citation is a guess about runtime behaviour, not a contract.

### 10. Temporal invariants name enforcement

Every invariant in § 5 names the mechanism that enforces the ordering — a lock from § 4, a wait-for-event step in a saga from § 2, a compensating action in an SM from § 1. Invariants without enforcement are wishes, not contracts.

### 11. Event emission timing is exact

§ 6 states, per event, whether emission is inside the DB transaction, after commit via outbox, or before commit optimistically. Every event from INTERFACES § 7 has a row. Ambiguity here causes split-brain between database and message bus — the commonest source of event-driven bugs.

### 12. No implementation syntax

BEHAVIOR.md never names try / catch blocks, async / await keywords, mutex / semaphore primitives, thread pools, goroutine counts, specific libraries. Those are SPEC / IMPLEMENTATION concerns. This document states *what* the contract is; *how* the code realises it lives downstream.

### 13. No wire schemas

Event payload shapes, HTTP request bodies, JSON field casing belong to INTERFACES.md. This document cites events by `EVT-name` and endpoints by `EP-name` — never inlines their shape. A saga block that inlines a JSON body is a rule violation.

### 14. Diagrams must parse

Every mermaid block in § 1 must be valid `stateDiagram-v2`. Before finalising, mentally trace every state listed in the `States:` bullet and confirm it appears in the diagram as either a source, target, terminal (`--> [*]`), or initial (`[*] -->`). A dangling diagram fails the checklist.

### 15. Single YAML frontmatter block

One YAML block at the top containing common fields (`skill`, `date`, `status`) and behavior-specific counts (`state_machines`, `sagas`, `idempotency_keys`, `concurrency_rules`, `temporal_invariants`, `tracked_events`, `open_questions`). Never emit a second block. Counts match the body exactly.

### 16. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "eventually". Use exact state names, exact IDs, exact durations. Unresolvable ambiguity surfaces in § 8 Open Questions with options, tradeoffs, and a recommendation.

---

## Output Format

The per-state-machine block template in § 1 and per-saga block template in § 2 are defined in `references/saga-template.md`. Read that file and apply both templates verbatim — field order, heading level, bullet labels, table column order, and fence style are the contract.

```markdown
---
skill: BEHAVIOR.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
state_machines: {N}
sagas: {N}
idempotency_keys: {N}
concurrency_rules: {N}
temporal_invariants: {N}
tracked_events: {N}
open_questions: {N}
---

# BEHAVIOR — {ProductName}

> Behavioral contract. Every state machine, saga, idempotency key, concurrency
> rule, temporal ordering invariant, and event emission timing lives here.
> Downstream artifacts cite by stable `SM-*` and `SAGA-*` IDs. Wire shapes of
> endpoints and events live in INTERFACES.md; error code definitions in
> ERRORS.md; domain invariants in DOMAIN.md.

## § 1. State Machines

{Opening note. If every aggregate in DOMAIN § 5 with a lifecycle has an SM,
 a single sentence: "Every stateful aggregate in DOMAIN § 5 has an SM block
 below." If any aggregate is SM-free, list it explicitly with a one-line
 reason: "Aggregates without state machines: `{AggregateName}` — {reason}."}

{State-machine blocks per `references/saga-template.md`, one per entity,
 sorted alphabetically by `SM-name`.}

---

## § 2. Sagas

{Saga blocks per `references/saga-template.md`, one per multi-step workflow,
 sorted alphabetically by `SAGA-name`. If the system has no multi-step
 workflows: "No sagas — every operation is a single state-machine transition
 or a single stateless endpoint call."}

---

## § 3. Idempotency Keys

| Key name | Scope | Storage | TTL | Generation | Collision handling | Used by |
|----------|-------|---------|-----|-----------|--------------------|---------|
| `{key_name}` | `per-{resource}` / `global` | `{store}` | `{duration}` | `{client-provided / server-derived / system-generated}` | `{cached result / ERR_CODE / serialise}` | `SAGA-{name}`, `EP-{name}` |

(One row per key, sorted alphabetically by key name. If the system has no
 idempotency keys — a pure read-only or strictly non-replayable system:
 "No idempotency keys — the system is strictly non-replayable; every
 mutating request is assumed unique." This is rare and demands rationale.)

---

## § 4. Concurrency Rules

### Per-aggregate locking

- `{AggregateName}`: `{lock kind — advisory / row-level / distributed}` on `{key}`; acquired at `{point}`, released at `{point}`; preserves `INV-{NN}`.

(Repeat for every aggregate with locking. If none: "No per-aggregate locking
 — every aggregate relies on optimistic concurrency or natural serialisation
 by natural key.")

### Optimistic concurrency

- `{AggregateName}`: carries `{version field name}`; stale-write attempts return `{ERR_CODE}`; enforced at `{point}`.

(Repeat for every aggregate using optimistic concurrency. If none:
 "No optimistic concurrency — every write path uses locking from the
 previous subsection.")

### Serializability expectations

- `{OperationA}` and `{OperationB}` must not interleave — preserves `INV-{NN}`; enforced by `{mechanism}`.

(Repeat for every such pair or set. If none: "No cross-operation
 serializability rules — every operation pair is independent.")

### Parallel-safe operations

- `{OperationName}` requires no coordination — `{justification, e.g., read-only / commutative}`.

(Repeat for every operation explicitly permitted to run in parallel. This
 list is positive: it prevents future agents from adding locks where none
 are needed.)

---

## § 5. Temporal Ordering Invariants

- `{One-sentence ordering fact — e.g., "A repository's first push creates the record; subsequent pushes require it to exist."}` (cites `INV-{NN}`; enforced by `{mechanism — lock from § 4 / wait-step in SAGA-{name} / compensating transition in SM-{name}}`)

(Repeat for every temporal invariant. If none: "No temporal ordering
 invariants beyond those enforced by state-machine transition guards in § 1.")

---

## § 6. Event Emission Timing

| `EVT-name` | Emitted by | Emission point | Transactional? | Durability |
|------------|-----------|----------------|----------------|------------|
| `EVT-{name}` | `SM-{entity}: {from} → {to}` / `SAGA-{name} step {N}` | `inside DB transaction` / `after commit (outbox)` / `before commit (optimistic)` | `yes` / `no` | `{durability statement}` |

(One row per event in INTERFACES § 7. No event is omitted. If the system has
 no events: "No events — the system is request/response only. See
 INTERFACES § 7.")

---

## § 7. Relationship to Other Artifacts

- **DOMAIN.md** owns the invariants; this document preserves them through state-machine transitions and saga compensations. Every `INV-NN` cited here exists in DOMAIN § 9.
- **ARCHITECTURE.md** owns components; this document sequences them in sagas. Every component named in § 2 exists in ARCHITECTURE § 2.
- **INTERFACES.md** owns per-endpoint and per-event wire shapes; this document owns the cross-endpoint sequencing (sagas) and the in-process state changes (SMs). Every `EP-name` and `EVT-name` cited here exists there.
- **ERRORS.md** owns the code registry; this document cites codes for illegal transitions, saga non-retryable failures, and concurrency conflicts. Every `ERR_CODE` cited here exists in ERRORS.md § 3.
- **USE_CASES.md** owns the actor-driven scenarios; this document names the `UC-NN` every SM and SAGA realises.
- **QUALITY.md** derives metrics and SLOs from the transitions and saga timings named here — saga duration, transition counts, compensation rates.
- **SECURITY.md** maps `THREAT-NN` entries onto state transitions and saga steps where security-relevant; citations flow from SECURITY into this document, not the reverse.
- **OPERATIONS.md** derives runbooks for stuck sagas and compensation playbooks from § 2 and § 4.
- **SPECs** for stateful units cite the exact `SM-*` or `SAGA-*` they implement and must use state names and transition event names from this document verbatim.
- **/system-verify** exercises saga happy paths and selected failure modes from § 2.

---

## § 8. Open Questions

- [ ] {Question — e.g., "Should compensation of `SAGA-push` step 3 also cancel step 2's side effects, or is step 2 designed to be idempotent across cancellations? Current block assumes idempotent step 2, but DOMAIN's `Push` aggregate description suggests step 2 holds a reservation that must be released."}
  - **Option A:** Step 2 is naturally idempotent; compensation of step 3 leaves step 2's effects in place. Simpler; matches current block. Requires explicit confirmation from `/domain`.
  - **Option B:** Step 2 holds a reservation; compensation chain must include step 2's release. More correct if reservation is observable; complicates the saga.
  - **Recommendation:** Option B, pending confirmation from DOMAIN that the reservation is observable. If confirmed, add a row to § 2's `SAGA-push` compensations table releasing the reservation.

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- State machines per stateful aggregate, with states, initial / terminal markers, full transition table (guards, effects, invariants preserved), illegal-transition table with `ERR_CODE` citations, and a parsing mermaid diagram
- Sagas per multi-step workflow, with trigger, participants (components + aggregates), numbered happy path with endpoint / service citations, per-step compensations, per-step and overall timeouts, retry policy citing `ERR_CODE` classes, failure modes with observable signals
- Idempotency-key registry with scope, storage, TTL, generation, collision handling, and bidirectional saga citations
- Concurrency rules — per-aggregate locking, optimistic concurrency with conflict `ERR_CODE`s, serializability expectations, and a positive list of parallel-safe operations
- Temporal ordering invariants with `INV-NN` citations and enforcement mechanism
- Event emission timing — per `EVT-name` whether inside transaction, after commit via outbox, or before commit
- Cross-artifact relationships and bidirectional citations with DOMAIN, ARCHITECTURE, INTERFACES, ERRORS, USE_CASES
- Genuinely ambiguous saga-design decisions surfaced in § 8 Open Questions

### Out of scope

- Wire shapes of endpoints and events — owned by `/interfaces`. This document cites by `EP-name` / `EVT-name`.
- Error code definitions (`code → http_status → user_action → retryable → log_level`) — owned by `/errors`. This document cites by code string.
- Persistence schema, DB-level locking details, transaction isolation levels, column-level constraints — owned by `/data`. This document names the *rule* ("acquire advisory lock on `repo_id`"), not the SQL that implements it.
- Implementation syntax — try / catch, async / await, mutex primitives, library choices — owned by `/spec` and `/implement`.
- Observability of transitions (metric names, SLO thresholds, alert rules) — owned by `/quality`. This document names *what* is observable; `/quality` names the signal shape.
- Threat modelling on state transitions (STRIDE categorisation, authorisation checks per transition) — owned by `/security`. This document provides the transition map; `/security` overlays threats.
- Runbooks, on-call procedures, manual saga intervention playbooks — owned by `/operations`. This document provides the compensations and failure modes from which runbooks are derived.
- UI-level flow (what screen appears when, which buttons are enabled) — owned by the per-surface IA skills.
- Domain glossary, bounded contexts, entity / aggregate definitions themselves — owned by `/domain`. This document cites by aggregate name and `INV-NN`.

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `state_machines`, `sagas`, `idempotency_keys`, `concurrency_rules`, `temporal_invariants`, `tracked_events`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "eventually")
- [ ] § 8 Open Questions is present (empty with "All questions resolved." or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files beyond the referenced registries (DOMAIN § 9, ERRORS § 3, INTERFACES §§ 6–7)
- [ ] All eight sections § 1 through § 8 are present with their exact headings
- [ ] § 1 opening note either confirms every stateful aggregate has an SM, or explicitly lists SM-free aggregates with one-line reasons
- [ ] Every `SM-*` block has `Entity`, `Owning context`, `States`, `Initial state`, `Terminal states`, `Use cases`, full transition table, illegal-transition table, and mermaid diagram
- [ ] Every `SM-*` state listed in `States:` appears in the transition table or as initial / terminal
- [ ] Every illegal-transition row names an `ERR_CODE` that exists in ERRORS.md § 3 (or is surfaced in § 8 as pending ERRORS.md creation)
- [ ] Every `SM-*` mermaid diagram is valid `stateDiagram-v2` and parses
- [ ] Every `SAGA-*` block has single-form `Trigger`, `Participating components`, `Participating aggregates`, `Idempotency key`, `Use cases`, numbered happy path, full compensations table (one row per happy-path step), per-step timeout, overall timeout, retry policy, non-retryable errors, and failure-modes table
- [ ] Every `SAGA-*` compensations table has one row per happy-path step — no gaps; `none (read-only)` is a legitimate value
- [ ] Every mutating `SAGA-*` cites an idempotency key that has a row in § 3; the § 3 row reciprocally names the saga in `Used by`
- [ ] § 3 every row has `Key name`, `Scope`, `Storage`, `TTL`, `Generation`, `Collision handling`, `Used by` — no blanks; TTL is concrete (`(none)` is forbidden)
- [ ] § 4 has all four subsections (per-aggregate locking, optimistic concurrency, serializability expectations, parallel-safe operations), each populated or stating "(none)" with rationale
- [ ] Every concurrency rule in § 4 cites the `INV-NN` it preserves or `ERR_CODE` it emits
- [ ] § 5 every temporal invariant cites its `INV-NN` and names the enforcement mechanism
- [ ] § 6 has a row for every `EVT-name` in INTERFACES § 7; no event missing; `Emission point` is one of the three literal values
- [ ] Every `INV-NN` cited anywhere exists in DOMAIN § 9
- [ ] Every `ERR_CODE` cited anywhere exists in ERRORS.md § 3 (or is listed in § 8 Open Questions pending ERRORS.md)
- [ ] Every `EP-name` cited in § 2 saga steps exists in INTERFACES § 6
- [ ] Every `EVT-name` cited anywhere exists in INTERFACES § 7 (or DOMAIN § 7 for pre-wire events)
- [ ] Every `UC-NN` cited exists in USE_CASES.md
- [ ] Every component named in § 2 exists in ARCHITECTURE § 2; every aggregate named exists in DOMAIN § 5
- [ ] No implementation syntax (grep-check: `try`, `catch`, `panic`, `async`, `await`, `mutex`, `semaphore`, `goroutine`, `thread`)
- [ ] No wire schema (grep-check: `camelCase`, `snake_case`, `JSON body`, `HTTP status`, `protobuf`)
- [ ] Retired `SM-*` / `SAGA-*` / key entries retain their row with `(retired — superseded by {new-id})` markers
- [ ] Frontmatter counts match the body: `state_machines`, `sagas`, `idempotency_keys`, `concurrency_rules`, `temporal_invariants`, `tracked_events`, `open_questions`
- [ ] `status` is `complete` if § 8 is "All questions resolved." and `has_open_questions` otherwise
- [ ] Document length is within the 300–1000 line target (hard cap 1500; split by aggregate if exceeded)
