---
name: ERRORS.md
description: Build the unified error / result taxonomy — classification tree, canonical error shape, alphabetical error-code registry with stable UPPER_SNAKE codes, wrapping rules across component boundaries, per-surface rendering, and a deprecation policy. Use when asked to create the error taxonomy, register error codes, build the error registry, define the unified error model, or produce an ERRORS.md.
---

# Task: Generate ERRORS.md — Unified Error / Result Taxonomy

## Objective

Produce an ERRORS.md that serves as the single source of truth for every error code the system can emit, its HTTP status, CLI exit code, UI message key, user action, retry semantics, log level, and owning bounded context. The document fixes a classification tree and per-class policy, pins the canonical error response body, publishes the complete alphabetical registry of stable UPPER_SNAKE error codes, names the wrapping and translation rules across component boundaries, states how each error is rendered on each surface, and commits a deprecation policy for retiring codes. An agent reading this document alone can — for any code the system emits — cite its HTTP status, CLI exit, UI key, user action, retry semantics, and owner, without opening INTERFACES.md or any IA.

ERRORS.md sits between INTERFACES.md (which owns the *shape* of error responses and per-endpoint error-code listings) and the per-surface IAs (which own user-facing error UX) and resolves them to one taxonomy. It is small, stable, widely cited, and read by IAs (for user-facing message copy), `/spec` (for the codes a unit may emit), `/review` (for contract conformance), and `/system-verify` (when diagnosing failures). The defining discipline — and the commonest violation — is **codes are stable**: once a code string is assigned, it is never reused, never re-meaning'd, never silently deleted.

---

## Inputs

