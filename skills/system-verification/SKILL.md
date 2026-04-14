---
name: SYSTEM_VERIFICATION.md
description: Verify the composed application end-to-end by bootstrapping the full stack and running cross-cutting user scenarios. Use when asked to do system verification, test the full application, run end-to-end integration tests, verify the system works as a whole, or produce a SYSTEM_VERIFICATION.md.
---

# Task: System Verification — End-to-End Application Testing

## Objective

Produce a SYSTEM_VERIFICATION.md that documents whether the fully composed application actually works by bootstrapping the entire stack and running cross-cutting user scenarios that span multiple components. This is the system-level equivalent of per-unit verification — it catches integration bugs, contract mismatches, missing configuration, and system glue issues that are invisible when each unit is tested in isolation. After this skill runs, the reader knows whether the application works as a whole and exactly where it breaks if it does not.

---

## Inputs

1. **Use cases or user scenarios** (required) — the cross-cutting user flows to test. Accepted formats: a USE_CASES.md document, user stories, feature descriptions, or freeform instructions. Each scenario should describe a complete user journey that exercises multiple components in sequence.
2. **Application start instructions** (optional) — how to bootstrap the full stack. If not provided, discover from the codebase: docker-compose files, Makefiles, package.json scripts, README setup instructions. The skill must start ALL services, not just one.
3. **Architecture documents** (optional) — system structure, component list, service dependencies. Helps identify what needs to start and in what order.
4. **CONTRACT_REGISTRY.md** (auto-discovered) — if present, used to verify that cross-boundary communication uses correct wire formats during live testing.
5. **Per-unit VERIFICATION.md files** (optional) — prior verification results. Useful for understanding what already works at the unit level, so system verification can focus on cross-boundary behavior.

---

## Workflow

System verification has two phases: bootstrap and scenario testing. Bootstrap must succeed before scenarios run — there is no point testing features against a system that cannot start.

### Phase A: Bootstrap

Verify the application can start as a complete system. This phase catches "system glue" issues — missing environment variables, unconfigured databases, broken service dependencies, missing migration scripts.

**Step 1: Environment check.**

- Identify all required environment variables across all components. Check `.env` files, `.env.example` files, docker-compose environment sections, and application config files.
- Verify that all required variables are set or have defaults. If `.env` files are missing, check whether `.env.example` files exist and can be copied.
- Check for database connection strings, API keys, service URLs, and auth configuration.
- Record every environment issue found.

**Step 2: Dependency startup.**

Start services in dependency order:

1. Infrastructure services first: databases, message queues, cache servers (typically via docker-compose or local install).
2. Internal services next: git engines, background workers, any internal APIs.
3. Application services last: the main server, web dev server.

For each service:
- Record the exact start command used.
- Wait for a readiness signal: a health check endpoint returning 200, a specific log line, a port becoming available.
- Timeout after 60 seconds per service. Record failure output if a service does not start.

**Step 3: Database setup.**

- Run database migrations if a migration command exists (e.g., `npx drizzle-kit push`, `diesel migration run`, `alembic upgrade head`).
- Seed initial data if a seed command or script exists.
- Record any migration failures with full error output.

**Step 4: Connectivity verification.**

Verify that services can reach each other:
- Can the server reach the database? (health check, simple query)
- Can the server reach internal services? (health endpoint, ping)
- Can a client reach the server? (HTTP request to a public endpoint)
- Do auth endpoints respond? (load login page, hit auth discovery endpoints)

**Bootstrap verdict:**

If any step fails in a way that prevents scenario testing, record all failures and stop. Set the overall verdict to FAIL with category "bootstrap failure." These are typically system glue issues — missing configuration, broken connectivity, missing migration scripts — that must be fixed before any feature can be tested.

If bootstrap succeeds with warnings (e.g., some optional services unavailable, some non-critical env vars missing), proceed to Phase B and note the warnings.

### Phase B: End-to-End Scenarios

Run cross-cutting user flows from the scenario input. Each scenario exercises a complete user journey across component boundaries — a single scenario may touch the CLI, the server, the database, and the web UI in sequence.

**Scenario execution approach:**

For each scenario:

1. **Parse into steps.** Decompose the user scenario into concrete interaction steps, identifying which component each step exercises and which boundaries are crossed.

2. **Execute steps sequentially.** Use the appropriate tool for each component:
   - Web UI: agent-browser (navigate, click, fill, verify page content)
   - CLI: Bash (run commands, check output and exit codes)
   - API: curl or equivalent (send requests, check response shapes and status codes)
   - Database: direct query if accessible (verify side effects)

3. **Verify at every boundary crossing.** When a scenario crosses a component boundary (e.g., "push from CLI, then verify in web UI"), the verification must confirm that data survived the boundary correctly — correct field names, correct values, correct format.

4. **Capture evidence.** Screenshots for web interactions, command output for CLI/API interactions, response bodies for HTTP calls.

5. **Record result.** Each scenario gets one of:

| Result | Meaning |
|--------|---------|
| **PASS** | Complete user journey works end-to-end, all boundary crossings verified. |
| **FAIL** | Journey breaks at a specific point. Record which step failed, the expected vs actual behavior, and which boundary or component is involved. |
| **PARTIAL** | Some steps in the journey succeed, others fail. Record which steps passed and which failed. |
| **BLOCKED** | Scenario cannot be executed: depends on a component that failed to start, requires auth that cannot be obtained, or depends on a failed scenario's side effects. |

**Continuation rules:**
- Continue testing all scenarios even if earlier ones fail. Later scenarios may reveal independent issues.
- If the entire system crashes during testing, attempt one restart of the full stack. If restart fails, mark remaining scenarios as BLOCKED.
- Group scenarios so that independent flows are tested even if one flow's failure blocks related flows.

