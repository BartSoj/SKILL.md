# Per-Command Blueprint Template

Every command template in the inventory gets one entry in the Per-Command Blueprints section of `CLI_IA.md`, following the template below. Every field is mandatory unless explicitly marked with an "N/A" fallback.

When iterating the inventory in Phase 4 of the workflow, copy the block below per command and fill in every field. Describe behavior by invocation contract and failure modes, not by code shape (Rule 6). Describe messages by role, not by literal string (Rule 8). Draw every exit code from the Exit-Code Taxonomy (Rule 14). Respect the scriptability contract — stdout carries data, stderr carries human-facing messages, the two never interleave (Rule 15).

---

## Template block (copy per command)

```markdown
### `{command path}` — {short name}

- **Synopsis:** `{usage line with positionals and flags, e.g., tool repo push [--force] [--json] [<path>]}`
- **Purpose:** {one sentence}
- **Kind:** {action | utility | diagnostic | shell-integration}
- **Audience:** {user types who invoke this command}
- **Auth requirement:** {unauthenticated | authenticated | authenticated-with-permission: {permission name}}
- **Referenced use cases:** {UC-IDs, or explicit classification: utility | diagnostic | auth | shell-integration}

**Positional arguments:**

| Name | Type | Required | Default | Validation |
|------|------|----------|---------|------------|
| `<name>` | {type} | {yes/no} | {default or "—"} | {validation rule} |

(Or "No positional arguments.")

**Flags (command-specific — global flags apply by reference):**

| Flag | Value | Default | Required | Purpose |
|------|-------|---------|----------|---------|
| `--{flag}` / `-{short}` | {boolean / string / integer / enum / repeated} | {default} | {yes/no} | {purpose} |

(Or "No command-specific flags — standard global flags apply.")

**Stdin:** {whether read, under what condition, expected format — or "Not read."}

**Environment variables read:** {list by name, referencing Global Flags & Environment where already documented — or "None beyond global."}

**Config values read:** {key names from the config file hierarchy — or "None."}

**Interactive prompts:** {which Shared Surfaces this command can invoke, and under what conditions — or "No interactive prompts."}

**Primary outcome:** {single effect on the world — e.g., "Uploads the repo at `<path>` to the configured remote."} *(Or "No primary outcome — {utility | diagnostic | shell-integration}.")*

**Output on stdout:**
- **Human mode (default):** {structural description of the success output — one line, multi-line, table, etc.}
- **Machine mode (`--json` / `--yaml` / etc.):** {structural description with field names — or "N/A — no machine mode."}

**Output on stderr:** {structural description of progress, warnings, logs, error messages — respects scriptability contract}

**Side effects:** {files written, network calls made, state mutated — high-level only, not implementation}

**Failure matrix:**

| State | What the user sees (stderr / stdout) | Exit code |
|-------|--------------------------------------|-----------|
| Success | {description} | {code from taxonomy} |
| Already-satisfied / no-op | {description or "N/A — not idempotent."} | {code} |
| Input validation failure | {description} | {code} |
| Authentication failure | {description or "N/A — no auth."} | {code} |
| Authorization failure | {description or "N/A — no permission gating."} | {code} |
| Resource not found | {description or "N/A — no named target."} | {code} |
| Conflict / precondition failed | {description or "N/A — no preconditions."} | {code} |
| Network / transient failure | {description or "N/A — no network."} | {code} |
| Interrupted (SIGINT / SIGTERM) | {description} | {code} |
| Non-interactive mode unavailable | {description or "N/A — no prompts."} | {code} |
| Partial success | {description or "N/A — single-item operation."} | {code} |

**Interactive vs non-interactive behavior:**
- **Interactive (TTY, no `--yes`):** {what the command does — e.g., "prompts for confirmation before pushing to a protected branch."}
- **Non-interactive with blanket confirmation (`--yes` or `CI=true`):** {what the command does — e.g., "proceeds without prompting."}
- **Non-interactive without blanket confirmation:** {what the command does — typically, exit with the "non-interactive mode unavailable" failure code.}

(Or "No interactive prompts — three-mode rule n/a.")

**Idempotency:** {safe (pure read) | idempotent (repeatable, same effect) | mutating (new effect each run) | destructive (irreversible)}

**Argument-dependent behavior:** {how positionals / flags change behavior beyond simple input variation — e.g., "`--force` skips the protected-branch precondition; without it, exit 5 on protected branch". Or "N/A — behavior uniform across inputs."}

**Shared surfaces invoked:** {names from Shared Surfaces section, or "None."}
```

