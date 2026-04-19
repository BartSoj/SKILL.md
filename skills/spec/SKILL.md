---
name: SPEC.md
description: Produce an implementation-complete specification for a single work unit — file manifest, public interface contract with full signatures, ordered behavioral paths, error catalog bound to registered `ERR_CODE`s, test specifications, and design-layer citations (`INV-NN`, `EVT-name`, `SM-*`, `SAGA-*`, `METRIC-*`, `SLO-*`, `THREAT-NN`, `MIT-NN`, `UC-NN`, data aggregates). Use when asked to create a SPEC.md for a work unit, write a unit specification, translate a work unit into an implementable contract, pin the public interface and error catalog for a unit, or produce a SPEC.md.
---

# Task: Create SPEC.md for a Work Unit

## Objective

Produce a SPEC.md that serves as the single source of truth for implementing one work unit: a file manifest, every public symbol with a full signature, every behavioral path as ordered prose, every error as an `ERR_CODE` row bound to ERRORS.md § 3, every test as a setup/action/assertion triple, and every design-layer citation (invariants, events, state machines, sagas, metrics, SLOs, threats, mitigations, use cases, data aggregates) the unit touches. The defining discipline — and the definition of "done" — is that **the gap between SPEC and code is purely mechanical**: an implementing agent reading only this file (and the dependency SPECs it cites) produces the implementation without making any architectural, behavioral, or naming decisions of its own.

The commonest violation is silently picking an answer for a design-level question instead of surfacing it. When the design suite is silent, ambiguous, or contradictory on a question the unit must answer — a wire shape, an error code, a lock granularity, a field name — the question belongs in § 10 Open Questions with options, tradeoffs, and a recommendation. Do not invent. The SPEC must never introduce a design decision that is not already traceable to DOMAIN, INTERFACES, BEHAVIOR, ERRORS, QUALITY, SECURITY, DATA, or USE_CASES.

---

## Inputs

1. **Work unit definition** (required) — the entry from WORK_UNITS.md for this unit. Contains the unit ID, concept, repo, dependencies, file list, test descriptions, estimated LOC, and interface summary.
2. **Architecture and design suite** (required) — ARCHITECTURE.md plus the design registries whose stable IDs section 1 cites: DOMAIN.md, INTERFACES.md, BEHAVIOR.md, ERRORS.md, QUALITY.md, SECURITY.md, DATA.md, USE_CASES.md. The subset actually loaded is whatever the unit's declared SPEC reads in WORK_UNITS require plus the registries whose IDs the unit touches.
3. **Decision documents** (optional, auto-discovered) — any `ADR-NN` referenced from the work unit entry or from ARCHITECTURE.md. Discovery mechanism: follow `ADR-NN` citations out of WORK_UNITS and ARCHITECTURE and read each referenced ADR.
4. **SPEC.md files of dependency units** (required, auto-discovered) — the specs of every unit this unit depends on. Discovery mechanism: walk the dependency list in the WORK_UNITS.md entry for this unit and load each dependency's SPEC.md. These are the source of truth for the exact type names, function signatures, and interface shapes the new unit may import.
5. **Prior SPEC.md for this unit** (optional, auto-discovered — only if refining) — read fully; preserve every resolved open question, every assigned internal-symbol name, and every `Size hint` already calibrated against earlier drafts. Re-run citations against the current registries — do not assume stale `INV-NN` or `ERR_CODE` are still valid.

Read-set size: WORK_UNITS + DOMAIN are always read; 3–6 further artifacts depending on declared SPEC reads in WORK_UNITS (dependency SPECs + the design-layer registries this unit actually cites). Read every dependency SPEC end-to-end — truncated reads on dependencies cause silent signature drift.

---

## Rules

### Completeness rules

- Every public symbol must have a full signature: name, parameters (with names and types), return type, async/sync.
- Every error condition must be enumerated with its trigger, error type, and user-facing consequence (HTTP status, exit code, message format).
- Every constant, default value, environment variable name, timeout, size limit, or magic number must be stated explicitly with its value.
- Every file must be described at the level where an agent can produce it without referring to other sections — each file entry is a self-contained mini-spec.
- Every behavioral path (happy path, each error path, edge cases) must be described step by step in prose.
- The Open Questions section must be empty. If you cannot resolve a question, surface it to the user rather than leaving it open.

### Precision rules

