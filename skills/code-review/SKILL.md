---
name: CODE_REVIEW.md
description: Review code changes using parallel subagents for security, bugs, quality, contracts, test coverage, and historical context. Use when asked to review code, review changes, do a code review, check code before merge, or produce a CODE_REVIEW.md.
---

# Task: Review Code Changes — Parallel Subagent Analysis

## Objective

Produce a CODE_REVIEW.md that captures every meaningful issue in the current code changes by launching specialized review subagents in parallel, each focused on a distinct quality dimension, then aggregating their findings into a single actionable document. The review covers the explicitly provided scope, or all uncommitted changes plus branch commits not yet in main when no scope is given.

---

## Inputs

1. **Code changes** — discovered automatically from git state (see Phase 1), or scoped explicitly by the caller.
2. **Review scope** (optional) — narrows what changes to review. Can be a specific commit hash, a commit range, specific files or directories, or "staged" for staged changes only. When provided, review only the specified changes. When omitted, auto-discover all uncommitted and branch changes (default behavior).
3. **Project guidelines** — CLAUDE.md, README.md, or equivalent files in the repository root.
4. **Existing codebase** — the full repository, available to subagents for context beyond the diff.

---

## Workflow

The review proceeds in three phases. All three are mandatory and sequential — Phase 2 cannot start before Phase 1 completes, Phase 3 cannot start before all Phase 2 subagents complete.

### Phase 1: Scope Discovery

Determine what code is under review.

**If an explicit scope was provided**, use it to determine the diff. Derive the appropriate git commands from the scope — a commit hash means diffing that commit against its parent, a range means diffing the endpoints, "staged" means `git diff --cached`, specific files means scoping the diff to those paths. Use judgment to translate the caller's scope into the right git operations.

**If no scope was provided** (default), discover changes from git state:

1. **Unstaged changes:** `git diff`
2. **Staged changes:** `git diff --cached`
3. **Branch changes:** Identify the main branch (`main` or `master`), then run `git log <main-branch>..HEAD --oneline` and `git diff <main-branch>...HEAD`

Combine into a single unified picture.

**In both cases**, produce:
- **Changed files list** — every file path that appears in the diff, deduplicated.
- **Full diff content** — the combined diff. For files that appear in both branch diff and working-tree diff, use the working-tree version (latest state).
- **Commit summary** — one-line description of each relevant commit for context.

Also read project guideline files from the repository root: CLAUDE.md, README.md, CONTRIBUTING.md, .editorconfig, linter configs, or any other convention-defining file. These will be passed to every subagent.

If the combined diff is empty (no changes anywhere), stop and report "No changes to review" instead of producing CODE_REVIEW.md.

### Phase 2: Parallel Subagent Review

Launch **six subagents in parallel**, one for each review dimension. Each subagent receives:

- The full changed files list
- The full diff content
- The commit summary (for branch changes)
- All project guideline file contents
- Its specific review instructions (from the corresponding file in `references/`)

Read each agent's instructions from the `references/` directory of this skill before launching:

| Subagent | Instructions file | What it reviews |
|----------|------------------|-----------------|
| Security Auditor | `references/security-auditor.md` | Vulnerabilities, injection, auth, data exposure, secrets, SSRF, dependency risks |
| Bug Hunter | `references/bug-hunter.md` | Logic errors, race conditions, silent failures, null paths, error swallowing, edge cases |
| Code Quality Reviewer | `references/code-quality-reviewer.md` | DRY, KISS, YAGNI, SOLID, naming, complexity, project convention compliance |
| Contracts Reviewer | `references/contracts-reviewer.md` | API design, type safety, breaking changes, invariants, backward compatibility |
| Test Coverage Reviewer | `references/test-coverage-reviewer.md` | Missing test paths, test quality, regression coverage, test anti-patterns |
| Historical Context Reviewer | `references/historical-context-reviewer.md` | Git blame hotspots, recurring bugs, past decisions, change frequency patterns |

Every subagent must return its findings using the exact output structure defined in its instructions file. Do not modify or reinterpret the subagent's output format — the aggregation step expects consistent structure.

Wait for ALL six subagents to complete before proceeding.

### Phase 3: Aggregation — Produce CODE_REVIEW.md

Collect findings from all six subagents and assemble CODE_REVIEW.md:

1. **Deduplicate.** If two subagents flag the same file and line for overlapping reasons (e.g., security-auditor and bug-hunter both flag an unvalidated input), merge them into one finding. Keep the more severe classification and combine the reasoning.

2. **Classify severity.** Every finding must be assigned exactly one severity:
   - **Critical** — data loss, security breach, crash in production, silent corruption. Must be fixed before merge.
   - **High** — real bugs, significant logic errors, missing error handling on critical paths. Should be fixed before merge.
   - **Medium** — code quality issues, minor bugs in non-critical paths, missing tests for important behavior. Worth fixing.
   - **Low** — style inconsistencies, minor naming issues, optional improvements. Fix if convenient.

3. **Discard noise.** Every finding must be directly caused by, introduced in, or meaningfully worsened by the code changes under review. Drop findings that are:
   - Pre-existing issues not introduced or worsened by the current changes
   - Issues in unchanged code that happens to be adjacent to or called by the changed code
   - Forward-looking concerns about behavior that depends on code not yet written — review what exists, not what might exist later
   - Aspirational improvements that go beyond what the changes set out to accomplish — the review evaluates whether the changes are correct, not whether they could be more ambitious
   - Style-only nitpicks in files with large functional changes (focus on substance)
   - Issues that linters or formatters would catch automatically
   - Theoretical issues that require implausible conditions to trigger
   - Intentionally suppressed issues (e.g., `// noinspection`, `@SuppressWarnings`, `# noqa`)

