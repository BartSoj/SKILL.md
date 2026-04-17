---
name: CLI_IA.md
description: Generate a command-line information architecture registry — command inventory, grammar, global flags and environment, per-command blueprints, flows, exit-code taxonomy, terminal capability strategy, and use-case traceability. Use when asked to create a CLI IA, design the command tree, produce a command inventory, define CLI grammar and exit codes, or produce a CLI_IA.md.
---

# Task: Generate CLI_IA.md — Command-Line Information Architecture Registry

## Objective

Produce a CLI_IA.md that serves as the single source of truth for a command-line application's information architecture: the command grammar, global flags and environment, full command-template tree, per-command blueprint (purpose, inputs, outputs, side effects, exit codes, failure matrix, interactive vs non-interactive behavior), multi-command flows, shared interactive surfaces (prompts, progress, pager, error format), exit-code taxonomy, terminal capability strategy, and bidirectional traceability from product use cases to commands. An agent or developer reading this document knows exactly what commands exist, how to invoke them, what they read and write, how they fail, and how every use case is realized on the CLI surface — without making any independent grammar, naming, or exit-code decisions. CLI_IA sits between product requirements (proposal, use cases) and per-unit CLI SPECs, the same way WEB_IA sits between product requirements and per-unit web SPECs.

---

## Inputs

1. **Product / proposal document** (required) — product purpose, target users, and rationale. Sets the scope and audience frame for the IA.
2. **Use cases document** (required) — enumerates user interactions and jobs-to-be-done. Primary input. Every use case with a CLI surface must map to one or more commands in the inventory.
3. **Architecture document** (optional but recommended) — CLI framework choice (clap, cobra, commander, argparse, click, etc.), binary distribution plan, auth model, language. Informs grammar conventions and shared-surface choices. If absent, infer conservative defaults and flag non-obvious choices as open questions.
4. **Existing CLI codebase** (auto-discovered) — discover current commands, subcommands, and flags from source (e.g., clap derive macros, cobra command tree, commander definitions, argparse calls). Treat as reference only — CLI_IA is the target state, not a mirror of the current state.
5. **Decision documents** (optional) — architectural decisions that constrain the IA (e.g., "all destructive commands require confirmation unless `--yes`", "auth via OAuth device flow").
6. **Web IA document** (optional) — if WEB_IA.md exists, use-case traceability must be consistent across both documents. Every use case surfaces on web, CLI, both, or is explicitly classified as having neither surface.
7. **Contract registry** (optional) — if present, per-command input/output may reference endpoints by name; wire formats are owned by CONTRACT_REGISTRY.md, not redefined here.

The user may override scope (e.g., "only the end-user CLI, not the internal admin CLI"). Support that by recording the narrowed scope in the Identity & Context section and excluding out-of-scope binaries from the inventory.

Scope note: this skill covers terminal-invoked binaries that read args / stdin / environment and write stdout / stderr and exit with a status code. Interactive REPLs are in scope. Full TUI applications with a richer UI surface than prompts and progress indicators are out of scope — declare them out of scope in Phase 1 if they arise.

---

## Workflow

CLI IA generation proceeds in seven phases: scope, grammar and globals, inventory, per-command blueprint, flows and shared surfaces, exit codes and terminal capability, traceability and validation. Phases are sequential — later phases depend on earlier decisions — but revisit earlier phases if a later phase reveals a gap.

### Phase 1: Scope & Audience

Derive from the product doc and use cases document the target CLI surface (single binary, multi-binary suite, subcommands of a host tool) and the audiences the IA serves (end user, automation / CI, contributor, operator). Record explicitly what is *not* in scope for this CLI_IA document — GUI wrappers, web surfaces, TUI apps with a richer UI than simple prompts. Keep audience treatment thin — named user types and their 3–7 top tasks per audience. No deep persona work.

### Phase 2: Command Grammar, Global Flags & Environment

Decide grammar and globals before enumerating commands. This prevents per-command drift where each command invents its own flag names or argument conventions.