- Use exact names — function names, type names, field names, file paths, route paths, crate/package names. Do not use placeholders like "appropriate error" or "relevant fields."
- When referencing something from a dependency unit, use its exact exported name and the file it comes from.
- Describe behavior as ordered steps ("1. Acquire write lock on repo_id. 2. Open repository. 3. ..."), not as vague summaries ("handles the request appropriately").
- Distinguish between "returns error" and "returns empty/default" — state which one, and the exact value.

### Scope rules

- The spec must explicitly state what is OUT of scope. Closely related functionality that belongs to other units must be named and excluded.
- Do not spec anything that is not part of this unit's deliverable. If a function will be implemented by a future unit, do not describe its behavior — only reference its expected signature if needed for type-checking.
- Do not add features, optimizations, or error handling beyond what the work unit definition calls for.

### Consistency rules

- All type names, function signatures, and route paths must be consistent with dependency units' SPECs. If a dependency exports `TreeEntry { path: String, sha: String, kind: EntryKind }`, use those exact field names.
- If the architecture documents specify a convention (e.g., vertical modules, error mapping pattern), the spec must follow it without restating the convention — just apply it.

### Interface contract rules

When the unit implements code on either side of an HTTP boundary (client that sends requests/deserializes responses, or server that defines route handlers/serializes responses):

- The spec must reference the specific INTERFACES.md entries (or equivalent contract document) that define the wire format for the endpoints this unit touches. If no interface contract exists, derive wire formats from architecture documents and state them explicitly in the spec — do not leave them implicit.
- Request/response types in the spec must use the exact field names and types from the interface contract. Do not independently derive wire formats from English prose descriptions.
- The spec must explicitly state the serialization convention the unit must follow (e.g., "Response struct uses `#[serde(rename_all = "camelCase")]` per interface contract wire convention" or "Route handler returns camelCase JSON per interface contract").
- When the unit is one side of a client-server pair, the spec must acknowledge the other side exists and reference the interface contract entry that both sides must conform to. This prevents each side from making independent naming choices.

### Design-layer citation rules

The SPEC is the bridge between the design suite (DOMAIN, INTERFACES, BEHAVIOR, ERRORS, QUALITY, SECURITY, DATA) and code. Every design layer the unit touches must be cited by stable ID in section 1 "Design References" — never by prose, never by re-stated content. Citations resolve to the authoritative registry; the implementer does not have to guess what `INV-7` or `METRIC-push-duration-ms` means.

- **Domain invariants (`INV-NN` from DOMAIN.md § 9).** If the unit's behavior preserves, enforces, or potentially threatens any invariant, list the IDs in section 1 and name the mitigation in section 4 wherever a behavioral step crosses the risk.
- **Domain events (`EVT-name` from DOMAIN § 7 / INTERFACES § 7).** If the unit emits or consumes any event, list it in section 1 with the emission or consumption point and specify the exact trigger in section 4.
- **State machines and sagas (`SM-entity: {from} → {to}` and `SAGA-name` from BEHAVIOR §§ 1–2).** If the unit implements a state-machine transition or a saga step, cite the exact transition or step ID in section 1 and match the transition's guards / effects / invariants in section 4.
- **Observability signals (`METRIC-*`, `SLO-*`, `SPAN-*` from QUALITY §§ 3–5).** If the unit emits metrics, participates in an SLO, or owns spans, list the IDs in section 1 and state emission locations (which function, which handler, which saga step) in section 4.
- **Security threats and mitigations (`THREAT-NN`, `MIT-NN` from SECURITY §§ 5–6).** If the unit is cited as the implementation location for any `MIT-NN`, or faces any `THREAT-NN` relevant to its surface, list both in section 1 and name the control implementation in section 4.
- **Error codes (`ERR_CODE` from ERRORS.md § 3).** Every error the unit may emit, map, or propagate uses the exact `ERR_CODE` UPPER_SNAKE string from the registry. Do not invent new codes in the SPEC. If a condition the unit must signal has no registered code, surface it as an open question for `/ERRORS.md` to resolve — then re-run this SPEC.

Layers that do not apply to this unit get an explicit `None — {reason}` line in section 1, not silent omission.

### Single YAML frontmatter block

