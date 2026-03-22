---
name: SPEC.md
description: Create a specification for the work unit. Use when asked to create a SPEC.md file for a work unit, write specs or technical documentation for a work unit.
---

# Task: Create SPEC.md for a Work Unit

## Objective

Given a work unit definition and its surrounding architecture documents, produce a SPEC.md file that is **implementation-complete**: an AI agent reading only this file can produce the implementation without making any architectural, behavioral, or naming decisions of its own. The gap between SPEC.md and code should be purely mechanical — translating unambiguous prose into syntax.

---

## Inputs

You will receive:

1. **A work unit definition** — contains the unit ID, concept, repo, dependencies, file list, test descriptions, estimated LOC, and interface summary.
2. **Architecture documents** — describe the overall system, file structure, conventions, and technology choices.
3. **Decision documents** — record architectural decisions and their rationale.
4. **Security documents** — describe auth model, access control, and trust boundaries.
5. **SPEC.md files of dependency units** — the specs of units this unit depends on. These are the source of truth for what types, functions, and interfaces are available to import.

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

### No-code rule

- The spec must not contain code or pseudocode unless a specific algorithm is non-obvious and cannot be described unambiguously in prose (e.g., a specific hash partitioning scheme, a binary format layout). In that case, minimal pseudocode is permitted with a note explaining why prose was insufficient.

---

## Output Format

The SPEC.md file must follow this exact structure. Every section is mandatory. If a section has no content (e.g., no constants for the unit), include the heading with "None." underneath.

```markdown
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

---

## 7. Error Catalog

| Error variant/type | Trigger condition | HTTP status (if applicable) | User-facing message format | Source |
|-------------------|-------------------|----------------------------|---------------------------|--------|
| ... | ... | ... | ... | this unit / propagated from {unit} |

For propagated errors: state whether they are re-wrapped, mapped to a different type, or passed through unchanged.

---

## 8. Test Specification

For each test:

### Test: {descriptive name}

- **Setup:** What state/fixtures must exist before the test runs. Be specific (e.g., "a bare git repo initialized at a temp directory", not "appropriate setup").
- **Action:** What function/route to call, with what arguments.
- **Assertion:** Exact expected outcome — return value, status code, side effects to verify, state changes.
- **Teardown:** Cleanup required (if any beyond default temp directory cleanup).

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