Decide:
- **Grammar shape** — `tool <noun> <verb>`, `tool <verb> <noun>`, or `tool <verb>`; subcommand depth limit
- **Flag conventions** — short vs long, boolean vs value, repeatable flags, `=` vs space between flag and value
- **Positional vs flag rules** — when a value is a positional argument versus a flag
- **Config-file hierarchy** — locations and precedence (system → user → project → env → flags)
- **Environment-variable naming prefix** — e.g., `TOOL_*` convention, reserved names
- **Global flags** available on every command (typical set: `--help`, `--version`, `--verbose`, `--quiet`, `--json`, `--no-color`, `--config`, `--profile`, `--yes`)
- **Globally honored environment variables** (e.g., `TOOL_*`, `NO_COLOR`, `CLICOLOR`, `CI`)

Use architecture-doc hints where available; fall back to conservative defaults. Surface non-obvious choices as open questions rather than guessing.

### Phase 3: Command Inventory

From use cases and architecture, enumerate the full set of **command templates** (not command instances — see Rule 1). Classify each command: action, utility, diagnostic, or shell-integration. Build a single command-tree table that makes the hierarchy visible (either via indentation or grouped subsections).

For a new product, derive commands from use cases: every use case with a CLI surface implies at least one command where the user accomplishes or progresses that use case. For an existing codebase, discover current commands and reconcile against use cases — adding missing commands, flagging orphan commands.

### Phase 4: Per-Command Blueprint

Iterate every command template in the inventory and produce its full entry. This is the bulk of the output. Each entry must include every mandatory field listed in the Output Format section: command path, synopsis, purpose, kind, audience, auth requirement, referenced use cases, positional arguments, flags, stdin behavior, environment variables, config values, interactive prompts, primary outcome, stdout and stderr output, side effects, full failure matrix with exit codes, interactive-vs-non-interactive behavior, idempotency classification, and argument-dependent behavior.

Work through commands in the order they appear in the inventory tree. When writing outputs, describe each stream by **role and content**, not by literal message strings (see Rule 8).

### Phase 5: Flows & Shared Surfaces

Identify multi-command journeys — `auth login` → `repo init` → `push`; setup wizards; multi-stage operations (prepare → confirm → execute); pipeline composition. For each, write a flow entry: ordered steps pointing to commands in the inventory, entry conditions, success exit, abort behavior, resume semantics, and the use case(s) the flow implements. Flows do not introduce new commands (Rule 11); they sequence existing ones.

Identify shared interactive surfaces that cross commands: confirmation prompts, single-select, multi-select, free-text input, masked password entry, progress indicators (spinner, progress bar, plain-text fallback), pager behavior, table formatting, error-message format, log-level mapping from `--verbose` / `--quiet`, editor invocation for long-form input. Consolidate these in the dedicated Shared Surfaces section (Rule 12). Per-command entries may list which shared surfaces they can invoke, but must not re-describe them.

### Phase 6: Exit Codes, Terminal Capability & Output Modes

Lock the exit-code taxonomy — a single table mapping codes to categories (success, generic error, misuse, auth failure, network failure, interrupt, etc.). Every per-command blueprint's failure matrix draws codes from this taxonomy. If a command needs a code outside the taxonomy, the taxonomy is incomplete — extend it (Rule 14).

Document terminal capability strategy at the IA level:
- **TTY vs non-TTY detection** — affects progress indicators, colors, prompts
- **Color strategy** — auto / always / never, and respect for `NO_COLOR` and `CLICOLOR`
- **Unicode vs ASCII fallback** — graceful degradation for legacy terminals
- **Terminal-width adaptation** — where commands produce wrapped or truncated output
- **Redirect / pipe detection** — suppress color, pager, animations when stdout is not a TTY
- **Pager invocation rules** — auto-pager in TTY for long output; disable via `--no-pager`
- **Scriptability contract** — stdout carries primary output (data, machine-parseable where possible), stderr carries progress, warnings, logs, errors (human-facing); the two streams never interleave

Document output-mode conventions if applicable (`--json`, `--yaml`, `--plain`, `--format`, field-stability contract for scripting, schema location).

### Phase 7: Traceability & Validation

Build the bidirectional use-case ↔ command matrix. Every use case in the input use cases document maps to at least one command, or is explicitly listed as having no CLI surface with a one-line reason. Every command in the inventory is referenced by at least one use case, or is explicitly classified (utility, diagnostic, auth, shell-integration). If WEB_IA.md is an input, cross-reference use cases that surface on both.

Verify every rule holds: every action command has a primary outcome, failure matrix complete per command, every failure exit code exists in the taxonomy, grammar respected across all commands, scriptability contract respected, no implementation detail, no visual-aesthetic decisions, conditional sections either complete or absent. Update frontmatter counts to reflect the final document.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Command templates, not command instances

