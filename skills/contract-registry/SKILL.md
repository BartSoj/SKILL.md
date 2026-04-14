---
name: CONTRACT_REGISTRY.md
description: Generate a wire-format contract registry for all HTTP boundaries between independently-developed components. Use when asked to create a contract registry, document API contracts, define wire formats, produce a CONTRACT_REGISTRY.md, or establish a shared source of truth for HTTP request/response shapes.
---

# Task: Generate CONTRACT_REGISTRY.md — Wire-Format Contract Registry

## Objective

Produce a CONTRACT_REGISTRY.md that serves as the single source of truth for HTTP request/response shapes across every boundary where independently-developed components communicate. An agent or developer reading this document knows the exact JSON field names, types, casing conventions, and structure for every endpoint — eliminating the class of bugs where each side independently derives wire formats from English prose and produces incompatible implementations.

---

## Inputs

1. **Architecture documents** (required) — system structure, component boundaries, endpoint tables, technology choices. The primary source for identifying HTTP boundaries and endpoint definitions.
2. **Decision documents** (optional) — architectural decisions that affect wire format (e.g., "push as core write operation", "optimistic concurrency via parentSha").
3. **Existing type definitions** (auto-discovered) — source files that define request/response types (TypeScript interfaces, Rust structs, Go structs, Protobuf messages). These reveal what the codebase has already implemented, useful as reference and for detecting existing mismatches.
4. **Existing codebase** (auto-discovered) — route handlers, API clients, serialization annotations. Used to infer conventions and discover undocumented contracts.
5. **Known issues** (optional) — bug reports, integration test failures, or triage reports that reveal specific contract mismatches. These ensure the registry explicitly pins the fields and shapes where problems occurred.

---

## Workflow

Contract registry generation proceeds in four phases: boundary identification, convention extraction, contract documentation, and validation.

### Phase 1: Boundary Identification

Identify every HTTP boundary in the system where independently-developed components communicate. A "boundary" exists wherever:

- Two components developed by different teams or agents communicate over HTTP
- A client and server are implemented in different languages or repos
- A component acts as an intermediary, receiving from one service and forwarding (possibly transformed) to another

For each boundary, record:

| Field | What to capture |
|-------|----------------|
| **Name** | Descriptive label (e.g., "Server API — Client Boundary", "Server — Git Engine Boundary") |
| **Components** | Which components are on each side (e.g., "TS Server ↔ Rust CLI, Web Client") |
| **Convention risk** | Why independent derivation is dangerous here (e.g., "different languages with different naming conventions") |
| **Coverage status** | Whether existing documentation already specifies wire formats for this boundary |

**Skip boundaries that are already fully documented** with complete wire-format JSON schemas in existing architecture documents. Note their location and move on. The registry fills gaps, not duplicates existing documentation.

**Focus on boundaries that lack wire-format documentation** — where endpoint tables list method/path/description but not request/response JSON shapes. These are where contract drift occurs.

### Phase 2: Convention Extraction

For each boundary that needs documentation, determine the wire-format conventions by examining the codebase and architecture documents.

**Discover:**