1. **INTERFACES.md** (required) — for the canonical error response shape and the per-endpoint error-code listings cited at `EP-name` boundaries. Every error code mentioned in an endpoint response in INTERFACES.md must have a registry row here.
2. **Surface IAs** (required — one per surface that exists in the project) — `WEB_IA.md`, `CLI_IA.md`, `MOBILE_IA.md`, `TUI_IA.md`, `VOICE_IA.md`. Only the IAs that were produced for this project are read; surfaces the project does not build have no IA and no column in § 5. Each IA's error-surface section names the `ui_message_key` or template it expects.
3. **DOMAIN.md** (required) — § 2 Bounded Contexts supply the `owner_context` value on every row; § 9 Invariants Index supplies the `INV-NN` references that certain codes signal (a uniqueness invariant violation surfaces as a specific code, named here).
4. **Existing error handling code** (optional, auto-discovered) — if source trees are present, scan `throw`, `raise`, `panic`, `return Err(...)`, `respondError(...)` sites and collect codes already emitted that are not yet documented. Surface undocumented codes as rows or as open questions if the mapping is ambiguous.
5. **Existing ERRORS.md** (auto-discovered — only if refining) — if an ERRORS.md already exists at the expected path, read it fully. Every assigned code string is permanent. New codes take new unused strings (never reuse a retired code's string). Deprecated codes retain their row with `deprecated_since` / `removed_in` columns populated.

Read set size: 3 required (INTERFACES, DOMAIN, each surface IA in scope) + optional source scan + optional prior ERRORS. Read all IAs and INTERFACES.md end-to-end before drafting. Omitted reading causes two specific failure modes: codes cited in INTERFACES or an IA without a registry row (contract drift) and codes emitted by code that never reach the registry (silent divergence).

---

## Workflow

Error taxonomy construction proceeds in seven phases: harvest, classification, canonical shape, registry, wrapping, per-surface mapping, and validation. Phases are sequential. Revisit earlier phases if a later one reveals a missing code or a contradiction.

### Phase 1: Code Harvest

Collect every error code candidate from four sources:

- **INTERFACES.md per-endpoint response listings** — every code named in the `errors:` field of an endpoint. These are the externally-visible codes; every one must land in § 3.
- **Surface IAs' error-surface sections** — every `ui_message_key` an IA expects to render; the corresponding code must exist here.
- **DOMAIN.md invariants** — every `INV-NN` that is enforced at runtime typically signals through a specific code when violated. Name one code per enforced invariant, citing the `INV-NN` in the row's `references` column.
- **Existing source code** (if available) — `throw`, `raise`, `panic`, `Err(...)`, error-response helpers. Normalise to UPPER_SNAKE. Codes found only in code with no INTERFACES / IA reference are either internal-only (log-level `error` or `critical`; never reach users) or missing from INTERFACES — surface as an open question if the boundary is unclear.

Deduplicate by meaning, not by spelling. `NOT_FOUND` and `MISSING` and `UNKNOWN_RESOURCE` are the same code in three forms; pick one canonical UPPER_SNAKE string. Record the rejected forms so the registry does not accumulate synonyms.

### Phase 2: Classification Tree

Partition the harvested codes into three top-level classes and their conventional HTTP families:

- **Client errors (4xx family)** — the request was invalid or unauthorised given server state: `auth`, `authz`, `validation`, `not-found`, `conflict`, `rate-limit`, `idempotency`.
- **Server errors (5xx family)** — the server or a dependency failed to complete a well-formed request: `unhandled bug`, `dependency failure`, `overload`, `degraded`.
- **Protocol errors** — the request could not be parsed or versioned: `malformed request`, `unsupported version`, `unsupported media type`.

Write § 1 as a short bulleted tree plus a policy table. The policy table has exactly these columns, one row per sub-class: `Class`, `Retryable?`, `Idempotent retry safe?`, `User-actionable?`, `Log level`, `Paged?`. Every cell is a single-word answer (`yes` / `no` / `warn` / `error` / `critical`); no prose.

Class totals land in frontmatter: `client_error_codes:`, `server_error_codes:`, `protocol_error_codes:`. They sum to `error_codes:`.

### Phase 3: Canonical Error Shape

State the JSON body the system emits on HTTP errors. If INTERFACES.md pins the shape, cite it verbatim and do not re-design. If INTERFACES.md delegates to ERRORS.md for the shape, define it here with these fields: `code` (stable UPPER_SNAKE, required), `message` (human-readable developer-facing English summary, stable across versions, required), `details` (optional structured payload — schema per code), `request_id` (correlation id, required), `docs_url` (optional).

Follow the shape block with a one-paragraph statement of how non-HTTP surfaces use the same taxonomy: CLI surfaces emit `message` to stderr and set `cli_exit`; UI surfaces look up `code` and render the template named by `ui_message_key` (allowing i18n without touching the API); log sinks record `code`, `request_id`, and the full structured context per `log_level`. The shape lives here; per-endpoint application of the shape lives in INTERFACES.md.

### Phase 4: Error Code Registry

§ 3 is the master registry — one row per code, sorted alphabetically by `code`. Every row populates every column:

| Column | Meaning | Allowed values |
|---|---|---|
| `code` | Stable UPPER_SNAKE string (`REPO_NOT_FOUND`, `PUSH_CONFLICT`, `AUTH_REQUIRED`). Never reused. | `[A-Z][A-Z0-9_]+` |
| `http_status` | HTTP status emitted at the INTERFACES boundary | 3-digit HTTP code |
| `cli_exit` | Process exit code for CLI surface. `0` is reserved for success; pick integers per CLI_IA's exit-code taxonomy | integer ≥ 1 or `—` if not emitted by CLI |
| `ui_message_key` | Key the UI maps to an i18n message template (or the literal template if no i18n) | short dotted string `error.repo.not_found` |
| `user_action` | One-line instruction to the user in the affected surface's voice ("Verify the repo name and try again.") | full sentence ending in `.` |
| `retryable` | Retry safety | `yes` / `yes-with-backoff` / `no` |
| `log_level` | Log level at emission | `warn` / `error` / `critical` |
| `owner_context` | Bounded context that owns this error (from DOMAIN.md § 2) | context name |
| `references` | `INV-NN` invariants this code signals, `THREAT-NN` if security-related, `EP-name` if emitted by a specific endpoint pattern | comma-separated IDs or `(none)` |

Write the table once; sort alphabetically by `code` for stability across regenerations. The row-count lands in frontmatter `error_codes:`. Retryable count lands in `retryable_codes:` (rows where `retryable` is `yes` or `yes-with-backoff`).

Keep `message` and `details` out of the registry — they are emitted at runtime from the shape in § 2, not enumerated here. The registry is a contract, not a message catalogue.

### Phase 5: Error Wrapping Rules

When an error crosses a component boundary (Worker → API, API → Client, Plugin → Host, external system → ACL), specify what happens. § 4 is a boundary × rule table with these columns: `Boundary`, `Add fields?`, `Translate codes?`, `Redact details?`, `Rationale`.

Typical boundary rules to document:
- **API ingress → client** — add `request_id`; never translate codes; in production redact `details` for codes with log level `critical`.
- **Internal component → API** — translate internal-only codes (`GIT_CORRUPT`, `CACHE_THRASH`) to the nearest external code (`INTERNAL` / `DEPENDENCY_FAILURE`); never expose internal codes to clients.
- **External system → ACL** — map foreign error vocabulary into this taxonomy; preserve the foreign code in `details.upstream_code` when useful for support.

One row per boundary. If a boundary has no wrapping rules (rare), write the row with `Add fields?` = `no`, `Translate codes?` = `no`, `Redact details?` = `no`, `Rationale` = "transparent passthrough; boundary preserves code semantics".

### Phase 6: Per-Surface Mapping

For each surface that exists in this project (one column per IA that was produced), state how codes are rendered. § 5 has one subsection per surface, in this fixed form:

- **API** — full error shape per § 2; `http_status` from the registry is the response status; `code` verbatim.
- **CLI** — `message` written to stderr; process exits with `cli_exit` from the registry; `user_action` is suffixed to stderr when surface is interactive (`isatty(stderr)`); `—` registry rows never trigger a CLI path.
- **Web UI** — surface picks `ui_message_key` from the registry and renders the IA's template for that key (toast / inline / modal per WEB_IA); `details` may be rendered for `log_level: warn` codes in developer mode.
- **Mobile** — same `ui_message_key`; rendering per MOBILE_IA (dialog / banner / toast). Offline codes render via cached templates.
- **TUI** — same `ui_message_key`; rendering per TUI_IA (status line / modal / error panel).
- **Voice** — same `ui_message_key`; TTS template per VOICE_IA; `user_action` is spoken.

Surfaces the project does not build are omitted — not listed with "(not in scope)". The scope is named in § 5's opening sentence: "Surfaces in scope: WEB, CLI." Omit the surfaces not in scope.

### Phase 7: Deprecation & Validation

**§ 6 Deprecation Policy.** State: how a code is retired (add `deprecated_since: YYYY-MM-DD` and `removed_in: vX.Y` columns to the row; do not delete the row), the minimum deprecation window (e.g., "one major version or 6 months, whichever is longer"), what consumers must do to migrate (typically: update switch statements on `code`, update i18n keys, update retry policy if the new code's retryable differs). Deprecated codes continue to appear in the registry — the row persists forever with its `deprecated_since` populated — so that consumers reading the registry after retirement still find the history.

**§ 7 Relationship to Other Artifacts.** One bullet per relationship, in this fixed order:
- INTERFACES.md cites codes by the `code` string in endpoint response listings; every such citation must match a registry row in § 3.
- IAs cite codes via `ui_message_key`; every key appearing in an IA must map to a row in § 3.
- SPECs list which codes the unit may emit; `/spec` reads this file to draw from the registry, not to invent new codes.
- SECURITY.md maps `THREAT-NN` entries to codes where the error is a security signal; the registry's `references` column records the linkage.
- QUALITY.md's alert catalogue may reference specific codes by string; the `log_level` and `retryable` semantics land there verbatim.
- BEHAVIOR.md may cite codes on state-machine edges where a transition is blocked; the code communicates the reason domain-side.

**§ 8 Validation.** Before finalising verify:
- Every code cited in INTERFACES.md per-endpoint listings appears in § 3 (sample at least five endpoints if the registry has > 20 codes).
- Every `ui_message_key` cited in every IA appears in § 3.
- Every aggregate in DOMAIN.md § 5 whose invariants are enforced at an API boundary has at least one code in § 3 referencing the relevant `INV-NN`.
- Every row has every column populated — no blanks, no `n/a`, no `tbd`.
- The registry is alphabetically sorted by `code`.
- Every code string is unique; no duplicate strings; no reused retired strings.
- Every `owner_context` value matches a bounded context name in DOMAIN.md § 2 exactly.
- Frontmatter counts are consistent: `client_error_codes + server_error_codes + protocol_error_codes == error_codes`.

Update frontmatter counts. `status` is `complete` if § 8 Open Questions is "All questions resolved." and `has_open_questions` otherwise.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Codes are stable — never reused, never re-meaning'd

Once a code string is assigned, it belongs to that meaning forever. Renaming a code is forbidden. Re-meaning a code (keeping the string, changing what it signals) is forbidden. Retiring a code removes it from emission but leaves the row in the registry with `deprecated_since` / `removed_in` populated so downstream consumers reading the registry after retirement find its history.

### 2. UPPER_SNAKE, never dotted, never spaced

Every `code` matches `[A-Z][A-Z0-9_]+`. No dots, no dashes, no spaces, no lowercase. Dotted keys exist — they are `ui_message_key`s — but they are a separate column and a separate namespace.

### 3. Codes are not HTTP statuses

One HTTP 404 can represent ten distinct codes (`REPO_NOT_FOUND`, `USER_NOT_FOUND`, `BRANCH_NOT_FOUND`, `OBJECT_NOT_FOUND`, ...). The `http_status` column is a mapping, not an identity. A row that has only an HTTP status and no code string is not a row — it is a leak from INTERFACES.md. Every row has a code.

### 4. Single-word-per-cell in the registry

The columns `retryable`, `log_level`, `http_status`, `cli_exit`, and `owner_context` carry one value each, from the defined enum or integer set. No prose, no qualifications, no footnotes. Nuance lives in the `user_action` column (a sentence) or in the wrapping / surface-mapping sections — never in the registry.

### 5. Every code has a user_action

Even internal-only errors have a user-facing action: "Contact support and quote request_id." The `user_action` column is never empty, never `n/a`, never "(internal only)". If the code is genuinely never rendered to any user on any surface, it does not belong in the registry — it is an internal log-only signal that INTERFACES.md does not need to cite.

### 6. Every code has an owner_context

`owner_context` names the bounded context from DOMAIN.md § 2 that owns the error's meaning. Orphan codes — codes with no owning context — reveal unmodeled domain concepts; surface them as open questions rather than assigning a stub context name.

### 7. No code in cited artifacts without a registry row

A code string appearing in INTERFACES.md (endpoint responses), an IA (user-facing copy), a SPEC (unit emissions), or SECURITY.md (threat responses) must have a registry row in § 3. This is the contract the document enforces. The validation phase samples citations and confirms each maps to a row.

### 8. Alphabetical sort, forever

§ 3 is sorted alphabetically by `code`. The sort is stable across regenerations — an agent regenerating the document must produce the same row order an earlier agent produced, modulo the new rows added at their alphabetical position. This reduces diff noise and makes review tractable.

### 9. Retired codes stay in the registry

Deleted rows break consumers that still recognise the string. Retire codes by adding `deprecated_since: YYYY-MM-DD` and `removed_in: vX.Y` columns to the row and leaving it in place. Only after `removed_in` has passed in the real world does the row move to an archival section at the bottom of § 3 — never silently deleted.

### 10. Every code's retryable semantics are testable

`retryable` is one of three values. `yes` means a mechanical retry is safe without operator intervention; `yes-with-backoff` means a retry is safe if and only if the client observes exponential backoff (per `/operations`' retry budget); `no` means any retry is a bug. Surfaces and clients read this value and wire retry policy from it — ambiguity here causes real outages.

### 11. Log level and pager gating per class, not per code

The policy table in § 1 sets the default log level and pager gating per class. Individual codes inherit the class default unless their row overrides it with a documented reason in `user_action` or in an inline note below the registry. Mass-override across a class is a sign the class boundary is wrong — revisit § 1.

### 12. No wire-schema design in ERRORS

The canonical shape in § 2 is *referenced* or *pinned* here, but field-by-field wire design — versioning, content negotiation, media type, status-code conventions at large — belongs to INTERFACES.md. An `ERRORS.md` that starts enumerating HTTP status conventions for the whole API is out of scope; delete that content.

### 13. No UI copy style in ERRORS

`ui_message_key` is the handle; the *template* it resolves to belongs to the IA. A registry that starts carrying long marketing-voice error copy is over-reaching — `user_action` stays to a single sentence, and `ui_message_key` stays to a dotted identifier. Layout (toast / inline / modal) is the IA's concern.

### 14. Single YAML frontmatter block

One YAML frontmatter block at the top containing common fields (`skill`, `date`, `status`) and errors-specific fields (`error_codes`, `retryable_codes`, `client_error_codes`, `server_error_codes`, `protocol_error_codes`, `open_questions`). Never emit a second YAML block. Counts match the body exactly; `client + server + protocol == error_codes`.

### 15. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "some", "a few". Use exact code strings, exact HTTP statuses, exact context names, exact `INV-NN` / `THREAT-NN` IDs. Unresolvable ambiguity surfaces in § 8 Open Questions with options, tradeoffs, and a recommendation.

---

## Output Format

```markdown
---
skill: ERRORS.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
error_codes: {N}
retryable_codes: {N}
client_error_codes: {N}
server_error_codes: {N}
protocol_error_codes: {N}
open_questions: {N}
---

# ERRORS — {ProductName}

> Unified error / result taxonomy. Every error the system emits is registered
> here with HTTP status, CLI exit, UI message key, user action, retry
> semantics, log level, and owning bounded context. Downstream artifacts cite
> codes by `code` string. Wire shape of error bodies lives in INTERFACES.md;
> UI rendering lives in the per-surface IAs; alert thresholds live in
> QUALITY.md; threat-to-code mapping lives in SECURITY.md.

## § 1. Classification Tree

- **Client errors (4xx family)** — the request was invalid or unauthorised.
  - `auth` — identity not established
  - `authz` — identity established, authority insufficient
  - `validation` — request syntactically or semantically invalid
  - `not-found` — addressed resource does not exist
  - `conflict` — request violates current server state
  - `rate-limit` — client exceeded quota
  - `idempotency` — replayed request with divergent payload
- **Server errors (5xx family)** — the server or a dependency failed.
  - `unhandled bug` — unexpected exception crossed a boundary
  - `dependency failure` — a downstream system failed
  - `overload` — resource budget exhausted
  - `degraded` — a subsystem is operating in reduced-capability mode
- **Protocol errors** — the request could not be parsed or versioned.
  - `malformed request` — request bytes did not parse
  - `unsupported version` — API version negotiation failed
  - `unsupported media type` — content-type not accepted

### Policy per class

| Class | Retryable? | Idempotent retry safe? | User-actionable? | Log level | Paged? |
|-------|------------|------------------------|------------------|-----------|--------|
| `auth` | no | — | yes | warn | no |
| `authz` | no | — | yes | warn | no |
| `validation` | no | — | yes | warn | no |
| `not-found` | no | — | yes | warn | no |
| `conflict` | no | — | yes | warn | no |
| `rate-limit` | yes-with-backoff | yes | yes | warn | no |
| `idempotency` | no | — | yes | warn | no |
| `unhandled bug` | no | — | no | error | yes |
| `dependency failure` | yes-with-backoff | yes | no | error | yes |
| `overload` | yes-with-backoff | yes | no | error | yes |
| `degraded` | no | — | no | warn | yes |
| `malformed request` | no | — | yes | warn | no |
| `unsupported version` | no | — | yes | warn | no |
| `unsupported media type` | no | — | yes | warn | no |

(Repeat/adjust the policy table if classes differ for this project. Every cell
 is a single-word answer.)

---

## § 2. Canonical Error Shape

HTTP error responses carry the following JSON body (pinned here; INTERFACES.md
cites this section verbatim).

```json
{
  "code": "string — stable UPPER_SNAKE code from § 3. Required.",
  "message": "string — human-readable developer-facing English summary. Stable across versions. Required.",
  "details": "object — optional structured payload; schema per code.",
  "request_id": "string — correlation id. Required.",
  "docs_url": "string — optional link to docs."
}
```

Non-HTTP surfaces consume the same taxonomy without the wire envelope:

- **CLI** renders `message` to stderr and sets the process exit to `cli_exit`
  from § 3.
- **UI surfaces** (WEB / MOBILE / TUI) look up `code` and render the IA
  template identified by `ui_message_key` from § 3. Layout (toast / inline /
  modal / banner) is the IA's concern; the key lands here.
- **VOICE** renders the TTS template identified by `ui_message_key` via
  VOICE_IA's prompt catalogue.
- **Log sinks** record `code`, `request_id`, `owner_context`, and `details`
  at the `log_level` from § 3.

---

## § 3. Error Code Registry

Surfaces in scope: {WEB | CLI | MOBILE | TUI | VOICE — list only those that
exist in this project.}

| `code` | `http_status` | `cli_exit` | `ui_message_key` | `user_action` | `retryable` | `log_level` | `owner_context` | `references` |
|--------|--------------|-----------|------------------|---------------|-------------|-------------|-----------------|--------------|
| `{CODE_STRING}` | `{3-digit}` | `{int or —}` | `{dotted.key}` | "{one sentence.}" | `{yes | yes-with-backoff | no}` | `{warn | error | critical}` | `{ContextName}` | `{INV-NN, THREAT-NN, EP-name or (none)}` |

(Alphabetical by `code`. One row per code. No blanks, no `n/a`. Deprecated
 rows retain their entry with two additional inline columns:
 `deprecated_since` and `removed_in`.)

---

## § 4. Error Wrapping Rules

| Boundary | Add fields? | Translate codes? | Redact details? | Rationale |
|----------|-------------|------------------|-----------------|-----------|
| `API ingress → client` | `request_id`, `docs_url` | `no` | yes for `log_level: critical` in production | `request_id` enables support; internal details never leave the edge |
| `Internal component → API` | `no` | `yes` — internal-only codes collapse to `INTERNAL` / `DEPENDENCY_FAILURE` | yes | internal vocabulary never leaks to clients |
| `External system → ACL` | `details.upstream_code`, `details.upstream_message` | `yes` — foreign codes map to taxonomy entries | no | preserves debugging trail without exposing foreign dialect |

(Repeat for every component boundary the system crosses. If a boundary is
 transparent, still include a row stating so — silence is ambiguous.)

---

## § 5. Per-Surface Mapping

### API
- Response body per § 2.
- `http_status` from § 3.
- `code` verbatim.
- `details` redacted per § 4 rules.

### CLI
- `message` to stderr.
- Process exit = `cli_exit` from § 3.
- `user_action` appended to stderr when `isatty(stderr)`.
- Registry rows with `cli_exit: —` never reach CLI surfaces.

### Web UI
- Template picked by `ui_message_key` from § 3; layout per WEB_IA.
- `details` rendered only in developer mode for `log_level: warn` codes.

### Mobile
- Template picked by `ui_message_key`; layout per MOBILE_IA.
- Offline codes render via cached templates; `user_action` is the primary affordance.

### TUI
- Template picked by `ui_message_key`; placement per TUI_IA (status line, modal, error panel).

### Voice
- TTS template per VOICE_IA prompt catalogue, keyed by `ui_message_key`.
- `user_action` is spoken; `request_id` is spelled on request.

(Omit subsections for surfaces the project does not build.)

---

## § 6. Deprecation Policy

- **How to retire a code:** add `deprecated_since: YYYY-MM-DD` and
  `removed_in: vX.Y` columns to the registry row. Do not delete the row.
- **Minimum deprecation window:** {one major version or 6 months, whichever
  is longer — adjust per project release cadence}.
- **Consumer migration:** consumers must update switch statements keyed on
  `code`, update i18n keys to the replacement `ui_message_key`, and adjust
  retry policy if the replacement code's `retryable` differs from the retired
  code.
- **Post-removal:** after `removed_in` ships, the row moves to the archival
  section at the bottom of § 3 and is never re-emitted.

---

## § 7. Relationship to Other Artifacts

- **INTERFACES.md** cites codes by `code` string in per-endpoint response
  listings; every such citation maps to a row in § 3.
- **IAs** (`WEB_IA.md`, `CLI_IA.md`, `MOBILE_IA.md`, `TUI_IA.md`,
  `VOICE_IA.md`) cite codes via `ui_message_key`; every key maps to a row
  in § 3.
- **SPECs** list which codes the unit may emit; `/spec` draws from the
  registry and must not invent new codes.
- **SECURITY.md** maps `THREAT-NN` entries to codes where the error is a
  security signal; linkage lives in the `references` column of § 3.
- **QUALITY.md** alert catalogue may reference codes directly; `log_level`
  and `retryable` semantics land there verbatim.
- **BEHAVIOR.md** may cite codes on state-machine edges where a transition
  is blocked by a runtime check; the code names the reason domain-side.
- **DOMAIN.md** supplies `owner_context` for every row; every `INV-NN`
  referenced in `references` lives in DOMAIN § 9.

---

## § 8. Open Questions

- [ ] {Question — e.g., "Should `USER_SUSPENDED` return HTTP 403 (forbidden)
      or 409 (conflict)? 403 matches the auth/authz family policy; 409
      signals state-driven refusal. The choice affects client retry logic:
      under 403 the CLI advises the user to contact an admin, under 409 it
      advises retrying after remediation."}
  - **Option A:** 403 — aligns with `authz` class policy; consistent with
    `FORBIDDEN` semantics; user-action is "contact your administrator".
  - **Option B:** 409 — signals that the *account* is in a state that
    refuses the request; differentiates from permission-denied errors.
  - **Recommendation:** 403, with `details.suspension_reason` carrying the
    nuance; keeps retry semantics aligned with the class policy.

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Classification tree (client / server / protocol) with per-class policy table
- Canonical error response body (referenced from or pinned by INTERFACES.md)
- Complete alphabetical registry of every error code with all nine columns populated
- Wrapping rules per component boundary (add fields, translate codes, redact details)
- Per-surface rendering rules for every surface in scope
- Deprecation policy with minimum window and migration requirements
- Cross-artifact relationships with INTERFACES, IAs, SPECs, SECURITY, QUALITY, BEHAVIOR, DOMAIN
- Genuinely ambiguous code-to-HTTP-status mappings surfaced in § 8 Open Questions

### Out of scope

- Field-by-field wire design of error responses, content negotiation, media types, versioning syntax — owned by `/interfaces`
- HTTP status code conventions in general (which status for which semantics across the whole API) — owned by `/interfaces`
- UI error layout, copy style, typography, animation of error states — owned by the per-surface IA skills; ERRORS supplies the key and the one-sentence `user_action`, not the template body
- Alert thresholds, SLO impact, pager routing policies — owned by `/quality` (this document supplies `log_level` and a paged/not-paged flag per class only)
- Threat modelling, STRIDE categorisation, security controls — owned by `/security` (this document records the linkage via the `references` column)
- Runbooks, on-call procedures, incident response templates — owned by `/operations`
- Full glossary, entities, aggregates, value objects, invariants, bounded contexts themselves — owned by `/domain`
- Per-unit error emission code, try/catch structure, panic handlers — owned by `/spec` and `/implement`
- Use case catalogue, cross-cutting scenario narratives — owned by `/use-cases`

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `error_codes`, `retryable_codes`, `client_error_codes`, `server_error_codes`, `protocol_error_codes`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "and so on", "many", "some")
- [ ] § 8 Open Questions is present (empty with "All questions resolved." or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] All eight sections § 1 through § 8 are present with their exact headings
- [ ] § 1 classification tree has client, server, and protocol branches with sub-classes, followed by a per-class policy table whose every cell is a single-word answer
- [ ] § 2 canonical error shape names every required field (`code`, `message`, `request_id`) and every optional field (`details`, `docs_url`); states how non-HTTP surfaces consume the taxonomy
- [ ] § 3 registry has every code with all nine columns populated — no blanks, no `n/a`, no `tbd`
- [ ] Every `code` matches `[A-Z][A-Z0-9_]+` (UPPER_SNAKE); no dots, no dashes, no spaces, no lowercase
- [ ] Every `code` string is unique; no duplicate strings; no reused retired strings
- [ ] § 3 rows are sorted alphabetically by `code`
- [ ] Every row's `retryable` is one of `yes` / `yes-with-backoff` / `no`
- [ ] Every row's `log_level` is one of `warn` / `error` / `critical`
- [ ] Every row's `owner_context` matches a bounded context name in DOMAIN.md § 2 exactly
- [ ] Every row's `user_action` is a single sentence ending in `.`; never empty, never `n/a`
- [ ] Every row's `references` either lists `INV-NN` / `THREAT-NN` / `EP-name` IDs or states `(none)`
- [ ] Every code cited in INTERFACES.md per-endpoint response listings appears as a row in § 3 (sample at least five endpoints if registry has > 20 codes)
- [ ] Every `ui_message_key` cited in every surface IA in scope appears as a row in § 3
- [ ] § 4 has one row per component boundary the system crosses; transparent boundaries are still listed explicitly
- [ ] § 5 contains exactly the surfaces in scope for this project (no subsections for surfaces not built)
- [ ] § 6 specifies the minimum deprecation window and the three consumer-migration steps (switch statements, i18n keys, retry policy)
- [ ] Deprecated codes retain their registry row with `deprecated_since` and `removed_in` populated; deleted rows do not exist
- [ ] § 7 names relationships with INTERFACES, IAs, SPEC, SECURITY, QUALITY, BEHAVIOR, DOMAIN
- [ ] Frontmatter counts satisfy `client_error_codes + server_error_codes + protocol_error_codes == error_codes`
- [ ] Frontmatter `retryable_codes` equals the count of rows where `retryable` is `yes` or `yes-with-backoff`
- [ ] `status` is `complete` if § 8 is "All questions resolved." and `has_open_questions` otherwise
- [ ] Document length is within the 200–600 line target (hard cap 1000)
