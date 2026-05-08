---
name: INTERFACES.md
description: Consolidate the machine-interface contract — global wire conventions, authentication, idempotency, versioning, endpoint catalogue with request/response shapes, event registry, and evolution policy under stable `EP-name` and `EVT-name` IDs. Use when asked to produce an INTERFACES.md, design machine interfaces, document HTTP and event contracts, define wire formats and event schemas, or consolidate API style, endpoints, events, and evolution policy into one artifact.
---

# Task: Generate INTERFACES.md — Machine-Interface Contract

## Objective

Produce an INTERFACES.md that serves as the single source of truth for how independently-developed components communicate across machine boundaries: HTTP endpoints, events, and the wire-level conventions that govern both. The document merges what older guidance often split across separate wire-format, API-style, event-registry, and evolution-policy documents, because those four are always read together by the same downstream agents. For every boundary it pins the field-casing, timestamp, ID, enum, null-versus-missing, pagination, filtering, and error-response conventions; names the authentication scheme and idempotency semantics; declares the versioning strategy; catalogues every endpoint under a stable `EP-name` with exact request/response shapes and the error codes it may emit (cited to ERRORS.md, never inlined); catalogues every event under a stable `EVT-name` with producer, consumers, delivery guarantee, and payload schema; and commits an evolution policy with compatibility promises and deprecation windows.

INTERFACES.md is the document that decides when code, tests, and other artifacts disagree about the wire. Downstream readers include every IA (for the endpoints and events they realise), `/data`, `/behavior`, `/security`, `/quality`, `/operations`, every boundary-touching per-unit `/SPEC.md`, every `/code-review`, and `/system-verification`. The defining discipline — and the commonest failure — is **every endpoint and event entry is self-sufficient**: an agent implementing either side of the boundary can produce compatible serialisation/deserialisation code from a single entry, without opening another file.

---

## Inputs

1. **DOMAIN.md** (required) — for entity and event names, bounded-context owners, and invariants that inform shapes. Every event name in § 7 must match a `EVT-name` from DOMAIN § 7; every endpoint's vocabulary must use glossary canonical forms.
2. **ARCHITECTURE.md** (required) — for components, boundaries, and cross-component flows. Boundaries enumerated in § 1 trace to architecture boundaries; transformation points named in § 6 per-endpoint entries trace to architecture data-flow sections.
3. **Surface IAs that call the system** (auto-discovered — one per surface that exists) — `WEB_IA.md`, `CLI_IA.md`, `MOBILE_IA.md`, `TUI_IA.md`, `VOICE_IA.md`. Each IA enumerates the `EP-name` it invokes; every such citation must match an endpoint entry in § 6.
4. **Existing type definitions in the codebase** (optional, auto-discovered) — TypeScript interfaces, Rust structs, Protobuf messages, OpenAPI specs. When refining, these are evidence of already-implemented shapes. Conflicts between sources are never resolved silently — they become open questions.
5. **Existing INTERFACES.md** (auto-discovered — only if refining) — read fully; every assigned `EP-name` and `EVT-name` is permanent. New endpoints and events take new unused IDs. Retired IDs remain with `(retired)` markers.
6. **Known contract issues** (optional) — bug reports pinning specific mismatch fixes. Every pinned fix must land in the relevant entry or be surfaced as an open question if the correct shape is ambiguous.

Read set size: 2 required + up to 5 surface IAs + optional existing sources. Read all required inputs and all IAs in scope end-to-end. Truncated reads cause silent contract drift.

---

## Workflow

Interface-contract construction proceeds in seven phases: boundary identification, global convention extraction, auth/idempotency/versioning policy, endpoint catalogue, event catalogue, evolution policy, validation. Phases are sequential; revisit earlier phases if a later phase reveals an inconsistency.

### Phase 1: Boundary Identification

Enumerate every machine boundary in the system. A boundary exists wherever:

- Two components developed by different teams or agents communicate over a network protocol (HTTP, WebSocket, message bus).
- A client and server are implemented in different languages or repositories.
- An intermediary component receives one shape and forwards another.

