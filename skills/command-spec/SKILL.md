---
name: COMMAND_SPEC.md
description: Produce a deep per-command design specification for a single command in a command-line application — argument-parsing rules beyond CLI_IA's grammar, stdin/stdout/stderr composition, every output mode's shape, table and block layout in human mode, interactive choreography (prompts, progress, pager) across the three-mode rule, signal handling (SIGINT/SIGTERM/SIGHUP/SIGPIPE), per-failure cleanup and recovery hints, capability degradation across color and Unicode tiers, and idempotency / re-run semantics. Use when asked to create a COMMAND_SPEC for a CLI command, write a deep command spec, extend a CLI_IA per-command blueprint with composition detail, design a command's output modes and signal handling, or produce a COMMAND_SPEC.md.
---

# Task: Create COMMAND_SPEC.md for a CLI Command

## Objective

Produce a COMMAND_SPEC.md that serves as the single source of truth for one command's *deep composition rules*: how arguments parse beyond the grammar declared in CLI_IA, how stdin / stdout / stderr compose under each output mode, how human-mode tables and blocks lay out using DESIGN.md tokens, how interactive prompts and progress and pager choreograph across the three-mode rule, how the four primary POSIX signals are handled, how each failure listed in CLI_IA's matrix is detected and cleaned up and recovered from, and how the rendering degrades across terminal capability tiers. The defining claim — and the definition of "done" — is that **an implementing agent reading CLI_IA's per-command blueprint plus DESIGN.md's terminal style tokens plus this COMMAND_SPEC can compose the command exactly as designed across all output modes and capability tiers, without inventing parsing rules, output shapes, prompt choreography, signal-cleanup steps, or capability-degradation rules.**

COMMAND_SPEC is the rarest of the per-item leaf specs in the framework — most CLI commands are simple enough that the inline blueprint at `CLI_IA.md` is sufficient. Use this skill only for commands that exceed the inline form: dense interactive prompts, materially different shapes across `--json` / `--quiet` / human modes, intricate flag interactions, non-trivial signal handling, multi-stage progress reporting, or extensive failure-recovery semantics. Most projects will have few or zero COMMAND_SPEC files.

The commonest violation is to overlap with CLI_IA. CLI_IA owns *identity* — that the command exists, its grammar, its global flags by reference, its primary outcome, its failure-matrix summary, its exit-code taxonomy, its use-case traceability. COMMAND_SPEC owns *composition* — the parsing rules beyond the grammar, the output composition beyond the contract, the interactive choreography beyond the surface name, the signal handling beyond the failure-row name, the capability degradation beyond the tier names. When COMMAND_SPEC restates CLI_IA, the two documents drift; cite by reference instead.

---

## Inputs

1. **Command identifier** (required) — the stable command identifier from CLI_IA.md. Conventionally the canonical command path (`tool repo push`) or an assigned ID. The skill produces one COMMAND_SPEC per command identifier.
2. **CLI_IA.md** (required) — the parent surface IA document. Read in full: command grammar, global flags, environment variables, exit-code taxonomy, terminal capability strategy, scriptability contract, shared interactive surfaces, and the target command's per-command blueprint entry. Every citation in this COMMAND_SPEC resolves into CLI_IA.
3. **DESIGN.md** (required if present in the project) — the terminal style-token document. Read in full: color palette and tiers, glyph set with ASCII fallbacks, table style, error-block style, prompt style, progress style. Every visual decision in COMMAND_SPEC's human-mode and capability-degradation sections cites a token by name. If DESIGN.md is absent for the project, surface the absence as an open question rather than inventing tokens.
4. **Sibling COMMAND_SPEC.md files** (optional, auto-discovered) — for each command this command hands off to or receives output from in a CLI_IA flow, load that command's COMMAND_SPEC if it exists. Use only as a cross-reference for naming consistency; never copy content.
5. **Prior `cli-commands/<cmd-id>/COMMAND_SPEC.md`** (optional, auto-discovered — only if refining) — read fully; preserve every resolved open question, every assigned mode shape, every cleanup step already specified. Re-run citations against the current CLI_IA and DESIGN — do not assume stale tokens or exit-code IDs are still valid.

Read-set size: CLI_IA's full target-command blueprint plus the surrounding policy sections (grammar, globals, exit codes, capability) and DESIGN.md when present are always read; sibling COMMAND_SPECs are read only when CLI_IA's flow entries name them.

The output file is conventionally placed at `cli-commands/<cmd-id>/COMMAND_SPEC.md` in the consuming project, where `<cmd-id>` is the stable identifier from CLI_IA.

---

## Selection Criteria

COMMAND_SPEC is opt-in and is the rarest of the per-item specs in the framework. Do **not** create one for a command unless at least one of the following holds:

- The composition would exceed roughly thirty lines if expressed inline in CLI_IA's per-command blueprint entry.
- The command supports two or more output modes whose shapes differ materially (not just "the same data, JSON vs human" — distinct top-level structures).
- The command has rich interactive prompts (multi-step wizard), multi-stage progress reporting (more than a single bar), or invokes a pager.
- The command launches an embedded TUI surface (in which case a separate VIEW_SPEC may also exist for the launched view; COMMAND_SPEC owns the launching command, VIEW_SPEC owns the view).
- Signal handling is non-trivial — graceful shutdown with cleanup, resume-from-interrupt, side-effect rollback.
- Failure recovery requires command-specific cleanup steps beyond exiting with a code (lockfiles, partial state, temp directories).
- Multiple work units touch the same command and benefit from a shared composition contract.

If none of the above holds, do not produce a COMMAND_SPEC. Stay with CLI_IA's inline blueprint. Most commands fall into this category.

---

## Rules

### Single source of truth — no overlap with CLI_IA

CLI_IA owns command identity and contract; COMMAND_SPEC owns composition. The boundary:

- **CLI_IA owns** — that the command exists; its grammar (`tool {verb} {noun}`); its place in the inventory tree; its global flags by reference; its primary outcome (one sentence); its failure-matrix summary (rows + exit code per row); its use-case traceability; the exit-code taxonomy; the shared-surface inventory by name; the terminal capability tier names.
- **COMMAND_SPEC owns** — argument-parsing order sensitivity and conflict resolution; default-from-context derivation; auto-detection rules; argument expansion semantics; per-stream composition rules; the exact JSON shape for each machine mode; the table layout, block ordering, and color usage in human mode; the prompt and progress and pager choreography; the signal-cleanup steps; the per-failure detection signal and stderr message structure and recovery hint and cleanup; the per-capability-tier rendering rules; the idempotency / lock / resume / dry-run semantics.

Do not redefine CLI_IA's grammar, globals, exit codes, primary outcome, failure-matrix summary, or traceability in this document. Reference them. When CLI_IA and COMMAND_SPEC appear to disagree on any owned concern, CLI_IA decides; this document is updated to match.

### Citation rules

Every cross-document reference uses the cited document's stable identifier verbatim:

- **Design tokens** are cited by name from DESIGN.md (`color-warning`, `glyph-bullet-primary`, `table-style-compact`, `prompt-style-default`, `progress-style-bar`). Do not paraphrase; do not introduce a new token.
- **Exit codes** are cited by the ID and integer assigned in CLI_IA's Exit-Code Taxonomy (e.g., `EX-AUTH (4)`). Never invent a code; never use a bare integer without its taxonomy ID.
- **Capability tiers** are cited by the names CLI_IA's Terminal Capability section uses (e.g., "24-bit color", "256 color", "16 color", "no color", "Unicode", "ASCII-only", "narrow terminal"). Do not invent tier names.
- **Shared interactive surfaces** are cited by name from CLI_IA's Shared Surfaces section (e.g., "confirmation prompt", "single-select", "progress bar"). Do not redefine the surface inline; reference it.
- **Flags** are cited by their long form (`--json`, `--quiet`, `--no-color`, `--yes`), never by short form alone. Short forms are mentioned only in the Flags table in section 2.
- **Environment variables** are cited in `UPPER_SNAKE_CASE` exactly as declared in CLI_IA (e.g., `NO_COLOR`, `TOOL_AUTH_TOKEN`).
- **Sibling commands** in flow citations use the canonical command path from CLI_IA's inventory (e.g., `tool repo push`).

### Forbidden patterns

- **Defining new exit codes.** CLI_IA's Exit-Code Taxonomy is authoritative. If a failure in this command needs a code that does not exist, surface it in section 12 Open Questions and re-run CLI_IA before re-running this skill.
- **Brand color decisions.** Hex codes, RGB triplets, named-from-thin-air color words. Cite DESIGN.md tokens by name only.
- **Implementation library names.** No `clap`, `cobra`, `commander`, `argparse`, `click`, `cli-ux`, `kingpin`, `pflag`, `yargs` — those belong in per-unit SPEC.
- **Final user-facing copy.** CLI_IA owns prompt and message structure at the role level; COMMAND_SPEC owns composition rules (where the prompt appears, how it composes with progress, how it degrades). Concrete English sentences are owned by a UX-writing phase.
- **Restating CLI_IA.** Do not redefine grammar, global flags, exit-code taxonomy, primary outcome, failure-matrix row presence, use-case traceability, or capability-tier definitions in this document.
- **Code or pseudocode.** Compose by prose and tables. Pseudocode is permitted only when a specific algorithm is non-obvious and cannot be described unambiguously in prose; in that case, include a one-line note explaining why prose was insufficient.