A parameterized command like `tool repo push <repo>` is one command template, not N commands. Document each command path once. Note the positional arguments and flags and enumerate argument-dependent behavior (e.g., "if `<repo>` is unreachable → exit 6; if private and viewer lacks access → exit 4 with an auth-failure message distinguishable from not-found to avoid information leak"). Do not enumerate concrete invocations.

### 2. Primary outcome is mandatory per non-utility command

Every action command must declare **exactly one** primary outcome — the single effect on the world the command is designed to produce (a file written, state mutated, data fetched and displayed, repo pushed). Commands with two or more legitimate primary outcomes must either be split or explicitly justify the pairing. Utility, diagnostic, and shell-integration commands (`--help`, `--version`, `completion`, `doctor`, `self-update`) carry no primary outcome and must be explicitly classified.

### 3. Failure matrix is mandatory per command

Every per-command blueprint must enumerate, at minimum: success (primary outcome achieved); already-satisfied / no-op (if idempotent); input validation failure; authentication failure (if command requires auth); authorization failure; resource not found (if command targets a named resource); conflict / precondition failed (e.g., dirty working tree, protected branch); network / transient failure (if command is networked); interrupted (SIGINT, SIGTERM mid-execution); non-interactive mode unavailable (prompt required but stdin is not a TTY and no `--yes`); partial success (for multi-item operations, if applicable). For each state describe **what the user sees on stdout / stderr at the category level** and **which exit code** from the taxonomy it uses. Do not describe terminal colors, fonts, or specific emoji — those are out of scope.

### 4. Every command maps to at least one use case or is explicitly classified

A command in the inventory either appears in the traceability matrix under at least one use case, or carries an explicit classification: utility, diagnostic, auth, or shell-integration. Silent orphan commands are not allowed.

### 5. Every use case maps to at least one command or is explicitly noted

Every use case in the input use cases document appears in the traceability matrix mapped to at least one command, or is listed as "no CLI surface" with a one-line reason (e.g., "web-only", "background job, no UI"). Silent drops are not allowed.

### 6. No implementation detail

Do not write parser library types, argument struct fields in code, internal error enum variants, module layout, or function signatures. Those belong in per-unit SPECs. Describe commands by **invocation contract and failure behavior**, not by code shape.

**Worked example — the IA-vs-SPEC line.** This is the single most important boundary in the document; internalize it.

> **In scope (CLI_IA):** "`tool repo push <path>`: positional `<path>` is a directory (defaults to cwd). The `--force` flag suppresses the protected-branch check. Reads `TOOL_AUTH_TOKEN` from environment. Writes progress to stderr as a progress bar in TTY mode and plain timestamped lines in non-TTY. On success, stdout is a one-line human summary; with `--json` it is a single JSON object with `commit_sha` and `url`. Exit 0 on success, 4 on auth failure, 5 on protected-branch rejection without `--force`, 6 on network failure after retries."
>
> **Out of scope (CLI_IA, belongs in per-unit SPEC):** "`PushCommand::execute(args: PushArgs) -> Result<PushSuccess, PushError>` — `PushArgs` has fields `path: PathBuf`, `force: bool`, `auth: AuthToken`; `PushError` is an enum with variants …"

Prefer contract-style descriptions over code-level descriptions throughout the output.

### 7. No visual / aesthetic decisions

No specific color codes, no emoji selection beyond "use sparingly and provide Unicode fallback", no specific box-drawing character or font choices. Those belong downstream in visual / UX execution.

### 8. No final copy

Describe message roles ("auth-failure message", "progress line", "summary line") rather than literal strings. Short disambiguating placeholders are acceptable — no more.

### 9. Command grammar stated once, then enforced

The Command Grammar section is authoritative. Every command in the inventory must conform to the declared conventions (grammar shape, subcommand depth, flag style, positional rules). If a command must deviate (e.g., matching an external-tool convention), document the exception inline in the blueprint entry and explain why.

### 10. Global flags and environment stated once

The Global Flags & Environment section is authoritative. Per-command blueprint entries reference global flags by name — "standard global flags apply" — and never re-document `--help`, `--verbose`, `--json`, etc. Commands that suppress a global flag (e.g., a JSON-only command that rejects `--quiet`) note the suppression in the entry.