For each boundary capture: descriptive name (`Server API ↔ Client`, `Server ↔ Git Engine`, `Worker ↔ Message Bus`), components on each side (from ARCHITECTURE.md), native casing on each side (the language-native convention — `snake_case` for Rust, `camelCase` for TypeScript), and coverage status (new | partially documented in ARCHITECTURE.md | fully documented elsewhere; if fully documented, the row cites the location and the boundary does not repeat the documentation).

Boundaries already fully documented elsewhere appear in § 1's table with a pointer — never re-documented below. The document fills gaps.

### Phase 2: Global Wire Conventions

For each boundary (or for the system as a whole if conventions are uniform), pin the defaults that apply to every endpoint and event unless explicitly overridden. § 2 carries one subsection per boundary covering every dimension in the Rules section below: field casing on the wire, timestamp format, ID format, enum representation, numeric types and JS safe-range, nullability convention, empty-array-vs-missing, pagination (cursor or offset; default and max page size), filtering and sorting query-parameter conventions, and the canonical error response shape.

The canonical error response shape is **pinned** in this document (the wire shape is INTERFACES' concern); error **codes** are cited by string from ERRORS.md (never inlined). State the shape once; every endpoint's error responses conform to it.

A per-endpoint override is legitimate when an external standard governs (OAuth2 endpoints use snake_case per RFC 6749) or when a legacy endpoint predates a convention change (explicitly noted with a deprecation plan). Every override appears both in § 2 (as an exception) and in the endpoint's § 6 entry.

### Phase 3: Authentication, Idempotency, Versioning

**§ 3 Authentication** — per boundary: scheme (`bearer JWT`, `session cookie`, `mTLS`, `HMAC-signed request`, `API key`), token shape and issuer (`JWT issued by IdentityService, 15-min TTL`), scope vocabulary (`repo:read`, `repo:write`, `admin:*`), refresh flow if applicable, and the list of endpoints that are explicitly unauthenticated (if any — listed by `EP-name`).

**§ 4 Idempotency** — which HTTP methods are idempotent by convention (`GET`, `PUT`, `DELETE`). The idempotency-key header name (`Idempotency-Key`), accepted formats (ULID, UUID), storage duration (e.g., 15 minutes post-completion), and replay semantics (server returns the stored response for replays within the window; rejects replays with divergent bodies with a specific error code cited from ERRORS.md).

**§ 5 Versioning** — the chosen strategy: URL path (`/v1/...`), version header (`Api-Version: 2026-04-19`), or additive-only (no version identifier; new fields always allowed). State the deprecation signals: `Sunset` and `Deprecation` response headers, a docs URL, and the advance-notice window per change class (detailed in § 8). Cite `ADR-NN` from ARCHITECTURE.md when the decision was architectural.

### Phase 4: Endpoint Catalogue

§ 6 is the endpoint registry. For each endpoint produce one block following the template in `references/endpoint-template.md`. Read that file and apply it verbatim — field order, heading level, bullet labels, and JSON fence style are the contract that downstream agents parse.

Assignment rules:

- **Stable IDs.** Assign `EP-{action}` in kebab-case naming the action, not the URL path (`EP-push-create`, `EP-repo-read`, `EP-user-list`). Once assigned, the ID is permanent. Retired endpoints keep their ID with a `(retired — superseded by EP-{name})` marker in § 6.
- **Sort within boundary.** Group endpoints by boundary (subsection per boundary from § 1). Within each boundary sort by `EP-name` alphabetically.
- **Traceability.** Every endpoint references at least one `UC-NN` from USE_CASES.md (via DOMAIN.md). Orphan endpoints reveal either an unneeded endpoint or a missing use case — surface as an open question.
- **Error code citations only.** The entry lists `ERR_CODE` values by string; every cited code must exist in ERRORS.md. Never inline error bodies — the canonical shape lives in § 2, the code registry in ERRORS.md.
- **Event emissions.** If the endpoint emits any events, list their `EVT-name`s; the events themselves live in § 7.

### Phase 5: Event Catalogue

§ 7 is the event registry. If the system uses no events at all (pure request/response), § 7 reads exactly: `No events — the system is request/response only.` and the `events:` frontmatter count is `0`.

Otherwise, for each event produce one block following the event template in `references/endpoint-template.md` verbatim. Assignment rules:

- **Stable IDs.** Kebab-case, past-tense, naming a state change (`EVT-push-accepted`, `EVT-order-placed`, `EVT-subscription-renewed`). Commands (`EVT-place-order`, `EVT-charge-customer`) are forbidden — commands belong to endpoint requests or BEHAVIOR.md.
- **Name alignment with DOMAIN.** Every event whose name appears in DOMAIN.md § 7 must match exactly. INTERFACES.md may add events that DOMAIN.md lacks (integration events, webhooks) — these are named here and noted as non-domain in the block.
- **Schema version.** Every event carries a `schema-version` header; state the current version in the block and the evolution rules (additive-only within a version; breaking change requires `EVT-{name}-v2`).

### Phase 6: Evolution Policy

§ 8 pins the promises the system makes to consumers. Four subsections:

- **Compatibility promise.** What the system promises never to do silently (rename fields, change field types, remove fields, change endpoint URLs, change enum values, reduce response status-code variety). State the promise in imperative terms: "we will never rename an existing wire field"; "adding optional fields is always allowed"; "removing fields requires a deprecation cycle per class C below".
- **Change classes.** Classify changes by impact: additive (new optional fields, new endpoints, new events) — allowed any time; compatible (loosening constraints, widening enums by adding values with a documented default-branch) — allowed with announcement; breaking (removals, renames, narrowings, type changes) — requires deprecation cycle.
- **Deprecation timeline.** Per change class, the minimum window between announcement and effect. Additive changes are immediate. Compatible changes ship with the next minor release. Breaking changes ship with a minimum 6-month deprecation window (or the project's documented equivalent), with `Deprecation` and `Sunset` headers on every affected response from announcement through sunset.
- **Migration guidance.** What consumers must do for each change class: update type definitions, migrate to the new endpoint/event, adjust retry logic if idempotency semantics differ, refresh cached templates if error `ui_message_key` mappings changed (per ERRORS.md § 6).

Cite `ADR-NN` where the evolution policy references an architectural decision.

### Phase 7: Validation

Before finalising, verify:

- Every boundary in § 1 has: name, components, native casing on each side, wire casing, endpoint count, event count, coverage status.
- § 2 covers every convention dimension (casing, timestamps, IDs, enums, numeric types, null-vs-missing, empty-array-vs-missing, pagination, filtering, error response shape) for every boundary.
- Every endpoint in § 6 has every bullet from the endpoint template populated — no blanks, no `(tbd)`, no `(n/a)` except where the template explicitly permits `(none)`.
- Every endpoint references at least one `UC-NN`.
- Every error code cited in any § 6 entry appears in ERRORS.md § 3 (sample at least five endpoints if the project has > 20 endpoints).
- Every event cited in any `Events emitted:` bullet appears in § 7.
- Every event in § 7 has every bullet from the event template populated.
- Every `EVT-name` that also appears in DOMAIN.md § 7 has a matching name (no drift).
- § 8 contains all four subsections: compatibility promise, change classes, deprecation timeline, migration guidance.
- No implementation details appear (grep-check: `serde`, `rename_all`, `@JsonProperty`, `Jackson`, `Gson`, `Pydantic`, `validator`, `zod`, `io-ts`).
- Frontmatter counts match the body: `boundaries`, `endpoints`, `events`, `open_questions`.

Update frontmatter. `status` is `complete` if § 10 reads `All questions resolved.`, `has_open_questions` otherwise.

---

## Rules

These rules govern the output. Violations are detected by the quality checklist.

### 1. Zero ambiguity per entry

An agent implementing either side of the boundary must produce compatible code from the entry alone — no inference, no cross-referencing another file except for the ERRORS.md code definitions. Every field carries its wire type, required/optional status, and a one-line description.

### 2. Stable IDs forever

`EP-name` and `EVT-name` are assigned once and never renumbered, never reused, never silently deleted. Retired IDs remain with `(retired — superseded by {new-id})` markers. The IDs outlive regenerations; quotes do not.

### 3. Field casing on the wire, always

The JSON in endpoint and event bodies shows the casing as it travels on the wire, regardless of the language-native casing on either side. If Rust's native `snake_case` becomes `camelCase` on the wire, the block shows `camelCase`. The language-native form is an implementation detail.

### 4. Error codes are citations, not inlined

Every endpoint lists `ERR_CODE` values from ERRORS.md. Never inline the message, never restate the HTTP status-to-meaning mapping. The canonical error response shape lives in § 2; the code registry in ERRORS.md. An endpoint entry that enumerates inline error bodies is a rule violation.

### 5. Conflicting sources → Open Questions

When existing type definitions disagree (TypeScript interfaces say `userId`, Rust structs say `user_id`, OpenAPI spec says `uid`), the agent does not silently pick one. The conflict becomes a question in § 10 with options, tradeoffs, and a recommendation. Silent resolution is exactly the failure INTERFACES.md exists to prevent.

### 6. No implementation hints

No "use `serde rename_all = camelCase`", no "use `@JsonProperty`", no "generate with OpenAPI code-gen", no library names, no decorator syntax. INTERFACES.md specifies the wire contract; SPECs specify how each unit implements it.

### 7. Transformation points are explicit

When a handler receives one shape and emits a different shape (server between client and internal service), both shapes appear in the entry and the transformation is named. Implicit reshaping causes contract drift.

### 8. Global conventions plus explicit overrides

§ 2 pins the defaults; every override appears both in § 2 (as a named exception with rationale) and in the overriding endpoint's § 6 entry. An endpoint that silently diverges from the boundary convention is a bug, not a feature.

### 9. Endpoint IDs name the action, not the path

`EP-push-create`, not `EP-POST-v1-repos-repoId-push`. Action-named IDs survive URL restructuring; path-named IDs churn with every versioning change.

### 10. Event names are past-tense state changes

`EVT-push-accepted`, not `EVT-accept-push` or `EVT-push-accept`. Commands (present-tense or imperative) belong to endpoints or BEHAVIOR.md. An imperative event name is a modelling error.

### 11. Self-sufficiency beats elegance

A reader agent cannot open files to resolve ambiguity. Prefer inlining a type note in every field (`string — required. ULID format.`) over cross-referencing a "types glossary" elsewhere. Duplication of type semantics across entries is acceptable; ambiguity is not.

### 12. No domain, data, or UI drift

INTERFACES is about the wire. Domain glossary stays in DOMAIN.md (never restate entity definitions here). Persistence schema stays in DATA.md (never name tables or columns here). UI layouts stay in IAs (never describe how a toast is rendered here). Cross-endpoint orchestration (sagas, state machines) stays in BEHAVIOR.md.

### 13. Single YAML frontmatter block

One YAML block at the top with `skill`, `date`, `status`, `boundaries`, `endpoints`, `events`, `error_shape_references`, `open_questions`. Never two blocks. Counts match the body exactly.

### 14. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "various", "and so on". Use exact IDs, exact field names, exact types, exact HTTP statuses. Unresolvable ambiguity surfaces in § 10 Open Questions.

---

## Output Format

The per-endpoint block template in § 6 and per-event block template in § 7 are defined in `references/endpoint-template.md`. The agent must read that file and apply the templates verbatim. Do not paraphrase, do not reorder bullets, do not drop fields.

```markdown
---
skill: INTERFACES.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
boundaries: {N}
endpoints: {N}
events: {N}
error_shape_references: ERRORS.md
open_questions: {N}
---

# INTERFACES — {ProductName}

> Consolidated machine-interface contract. Every HTTP endpoint, every event,
> every wire-level convention, and the evolution policy live here. Downstream
> artifacts cite endpoints and events by stable `EP-name` / `EVT-name` IDs.
> Error code definitions live in ERRORS.md; this document pins the shape of
> error responses and lists which codes each endpoint may emit.

## § 1. Overview & Boundaries

| Boundary | Components | Native casing (each side) | Wire casing | Endpoints | Events | Coverage |
|----------|-----------|---------------------------|-------------|-----------|--------|----------|
| `{boundary name}` | `{Component A}` ↔ `{Component B}` | `snake_case` / `camelCase` | `camelCase` | `{N}` | `{N}` | `new` / `partial (see ARCHITECTURE § X)` / `external (see {path})` |

(One row per boundary. Boundaries fully documented elsewhere cite the
 location and are not re-documented in §§ 6–7.)

---

## § 2. Global Wire Conventions

### `{boundary name}`

- **Field casing:** `camelCase` on the wire; servers receive / emit `camelCase` regardless of native representation.
- **Timestamp format:** ISO 8601 with timezone, always UTC (`2026-04-19T09:00:00Z`). Numeric epochs are forbidden.
- **ID format:** `{kind}` — ULID 26 chars / UUIDv4 / prefixed (`repo_abc123...`). State the format once; every ID-typed field conforms.
- **Enum representation:** `lowercase-string` — e.g., `"active"`, `"archived"`. Numeric enums forbidden.
- **Numeric types:** integers are JSON `number` and stay within JS safe range (`< 2^53`); larger integers travel as `string`. Decimals name their precision and scale.
- **Nullability:** absence of a field means unknown; explicit `null` means "known to be none". State which convention this boundary uses and apply it uniformly.
- **Empty arrays vs missing:** empty `[]` means "queried; no items found"; missing field means "not queried / not applicable".
- **Pagination:** cursor-based (`cursor`, `limit`; max `limit` = `100`, default `20`). Response carries `nextCursor` (or absent when exhausted).
- **Filtering and sorting:** query params `filter[field]=value` and `sort=field` or `sort=-field` for descending. Multiple sort keys comma-separated.
- **Error response shape:** every non-2xx response carries the § 2 error shape (field names follow this boundary's wire casing — the example below shows camelCase; snake_case boundaries use `request_id`, `docs_url`):
  ```json
  {
    "code": "string — required. UPPER_SNAKE code from ERRORS.md.",
    "message": "string — required. Human-readable developer-facing English summary.",
    "details": "object — optional. Structured payload; schema per code.",
    "requestId": "string — required. Correlation id.",
    "docsUrl": "string — optional. Link to docs for this code."
  }
  ```
  Code-to-meaning mapping lives in ERRORS.md § 3.

(Repeat for each boundary. If all boundaries share conventions, one subsection
 named `All boundaries` is sufficient.)

---

## § 3. Authentication Model

### `{boundary name}`

- **Scheme:** `{Bearer JWT | session cookie | mTLS | HMAC-signed request | API key}`
- **Token shape:** `{e.g., JWT issued by IdentityService, RS256, 15-min TTL, claims: sub, scopes, iat, exp}`
- **Scopes:** `{enumerate — "repo:read", "repo:write", "admin:*"}`
- **Refresh:** `{refresh endpoint EP-name or "clients re-authenticate on expiry"}`
- **Unauthenticated endpoints:** `EP-{name}`, `EP-{name}` — or `(none)`.

(Repeat per boundary.)

---

## § 4. Idempotency

- **Idempotent methods (convention):** `GET`, `PUT`, `DELETE`.
- **Idempotency-Key header:** `Idempotency-Key` — ULID or UUID accepted.
- **Retention window:** `{15 minutes post-completion}` — during this window a replay returns the stored response.
- **Replay semantics:** identical body within window → stored response replayed; divergent body within window → `{ERR_CODE_FROM_ERRORS_MD}` (`409`).
- **Non-idempotent endpoints:** `POST` endpoints without idempotency keys are safe to retry only after reading the response; state per-endpoint in § 6.

---

## § 5. Versioning

- **Strategy:** `{URL path /v1/... | Api-Version header (date) | additive-only}`. See `ADR-{NN}` in ARCHITECTURE.md.
- **Current version:** `{v1 | 2026-04-19}`.
- **Deprecation signals:** `Deprecation: true` header on affected responses; `Sunset: <date>` header names the removal date; `Link: <docs-url>; rel="deprecation"` points to migration docs.
- **Advance notice:** per change class — see § 8.

---

## § 6. Endpoints

(One subsection per boundary. Within each boundary, one block per endpoint,
 sorted alphabetically by `EP-name`. Each block follows the template in
 `references/endpoint-template.md` verbatim.)

### Boundary: `{boundary name}`

{Endpoint blocks per template, one after another.}

---

## § 7. Events

(Either the fixed sentence below or one block per event following the template
 in `references/endpoint-template.md`. Sort alphabetically by `EVT-name`.)

`{If none: "No events — the system is request/response only."}`

{Otherwise: event blocks per template.}

---

## § 8. Evolution Policy

### Compatibility promise

- We will never rename an existing wire field.
- We will never change the type of an existing wire field.
- We will never remove a field, endpoint, or event without a deprecation cycle per the timeline below.
- We will never narrow an enum's values without a deprecation cycle.
- Adding optional fields, new endpoints, and new events is always allowed.

### Change classes

- **Additive** — new optional fields, new endpoints, new events, new enum values at the *wide* end. Allowed any release.
- **Compatible** — loosening a constraint, widening an enum with a default-branch contract. Allowed with release notes.
- **Breaking** — removals, renames, narrowings, type changes, status-code changes for existing meanings. Requires the deprecation cycle below.

### Deprecation timeline

- **Additive:** immediate.
- **Compatible:** ship in the next minor release with RELEASE_NOTES entry.
- **Breaking:** minimum `{6 months}` advance notice. From announcement, every affected response carries `Deprecation: true` and `Sunset: <date>`. Sunset date is no earlier than announcement + window.

### Migration guidance

- **Additive:** none required.
- **Compatible:** update type definitions to accept the new shape; no runtime change.
- **Breaking:** update callers to the replacement `EP-name` / `EVT-name`; update switch statements keyed on enum values; adjust retry logic if idempotency semantics differ; refresh UI templates if error `ui_message_key` mappings changed (per ERRORS.md § 6).

---

## § 9. Relationship to Other Artifacts

- **DOMAIN.md** owns the ubiquitous language. Every term used in this document is defined there. Every `EVT-name` that also appears in DOMAIN § 7 must match exactly.
- **ARCHITECTURE.md** owns components and boundaries. Boundaries in § 1 trace to architecture boundaries; transformation points trace to architecture data-flow sections.
- **IAs** (`WEB_IA.md`, `CLI_IA.md`, `MOBILE_IA.md`, `TUI_IA.md`, `VOICE_IA.md`) cite `EP-name` for the endpoints they invoke and `EVT-name` for the events they subscribe to. Every such citation maps to a block here.
- **ERRORS.md** owns the error-code registry; this document owns the error response *shape* (§ 2) and the per-endpoint *code citations* (§ 6).
- **DATA.md** owns persistence schemas — this document never names tables or columns.
- **BEHAVIOR.md** owns state machines and saga-level sequencing across multiple endpoints — this document owns only per-endpoint contracts.
- **SECURITY.md** maps `THREAT-NN` entries to endpoints and events where relevant; endpoints cite threat refs via their authentication bullet where material.
- **QUALITY.md** owns metric and SLO catalogues; this document does not name metrics.
- **SPECs** for boundary-touching units cite the exact `EP-name` they implement and must use the field names from this document verbatim.

---

## § 10. Open Questions

- [ ] {Question — e.g., "`EP-repo-read` response includes both `createdAt` (camelCase, ISO 8601) in the spec and `created_at` (snake_case, epoch seconds) in existing Rust types. Which is authoritative for the public boundary?"}
  - **Option A:** camelCase ISO 8601 — matches § 2 global convention; requires Rust-side transformation.
  - **Option B:** snake_case epoch — matches current implementation; violates § 2; requires spec exception.
  - **Recommendation:** Option A; transformation is already required at the boundary for other fields, so adding one field does not change implementation complexity materially.

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Boundary enumeration with native-casing/wire-casing convention
- Global wire conventions (field casing, timestamps, IDs, enums, numeric types, nullability, empty-array-vs-missing, pagination, filtering, error response shape)
- Authentication model per boundary (scheme, tokens, scopes, refresh, unauthenticated endpoints)
- Idempotency rules (method conventions, idempotency-key header, retention window, replay semantics)
- Versioning strategy and deprecation signalling
- Endpoint catalogue with stable `EP-name` IDs and full per-endpoint blocks (request/response shapes, auth, idempotency, error code citations, events emitted, transformation notes, rate limits)
- Event catalogue with stable `EVT-name` IDs and full per-event blocks (kind, producer, consumers, delivery guarantee, ordering, payload, headers, version, evolution rules)
- Evolution policy (compatibility promise, change classes, deprecation timeline, migration guidance)
- Genuinely ambiguous shapes surfaced in § 10 Open Questions

### Out of scope

- Error code registry (code → status → message → user action → retry) — owned by `/errors`. This document pins the shape in § 2 and cites codes in § 6.
- Bounded contexts, entities, aggregates, value objects, glossary — owned by `/domain`.
- Persistence schema (tables, columns, indexes, migrations) — owned by `/data`.
- State machines, sagas, compensating actions, cross-endpoint orchestration — owned by `/behavior`.
- UI surfaces (pages, screens, commands, dialogs, prompts) — owned by IA skills.
- Implementation details (serialisation library choices, decorators, code generators, validator packages) — owned by `/spec`.
- Threat modelling, STRIDE categorisation, security controls — owned by `/security`.
- Metric catalogues, SLO definitions, alert thresholds — owned by `/quality`.
- Config vars, deployment topology, runbooks — owned by `/operations`.
- Use case catalogue — owned by `/use-cases`.

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `boundaries`, `endpoints`, `events`, `error_shape_references`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "and so on")
- [ ] § 10 Open Questions is present (empty with "All questions resolved." or with genuine ambiguities only)
- [ ] Output is self-contained — every endpoint and event entry is readable without opening another file except ERRORS.md for code definitions
- [ ] All ten sections § 1 through § 10 are present with their exact headings
- [ ] § 1 table has one row per boundary with components, native casing, wire casing, endpoint count, event count, coverage status
- [ ] § 2 covers every convention dimension (field casing, timestamps, IDs, enums, numeric types, nullability, empty-array-vs-missing, pagination, filtering, error response shape) for every boundary
- [ ] § 2 pins the canonical error response shape; no endpoint re-defines it
- [ ] § 3 names scheme, token shape, scopes, refresh, and unauthenticated endpoints per boundary
- [ ] § 4 names idempotent methods, idempotency-key header, retention window, and replay semantics
- [ ] § 5 states the versioning strategy and names the deprecation signals
- [ ] § 6 contains every endpoint block following `references/endpoint-template.md` verbatim — every bullet populated, no blanks
- [ ] Every `EP-name` is kebab-case and names the action (`EP-push-create`), not the URL path
- [ ] Every endpoint references at least one `UC-NN`
- [ ] Every endpoint's `Response errors:` lists only `ERR_CODE` citations (no inline bodies, no status-to-meaning mappings)
- [ ] Every endpoint's `Events emitted:` lists `EVT-name` values from § 7 — or `(none)`
- [ ] § 7 either reads `No events — the system is request/response only.` or contains event blocks following the template verbatim
- [ ] Every `EVT-name` is kebab-case and past-tense, naming a state change (`EVT-push-accepted`, never `EVT-place-order`)
- [ ] Every `EVT-name` that appears in DOMAIN.md § 7 matches exactly
- [ ] § 8 contains all four subsections: compatibility promise, change classes, deprecation timeline, migration guidance
- [ ] § 9 names relationships with DOMAIN, ARCHITECTURE, IAs, ERRORS, DATA, BEHAVIOR, SECURITY, QUALITY, SPEC
- [ ] No implementation details leaked (grep-check: `serde`, `rename_all`, `@JsonProperty`, `Jackson`, `Gson`, `Pydantic`, `validator`, `zod`, `io-ts`, `openapi-generator`)
- [ ] No UI vocabulary leaked (grep-check: `page`, `screen`, `button`, `modal`, `toast`, `dashboard`)
- [ ] No persistence vocabulary leaked (grep-check: `table`, `column`, `index`, `migration`, `primary key`, `foreign key`)
- [ ] Conflicts between sources are either resolved with rationale in the entry or surfaced in § 10
- [ ] Retired `EP-name` / `EVT-name` entries retain their row with `(retired — superseded by {new-id})` markers
- [ ] Frontmatter counts match the body: `boundaries`, `endpoints`, `events`
- [ ] `status` is `complete` if § 10 is "All questions resolved." and `has_open_questions` otherwise
- [ ] Document length is within the 500–2000 line target; if exceeding 2000, split by bounded context into `INTERFACES_{context}.md` with an `INTERFACES_INDEX.md`
