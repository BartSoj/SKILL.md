# Running Claude Code from Python

Guidelines for building Python scripts that orchestrate one or more Claude Code CLI sessions, run them in parallel, and collect structured file outputs.

## Core Principle: File-Based Contracts

Treat Claude Code as a CLI that takes a prompt and produces files — not as a conversational agent. Your Python script defines an output directory structure, tells each agent exactly which files to write, and validates those files after the run. Stdout is a debug channel, not the data channel.

```python
# The prompt tells the agent: "Write your results to {output_path}"
# The orchestrator checks:
if output_file.exists():
    data = json.loads(output_file.read_text())  # validate format
```

This gives you a stable contract: the script depends on file existence and schema, not on parsing natural language output.

## CLI Invocation

### Minimal invocation

```python
cmd = [
    "claude",
    "-p", prompt,                              # non-interactive, single-shot
    "--permission-mode", "bypassPermissions",  # no interactive permission prompts
]
result = subprocess.run(cmd, capture_output=True, text=True, timeout=600, env=claude_env())
```

### Key flags

| Flag | Purpose |
|---|---|
| `-p <prompt>` | Non-interactive prompt mode. Essential for automation — the agent runs once and exits. |
| `--permission-mode bypassPermissions` | Suppresses interactive permission dialogs that would hang the subprocess. Only use when you control the prompt. |
| `--add-dir <path>` | Gives the agent read/write access to a directory. Use multiple times for multiple directories. |
| `--model <model>` | Override the model per agent (e.g., `haiku` for cheap tasks, `opus` for complex ones). |
| `--output-format json` | Forces Claude's stdout to be JSON. Useful if you want to parse the conversational output as a secondary channel. |

### Environment isolation

When spawning `claude` from within a Claude Code session or any nested context, the `CLAUDECODE` environment variable causes session conflicts. Always strip it:

```python
def claude_env() -> dict[str, str]:
    """Return a clean environment for subprocess claude calls."""
    env = os.environ.copy()
    env.pop("CLAUDECODE", None)
    return env
```

This is easy to miss and causes cryptic failures. Apply it to every `subprocess.run` call.

### Preflight check

Verify the CLI is available before launching any work:

```python
if not shutil.which("claude"):
    raise SystemExit("Error: 'claude' CLI not found on PATH.")
```

## Parallel Execution

Use `ThreadPoolExecutor` with `as_completed` for progress reporting:

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

with ThreadPoolExecutor(max_workers=len(tasks)) as pool:
    futures = {}
    for task in tasks:
        future = pool.submit(run_one_agent, task["id"], task["prompt"], ...)
        futures[future] = task["id"]

    for future in as_completed(futures):
        task_id = futures[future]
        status = future.result()
        print(f"  {task_id}: {status['status']} ({status.get('elapsed', 0):.0f}s)")
```

Design choices:
- **`as_completed`** over `map` — reports progress as agents finish, not in submission order.
- **Each worker is self-contained** — receives all inputs as arguments, writes to its own output path, returns a status dict. No shared mutable state.
- **`max_workers`** — set to the number of tasks by default. Reduce if you hit API rate limits (e.g., `max_workers=6`).

## Output Directory as Single Source of Truth

The output directory is the entire interface between pipeline stages. Structure it so every piece of state is inspectable:

```
output/
  input.txt               # saved input (for reproducibility)
  dispatch.json           # stage 1: what was dispatched and why
  agents/
    agent-1.json          # stage 2: one file per parallel agent
    agent-2.json
    agent-1.error.txt     # errors as sibling files, not mixed into data
    agent-1.debug.txt     # raw stdout for debugging failed agents
  results.json            # stage 3: aggregated final output
  results.md              # stage 3: human-readable version
```

Rules:
- **Save inputs alongside outputs** for reproducibility. Anyone can re-run the pipeline from the saved state.
- **One file per parallel agent** in a subdirectory — easy to inspect, retry individually, or skip.
- **Error and debug files as siblings** — same name, different extension (`.error.txt`, `.debug.txt`).
- **Require the output directory to be empty** at start. Prevents stale data from previous runs contaminating results.

```python
if output_dir.exists() and any(output_dir.iterdir()):
    raise SystemExit(f"Error: output directory is not empty: {output_dir}")
```

## Worker Function Pattern

Every parallel worker should follow this structure:

```python
def run_one_agent(task_id: str, prompt: str, output_file: Path,
                  work_dirs: list[str], timeout: int) -> dict:
    """Run a single Claude Code instance. Returns a status dict."""
    cmd = ["claude", "--permission-mode", "bypassPermissions", "-p", prompt]
    for d in work_dirs:
        cmd.extend(["--add-dir", d])

    t0 = time.time()
    try:
        result = subprocess.run(
            cmd, capture_output=True, text=True,
            timeout=timeout, env=claude_env(),
        )
        elapsed = time.time() - t0

        if result.returncode != 0:
            output_file.with_suffix(".error.txt").write_text(
                f"Exit code: {result.returncode}\n\n{result.stderr[:2000]}")
            return {"id": task_id, "status": "failed", "elapsed": elapsed,
                    "error": result.stderr[:500]}

        if not output_file.exists():
            output_file.with_suffix(".debug.txt").write_text(
                result.stdout[:5000] if result.stdout else "(empty)")
            return {"id": task_id, "status": "no_output", "elapsed": elapsed}

        # Validate output format
        data = json.loads(output_file.read_text())
        return {"id": task_id, "status": "done", "elapsed": elapsed,
                "result_summary": summarize(data)}

    except subprocess.TimeoutExpired:
        return {"id": task_id, "status": "timeout", "elapsed": timeout}
    except json.JSONDecodeError as e:
        output_file.with_suffix(".debug.txt").write_text(
            output_file.read_text()[:5000])
        output_file.unlink()
        return {"id": task_id, "status": "invalid_output", "elapsed": elapsed,
                "error": str(e)}
