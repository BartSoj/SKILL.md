---
name: VERIFICATION.md
description: Verify a running application by interacting with it as a QA tester. Use when asked to verify the app works, test the application, do QA testing, validate that implementation works, or produce a VERIFICATION.md.
---

# Task: Verify Application Behavior Through Interaction

## Objective

Produce a VERIFICATION.md that documents the results of testing a running application by actually using it — launching it, interacting with its interface (browser, mobile emulator, or CLI), and recording pass/fail outcomes with evidence for each scenario. This is QA testing: the skill does not read or review source code. It starts the application and operates it as an end user would, then reports what works and what does not.

---

## Inputs

1. **Verification scenarios** (required) — what to test. Accepted formats: acceptance criteria, user stories, a specification document, a test plan, or freeform instructions like "verify that login works and the dashboard loads." Each scenario must describe a behavior that can be verified through interaction.
2. **Application start instructions** (optional) — how to start the app. If not provided, discover from the codebase: package.json scripts (`dev`, `start`, `serve`), Makefile targets, docker-compose.yml services, Cargo.toml, go run targets, or README instructions. If the application is already running, the user provides the URL or entry point and startup is skipped entirely.
3. **Application type** (optional) — web, mobile, or CLI. If not provided, infer from the start instructions and codebase structure.
4. **Base URL or entry point** (optional) — where to reach the running application (e.g., `http://localhost:3000`, a CLI command name). If not provided, extract from the start command output or fall back to common defaults (`localhost:3000`, `localhost:8080`, `localhost:5173`).

---

## Workflow

Verification proceeds in four phases: discovery, setup, testing, and documentation.

### Phase 1: Discovery

Determine three things before touching the application.

**1. What to test.** Parse verification scenarios from the user's input. Each scenario becomes a test case with four attributes:

| Attribute | What it captures |
|-----------|-----------------|
| **Name** | Descriptive label, e.g., "User can log in with valid credentials" |
| **Steps** | Ordered interaction sequence: navigate here, click this, type that, submit |
| **Expected outcome** | Observable result: text appears, page changes, exit code is 0, file is created |
| **Evidence strategy** | What to capture: screenshot, command output, console log, response body |

If the user's input is high-level ("verify login works"), decompose it into concrete steps using your knowledge of the application type and common UX patterns. If the application type is unknown at this point, defer step decomposition to Phase 3 after startup reveals the interface.

**2. How to start the app.** If the user provided start instructions or a URL to an already-running app, use those. Otherwise, explore the codebase:

- Read package.json, Makefile, docker-compose.yml, Cargo.toml, pyproject.toml, go.mod, or equivalent
- Look for `dev`, `start`, `serve`, `run`, `up` scripts or targets
- Identify prerequisite steps: dependency installation, database migration, env file creation, build step
- Identify the readiness signal: a specific log line ("ready on port 3000"), an HTTP 200 response, a CLI prompt

**3. What interaction tool to use.**

| Application type | Tool | Prerequisite |
|-----------------|------|-------------|
| Web application | agent-browser | Run `npx agent-browser --version` to confirm availability. If unavailable, stop and report: "agent-browser is required for web application verification. Install it with `npm install -g agent-browser`." |
| Mobile application | agent-browser (with device emulation or simulator flags) | Same as web. Confirm simulator/emulator availability for the target platform. |
| CLI application | Bash | No additional tooling required. |
| Mixed (CLI that spawns web UI) | Both — Bash for startup, agent-browser for UI | Confirm agent-browser availability. |

### Phase 2: Setup

Start the application and confirm it is ready.

1. **Install dependencies** if Phase 1 identified a required install step (e.g., `npm install`, `pip install -e .`, `cargo build`).
2. **Start the application** in the background. Record the exact start command used. For web/mobile apps, run the dev server. For CLI apps, verify the binary or script is available.
3. **Wait for readiness:**
   - Web/mobile: poll the base URL with agent-browser (`agent-browser open <url>`) or curl until a successful response. Timeout after 60 seconds.
   - CLI: run `<command> --help` or equivalent to confirm the binary responds.
4. **Capture initial state:**
   - Web/mobile: take a screenshot of the landing page via agent-browser.
   - CLI: capture the help output or version string.

If the application fails to start within the timeout, capture the full error output (stderr, exit code, last 50 lines of stdout) and skip directly to Phase 4 to document the startup failure. Do not attempt to debug or fix the application.

If the user provided a URL to an already-running application, skip steps 1-2 and verify reachability in step 3.

### Phase 3: Testing

Execute each scenario from Phase 1 sequentially.