1. **Field naming convention.** What casing does each side use natively? What casing appears on the wire? (e.g., "TypeScript server returns camelCase JSON; Rust CLI expects snake_case natively but must deserialize camelCase with serde rename annotations")
2. **Exceptions to the convention.** Are there endpoints governed by external standards that override the project convention? (e.g., OAuth2 endpoints use snake_case per RFC 6749 regardless of the project's camelCase convention)
3. **Serialization patterns.** How does each side serialize/deserialize? (e.g., `JSON.stringify` preserves JS property names; Rust serde defaults to field names but supports `rename_all`)
4. **Transformation points.** Where does a component receive one shape and forward a different shape? (e.g., server receives snake_case from git engine, transforms to camelCase before sending to clients)

**Document these as a "Global Wire Conventions" section** at the top of the registry. This section applies to all endpoints unless explicitly overridden per-endpoint.

### Phase 3: Contract Documentation

For each endpoint on each under-documented boundary, produce a contract entry. The goal is zero ambiguity — an agent implementing either side of the boundary can produce correct serialization/deserialization code from this entry alone.

**Per-endpoint contract structure:**

1. **Method and path** — exact HTTP method and URL pattern (e.g., `POST /api/repos/{repoId}/push`)
2. **Authentication** — what auth is required (e.g., "Bearer token, authenticated user must have write access to repo")
3. **Request body** — exact JSON shape with:
   - Every field name (in wire-format casing)
   - Type for each field (string, number, boolean, array of X, object with shape Y)
   - Required vs optional for each field
   - Constraints (max length, allowed values, format patterns)
   - Example value where the type alone is ambiguous
4. **Response body per status code** — exact JSON shape for each response status:
   - Success response (200/201/204) with every field
   - Error responses (400/401/403/404/409/500) with error shape
5. **Transformation notes** — if the endpoint sits between two boundaries (e.g., server between client and internal service), document both the incoming and outgoing shapes and the transformation logic

**Sources for deriving contracts (in priority order):**

1. Existing type definitions in the codebase (highest fidelity — actual implemented types)
2. Architecture document endpoint descriptions and data flow sections
3. Decision documents that specify wire-format choices
4. Codebase route handlers and API client code
5. Known issues that reveal the correct shape (when a bug report says "server returns X but client expects Y", the correct shape goes in the registry)

When sources conflict, flag the conflict as an open question. Do not silently pick one — the conflict itself is the bug that the registry exists to resolve.

### Phase 4: Validation

After documenting all contracts, validate the registry:

1. **Completeness check.** Every endpoint listed in architecture document endpoint tables has a corresponding entry in the registry (or is explicitly noted as covered by existing documentation elsewhere).
2. **Internal consistency.** Field names and types are consistent within each boundary's convention. No endpoint accidentally uses snake_case in a camelCase boundary without an explicit exception note.
3. **Cross-boundary consistency.** Where the same data flows through multiple boundaries (e.g., client → server → internal service), the transformation chain is documented end-to-end and the shapes are compatible.
4. **Conflict resolution.** Every discovered conflict between sources is either resolved (with rationale) or surfaced as an open question.

---

## Output Format

```markdown
---
skill: CONTRACT_REGISTRY.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions}
boundaries_documented: {N}
endpoints_documented: {N}
conflicts_found: {N}
conflicts_resolved: {N}
---

# CONTRACT_REGISTRY

> Single source of truth for HTTP wire formats across all component boundaries.
> Every request/response shape is authoritative — implementations and test mocks
> must conform to these definitions.

## Boundaries Overview

| Boundary | Components | Convention | Endpoints | Coverage |
|----------|-----------|------------|-----------|----------|
| {name} | {component A} ↔ {component B} | {camelCase / snake_case / mixed} | {N} | {new / existing-docs-at-path} |

(Repeat for each boundary. Boundaries with existing complete documentation
reference the document location and are not re-documented below.)

---

## Global Wire Conventions

{State the default field naming convention for each boundary.
 State the serialization approach for each language/framework involved.
 State any global exceptions (e.g., OAuth2 endpoints use snake_case per RFC).
 State transformation rules for intermediary components.}

---

## {Boundary Name}

### `{METHOD} {/path}`

**Auth:** {requirement}

**Request:**

```json
{
  "fieldName": "type — required/optional. Description. Constraints.",
  "anotherField": "type — required/optional. Description."
}
```

**Response `{status code}`:**

```json
{
  "fieldName": "type. Description.",
  "anotherField": "type. Description."
}
```

**Response `{error status code}`:**

```json
{
  "error": "string. Error message."
}
```

**Transformation notes:** {If this endpoint transforms data between boundaries,
describe the incoming shape, outgoing shape, and what changes. If no
transformation: omit this line.}

(Repeat for each endpoint on this boundary.)

---

(Repeat the boundary section for each documented boundary.)

---

## Relationship to Other Documents

- **Architecture documents:** This registry supplements architecture endpoint tables
  with complete wire-format schemas. Architecture documents should reference this
  registry for JSON shapes.
- **Unit SPECs:** SPECs for units on either side of a boundary must reference the
  specific CONTRACT_REGISTRY entries they implement and use wire-format field names
  from this document.
- **Test mocks:** All HTTP response mocks in tests must use field names and casing
  from this registry, not the implementing language's native convention.

---

## Open Questions

- [ ] {Question — e.g., "Field X is `camelCase` in server types but `snake_case` in the endpoint table. Which is authoritative?"}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Identifying HTTP boundaries between independently-developed components
- Documenting wire-format conventions (field casing, serialization, exceptions)
- Producing per-endpoint contract entries with exact JSON request/response shapes
- Flagging conflicts between existing sources and surfacing as open questions
- Documenting transformation logic where a component reshapes data between boundaries

### Out of scope

- Internal API contracts already fully documented in architecture documents — reference them, do not duplicate
- Business logic descriptions — those belong in architecture documents and unit SPECs
- Implementation guidance — that belongs in PLANs
- Generating or modifying code — the registry is a design document, not a code generator
- Database schemas or internal data models — only the HTTP wire format matters here
- Non-HTTP communication (message queues, file-based, in-process) — future extension if needed

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `boundaries_documented`, `endpoints_documented`)
- [ ] Every HTTP boundary in the system is identified, with coverage status noted
- [ ] Global wire conventions section states field naming convention, serialization approach, and exceptions
- [ ] Every endpoint on under-documented boundaries has a contract entry with method, path, auth, request shape, and response shape(s)
- [ ] Every field in every request/response shape has a name (in wire-format casing), type, and required/optional status
- [ ] Transformation notes are present for endpoints where a component reshapes data between boundaries
- [ ] Conflicts between sources are either resolved with rationale or surfaced as open questions
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] Boundaries already documented elsewhere are referenced, not re-documented