```

Key properties:
- **Always returns a status dict** — never raises. The orchestrator can aggregate statuses without try/except around every future.
- **Always measures elapsed time** — essential for understanding cost and identifying slow agents.
- **Saves debug artifacts on failure** — stderr for crashes, stdout for missing output, raw content for invalid JSON.
- **Validates output** — at minimum `json.loads()`, ideally schema checks (e.g., assert required keys exist).

## Atomic Locking

If you support retries or re-runs against the same output directory, prevent two workers from racing on the same task:

```python
lock_file = output_file.with_suffix(".lock")
try:
    fd = os.open(str(lock_file), os.O_CREAT | os.O_EXCL | os.O_WRONLY)
    os.write(fd, str(os.getpid()).encode())
    os.close(fd)
except FileExistsError:
    return {"id": task_id, "status": "locked"}
try:
    # ... run agent ...
finally:
    lock_file.unlink(missing_ok=True)
```

`O_EXCL` makes the create-or-fail atomic at the filesystem level.

## Prompt Construction

Build prompts programmatically. The prompt is your API call — be explicit about the output contract:

```python
def build_prompt(task_context: str, reference_content: str,
                 output_path: str) -> str:
    return f"""You are a specialist in {task_context}.

<reference-material>
{reference_content}
</reference-material>

<input-data>
{input_data}
</input-data>

Instructions:
1. Analyze the input according to the reference material.
2. Write your results as JSON to {output_path}.
3. Follow this exact schema:
{{
  "task_id": "...",
  "findings": [...],
  "summary": "..."
}}

Write ONLY this file. Do NOT write any other files. Do NOT ask questions."""
```

Prompt rules:
- **Specify the exact output file path.** The agent writes; the orchestrator reads.
- **Include the output schema in the prompt.** Show the exact JSON structure, field names, and types. Use example values where helpful.
- **End with constraints.** "Write ONLY this file. Do NOT ask questions." prevents the agent from going off-script.
- **Use XML tags** (`<reference-material>`, `<input-data>`) to structure large prompt sections. Claude parses these well.
- **Embed reference content directly** rather than relying on the agent to find files. The prompt is the complete input.

## Pipeline Stages

Most multi-agent scripts follow a three-stage pattern:

### Stage 1: Dispatch (pure Python)

Analyze the input, decide what agents to launch, compute context hints for each. No Claude calls — this is deterministic logic. Write a `dispatch.json` so you can inspect decisions without re-running.

### Stage 2: Parallel agents (Claude Code)

Launch N agents in parallel, each with its own prompt and output file. Each agent is independent — no shared state, no inter-agent communication. Collect status dicts as agents complete.

### Stage 3: Assembly (Claude Code or Python)

Aggregate individual agent outputs into a final result. This can be another Claude call (for intelligent deduplication and summarization) or pure Python (for simple concatenation). If you use a Claude agent for assembly, treat it like any other agent: give it a prompt, expect a file.

## Failure Handling

Let failures surface. Do not silently degrade:

- **If an agent fails, report it.** The orchestrator prints which agents failed and why. The user decides whether to re-run, adjust the prompt, or skip.
- **Save all debug artifacts.** Error files, raw stdout, partial outputs — everything needed to diagnose without re-running.
- **Exit with non-zero status** if critical agents fail, so CI pipelines and wrapper scripts can detect failure.

```python
failed = [r for r in results if r["status"] != "done"]
if failed:
    print(f"\n{len(failed)} agent(s) failed:")
    for r in failed:
        print(f"  {r['id']}: {r['status']} — {r.get('error', 'see debug files')}")
    sys.exit(1)
```

## Model Selection

Different agents in the same pipeline can use different models. Pass `--model` per invocation:

```python
cmd = ["claude", "-p", prompt, "--permission-mode", "bypassPermissions"]
if model:
    cmd.extend(["--model", model])
```

Use cheaper/faster models (haiku) for simple or low-stakes agents, and more capable models (opus) for complex reasoning. The orchestrator can decide model assignment based on task difficulty or relevance scoring.

## Timeouts

Always set timeouts. Claude can get stuck or produce unexpectedly long output:

```python
subprocess.run(cmd, ..., timeout=600)  # 10 minutes per agent
```

Choose timeouts based on expected task complexity. Simple extraction tasks: 60–120s. Deep analysis over a large codebase: 600–3600s. The timeout should be generous enough to avoid false positives but strict enough to prevent runaway costs.

## Checklist

Before running your pipeline in production:

- [ ] `CLAUDECODE` env var is stripped from subprocess environment
- [ ] Every agent writes to a unique output file path (no collisions)
- [ ] Output directory is verified empty before starting
- [ ] Every `subprocess.run` has a `timeout`
- [ ] Failed agents produce debug artifacts (`.error.txt`, `.debug.txt`)
- [ ] Pipeline exits non-zero when agents fail
- [ ] Prompts specify exact output file path and schema
- [ ] Prompts end with "Write ONLY this file. Do NOT ask questions."
- [ ] Inputs are saved to the output directory for reproducibility