### 11. Flows do not introduce new commands

Every flow step references an existing command template in the inventory. If a flow needs a step that has no corresponding command, add that command to the inventory first (with a full blueprint), then reference it from the flow.

### 12. Shared surfaces are consolidated

Prompts, progress indicators, pager, tables, and error-message patterns live in the Shared Surfaces section — not scattered across per-command entries. Per-command entries may list which shared surfaces they can invoke by name; they must not redefine them.

### 13. Conditional sections must be omitted, not left empty

If Output Modes, Shell Integration, Installation & Update Channels, Authentication State & Profiles, Long-Running Operations, Accessibility, Internationalization, or Telemetry sections are not applicable for the project, omit them entirely. Do not leave them present with "N/A" content. Tight output is the house style.

### 14. Exit-code taxonomy stated once, enforced across commands

The Exit-Code Taxonomy section is authoritative. Every failure-matrix row's exit code must exist in the taxonomy. A command that needs a code outside the taxonomy means the taxonomy is incomplete — extend the taxonomy, do not silently invent new codes. This skill does not prescribe a specific taxonomy (sysexits, HTTP-like three-digit, simple 0–127) — the product decides; this document enforces that one exists, is stated once, and every command conforms.

### 15. Scriptability contract is universal

Stdout carries the command's primary output (data, machine-parseable where possible). Stderr carries progress, warnings, logs, and error messages (human-facing). The two streams never interleave — a consumer piping stdout to another command must not receive log noise. Every per-command blueprint's stdout / stderr descriptions must respect this contract, and every command that can prompt the user must document its behavior across the three modes defined in the Interactive Behavior sub-section (TTY, `--yes` / `CI=true`, non-interactive without blanket confirmation).

### 16. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc." Use exact command paths, flag names, exit codes, and use-case references. If you cannot be exact, flag it as an open question rather than hiding behind a placeholder word.

---

## Output Format