4. **Preserve positive observations.** Good patterns, solid architectural decisions, and well-written code sections noted by any subagent go into the Positive Observations section. Acknowledging good work is part of a thorough review.

5. **Write CODE_REVIEW.md** using the output format below.

---

## Output Format

```markdown
---
skill: CODE_REVIEW.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions}
verdict: {pass | concerns | fail}
critical_issues: {N}
high_issues: {N}
files_reviewed: {N}
---

# CODE_REVIEW: {brief description of what was reviewed}

## Review Scope

- **Branch:** {branch name} ({N} commits ahead of {main branch})
- **Uncommitted changes:** {N} files with unstaged changes, {N} files staged
- **Total files reviewed:** {N}
- **Review agents:** Security Auditor, Bug Hunter, Code Quality, Contracts, Test Coverage, Historical Context

### Files Reviewed

| File | Change Type | Lines Changed |
|------|-------------|---------------|
| `path/to/file` | modified / added / deleted | +N / -N |

### Commit Summary

- `abc1234` — {commit message}
- `def5678` — {commit message}

(Omit this section if reviewing only uncommitted changes with no branch commits.)

---

## Critical Issues

Issues that must be resolved before merge. Each issue includes location, explanation, and a concrete fix.

### {N}. {Issue title}

**File:** `path/to/file.ext:{line range}`
**Found by:** {agent name}
**Category:** {e.g., SQL Injection, Race Condition, Data Loss}

**Problem:** {Precise description of what is wrong. Reference exact variable names, function names, and values.}

**Impact:** {What breaks, who is affected, under what conditions.}

**Fix:** {Specific change to make. Not "add validation" — state what validation, on which field, with what error.}

(Repeat for each critical issue. If none: "No critical issues found.")

---

## High-Priority Issues

Issues that carry real risk and should be fixed before merge.

### {N}. {Issue title}

**File:** `path/to/file.ext:{line range}`
**Found by:** {agent name}
**Category:** {category}

**Problem:** {description}

**Impact:** {description}

**Fix:** {specific fix}

(Repeat for each. If none: "No high-priority issues found.")

---

## Medium-Priority Issues

Issues worth fixing but not blocking.

### {N}. {Issue title}

**File:** `path/to/file.ext:{line range}`
**Found by:** {agent name}
**Category:** {category}

**Problem:** {description}

**Suggestion:** {specific improvement}

(Repeat for each. If none: "No medium-priority issues found.")

---

## Low-Priority Issues

Optional improvements.

| # | File | Category | Issue | Suggestion |
|---|------|----------|-------|------------|
| 1 | `path:line` | {cat} | {brief} | {brief} |

(If none: "No low-priority issues found.")

---

## Positive Observations

Patterns, decisions, and code quality worth acknowledging.

- **{File or component}:** {What was done well and why it matters.}

(Always include at least one positive observation. Every changeset has something done right.)

---

## Historical Context

Relevant findings from git history analysis. Omit this section if the historical-context-reviewer found nothing actionable.

- **{File}:** {Historical pattern, past issue, or architectural decision that is relevant to the current changes.}

---

## Summary

| Severity | Count |
|----------|-------|
| Critical | {N} |
| High | {N} |
| Medium | {N} |
| Low | {N} |

**Verdict:** {PASS — no critical or high issues / CONCERNS — high issues exist, recommend fixing / FAIL — critical issues must be resolved}

---

## Open Questions

Genuinely ambiguous items where multiple valid interpretations exist and the review cannot determine the correct one from context alone.

- {Question} — {Options with tradeoffs and a suggestion.}

(If none: "No open questions.")
```

---

## Scope

### In scope

- Changes identified by the explicit scope, or all uncommitted changes and branch commits when no scope is given
- Security vulnerabilities, bugs, code quality, API contracts, test coverage, and historical context analysis — limited to what the reviewed changes introduce or worsen
- Producing a single CODE_REVIEW.md aggregating all findings

### Out of scope

- Fixing any of the issues found (the review identifies, it does not implement)
- Reviewing code that has not changed (pre-existing issues are excluded unless the changes worsen them)
- CI/CD pipeline configuration review
- Performance benchmarking or profiling
- Dependency version auditing beyond what appears in the diff

---

## Quality Checklist

Before considering CODE_REVIEW.md complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields
- [ ] Phase 1 used the provided scope (or ran default discovery) and the scope section accurately reflects the changes
- [ ] All six subagents were launched in parallel and all six completed
- [ ] Duplicate findings across subagents are merged, not listed twice
- [ ] Every finding has a specific file path and line range, not a vague reference
- [ ] Every Critical and High finding has a concrete fix, not a vague suggestion like "add validation" or "handle the error"
- [ ] Every finding points to a specific line that was changed — no findings about unchanged code, future code, or aspirational improvements
- [ ] No findings refer to pre-existing issues untouched by the current changes
- [ ] No findings duplicate what a linter or formatter would catch automatically
- [ ] The Positive Observations section contains at least one entry
- [ ] The Summary verdict is consistent with the issue counts (FAIL if any Critical, CONCERNS if any High, PASS otherwise)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] The document is self-contained — readable and actionable without opening any other file