- Exactly one YAML frontmatter block at the top of the SPEC.md output, never two — even when refining an existing SPEC, merge into a single block. The block carries every field enumerated in the Output Format template (`skill`, `date`, `status`, `unit`, `files_specified`, `tests_specified`, `errors_specified`, `estimated_loc_prod`, `estimated_loc_test`, `open_questions`).
- Every count in the frontmatter matches the body exactly: `files_specified` is the number of file entries in § 9; `tests_specified` is the number of tests in § 8; `errors_specified` is the number of rows in § 7's table; `open_questions` is the number of unresolved questions in § 10 (zero when § 10 reads "All questions resolved."); `estimated_loc_prod` and `estimated_loc_test` are the sums of per-file `Size hint` values for production and test files respectively.
- `status` is `complete` only when § 10 reads "All questions resolved."; `has_open_questions` when one or more questions remain unresolved; `blocked` only when a missing input (e.g., an unregistered `ERR_CODE`, an ungranted dependency SPEC) prevents authoring the SPEC and the gap is explicit in § 10.

### No-code rule

- The spec must not contain code or pseudocode unless a specific algorithm is non-obvious and cannot be described unambiguously in prose (e.g., a specific hash partitioning scheme, a binary format layout). In that case, minimal pseudocode is permitted with a note explaining why prose was insufficient.

---

## Output Format

The SPEC.md file must follow this exact structure. Every section is mandatory. If a section has no content (e.g., no constants for the unit), include the heading with "None." underneath.

