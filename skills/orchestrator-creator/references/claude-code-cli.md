# Running Claude Code from the Command Line

## New Session

```bash
unset CLAUDECODE && claude -p "<prompt>" --permission-mode bypassPermissions
```

Required:
- `-p <prompt>` — non-interactive single-shot mode.
- `--permission-mode bypassPermissions` — prevents permission dialogs that hang the subprocess.

Optional flags worth knowing:
- `--add-dir <path>` — extra directory the agent may read/write (repeatable).
- `--model <alias>` — `opus`, `sonnet`, `haiku`, or full model name.
- `--effort <level>` — `low | medium | high | xhigh | max`.
- `--max-budget-usd <n>` — hard spend cap (only with `-p`).
- `--no-session-persistence` — don't write a session file (only with `-p`); use when you don't want the run to be resumable.
- `--output-format json` — emit a single JSON object containing `result`, `session_id`, `total_cost_usd`, etc. Use this whenever you need the session id or want machine-parseable output.
- `--agent <name>` — pick a configured agent. `--agents '<json>'` defines an inline custom agent.

Prompts can invoke skills: `-p "/SKILL.md Read input from ..."`. Prefer this over inlined instructions when a matching skill exists.

## Continuing or Resuming a Session

### Storage layout (the key fact)

Sessions are stored at `~/.claude/projects/<encoded-cwd>/<session-uuid>.jsonl` where `<encoded-cwd>` is the absolute cwd with `/` replaced by `-`. **Both `-c` and `-r` look only inside this per-cwd folder.** Every other rule below follows from that.

### `-c` (continue most recent in current directory)

```bash
unset CLAUDECODE && claude -c -p "<follow-up>" --permission-mode bypassPermissions
```

Picks the most recently modified `.jsonl` in `~/.claude/projects/<encoded-cwd>/`. Caveats:
- From a directory with no prior session in that folder, it **silently creates a new session** — no warning, no error.
- A subdirectory is a different cwd. `-c` in `/foo/bar/sub` will not find a session created in `/foo/bar`.
- **Do not run `-c` from a cwd where another claude session is currently active.** The active parent's `.jsonl` keeps getting newer mtimes, so a child invoked with `-c` there will load the parent's full context as its starting point and append its question-and-answer pair into the parent's session file. This pollutes the parent's session log and wastes tokens cold-loading the parent's history.
- When several `.jsonl` files exist in the same cwd folder (e.g., from earlier non-interactive runs), `-c` picks the one with the latest mtime — i.e., the one that *finished* last. Track this if you launch sessions in parallel from the same cwd.

### `-r <uuid>` (resume by id)

```bash
unset CLAUDECODE && claude -r <uuid> -p "<follow-up>" --permission-mode bypassPermissions
```

Same per-cwd lookup as `-c`. Even with an explicit UUID, the call returns "No conversation found" if `<uuid>.jsonl` is not in the current cwd's encoded folder. To resume from a different directory, either run from the original cwd or copy/move the `.jsonl` into the new cwd's encoded folder first.

`-r` with no argument opens an interactive picker (only useful in a TTY, not from `-p`).

### `--session-id <uuid>`

Forces a specific UUID for a **new** session. Combine with `--output-format json` to emit and capture it. Useful when you want to resume the exact session later without parsing the picker output.

```bash
UUID=$(uuidgen | tr 'A-Z' 'a-z')
claude --session-id "$UUID" -p "..." --output-format json --permission-mode bypassPermissions
```

### `--fork-session`

Used together with `-c` or `-r`. Preserves history but mints a new session id, so the original session file isn't extended. Use when you want to branch off without disturbing the original.

### `--no-session-persistence`

Skips writing a `.jsonl` entirely. The session can't be resumed; nothing pollutes the cwd's session folder. Use for one-shot calls from inside an active claude session to avoid any interaction with the parent's cwd folder.

## Rules

- **Always `unset CLAUDECODE`** before invoking `claude`. Prevents session conflicts when called from within an existing Claude Code session.
- **Skill name and instructions on the same line.** When invoking a skill via `-p "/SKILL.md ..."`, the entire prompt must be one line with no newline after the skill name.
- **Prefer file-based output.** Tell the agent to write results to a specific file path rather than parsing stdout. Use `--output-format json` only when you need metadata like `session_id`.
- **Choose the right resume mechanism.** From the original cwd → `-c` (latest) or `-r <uuid>` (specific). From a different cwd → must capture `session_id` upfront via `--output-format json` and use `-r` from that original cwd, or move the `.jsonl` first.
- **Never run `-c` retries from a cwd that hosts another active claude session.** Either `cd` to a dedicated empty directory before retrying, or use `--session-id` upfront so you can `-r` it deliberately, or replace the `-c` retry with a fresh invocation that re-passes the failure context.
- **Use `claude --help`** to discover additional flags.