Repeat the block for every command template in the inventory.

---

## Field-by-field guidance

- **Command path** — the full invocation path including parent commands (e.g., `tool repo push`, not just `push`).
- **Synopsis** — the usage line as a user would see it in `--help`. Include square brackets for optional flags, angle brackets for positionals, and ellipsis for repeated args. Do not invent short-flag aliases that do not exist.
- **Kind** — *action* mutates or fetches data; *utility* is a help / version / completion / doctor-style command with no primary outcome; *diagnostic* reports state (`status`, `check`, `list`) without mutation; *shell-integration* is `completion`, man-page generation, etc.
- **Auth requirement** — "unauthenticated" means the command works without credentials; "authenticated" requires valid credentials; "authenticated-with-permission" requires a specific scope or role — name it.
- **Positional arguments** — if a positional defaults to something (e.g., `<path>` defaults to cwd), say so. Validation column captures format constraints (path must exist, name must match regex, integer must be positive).
- **Flags** — list only command-specific flags. Do not restate global flags; reference them via "standard global flags apply". Boolean flags have no value column entry beyond "boolean". Enum flags list allowed values.
- **Stdin** — state whether and when stdin is read (e.g., "read when `-` is passed as `<path>`"), and the expected format (e.g., "newline-delimited list of identifiers", "raw bytes", "JSON object matching schema X"). If not read, write "Not read."
- **Primary outcome** — one sentence describing the single effect on the world. If the command has no primary outcome, classify it (utility / diagnostic / shell-integration). Commands with two or more legitimate outcomes should be split or justified.
- **Output on stdout** — describe the structural shape, not literal strings. For machine mode, name the top-level fields. Respect the scriptability contract: stdout is data, never logs.
- **Output on stderr** — progress, warnings, log lines, error messages. Human-facing. Never carries the command's primary data output.
- **Side effects** — files written (with paths), network calls made (by endpoint name if a contract registry exists), state mutated. High-level only — not "opens a database transaction and commits after fsync".
- **Failure matrix** — every row from the template, each mapped to an exit code from the taxonomy. Use "N/A — {reason}" for rows that genuinely do not apply to this command. Do not silently drop rows.
- **Interactive vs non-interactive behavior** — three-mode rule. If the command never prompts, write "No interactive prompts — three-mode rule n/a." and skip the three bullets.
- **Idempotency** — *safe* for pure reads (no state change ever), *idempotent* for repeatable writes with the same end state, *mutating* for writes that produce new effects each run (e.g., appending), *destructive* for irreversible operations (delete, overwrite).
- **Argument-dependent behavior** — the key field for preventing per-instance enumeration (Rule 1). Describe how each positional or flag changes behavior beyond trivial input variation. At minimum, commands targeting a named resource must document the not-found and authorization-failure branches.
- **Shared surfaces invoked** — reference by name from the Shared Surfaces section of `CLI_IA.md`. Do not redescribe the surface.

---

## Common mistakes to avoid

- Enumerating concrete invocations (e.g., `tool repo push myrepo`, `tool repo push otherrepo`) instead of the single template (`tool repo push <repo>`).
- Writing code-shaped descriptions ("`PushArgs` struct", "`PushError` enum variant"). Those belong in per-unit SPECs (Rule 6).
- Inventing exit codes outside the taxonomy (Rule 14). If a command needs a code that is not in the taxonomy, extend the taxonomy.
- Mixing progress / log output onto stdout (Rule 15). If in doubt, stdout is data; stderr is everything else.
- Redescribing a shared surface (e.g., confirmation prompt) inline in the per-command entry (Rule 12). Reference it by name.
- Leaving a failure-matrix row as "TBD" or omitting rows entirely. Use "N/A — {reason}" when genuinely inapplicable.
- Writing literal UX copy ("Error: authentication failed") instead of a role description ("auth-failure message on stderr") (Rule 8).
