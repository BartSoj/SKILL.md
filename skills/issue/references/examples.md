# ISSUE.md — Worked Examples

Two complete worked issues illustrating the shape `/ISSUE.md` produces. The first is a high-severity contract-mismatch bug discovered during system verification; the second is a medium-high CLI exit-code regression discovered during code review. Both demonstrate every required body section and the citation discipline the skill enforces.

The examples are presented as the contents of files that would live at:

- `issues/057-push-200-commit-lost/ISSUE.md`
- `issues/058-clone-exit-code-zero-on-fail/ISSUE.md`

Read them when authoring an issue against an unfamiliar surface — the examples concretize Rule 1 (source-of-truth citation), Rule 3 (concrete reproduction), Rule 4 (severity justification), Rule 8 (suggested-fix discipline), and Rule 10 (optional-section discipline).

---

## Example A — Contract-mismatch bug, severity high

A push to a repository returns 200 but the commit is silently dropped under load. The defect violates the documented contract on the push endpoint and the saga's terminal-state guarantee. Discovered during a system verification pass; fix sketch known; one open question about retry semantics.

The example is wrapped in a 4-backtick fence so that its inner 3-backtick code block (the server logs in `## Observed`) renders correctly.

````markdown
---
skill: ISSUE.md
date: 2026-05-08
status: open

id: "057"
title: "POST /repos/{name}/push returns 200 but commit not persisted on subsequent fetch"
kind: bug
severity: high
severity_note: "Corrupts user data — a successful 200 response indicates the push succeeded, but `git fetch` afterward shows the commit absent. The user has no way to detect the loss without manually re-checking."
component: "apps/server/src/modules/repos/push.ts, syns-git/src/handlers/push.rs"
discovered: 2026-05-08
discovered_by: bartsoj
discovered_in: "system verification pass for milestone 2026-05"

promoted_to_units: []
related: []

fixed_in: null
fixed_date: null
last_reconfirmed: null
last_reconfirmed_in: null
deferred_reason: null
deferred_date: null
wontfix_reason: null
---

# ISSUE — POST /repos/{name}/push returns 200 but commit not persisted on subsequent fetch

## Summary

A push to a repository returns HTTP 200 with `{accepted: true, sha: "..."}`
but a subsequent `git fetch` does not include the commit. The server logs the
push acceptance, but the git engine never receives the pack file. The handoff
between the TS server and the Rust git engine is silently dropping the request
under load.

## Reproduction

1. `syns init my-repo` to create a fresh repository.
2. Make a commit locally: `echo "test" >> README.md && git commit -am "test"`.
3. Push: `git push origin main`. Observe HTTP 200 response with `{accepted: true, sha: "abc123..."}`.
4. From a different working directory, `git clone` the same repository.
5. Run `git log` in the cloned repo. Observe the commit `abc123...` is **not** present.

Reproducible 100% of the time when the server is under sustained load (≥ 50 RPS push throughput); 0% under 10 RPS.

## Expected

Per `INTERFACES § PUT /repos/{owner}/{name}/push`, a 200 response means the
commit is durably persisted and visible to subsequent fetches. Per
`BEHAVIOR § SAGA-push`, the saga's `completed` terminal state requires the
git engine to acknowledge the pack file write before the server returns 200.

## Observed

Server logs (timestamps abbreviated):

```
[14:32:01] POST /repos/bart/my-repo/push received 1 commit (sha=abc123...)
[14:32:01] forwarding pack to git engine via httpgit-engine-internal://pack
[14:32:01] returned 200 to client
[14:32:01] (no follow-up acknowledgment from git engine)
```

The git engine logs show no record of receiving the pack. Network capture
confirms the request was sent but timed out at the engine's HTTP layer at
2.5s, with no retry by the server.

## Root cause

In `apps/server/src/modules/repos/push.ts:78`, the engine call is
fire-and-forget — the server does not await the engine's acknowledgment
before returning 200. The git engine handler at
`syns-git/src/handlers/push.rs:42` correctly waits for the pack write but
is never reached because the engine's HTTP server timeout (2s default)
drops the request under load.

The contract in `INTERFACES § PUT /repos/{owner}/{name}/push` says 200 =
durable persistence, but the implementation treats the engine call as
best-effort.

## Scope / impact

- **Affected:** all Collaborators pushing to any repository under sustained load.
- **Frequency:** 100% reproducible at ≥ 50 RPS, 0% reproducible at < 10 RPS.
- **Workaround:** none — the user has no signal that the push silently failed.

## Suggested fix

Two changes:

1. In `apps/server/src/modules/repos/push.ts:78`, change the engine call to
   `await engine.push(packFile)` and only return 200 after the engine returns
   its acknowledgment. Return 503 on engine timeout.
2. In `syns-git/src/handlers/push.rs:42`, increase the HTTP server's read
   timeout to 30s; the pack write is the slow path, not the network.

The 503 path needs a new entry in `ERRORS.md` (`ERR_PUSH_ENGINE_TIMEOUT`)
and the client retry semantics need confirmation — see Open Questions.

## Alternatives considered

- **Decouple via queue** — server enqueues, worker calls engine, asynchronous acknowledgment via webhook. Rejected: violates the `INTERFACES § push` contract that 200 = durable.
- **Stay fire-and-forget but log loudly** — rejected: silent data loss with logs is still data loss; the contract guarantees durability, not best-effort.

## Open Questions