For each scenario:

**1. Set up preconditions.** Navigate to the correct page, clear relevant state, or prepare whatever the scenario requires. If a scenario depends on a previous scenario's side effects (e.g., "verify dashboard shows the item created in scenario 2"), execute them in order. If a scenario is independent, reset to a clean starting point.

**2. Execute the interaction sequence.**

For web and mobile applications, use agent-browser:
- `agent-browser open <url>` to navigate
- `agent-browser snapshot` to inspect the current page and get element refs
- `agent-browser click @ref`, `agent-browser fill @ref "value"`, `agent-browser select @ref "option"` to interact
- `agent-browser wait --text "expected text"` or `agent-browser wait --url "expected/path"` to wait for results
- Chain commands with `&&` to execute multi-step interactions efficiently

For CLI applications, use Bash:
- Run the command with appropriate arguments
- Pipe input for interactive CLIs
- Capture stdout, stderr, and exit code separately

**3. Check the expected outcome.** Verify the result matches the scenario's expectation. Be specific:
- Web/mobile: check for exact text content, element visibility, URL change, absence of error banners
- CLI: check exit code, stdout content, stderr content, file system side effects

**4. Capture evidence.**
- Web/mobile: take a screenshot after the key assertion point. If the page has console errors, capture them via `agent-browser console` or equivalent. Save screenshots to `verification-evidence/` with descriptive names (e.g., `01-login-success.png`).
- CLI: capture the full command invocation and its output.

**5. Record the result:**

| Result | Meaning |
|--------|---------|
| **PASS** | Expected outcome observed exactly as specified. |
| **FAIL** | Outcome differs from expectation. Record both expected and actual. |
| **BLOCKED** | Scenario could not be executed: dependency on a failed scenario, app crashed, required state unavailable, or missing capability (e.g., no mobile emulator). Record the reason. |

**Continuation rules:**
- If a scenario fails, continue testing remaining scenarios. Do not stop at the first failure.
- If the application crashes during testing, attempt one restart. If restart succeeds, continue from the next scenario. If restart fails, mark all remaining scenarios as BLOCKED with reason "application crashed and could not be restarted."
- If a scenario is blocked because it depends on a failed scenario, mark it BLOCKED and note the dependency.

### Phase 3.5: End-to-End and Mock Fidelity Checks

After running the scenario-based tests from Phase 3, perform these additional checks when the infrastructure supports them. These are strong guidance, not hard requirements — early-stage units or components without running dependencies may not be able to perform them.

**End-to-end interaction.** After verifying scenarios through the unit's own interface, attempt to test the feature through the actual running system when feasible:

- If enough infrastructure exists to start related services (server, database, dependent APIs), start them and test the feature as it would work in the composed system — not just in isolation.
- If the feature has an observable effect through cross-component interaction (e.g., a push endpoint can be called via curl and the result read back through a list endpoint), test that path.
- Prefer real dependencies over mocks when the infrastructure supports it: if the database is available, test against it rather than an in-memory stub; if the server is running, test the CLI against the real server rather than wiremock.
- If e2e testing is not feasible (system cannot start yet, dependencies are not built, no way to observe the feature externally), document why in the VERIFICATION.md and proceed with unit-level verification only. This is acceptable for foundation and scaffold units.

**Mock fidelity verification.** If the unit's tests use HTTP mocks (wiremock, MSW, nock, test doubles), and a CONTRACT_REGISTRY.md (or equivalent contract document) exists in the project:

- Check that mock response bodies use wire-format field names and casing from the contract registry, not the implementing language's native convention.
- Flag any mock that uses field names or structure inconsistent with the registry — these mocks cause tests to verify against unrealistic data, masking integration bugs.
- Record mock fidelity findings in the VERIFICATION.md under a dedicated section.

### Phase 4: Produce VERIFICATION.md

After all scenarios are tested, shut down the application (if this skill started it), and write VERIFICATION.md.

---

## Output Format