**Failure categorization:**

When a scenario fails, categorize the root cause:

| Category | Description | Example |
|----------|-------------|---------|
| **Contract mismatch** | Client and server disagree on wire format — field names, casing, structure, types | CLI sends `parent_sha`, server expects `parentSha` |
| **Missing configuration** | Environment variable, database table, library config, dev script not set up | No `.env` loading, missing OAuth callback URL, no migration runner |
| **Logic bug** | Code does the wrong thing despite correct contracts | Push succeeds but returns wrong commit SHA |
| **Missing feature** | Something that needs to exist but was never implemented | No device approval web page, no migration CLI script |
| **Third-party library** | Library requires setup or config not documented in architecture | Auth library needs `usePlural: true` for ORM adapter, plugin needs specific DB table |

---

## Output Format

```markdown
---
skill: SYSTEM_VERIFICATION.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions}
verdict: {pass | partial | fail}
bootstrap: {pass | fail}
scenarios_total: {N}
scenarios_passed: {N}
scenarios_failed: {N}
scenarios_blocked: {N}
failure_categories: {comma-separated list of categories found, e.g., "contract_mismatch, missing_config"}
---

# SYSTEM_VERIFICATION: {project name}

## Summary

| Result | Count |
|--------|-------|
| PASS | {N} |
| FAIL | {N} |
| PARTIAL | {N} |
| BLOCKED | {N} |
| **Total** | **{N}** |

**Bootstrap:** {PASS — all services started / FAIL — system could not start (see Bootstrap section)}
**Verdict:** {PASS — all scenarios passed / PARTIAL — some failures / FAIL — critical scenarios failed or system did not start}

---

## Bootstrap

### Environment

| Variable/Config | Status | Issue |
|----------------|--------|-------|
| {name} | {ok / missing / wrong value} | {description if not ok} |

(If all environment checks pass: "All environment variables and configuration present.")

### Services

| Service | Start command | Status | Notes |
|---------|--------------|--------|-------|
| {name} | `{command}` | {running / failed / skipped} | {error output or readiness signal} |

### Database

- **Migrations:** {passed / failed / not applicable}
- **Seed data:** {loaded / failed / not applicable / no seed script}

### Connectivity

| Connection | Status | Evidence |
|-----------|--------|----------|
| {component A} → {component B} | {ok / failed} | {response or error} |

**Bootstrap verdict:** {PASS / FAIL — with list of blocking failures}

---

## Scenario Results

### {N}. {Scenario name} — {PASS / FAIL / PARTIAL / BLOCKED}

**Flow:** {One-line description of the user journey and which components it exercises}

**Steps:**
1. **[{component}]** {What was done} — {result}
2. **[{component}]** {What was done} — {result}
3. **[{component}]** {Boundary crossing: component A → component B} — {result}

**Expected:** {What the complete journey should produce}
**Actual:** {What happened — confirm match for PASS, describe discrepancy for FAIL}
**Evidence:** {Screenshot paths, command output, HTTP response bodies}

(For FAIL/PARTIAL, add:)
**Failure point:** {Which step failed and which boundary/component}
**Category:** {contract_mismatch / missing_config / logic_bug / missing_feature / third_party_library}
**Root cause:** {Brief analysis of why it failed}

(Repeat for each scenario.)

---

## Failure Summary

(Omit this section if all scenarios passed.)

### By Category

| Category | Count | Scenarios affected |
|----------|-------|--------------------|
| Contract mismatch | {N} | {scenario numbers} |
| Missing configuration | {N} | {scenario numbers} |
| Logic bug | {N} | {scenario numbers} |
| Missing feature | {N} | {scenario numbers} |
| Third-party library | {N} | {scenario numbers} |

### By Component Boundary

| Boundary | Failures | Description |
|----------|----------|-------------|
| {component A} → {component B} | {N} | {brief description of issues at this boundary} |

---

## Positive Observations

Aspects of the system that work well across component boundaries.

- **{Feature or area}:** {What works well and why it is noteworthy.}

(Include at least one positive observation when any scenario passes.)

---

## Open Questions

- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Bootstrapping the full application stack (all services, databases, infrastructure)
- Running cross-cutting user scenarios that span multiple components and boundaries
- Categorizing failures by root cause (contract mismatch, missing config, logic bug, missing feature, third-party library)
- Capturing evidence for every scenario (screenshots, command output, HTTP responses)
- Producing a SYSTEM_VERIFICATION.md with structured pass/fail results

### Out of scope

- Per-unit verification — that is handled by the VERIFICATION skill during the per-unit pipeline
- Fixing any failures found — this skill reports, it does not fix (TRIAGE handles analysis and fix planning)
- Performance testing, load testing, or benchmarking
- Security testing or penetration testing
- Deployment to staging or production
- Writing automated test suites — this is manual QA-style verification

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `verdict`, `bootstrap`, `scenarios_total`, `scenarios_passed`, `scenarios_failed`, `scenarios_blocked`)
- [ ] Bootstrap section documents environment check, service startup, database setup, and connectivity — no step skipped silently
- [ ] Every scenario from the input has a corresponding result — none silently skipped
- [ ] Every FAIL/PARTIAL result identifies the failure point, category, and root cause
- [ ] Every scenario documents which components and boundaries it exercises
- [ ] Evidence is captured for every scenario (screenshots, output, response bodies)
- [ ] Failure Summary groups issues by category AND by component boundary
- [ ] Positive Observations section has at least one entry when any scenario passes
- [ ] The system was actually started and interacted with — no results based on reading source code
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