```markdown
---
skill: CLI_IA.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
commands_documented: {N}
flows_documented: {N}
shared_surfaces: {N}
exit_codes_defined: {N}
use_cases_covered: {N}
use_cases_total: {N}
auth_commands: {N}
destructive_commands: {N}
open_questions: {N}
---

# CLI_IA

> Single source of truth for the command-line application's information architecture.
> Every command template, grammar rule, global flag, flow, shared surface, and exit
> code is authoritative — per-unit CLI SPECs must reference these entries.

## Identity & Context

{One paragraph: product name, the binary or binaries this document covers, and
 the audience frame. Reference the product proposal by path. State explicitly
 what surfaces are out of scope — GUI wrappers, web surfaces, TUI apps,
 internal-only admin binaries, etc.}

## Audience & Top Tasks

| Audience | Top tasks on the CLI |
|----------|---------------------|
| {audience name} | {3–7 comma-separated top tasks} |

(Repeat for each audience identified in the product doc.)

## Command Grammar

- **Grammar shape:** {`tool <noun> <verb>` | `tool <verb> <noun>` | `tool <verb>` — with rationale}
- **Subcommand depth:** {maximum depth — e.g., two levels}
- **Flag conventions:** {short / long rules; boolean vs value; repeatable flag behavior; `=` vs space between flag and value}
- **Positional vs flag rules:** {when a value is positional versus a flag}
- **Config-file hierarchy:** {locations and precedence — e.g., system → user → project → env → flags}
- **Environment-variable prefix:** {e.g., `TOOL_*` — reserved names, casing}
- **Exceptions:** {any commands that deviate from the grammar, with reason — or "None."}

## Global Flags & Environment

### Global flags (available on every command)

| Flag | Value | Purpose |
|------|-------|---------|
| `--help` / `-h` | boolean | {purpose} |
| `--version` | boolean | {purpose} |
| `{flag}` | {type} | {purpose} |

(Repeat for every global flag.)

### Globally honored environment variables

| Variable | Purpose | Precedence vs flags |
|----------|---------|---------------------|
| `{VAR}` | {purpose} | {rule} |

(Repeat for every global env var.)

### Config file load order

{Ordered list — e.g., "1. `/etc/tool/config.toml` (system), 2. `~/.config/tool/config.toml` (user), 3. `./.toolrc` (project), 4. environment, 5. flags."}

## Command Inventory (Tree)

| Command path | Kind | Auth | Audience | Use cases | Purpose |
|--------------|------|------|----------|-----------|---------|
| `tool {verb}` | action / utility / diagnostic / shell-integration | unauthenticated / authenticated / authenticated-with-permission | {audience} | {UC-IDs or classification} | {one line} |
| `tool {noun} {verb}` | {kind} | {auth} | {audience} | {UC-IDs} | {one line} |

(Group by area — user-facing actions, admin, utilities, diagnostics — via
 subsection headings or visible indentation. Repeat for every command template.)

---

## Per-Command Blueprints

Every command template in the inventory gets a full blueprint entry here. The
blueprint template — with every required field, its purpose, and common
mistakes to avoid — lives in
[`references/per-command-blueprint.md`](references/per-command-blueprint.md).
Read that file before writing the first entry; then copy the block per command
and fill in every field.

Entries in the output appear in the order they appear in the Command Inventory
tree, grouped by the same area headings.

### `{command path}` — {short name}

{Full blueprint block per `references/per-command-blueprint.md`.}

---

(Repeat the blueprint block for every command template in the inventory.)

---

## Flows

### {Flow name}

- **Purpose:** {one sentence}
- **Use cases implemented:** {UC-IDs}
- **Entry condition:** {what triggers the flow — e.g., "user runs `tool init` in a non-configured directory"}
- **Steps:**
  1. `{command path}` — {what the user does here; what state is produced for the next step}
  2. `{command path}` — {what the user does here}
- **Success exit:** {terminal state — e.g., "remote repo created, local `.tool/config` written, first push succeeded"}
- **Abort behavior (Ctrl-C mid-flow):** {what happens if the user interrupts; what is preserved}
- **Resume semantics:** {idempotent re-run? checkpoint file? start from last completed step?}

(Repeat for each flow. If none: "No multi-command flows in scope.")

---

## Shared Surfaces

| Surface | Kind | Trigger | Dismissal / completion | TTY vs non-TTY | Invoked by commands |
|---------|------|---------|------------------------|----------------|---------------------|
| {name} | {confirmation prompt / single-select / multi-select / free-text / masked password / spinner / progress bar / pager / table / error block / editor} | {what opens it} | {how it closes} | {behavior in each mode} | {command paths or "multiple"} |

(Repeat for every shared surface. If none: "No shared surfaces in scope.")

---

## Exit-Code Taxonomy

| Code | Category | Description |
|------|----------|-------------|
| `0` | success | Primary outcome achieved. |
| `{code}` | {category — e.g., misuse, auth, network, conflict, interrupted, non-interactive, generic error} | {one-line description of when this code is used} |

(Every exit code that appears in any command's failure matrix must be listed here.
 A command that needs a code outside this taxonomy means the taxonomy is incomplete
 — extend it.)

---

## Terminal Capability & Output Modes

- **TTY detection:** {when stdout / stderr is a TTY vs not; what changes based on detection — e.g., progress indicators, colors, prompts}
- **Color strategy:** {auto / always / never rule; respect for `NO_COLOR` and `CLICOLOR`; override flag name}
- **Unicode vs ASCII fallback:** {when ASCII fallback is used; what is substituted — e.g., progress characters, tree glyphs}
- **Terminal-width adaptation:** {which commands produce width-adapted output; truncation vs wrap rule}
- **Redirect / pipe detection:** {what is suppressed when stdout is not a TTY — color, pager, animations, spinners}
- **Pager invocation:** {auto-pager trigger — e.g., output exceeds terminal height; `--no-pager` override; which pager is invoked is implementation}

### Scriptability Contract

- **stdout** carries the command's primary output — data, machine-parseable where possible.
- **stderr** carries progress, warnings, logs, and error messages — human-facing.
- **The two streams never interleave.** A consumer piping stdout to another command must not receive log noise on stdout.

Every per-command blueprint's stdout / stderr descriptions respect this contract.

---

## Output Modes

(Omit this section entirely if the CLI has only human-readable output.)

- **Supported modes:** {list — e.g., `--json`, `--yaml`, `--plain`, `--format <template>`}
- **Default mode:** {rule — e.g., "human-readable in TTY, JSON when stdout is not a TTY"}
- **Field-stability contract:** {which fields are stable for scripting; deprecation policy for renames / removals}
- **Schema location:** {where JSON schemas or field reference docs live, or "generated inline via `--help`"}

---

## Shell Integration

(Omit this section entirely if the CLI ships no completions or man pages.)

- **Completions:** {shells supported — bash / zsh / fish / powershell; generation command — e.g., `tool completion bash`; install location}
- **Man pages:** {generation command; install location; auto-install rule}
- **Dynamic completion:** {which commands have dynamic completions — e.g., remote resource names, config keys}

---

## Installation & Update Channels

(Omit this section entirely if standard distribution with nothing CLI-IA-level to say.)

- **Distribution channels:** {list — brew, npm, cargo, go install, signed binary downloads}
- **Version channels:** {stable / beta / nightly rules}
- **Self-update command:** {command path, or "None — out-of-band updates only"}
- **Upgrade prompts:** {when the CLI nags about stale versions; how to disable}

---

## Authentication State & Profiles

(Omit this section entirely if the CLI has no authenticated operations.)

- **Authentication flows:** {interactive login command, token env var, OAuth device flow — pick one or document multiple}
- **Credential storage:** {location — e.g., OS keychain, `~/.config/tool/credentials`; encryption rule}
- **Profile switching:** {command; `--profile` flag behavior; default-profile rules}
- **Commands honoring the profile:** {which commands read credentials and which bypass profile scope}

---

## Long-Running Operations

(Omit this section entirely if no command runs long enough to matter.)

- **Which commands can run long:** {list with expected duration bands}
- **Progress indication:** {which shared surface — spinner, progress bar, plain-text fallback}
- **Cancellation semantics:** {per command — does Ctrl-C roll back, commit partial, or prompt?}
- **Resumption:** {restart-from-checkpoint rule; idempotent replay rule; resume command if any}

---

## Accessibility

(Omit this section entirely if accessibility is out of scope for this phase.)

- **Screen-reader friendliness:** {prompt and progress rules — e.g., no spinner-only progress; text status lines alongside any animated indicator}
- **Unicode fallback:** {ASCII equivalent for every Unicode glyph used}
- **Color contrast:** {minimum contrast when color is used; never-color-only-cue rule}

---

## Internationalization

(Omit this section entirely if single-locale.)

- **Locale detection:** {rule — e.g., `LC_ALL` / `LANG` / `TOOL_LOCALE` precedence}
- **Message catalog:** {framework — gettext or equivalent; file layout}
- **RTL considerations:** {rule for tables and boxed output, or "No RTL locales in scope."}

---

## Telemetry

(Omit this section entirely if no telemetry.)

- **Opt-in vs opt-out:** {rule}
- **What is collected:** {high-level — e.g., command name, exit code, duration. Never argument values.}
- **Disable mechanism:** {env var — e.g., `TOOL_NO_TELEMETRY=1`; flag; config key}
- **Privacy contract:** {what the CLI promises not to send}

---

## Traceability Matrix

### Use cases → commands

| Use case | Command(s) | Notes |
|----------|-----------|-------|
| {UC-ID} | `{command path}`, `{command path}` | {brief note, or "Primary surface."} |
| {UC-ID} | — | No CLI surface — {one-line reason}. |

(Every use case from the input document appears here. If WEB_IA.md is present,
 note in the third column when a use case also surfaces on web.)

### Commands → use cases

| Command | Use case(s) | Classification if none |
|---------|-------------|------------------------|
| `{command path}` | {UC-IDs} | — |
| `{command path}` | — | utility / diagnostic / auth / shell-integration |

(Every command from the inventory appears here.)

---

## Relationship to Other Documents

- **Product proposal and use cases:** This document realizes use cases on the CLI surface. Use-case IDs in the traceability matrix match the source use cases document.
- **Per-unit SPECs:** SPECs for CLI units must reference the CLI_IA entry for the command(s) or shared surface(s) they implement, and use command paths, flag names, and exit codes from this document verbatim.
- **CONTRACT_REGISTRY.md** (if present): per-command network calls may reference endpoint names; wire-format shapes are owned by the contract registry, not redefined here.
- **WEB_IA.md** (if present): web surfaces are a parallel document. The use-case traceability must be consistent across both — every use case surfaces on web, CLI, both, or is explicitly classified as having neither surface.
- **Visual / UX execution:** specific color codes, emoji, box-drawing characters, and literal message strings are owned downstream of this document, not here.

---

## Open Questions

- [ ] {Question — e.g., "Should destructive commands default to dry-run, requiring an explicit `--execute`, or default to execute with `--dry-run` available?"}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Full command-template inventory with grammar, hierarchy, and classification (action / utility / diagnostic / shell-integration)
- Per-command blueprint: purpose, invocation contract (positionals, flags, stdin, env, config), output contract (stdout, stderr, side effects), failure matrix with exit codes, interactive-mode behavior, idempotency classification, argument-dependent behavior
- Command grammar and global-flag / environment conventions
- Multi-command flow descriptions
- Shared interactive-surface inventory (prompts, progress, pager, tables, error patterns, editor invocation)
- Exit-code taxonomy
- Terminal capability strategy (TTY, color, Unicode, width, redirect, pager) and the universal scriptability contract
- Bidirectional traceability between use cases and commands
- Conditional sections (output modes, shell integration, installation, auth state, long-running operations, accessibility, i18n, telemetry) when applicable to the product

### Out of scope

- Implementation details — parser library, command struct layout, error enum variants, module organization, function signatures — owned by per-unit `SPEC.md`
- Final UX copy for messages and prompts — owned by a UX-writing phase
- Visual terminal aesthetics — specific colors, emoji, box-drawing character choices, spinner frames — owned downstream of this document
- Backend / API wire formats — owned by `CONTRACT_REGISTRY.md`
- Web information architecture — owned by `WEB_IA.md`
- Native, mobile, GUI, and full TUI information architecture — this skill is CLI-only (terminal-invoked binaries; interactive REPLs included; richer TUIs excluded)
- Database schema, business logic, algorithm design — owned by architecture and unit SPECs
- Binary packaging internals, signing infrastructure, release pipeline — owned by DEPLOYMENT docs (distribution *channels* may be named in the conditional Installation section; the *pipeline* is not)
- Persona research and user research — owned by a separate research phase

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `commands_documented`, `flows_documented`, `shared_surfaces`, `exit_codes_defined`, `use_cases_covered`, `use_cases_total`, `auth_commands`, `destructive_commands`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] Every command template in the inventory tree has a full per-command blueprint entry below
- [ ] Every per-command entry declares command path, synopsis, kind, auth requirement, positionals, flags, stdin behavior, env vars, config values, primary outcome (or utility / diagnostic / shell-integration classification), stdout and stderr output, side effects, full failure matrix with exit codes, interactive-mode behavior, idempotency, and argument-dependent behavior
- [ ] Every command with positional arguments or flags documents argument-dependent behavior (at minimum: validation-failure and not-found branches for commands targeting named resources)
- [ ] Every use case in the input use cases document maps to at least one command in the traceability matrix, or is explicitly listed as having no CLI surface with a one-line reason
- [ ] Every command in the inventory is referenced by at least one use case in the traceability matrix, or is explicitly classified (utility, diagnostic, auth, shell-integration)
- [ ] Command grammar is stated once and every command in the inventory conforms (or carries an explicit exception note)
- [ ] Global flags and environment variables are documented in exactly one section; per-command entries reference them by name and do not re-document them
- [ ] Exit-code taxonomy is present; every exit code used in any failure matrix exists in the taxonomy
- [ ] Scriptability contract (stdout = data, stderr = human, no interleaving) is stated in the Terminal Capability section; every command's stdout / stderr description respects it
- [ ] Every command that can prompt documents its behavior across all three modes (TTY, `--yes` / `CI=true`, non-interactive without blanket confirmation)
- [ ] Every flow step references an existing command template from the inventory (no flow introduces new commands)
- [ ] Shared surfaces appear only in the dedicated Shared Surfaces section, not scattered in per-command entries
- [ ] Conditional sections (output modes, shell integration, installation, auth state, long-running, accessibility, i18n, telemetry) are either fully completed or fully omitted — never present-and-empty
- [ ] No implementation detail (parser types, internal enums, module layout, function signatures) appears anywhere in the document
- [ ] No specific color codes, emoji selections, box-drawing or spinner-frame choices appear anywhere
- [ ] No final literal UX copy — message roles are described, not literal strings
- [ ] Frontmatter `use_cases_covered` and `use_cases_total` are present; any gap is explained in the Traceability Matrix
- [ ] If `WEB_IA.md` is an input, the traceability matrix cross-references use cases that surface on both web and CLI