- [ ] Should the new `ERR_PUSH_ENGINE_TIMEOUT` be retryable or not?
  - **Option A:** Retryable — client retries with exponential backoff.
  - **Option B:** Not retryable — client must investigate and re-push manually.
  - **Recommendation:** Option A. The push is content-addressed and idempotent, so retry is safe.

## References

- `INTERFACES § PUT /repos/{owner}/{name}/push` — source-of-truth for the durability contract.
- `BEHAVIOR § SAGA-push` — saga terminal-state semantics.
- `u044` — server-side push handler (likely owns the fix on the TS side).
- `u051` — git-engine push handler (likely owns the fix on the Rust side).
````

---

## Example B — CLI exit-code regression, severity medium-high

A CLI command that previously returned non-zero on failure now returns 0, breaking scripts that check `$?`. The defect violates the CLI's documented exit-code taxonomy and is a regression introduced by a recent error-handling refactor. The fix is small, the suggested approach is clear, and the only related work is a closed issue from earlier in the project. No open questions.

The example is wrapped in a 4-backtick fence so that its inner 3-backtick blocks (`bash` reproduction and the shell-output `## Observed` block) render correctly.

````markdown
---
skill: ISSUE.md
date: 2026-05-08
status: open

id: "058"
title: "syns clone exit code 0 even when clone fails"
kind: regression
severity: medium-high
severity_note: "Visible UX failure on a primary path with no workaround — scripts that check $? after `syns clone` cannot detect failures."
component: "syns-cli/src/commands/clone.rs"
discovered: 2026-05-08
discovered_by: bartsoj
discovered_in: "code review of u062"

promoted_to_units: []
related: ["issues/041-push-exit-code-drift"]

fixed_in: null
fixed_date: null
last_reconfirmed: null
last_reconfirmed_in: null
deferred_reason: null
deferred_date: null
wontfix_reason: null
---

# ISSUE — syns clone exit code 0 even when clone fails

## Summary

`syns clone <url>` returns exit code 0 when the clone fails (network error,
auth failure, repository not found). The CLI prints an error message to
stderr but the exit code is misleading. This is a regression — the previous
version (CLI v0.3.x) correctly returned non-zero exit codes per the
documented mapping.

## Reproduction

```bash
syns clone https://syns.example/owner/nonexistent-repo; echo "exit=$?"
```

Observed: stderr shows `Error: repository not found` but `exit=0`.
Expected: `exit=2` (per CLI_IA exit-code taxonomy for resource-not-found).

100% reproducible on any clone failure path (network error, auth failure,
nonexistent repository).

## Expected

Per `CLI_IA § Exit Code Taxonomy`, `syns clone` returns:

- `0` — clone succeeded.
- `2` — repository not found, network failure, auth failure.
- `64` — invalid arguments.

## Observed

```
$ syns clone https://syns.example/owner/nonexistent-repo
Error: repository not found
$ echo $?
0
```

## Root cause

In `syns-cli/src/commands/clone.rs:42`, the error handler logs the error to
stderr but calls `Ok(())` instead of `Err(...)`. Likely a regression from
`u062` (CLI error-handling refactor) which moved error formatting into a
shared formatter and lost the exit-code propagation in the clone path.

## Scope / impact

- **Affected:** all CLI users running scripts that check `syns clone`'s exit code.
- **Frequency:** 100% on any clone failure path.
- **Workaround:** parse stderr — fragile; the error message format is not part of any contract and may change.

## Suggested fix

In `syns-cli/src/commands/clone.rs:42`, change the error handler from
`Ok(())` to `Err(SynsError::ResourceNotFound)` and let the main error
handler map to exit code 2 via the existing `SynsError → exit-code`
mapping in `syns-cli/src/error.rs`.

## Open Questions

All questions resolved.

## References

- `CLI_IA § Exit Code Taxonomy` — source-of-truth for the exit-code contract.
- `u062` — likely shipped the regression (CLI error-handling refactor).
- `u038` — shipped the original `syns clone` command.
- `issues/041-push-exit-code-drift` — similar exit-code drift for `syns push`; closed; verify `u062` did not reintroduce the same defect there.
````

---

## What the examples illustrate

| Discipline | Example A | Example B |
|---|---|---|
| Source-of-truth citation (Rule 1) | Two artifacts: `INTERFACES § push`, `BEHAVIOR § SAGA-push` | One artifact: `CLI_IA § Exit Code Taxonomy` |
| Concrete reproduction (Rule 3) | Numbered CLI steps + git commands; explicit load conditions | Single shell line with exit-code check |
| Severity justification (Rule 4) | "Corrupts user data" — high | "No workaround" — medium-high |
| Component as code path (Rule 5) | Two paths spanning the bug's split between TS server and Rust engine | One file path |
| Discovery reflected (Rule 6) | Two units cited in `References`; no duplicate | One related issue in `related:` (closed sibling); two units cited |
| Suggested-fix discipline (Rule 8) | Sketch with two file:line targets and an ERR_CODE proposal | One file:line target with reuse of an existing mapping |
| Alternatives considered | Two alternatives, both rejected with one-line rationale | Omitted entirely (Rule 10) — only one obvious approach |
| Open Questions | One genuine question with options + recommendation | "All questions resolved." |
| Optional sections | None included — issue is freshly filed | None included — issue is freshly filed |

The two examples between them exercise every required section and most rules. When in doubt about format, copy the closest example's shape and substitute the new defect's content.