```markdown
# VERIFICATION: {brief description of what was verified}

## Summary

| Result | Count |
|--------|-------|
| PASS | {N} |
| FAIL | {N} |
| BLOCKED | {N} |
| **Total** | **{N}** |

**Verdict:** {PASS — all scenarios passed / PARTIAL — some failures but core functionality works / FAIL — critical scenarios failed or app did not start}

## Environment

- **Application:** {name and version if discoverable}
- **Type:** {web / mobile / CLI / mixed}
- **Start command:** `{the exact command used}` (or "Application was already running at {URL}")
- **Base URL / entry point:** {URL or command path}
- **Interaction tool:** {agent-browser vX.X.X / Bash / both}
- **Date:** {YYYY-MM-DD}

---

## Passed Scenarios

### {N}. {Scenario name}

**Steps performed:**
1. {What was done — exact commands or interactions}
2. {Next step}

**Expected:** {What should happen}
**Actual:** {What happened — confirming the match}
**Evidence:** {Screenshot path or output snippet}

(Repeat for each passing scenario. If none: "No scenarios passed.")

---

## Failed Scenarios

### {N}. {Scenario name}

**Steps performed:**
1. {What was done}
2. {Next step}

**Expected:** {What should happen}
**Actual:** {What actually happened — the specific discrepancy}
**Impact:** {What this failure means for the end user}
**Evidence:** {Screenshot path or output showing the failure}

(Repeat for each failing scenario. If none: "No failures — all scenarios passed.")

---

## Blocked Scenarios

### {N}. {Scenario name}

**Reason:** {Why this scenario could not be executed}
**Dependency:** {What prerequisite failed or was unavailable}

(Repeat for each blocked scenario. If none: "No blocked scenarios.")

---

## End-to-End Testing

(Include this section when e2e testing was attempted or when documenting why it was not feasible.)

**E2E attempted:** {yes / no}
**Reason (if no):** {e.g., "Server and database are not yet available — this is a Tier 0 foundation unit."}

(If e2e was attempted, list cross-component interactions tested and their results using the same PASS/FAIL/BLOCKED format as scenario results above.)

---

## Mock Fidelity

(Include this section when the unit's tests use HTTP mocks and a contract registry exists.)

**Contract registry found:** {yes — path / no}
**Mocks checked:** {N}
**Mismatches found:** {N}

| Mock location | Field/Issue | Registry says | Mock says |
|---------------|------------|---------------|-----------|
| `path/to/test:line` | {field name or casing issue} | {wire-format value} | {mock value} |

(If no contract registry exists: "No contract registry found — mock fidelity check skipped.")
(If no mocks exist: "No HTTP mocks found in this unit's tests.")
(If all mocks match: "All mock response bodies conform to the contract registry.")

---

## Positive Observations

Noteworthy quality observations from testing — aspects of the application that work well, feel polished, or exceed expectations.

- **{Feature or area}:** {What was done well and why it stands out.}

(Include at least one positive observation when any scenario passes.)

---

## Application Logs

Relevant log output captured during testing — startup logs, errors, warnings.

```
{log output}
```

(If no relevant logs were captured: "No notable log output.")

---

## Open Questions

Items where the verification could not determine correctness from the provided scenarios alone.

- {Question} — {Why it is ambiguous and what clarification would resolve it.}

(If none: "No open questions.")
```

---

## Scope

### In scope

- Starting the application in a development environment from the codebase
- Interacting with the running application through its user-facing interface: browser, mobile emulator, or CLI
- Verifying observable behavior against provided scenarios
- Capturing evidence (screenshots, command output, logs) for every tested scenario
- Producing a VERIFICATION.md documenting all results with pass/fail verdicts

### Out of scope

- Reading, reviewing, or modifying source code (code review is a separate concern)
- Writing automated test files or test suites (test authoring is a separate concern)
- Debugging or fixing failures discovered during verification — report them, do not fix them
- Performance testing, load testing, or benchmarking
- Security testing or penetration testing
- Deploying to staging or production environments
- Installing or configuring agent-browser — it must be available before this skill runs

---

## Quality Checklist

Before considering VERIFICATION.md complete, verify:

- [ ] Every scenario from the user's input has a corresponding result (PASS, FAIL, or BLOCKED) — none were silently skipped
- [ ] Every FAIL result includes both the expected and actual outcome with the specific discrepancy, not just "failed"
- [ ] Every PASS and FAIL result has captured evidence (screenshot path or output snippet)
- [ ] Every BLOCKED result explains exactly why it could not be executed
- [ ] The application was actually started and interacted with — no results are based on reading source code or making assumptions
- [ ] The Verdict is consistent with results: PASS only if all passed, FAIL if any critical scenario failed or app did not start, PARTIAL otherwise
- [ ] The Positive Observations section has at least one entry when any scenario passed
- [ ] The Steps Performed for each scenario list the actual commands or interactions used, not abstract descriptions
- [ ] The application was shut down after testing (if this skill started it)
- [ ] End-to-End Testing section is present — either with e2e results or a documented reason why e2e was not feasible
- [ ] Mock Fidelity section is present — either with check results, a note that no registry/mocks exist, or confirmation of conformance
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] The document is self-contained — readable and actionable without opening any other file