```markdown
---
skill: SPEC.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
unit: {U-NN}
files_specified: {N}
tests_specified: {N}
errors_specified: {N}
estimated_loc_prod: {N}
estimated_loc_test: {N}
open_questions: {N}
---

# SPEC: {Unit ID} — {Unit Name}

## 1. Identity & Context

**Unit:** {ID}
**Name:** {name}
**Repo:** {repository name}
**Concept:** {one-paragraph purpose — why this unit exists, what problem it solves, where it sits in the system}

### Dependencies

For each dependency, state:
- Unit ID and name
- What specifically is consumed (list exact type names, function signatures, config values)
- Where it is imported from (file path)

If no dependencies: "None — this is a foundation unit."

### Design References

Cite the stable IDs from the design suite that this unit touches. For layers that do not apply to this unit, state "None — {one-line reason}" — silent omission is forbidden.

- **Domain invariants preserved:** `INV-{NN}` — one-line statement recalled from DOMAIN § 9 for each; how the unit preserves it. (From DOMAIN.md § 9.)
- **Domain events produced / consumed:** `EVT-{name}` produced at `{function / step}`; `EVT-{name}` consumed by `{handler}`. (From DOMAIN § 7 / INTERFACES § 7.)
- **State machines / sagas:** `SM-{entity}: {from} → {to}` implemented by this unit; `SAGA-{name}` step `{N}` implemented. (From BEHAVIOR §§ 1–2.)
- **Observability signals emitted:** `METRIC-{name}` (counter / histogram / gauge) at `{emission point}`; owns span `SPAN-{name}`; participates in `SLO-{name}` as a backend of its SLI. (From QUALITY §§ 3–5.)
- **Security threats faced / mitigations implemented:** faces `THREAT-{NN}` at `{surface}`; implements `MIT-{NN}` at `{function / middleware / guard}`. (From SECURITY §§ 5–6.)
- **Use cases realized:** `UC-{NN}`, `UC-{NN}`. (From USE_CASES.md.)
- **Data aggregates / tables touched:** `{table_name}` (represents `{AggregateName}`); access patterns used: `{query summary → idx_name}`. (From DATA §§ 3–5.)

---

## 2. Scope Boundary

### In scope

Exhaustive bulleted list of what this unit delivers. Each item should be concrete and verifiable (e.g., "a function that...", "an HTTP route that...", "a type definition for...").

### Out of scope

Bulleted list of closely related things this unit does NOT do. Name the unit that owns each excluded item where applicable. This prevents the implementing agent from adding helpful but out-of-scope functionality.

---

## 3. Public Interface Contract

For each exported symbol, grouped by file:

### Functions

For each function:
- Full signature: `fn name(param: Type, ...) -> ReturnType`
- Async: yes/no
- Brief description of what it does (one line)
- Reference to Behavioral Specification section for detailed behavior

### Types / Structs / Enums

For each type:
- Full definition with every field/variant
- Each field: name, type, description, and whether optional
- Any trait implementations required (Display, Serialize, etc.)

### HTTP Routes (if applicable)

For each route:
- Method and path: `GET /repos/{id}/tree`
- Auth requirement: authenticated/public
- Request: path params, query params, body schema (field names, types, required/optional, constraints)
- Response: status code(s), body schema for each status
- Reference to Behavioral Specification section for detailed behavior

---

## 4. Behavioral Specification

For each public function or route, a step-by-step prose description:

### `function_name` / `METHOD /path`

**Happy path:**
1. Step one (be precise about what happens, what is called, what is checked)
2. Step two
3. ...
n. Return value / response

**Error paths:**
- **{Error condition}:** {what triggers it} → {what is returned/thrown, with exact error variant and message format}
- ...

**Edge cases:**
- {Description of edge case} → {expected behavior}
- ...

**Concurrency:**
- What locks are acquired/released and when (if applicable)
- What can safely run in parallel

---

## 5. Internal Design Decisions

Decisions that an implementer would otherwise have to make arbitrarily. Each entry:

- **Decision:** {what was decided}
- **Rationale:** {one-line why}

Examples of what belongs here:
- Choice of data structure when multiple would work
- Error handling strategy (propagate vs. map vs. wrap)
- Whether to use a helper function or inline logic
- Ordering of operations when multiple orderings are correct
- Naming of private/internal symbols

---

## 6. Dependencies & Integration

### External packages/crates

| Package | Version constraint | Features used | Purpose |
|---------|-------------------|---------------|---------|
| ... | ... | ... | ... |

### Internal imports

| Symbol | Source unit | Source file | Used in |
|--------|-----------|-------------|---------|
| ... | ... | ... | ... |

### Registration / Mounting

How this unit is integrated into its parent:
- Module declaration (e.g., `pub mod X;` added to which file)
- Route mounting (e.g., `router.merge(X_routes())` in which file)
- Any other wiring required

### HTTP Contract References

(Include this subsection when the unit implements code on either side of an HTTP boundary. If the unit has no HTTP boundary involvement: "Not applicable — this unit does not interact with an HTTP boundary.")

For each INTERFACES entry this unit implements:

| Endpoint | Role | Registry entry | Wire convention | Transformation |
|----------|------|----------------|-----------------|----------------|
| `METHOD /path` | client / server | INTERFACES.md § {section} | {e.g., camelCase JSON} | {e.g., "Server transforms snake_case from internal service to camelCase for client" or "None"} |

---

## 7. Error Catalog

| `ERR_CODE` | Language-level error type | Trigger condition | HTTP status | CLI exit | Source |
|------------|--------------------------|-------------------|-------------|----------|--------|
| `ERR_CODE_FROM_REGISTRY` | `RustErrorVariant` / `TsExceptionClass` / … | ... | `{3-digit or —}` | `{int or —}` | this unit / propagated from `U{NN}` |

Every `ERR_CODE` must exist in ERRORS.md § 3 — do not invent codes in the SPEC. If a condition the unit must signal has no registered code, surface it as an open question in section 10 and re-run `/ERRORS.md` before regenerating this SPEC.

For propagated errors: state whether they are re-wrapped (into which code), mapped to a different code, or passed through unchanged. The mapping, if any, is recorded in ERRORS.md § 4 Wrapping Rules; cite it here.

---

## 8. Test Specification

For each test:

### Test: {descriptive name}

- **Setup:** What state/fixtures must exist before the test runs. Be specific (e.g., "a bare git repo initialized at a temp directory", not "appropriate setup").
- **Action:** What function/route to call, with what arguments.
- **Assertion:** Exact expected outcome — return value, status code, side effects to verify, state changes.
- **Teardown:** Cleanup required (if any beyond default temp directory cleanup).

**Mock fidelity requirement:** When tests use HTTP mocks (wiremock, MSW, nock, test doubles that simulate HTTP responses), mock response bodies MUST use wire-format field names and casing as defined in INTERFACES.md (or the contract source of truth for the project), not the implementing language's native convention. This ensures tests verify against realistic wire data, not fantasy responses that happen to match the client's struct layout.

---

## 9. File Manifest

For each file this unit creates or modifies:

### `{full/path/to/file.ext}` {creates | modifies}

**Purpose:** Why this file exists as a separate file (one line).

**Exports:**
- Every public symbol with full signature (mirrors section 3 but anchored to this file)

**Internals:**
- Private functions, types, constants — not their implementation but their existence and role
- Description of internal logic organization

**Imports:**
| Symbol | From |
|--------|------|
| ... | ... |

**Invariants:**
- Conditions that must always hold within this file (e.g., "every public function validates input before processing", "all filesystem access goes through the lock manager")

**Constants & Configuration:**

| Name | Value | Configurable | Source |
|------|-------|-------------|--------|
| ... | ... | hardcoded / env var / config | ... |

If no constants: "None."

**Size hint:** ~{N} lines

---

## 10. Open Questions

This section must be EMPTY before implementation begins. If any questions remain unresolved, the spec is not ready. The user will resolve open questions after reviewing the spec — do not ask during generation.

When drafting the spec, you will encounter decisions where multiple approaches are defensible and the input documents do not clearly favor one. Do NOT silently pick an answer to keep this section empty. Instead:

- If a question has one obviously correct answer given the input documents, resolve it yourself and write the decision into the appropriate section. Do not list it here.
- If a question has no clear answer — multiple valid approaches exist, or the input documents are ambiguous or silent — list it here with proposed options and tradeoffs so the user can make an informed decision.

Format for open questions:

- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {if you lean one way, say so and why — but leave the decision to the user}

Once the user resolves all questions, move each resolution into the appropriate section and replace this section with:

"All questions resolved."
```