### Strict template

Twelve sections, every section mandatory. If a concern genuinely does not apply — the command is not interactive, has no signal-specific behavior beyond defaults, or supports only one output mode — the section heading remains and its body states the no-op explicitly (e.g., "Non-interactive only; no prompts, progress, or pager.", "Default signal disposition; no command-specific cleanup."). Silent omission is forbidden; an empty section is a quality-checklist failure.

### Single YAML frontmatter block

- Exactly one YAML frontmatter block at the top of the COMMAND_SPEC.md output, never two — even when refining an existing COMMAND_SPEC, merge into a single block. The block carries every field enumerated in the Output Format template (`skill`, `date`, `status`, `command_id`, `flags_specified`, `output_modes_specified`, `interactive_surfaces_specified`, `signals_handled`, `failure_modes_specified`, `capability_tiers_specified`, `open_questions`).
- Every count in the frontmatter matches the body exactly: `flags_specified` is the row count of section 2's Flags table; `output_modes_specified` is the number of output-mode subsections in section 5; `interactive_surfaces_specified` is the number of surfaces enumerated in section 7 (zero when the section reads "Non-interactive only"); `signals_handled` is the number of signal entries in section 8 (always at minimum the four primary signals SIGINT / SIGTERM / SIGHUP / SIGPIPE — when one has default disposition, it still counts as a row); `failure_modes_specified` is the number of failures detailed in section 9, which must equal the row count of CLI_IA's failure-matrix summary for this command minus any "N/A" rows; `capability_tiers_specified` is the number of tier subsections in section 10 (always the full set CLI_IA's Terminal Capability section enumerates); `open_questions` is the count of unresolved questions in section 12 (zero when section 12 reads "All questions resolved.").
- `status` is `complete` only when section 12 reads "All questions resolved."; `has_open_questions` when one or more questions remain; `blocked` only when a missing input (an absent CLI_IA, an absent DESIGN.md when the project requires one, an unregistered exit code the command needs) prevents authoring and section 12 makes the gap explicit.

### Precision rules

- Use exact names — flag long forms, environment-variable names, capability-tier names, shared-surface names, design-token names, exit-code IDs, sibling-command paths. No "appropriate", "relevant", "as needed", "etc."
- Describe stream composition by ordered structure ("First line: header summary; subsequent lines: one record per row; final line: total summary"), not by vague summary ("appropriate tabular output").
- Distinguish between "writes nothing" and "writes empty" — state which one, and what triggers the empty-output case.
- Distinguish between "exits silently" and "exits with a recovery hint" — state which one per failure row.

### Three-mode interactive rule

Every command that can prompt must document its behavior across all three modes — interactive (TTY, no `--yes`), non-interactive with blanket confirmation (`--yes` or `CI=true`), and non-interactive without blanket confirmation (pipe / `--no-input` / no TTY and no `--yes`). The third mode typically resolves to "exit with the non-interactive mode unavailable failure code from CLI_IA's taxonomy". State the resolution per surface, not just generally. If the command never prompts, state "Non-interactive only; no prompts, progress, or pager." in section 7 and skip the three-mode bullets.

### Capability degradation completeness

Every tier listed in CLI_IA's Terminal Capability section must appear as a subsection in section 10 — full color tier, mid-color tiers, no-color, Unicode, ASCII-only, narrow-terminal. For each tier, state what is rendered differently versus the previous-richer tier (palette swap, glyph substitution, column collapse, wrap policy). Do not omit a tier on the grounds that "behavior is the same as the higher tier" — state the no-op explicitly ("Same palette as 24-bit; no degradation.").

### Failure-matrix completeness

Section 9 details every failure row CLI_IA's failure-matrix summary lists for this command, in the same order, by the same name. Each row provides: the detection signal (what condition triggers it), the exit code (citing the taxonomy ID and integer), the stderr message format (one-line summary; optional detail block; optional remediation hint — describe the structure, not the literal copy), the recovery hint (what the user should try), and the cleanup steps (what state must be removed before exiting — lockfiles, temp files, partial output). Rows CLI_IA marks "N/A — {reason}" are skipped here, preserving the reason in a one-line note. Rows that exist in CLI_IA but cannot be detailed must surface as open questions in section 12.

---

## Output Format

The COMMAND_SPEC.md file must follow this exact structure. Every section is mandatory. If a section's concern does not apply to this command, include the heading with an explicit no-op statement underneath.

```markdown
---
skill: COMMAND_SPEC.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
command_id: {command-id-from-CLI_IA}
flags_specified: {N}
output_modes_specified: {N}
interactive_surfaces_specified: {N}
signals_handled: {N}
failure_modes_specified: {N}
capability_tiers_specified: {N}
open_questions: {N}
---

# COMMAND_SPEC: `{command path}` — {short name}

## 1. Identity & Cross-references

- **Command ID:** {ID from CLI_IA inventory}
- **Canonical invocation:** `{full grammar — e.g., tool repo push [<path>] [--force] [--json]}`
- **CLI_IA blueprint:** `CLI_IA.md` § {section anchor} — read this entry first; this COMMAND_SPEC extends it with composition rules.
- **Tokens cited from DESIGN.md:**
  - **Colors:** `{token-name}`, `{token-name}`
  - **Glyphs:** `{token-name}`, `{token-name}`
  - **Table style:** `{token-name}`
  - **Error-block style:** `{token-name}`
  - **Prompt style:** `{token-name}`
  - **Progress style:** `{token-name}`
  *(If DESIGN.md is absent: state "DESIGN.md not present in project — visual choices surfaced as open questions in section 12.")*
- **Sibling commands referenced in flows:** `{command path}` (handoff target / handoff source); `{command path}` (...). *(Or "None — command stands alone in CLI_IA flows.")*

---

## 2. Synopsis & Argument Structure

**Synopsis:** `{usage(1)-style synopsis — e.g., tool repo push [<path>] [--force] [--json]}`

### Positional arguments

| Name | Type | Required | Default | Validation | Env-var fallback |
|------|------|----------|---------|------------|------------------|
| `<name>` | {type} | {yes/no} | {default or "—"} | {validation rule} | {`UPPER_SNAKE` or "—"} |

*(Or "No positional arguments.")*

### Flags

| Long form | Short | Type | Default | Env-var | Mutually exclusive with | Required with | Purpose |
|-----------|-------|------|---------|---------|-------------------------|---------------|---------|
| `--{flag}` | `-{s}` | {boolean / string / integer / enum / repeated} | {default} | `UPPER_SNAKE` or "—" | `--{flag}` or "—" | `--{flag}` or "—" | {one-line role} |

*(Standard global flags are documented in CLI_IA § Global Flags & Environment and apply by reference. Only command-specific flags appear here.)*

### Subcommand structure

{If the command has subcommands: enumerate them by name with one-line role each. Each subcommand should have its own COMMAND_SPEC if it meets the selection criteria. Otherwise: "No subcommands.")}

### Argument normalization

- **Case folding:** {rule — e.g., "flag names case-sensitive; positional `<name>` lowercased before validation"}
- **Whitespace stripping:** {rule}
- **Prefix expansion:** {rule for unique-prefix flag matching, or "Not supported — full flag names required."}

---

## 3. Argument Parsing Rules

Beyond the grammar declared in CLI_IA.

- **Order sensitivity:** {does flag order matter; can flags appear after positionals; can `--` terminate flag parsing}
- **Conflict resolution:** {when two mutually-exclusive flags are both present, behavior — last-wins, first-wins, or error with which exit code from the taxonomy}
- **Default-from-context:** {when a flag is absent, what context supplies a default — config-file key, environment variable, a previous-invocation marker file. Per-flag rules.}
- **Auto-detection:** {behavior that depends on context — TTY check on stdout, pipe detection on stdin, presence of a marker env var like `CI`, presence of a config file. Per-rule.}
- **Argument expansion:** {glob expansion rules; brace expansion if supported; file-list expansion via `@filename`; or "No expansion — arguments are taken literally."}

---

## 4. Stdin / Stdout / Stderr Composition

### Stdin

- **When read:** {condition — e.g., "read when `<path>` is `-`", "always read when stdin is not a TTY", "never read"}
- **Format expected:** {format — e.g., "newline-delimited list of identifiers", "raw bytes", "JSON object matching the shape declared in section 5"}
- **End-of-input detection:** {EOF / blank line / sentinel marker}
- **Malformed input handling:** {which failure row in section 9 it triggers; what is salvaged before exit}
- *(Or "Not read.")*

### Stdout

- **Reservation:** Stdout is reserved exclusively for the command's primary output per CLI_IA's scriptability contract. No progress, no warnings, no logs ever appear on stdout.
- **Mode-dependent shape:** Per-mode structural composition is detailed in section 5.
- **Empty-output trigger:** {when stdout is intentionally empty — e.g., "no records match the query in machine mode"; or "Stdout is never empty in any mode."}

### Stderr

- **Composition order:** {what stderr emits and in what order — e.g., "1. progress lines during execution; 2. warnings as they arise; 3. on failure, the error block; 4. on success in `--verbose`, the summary line"}
- **Color rules:** {which lines colored, by DESIGN.md token name; suppression rules per capability tier — full detail lives in section 10}
- **Stream interleaving:** {explicit confirmation that stdout and stderr never interleave; if the command writes both, state which is flushed when}

---

## 5. Output Modes

For each output mode this command supports, a subsection. Common modes: human-readable (default), `--json`, `--yaml`, `--quiet`, `--verbose`, `--plain`. Document only the modes the command actually supports per CLI_IA's blueprint.

### Human mode (default)

- **Triggering:** Default mode when stdout is a TTY and no machine flag is set. Suppressed when {condition — e.g., "stdout is not a TTY and `--json` was not requested → defaults to `--json` per CLI_IA convention"}.
- **Shape:** {top-level structural composition — e.g., "header line; tabular body; footer summary line"}. Detailed table layout in section 6.
- **Color usage:** Cited by token name; full rules in section 6.
- **Truncation:** {what wraps, what truncates, what triggers a pager — full pager rules in section 7}

### `--json` mode

- **Triggering:** `--json` flag, or auto-selected when stdout is not a TTY and the command supports it. Per CLI_IA's Output Modes section.
- **Top-level shape:**
  ```json
  {
    "{field}": "{type}",
    "{field}": "{type or nested object}"
  }
  ```
- **Field naming convention:** {snake_case / camelCase per CLI_IA's wire-format conventions; cite the section if applicable}
- **Pretty-print rules:** {indent, key ordering, line breaks. Default pretty-print in TTY; compact one-line per record otherwise.}
- **Truncation rules:** {none — JSON is never truncated; or specific truncation policy if applicable}

### `--quiet` mode

- **Triggering:** `--quiet` flag, or `TOOL_QUIET=1` env var.
- **Shape:** {minimal output — e.g., "stdout: empty on success, error code only on failure; stderr: error block on failure"}
- **Suppression:** {what is suppressed vs default — progress, summary lines, warnings}

*(Repeat for each additional mode the command supports — `--verbose`, `--yaml`, `--plain`, `--format <template>`, etc. Or "Only human mode and `--json` mode supported.")*

---

## 6. Output Layout (human mode)

### Table composition

*(Include this subsection only if the human-mode output is tabular. Otherwise: "Output is not tabular.")*

- **Column order:** {ordered list with one-line role per column}
- **Alignment:** {left / right / center, per column}
- **Color, by column:** {DESIGN.md token name per column-or-cell-role}
- **Max width per column:** {policy — fixed, content-derived, capped at terminal-width fraction}
- **Truncation glyph:** {DESIGN.md token name — e.g., `glyph-truncate-ellipsis`; ASCII fallback}
- **Header treatment:** {style — bold, color, separator-row; DESIGN.md token name}
- **Empty-table treatment:** {what is rendered when zero rows — empty-state line, count summary, suppressed entirely}

### Block composition

*(Include this subsection only if the human-mode output is multi-section. Otherwise: "Output is single-section.")*

- **Section headings:** {DESIGN.md token name; ordering rule}
- **Separators between sections:** {DESIGN.md token name — e.g., `glyph-divider-horizontal`}
- **Block ordering rule:** {fixed order, dependency-driven order, or rule-based}

### Color usage

| Element | DESIGN.md token | Rendered when |
|---------|-----------------|---------------|
| {element role} | `{token-name}` | {capability tier — see section 10} |

### Glyph usage

| Element | DESIGN.md token | ASCII fallback |
|---------|-----------------|----------------|
| {element role — bullet, status icon, divider, indicator} | `{token-name}` | {ASCII char/sequence — full rules in section 10} |

### Density

- **Compact vs spaced:** {rule — compact by default; `--verbose` adds blank-line separators between records; or fixed}
- **One-line vs multi-line per record:** {rule — one record one line, or one record many lines depending on field count}

---

## 7. Interactive Behavior

For each interactive surface CLI_IA's Shared Surfaces section names that this command can invoke, a subsection citing the surface by name.

*(If the command never prompts and has no progress or pager: "Non-interactive only; no prompts, progress, or pager." Skip the rest of section 7.)*

### {Shared surface name — e.g., "confirmation prompt"}

- **When shown:** {trigger — e.g., "before any push to a protected branch"}
- **Prompt structure:** {role-level structure of the prompt text, not literal copy — e.g., "summary of the action; options; default option marker"}
- **Default behavior on Enter without input:** {default option resolves to which action}
- **Accepted answers:** {answer set — e.g., "y / yes / n / no, case-insensitive"}
- **Rejection / re-prompt:** {what happens when the answer does not match — re-prompt count, fallback to abort with which failure row}
- **Three-mode rule:**

  | Mode | Behavior |
  |------|----------|
  | Interactive (TTY, no `--yes`) | {behavior} |
  | Non-interactive with blanket confirmation (`--yes` or `CI=true`) | {behavior — typically "proceeds without prompting"} |
  | Non-interactive without blanket confirmation | {behavior — typically "exits with the non-interactive-mode-unavailable failure row"} |

### {Shared surface name — e.g., "progress bar"}

- **When shown:** {trigger — e.g., "during the upload phase, when the operation duration exceeds {threshold}"}
- **Stages:** {list of phases the bar tracks, with completion rules per phase}
- **Refresh rate:** {policy — e.g., "every 100 ms or on phase transition, whichever sooner"}
- **Final summary on completion:** {summary-line shape}
- **Capability fallback:** {what is rendered when stdout is not a TTY or terminal is narrow — full rules in section 10}

### {Shared surface name — e.g., "pager"}

- **When invoked:** {trigger — e.g., "human-mode output exceeds terminal height and stdout is a TTY and `--no-pager` is not set"}
- **Disable mechanism:** `--no-pager` flag or `TOOL_NO_PAGER=1`
- **Pager program:** {selection rule — e.g., "`PAGER` env var if set, else `less -R`, else `more`"}
- **Capability fallback:** {when no TTY or no pager available — write directly to stdout}

*(Repeat per surface this command invokes.)*

---

## 8. Signal Handling

| Signal | Disposition | Cleanup performed | Exit code emitted |
|--------|-------------|-------------------|-------------------|
| `SIGINT` (Ctrl-C) | {graceful / immediate / ignored} | {what is cleaned — lockfiles, temp dirs, partial output} | `{EX-ID (N)}` |
| `SIGTERM` | {graceful / immediate} | {cleanup} | `{EX-ID (N)}` |
| `SIGHUP` | {re-read config / continue / abort} | {cleanup} | `{EX-ID (N)}` |
| `SIGPIPE` | {detect-and-exit-cleanly / ignored / default} | {cleanup} | `{EX-ID (N)}` or `0` |

For each signal that triggers non-default behavior, expand:

### `SIGINT` (Ctrl-C)

- **Detection signal:** {how the handler is registered}
- **State preserved:** {what is written to disk before exit so a subsequent `--resume` invocation can pick up; or "No state preserved — interruption discards in-flight work"}
- **Cleanup steps in order:** {ordered list — close handles, release locks, remove temp directory, flush logs}
- **Exit code:** `{EX-ID}` (`{N}`) per CLI_IA's taxonomy

*(Repeat per signal with non-default behavior. Signals at default disposition state "Default disposition; no command-specific cleanup." in their row and need no expansion.)*

---

## 9. Failure Matrix Detail

For each failure row CLI_IA's failure-matrix summary lists for this command (in the same order, by the same name), expand into a detailed entry. Rows CLI_IA marks "N/A — {reason}" are skipped here with a one-line preserved reason.

### {Failure name — e.g., "Authentication failure"}

- **Detection signal:** {exact condition — e.g., "server returned 401 from `EP-name`"; "missing or empty `TOOL_AUTH_TOKEN`"; "`auth login` has never been run for the active profile"}
- **Exit code:** `{EX-ID}` (`{N}`) per CLI_IA Exit-Code Taxonomy
- **Stderr message structure:**
  - **One-line summary:** {role — e.g., "auth-failure summary citing the active profile"}
  - **Optional detail block:** {role — e.g., "server response body when `--verbose`"}
  - **Optional remediation hint:** {role — e.g., "instruction to run `tool auth login`"}
- **Recovery hint:** {what the user should try — concrete action]}
- **Cleanup performed before exit:** {ordered list — lockfile release, temp file removal, partial output rolled back; or "Nothing to clean."}

*(Repeat for every non-N/A row in CLI_IA's failure matrix.)*

---

## 10. Output-Mode Switching & Capability Degradation

Per terminal capability tier CLI_IA's Terminal Capability section enumerates. Each tier has a subsection. State the no-op explicitly when a tier inherits the higher tier's rendering with no changes.

### 24-bit color (truecolor)

- **Palette:** Full DESIGN.md palette — every cited token rendered at its native value.
- **Glyph set:** Full Unicode set per cited tokens.

### 256 color

- **Palette mapping:** {how 24-bit tokens degrade to 256-color — e.g., "nearest-256 quantization"; or per-token override map}
- **Glyph set:** Same as 24-bit.

### 16 color

- **Palette mapping:** {how palette tokens degrade to ANSI 16 — e.g., "warning → yellow, danger → red, success → green, primary → cyan"}
- **Glyph set:** Same as 24-bit.

### No color

- **Trigger:** `NO_COLOR` env var set, `--no-color` flag, or stdout not a TTY.
- **Emphasis substitutes:** {how color emphasis is replaced — e.g., "bold for headings; underline for links; bracket-glyph prefix for status; never color-only-cue"}
- **Glyph set:** Same as the prevailing Unicode/ASCII tier.

### Unicode

- **Glyph set:** Full Unicode set per cited tokens.

### ASCII-only

- **Trigger:** `LANG=C` or `LC_ALL=C` or terminal reports no UTF-8 support.
- **Glyph fallback map:** {per-token Unicode → ASCII substitution — e.g., `glyph-bullet-primary` (`•`) → `*`; `glyph-divider-horizontal` (`─`) → `-`; `glyph-truncate-ellipsis` (`…`) → `...`}

### Narrow terminal (< 80 columns)

- **Trigger:** `COLUMNS` env var or terminal-size probe returns less than 80.
- **Column collapse rule:** {which columns drop first; ordering when collapsing further}
- **Wrap policy:** {wrap to next line, truncate with ellipsis, or stack rows vertically}

*(Repeat per tier CLI_IA enumerates. State "Same as {higher tier}; no degradation." for tiers that inherit unchanged.)*

---

## 11. Idempotency & Re-Run Behavior

- **Idempotent on repeated invocation?** {yes / no / yes-with-flags — e.g., "yes when `<path>` is unchanged and no new commits exist; no otherwise"}
- **Lock or guard:** {file lock at `{path}`, optimistic lock via server-side concurrency token, or "None — command relies on caller serialization"}
- **Lock acquisition / release order:** {acquire-then-act-then-release; release on every exit path including signals — cross-references section 8}
- **Resume from interruption:** {resume mechanism — `--resume` flag re-reads checkpoint at `{path}`; or "No resume — re-run is full restart"}
- **Dry run:** {`--dry-run` flag behavior — what side effects are skipped; what output is produced; whether stdout shape differs from non-dry mode; or "No dry-run mode — command always commits"}

*(If idempotency, locking, resume, and dry-run are all genuinely irrelevant — e.g., a pure read-only command — state: "Read-only command; idempotent by nature; no lock, no resume, no dry-run.")*

---

## 12. Open Questions

This section lists genuine ambiguities only. When the inputs answer the question, resolve it in the appropriate section above and do not list it here. List only when multiple valid approaches exist and CLI_IA, DESIGN, sibling COMMAND_SPECs, and a prior version (if any) do not decide.

Format:

- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {leaning, with reasoning — but the user decides}

When the user resolves all questions, move each resolution into the appropriate section above and replace this section with:

"All questions resolved."
```

---

## Scope

### In scope

- One command per invocation, identified by stable CLI_IA command ID (canonical command path)
- Argument-parsing rules beyond CLI_IA's grammar — order sensitivity, conflict resolution, default-from-context, auto-detection, argument expansion
- Stdin / stdout / stderr composition — what each stream carries, in what format, with the scriptability contract honored
- Every output mode the command supports — triggering, top-level shape, field naming, pretty-print, truncation
- Output layout in human mode — table composition, block composition, color and glyph usage, density
- Interactive behavior across the three-mode rule — prompts, confirmations, progress, pager
- Signal handling — `SIGINT`, `SIGTERM`, `SIGHUP`, `SIGPIPE`, with per-signal cleanup and exit code
- Failure-matrix detail — per failure: detection, exit code (cited from CLI_IA), stderr format, recovery hint, cleanup
- Capability degradation across color and Unicode tiers, narrow-terminal reflow
- Idempotency, lock/guard semantics, resume from interruption, dry-run behavior
- Citations of CLI_IA's command grammar, exit-code taxonomy IDs, capability-tier names, and DESIGN.md tokens by name only

### Out of scope

- Command identity (canonical name, grammar summary, primary outcome, top-level positionals and flags listed in CLI_IA, exit-code taxonomy, failure-matrix summary, use-case traceability) — owned by `CLI_IA.md`
- Global flags and environment variables — owned by `CLI_IA.md` `§ Global Flags & Environment`
- Design system — color palette and tiers, glyph set and ASCII fallbacks, table style, prompt style, progress style — owned by `DESIGN.md`
- Brand decisions — color choices, glyph choices, voice — owned by `DESIGN.md`
- Final user-facing copy — prompt strings, message wording, help text — owned by a UX-writing phase or by CLI_IA's per-command surface text
- Implementation — argument-parser library choice (`clap`, `cobra`, `argparse`, etc.), error types, logging frameworks — owned by per-unit `SPEC.md`
- Wire formats for backend communication — owned by `INTERFACES.md` (or the contract registry of the project)
- Multi-command flows — owned by `CLI_IA.md` `§ Flows`; COMMAND_SPEC may cite a sibling command's hand-off but does not redefine the flow
- Commands that do not meet the Selection Criteria — those stay with inline blueprints in `CLI_IA.md`
- Multiple commands — exactly one command per COMMAND_SPEC invocation
- Per-view TUI layout when the command launches a terminal UI — owned by a separate `VIEW_SPEC.md` for the launched view; COMMAND_SPEC documents only the launch and re-entry behavior

---

## Quality Checklist

Before considering a COMMAND_SPEC.md complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `command_id`, `flags_specified`, `output_modes_specified`, `interactive_surfaces_specified`, `signals_handled`, `failure_modes_specified`, `capability_tiers_specified`, `open_questions`)
- [ ] Frontmatter counts match the body exactly (`flags_specified` = rows in § 2 Flags table; `output_modes_specified` = subsections in § 5; `interactive_surfaces_specified` = surfaces in § 7 (zero when § 7 reads "Non-interactive only"); `signals_handled` = rows in § 8's table; `failure_modes_specified` = entries in § 9; `capability_tiers_specified` = subsections in § 10; `open_questions` = unresolved items in § 12)
- [ ] `status` is `complete` if section 12 reads "All questions resolved." and `has_open_questions` otherwise (use `blocked` only when a missing input prevents authoring and § 12 makes the gap explicit)
- [ ] All twelve sections are present; sections that do not apply state the no-op explicitly (e.g., "Non-interactive only", "Output is not tabular", "Same as {higher tier}; no degradation.") rather than being omitted
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.") anywhere in the document
- [ ] Open Questions section is present (empty with "All questions resolved." or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable when read alongside CLI_IA.md and DESIGN.md, without opening other COMMAND_SPECs or unit SPECs
- [ ] Section 1 cites the CLI_IA blueprint by section anchor; cites every DESIGN.md token used downstream by name; lists sibling commands for every flow handoff (or states "None")
- [ ] Section 2's Flags table lists only command-specific flags; global flags are not re-documented; every flag's mutually-exclusive and required-with relations are stated explicitly
- [ ] Section 3 documents order sensitivity, conflict resolution, default-from-context, auto-detection, and argument expansion — each with an explicit rule or "Not supported" / "No expansion"
- [ ] Section 4 explicitly affirms the scriptability contract (stdout = data, stderr = human, no interleaving) and states what each stream emits in this command
- [ ] Section 5 has one subsection per output mode the command supports per CLI_IA; each subsection states triggering, shape, field naming, pretty-print, and truncation
- [ ] Section 5's machine-mode subsections describe the top-level JSON / YAML shape with field names and types
- [ ] Section 6 cites DESIGN.md tokens by name for every color and glyph element in the human-mode tables; no hex codes, no ad-hoc color words
- [ ] Section 7 documents the three-mode rule (interactive / blanket-confirmation / non-interactive) for every interactive surface, or states "Non-interactive only; no prompts, progress, or pager."
- [ ] Section 7 cites every interactive surface by its name from CLI_IA's Shared Surfaces inventory; no surface is redefined inline
- [ ] Section 8 lists all four primary signals (`SIGINT`, `SIGTERM`, `SIGHUP`, `SIGPIPE`); signals at default disposition state "Default disposition; no command-specific cleanup." rather than being omitted
- [ ] Section 9 details every non-N/A failure row in CLI_IA's failure-matrix summary for this command, in the same order, by the same name; every entry cites an exit code by its CLI_IA Exit-Code Taxonomy ID and integer
- [ ] No section 9 entry invents a new exit code; missing-code needs surface as open questions in § 12
- [ ] Section 10 has one subsection per capability tier CLI_IA's Terminal Capability section enumerates; tiers that inherit unchanged state the no-op
- [ ] Section 10 provides a per-token Unicode → ASCII fallback map in the ASCII-only subsection
- [ ] Section 11 states idempotency, lock, resume, and dry-run rules — each explicitly, or affirms the read-only no-op when all four are irrelevant
- [ ] No code or pseudocode (or, if present, accompanied by a one-line note explaining why prose was insufficient)
- [ ] No implementation library names (no `clap`, `cobra`, `commander`, `argparse`, `click`, `yargs`, etc.)
- [ ] No final user-facing copy — message structures are described, not literal strings
- [ ] No restating of CLI_IA's grammar, global flags, exit-code taxonomy, primary outcome, failure-matrix row presence, use-case traceability, or capability-tier definitions
- [ ] Every flag is referenced by its long form (`--json`, never `-j` alone); every environment variable in `UPPER_SNAKE_CASE`; every exit code by its `EX-ID (N)` form
- [ ] Selection criteria justified — this command meets at least one of the criteria in the Selection Criteria section; otherwise the COMMAND_SPEC should not exist and the inline blueprint in CLI_IA suffices
