# Running Claude Code from the Command Line

## New Session

```bash
unset CLAUDECODE && claude -p "<prompt>" --permission-mode bypassPermissions
```

Required flags:
- `-p <prompt>` — non-interactive single-shot mode
- `--permission-mode bypassPermissions` — prevents interactive permission dialogs that hang the process

Optional flags:
- `--model <model>` — override the model (`opus`, `sonnet`, `haiku`)
- `--add-dir <path>` — give the agent access to a directory (repeatable)

Prompts can invoke skills: `-p "/SPEC.md Read the input from ..."`. Prefer this over writing instructions inline when a matching skill exists.

## Continue Previous Session

```bash
unset CLAUDECODE && claude -c -p "<follow-up prompt>" --permission-mode bypassPermissions
```

The `-c` flag continues the most recent conversation from the same working directory. Use this to iterate on previous output without losing context.

## Rules

- **Always `unset CLAUDECODE`** before invoking `claude`. Prevents session conflicts when called from within an existing Claude Code session.
- **Skill name and instructions on the same line.** When invoking a skill via `-p "/SKILL.md ..."`, the entire prompt must be a single line with no newline after the skill name. A newline immediately after the skill name prevents the skill from being triggered.
- **Prefer file-based output.** Tell the agent to write results to a specific file path rather than parsing stdout. Stdout may contain progress messages and formatting that breaks parsers.
- **Use `claude --help`** to discover additional flags.