---

## Quality Checklist

Before considering a SPEC.md complete, verify:

- [ ] Every function/route in section 3 has a corresponding behavioral description in section 4
- [ ] Every file in section 9 has exports that match section 3
- [ ] Every error in section 7 is referenced in at least one error path in section 4
- [ ] Every dependency in section 6 is either used in section 9's import maps or explicitly justified (e.g., declared for downstream units)
- [ ] Every test in section 8 maps to a behavior described in section 4
- [ ] Section 10 (Open Questions) has only genuine ambiguities — not questions with obvious answers
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] No code or pseudocode (unless justified with a note)
- [ ] Out-of-scope section names at least 3 items that an eager implementer might accidentally include
- [ ] Every constant/magic value used anywhere in the spec has an explicit value in section 9's Constants table
- [ ] If the unit touches an HTTP boundary: section 6 includes an HTTP Contract References subsection citing specific interface contract entries
- [ ] If the unit touches an HTTP boundary: request/response types use exact field names from the interface contract, not independently derived names
- [ ] If the unit uses HTTP mocks in tests: section 8 states mock fidelity requirement with wire-format field names
- [ ] Section 1 "Design References" subsection is present with all seven bullets (Domain invariants, Domain events, State machines/sagas, Observability signals, Security threats/mitigations, Use cases, Data aggregates/tables); bullets that do not apply to this unit state `None — {reason}` explicitly
- [ ] Every `INV-NN` cited in section 1 exists in DOMAIN.md § 9
- [ ] Every `EVT-name` cited in section 1 exists in DOMAIN.md § 7 or INTERFACES.md § 7
- [ ] Every `SM-{entity}: {from} → {to}` and `SAGA-{name}` cited in section 1 exists in BEHAVIOR.md §§ 1–2
- [ ] Every `METRIC-*`, `SLO-*`, and `SPAN-*` cited in section 1 exists in QUALITY.md §§ 3–5
- [ ] Every `THREAT-NN` and `MIT-NN` cited in section 1 exists in SECURITY.md §§ 5–6
- [ ] Every `UC-NN` cited in section 1 exists in USE_CASES.md
- [ ] Every table / access pattern cited in section 1 exists in DATA.md §§ 3–5
- [ ] Section 7 Error Catalog uses `ERR_CODE` UPPER_SNAKE format for every row; every `ERR_CODE` exists in ERRORS.md § 3 (no invented codes)
- [ ] Any missing-code need is surfaced in section 10 Open Questions (not silently invented in section 7)
- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `unit`, `files_specified`, `tests_specified`, `errors_specified`, `estimated_loc_prod`, `estimated_loc_test`, `open_questions`)
- [ ] Frontmatter counts match the body exactly (`files_specified` = rows in § 9, `tests_specified` = tests in § 8, `errors_specified` = rows in § 7, `open_questions` = unresolved items in § 10, `estimated_loc_prod` / `estimated_loc_test` = summed `Size hint` values)
- [ ] `status` is `complete` if section 10 Open Questions reads "All questions resolved." and `has_open_questions` otherwise (use `blocked` only when a missing input prevents authoring and § 10 makes the gap explicit)
