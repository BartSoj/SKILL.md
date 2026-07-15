# Invoking Skills with Dynamic Workflows

The orchestrator runs skills by authoring **dynamic workflows** and executing them with the `Workflow` tool. A workflow is a small JavaScript script that spawns one subagent per skill invocation via `agent()`, running them sequentially or in parallel. Each subagent is a fresh Claude Code agent with no memory of the orchestrator's context — it communicates through files only.

The orchestrator does **not** fold the whole pipeline into one script. The workflow runtime is deterministic — it cannot read a skill's output and form judgment about whether it's ready. **That judgment is the orchestrator's job.** So the orchestrator runs a **sequence of small workflows, one per inspection boundary**: author a workflow, wait for it to finish, `Read` the files it produced, decide (proceed / regenerate / loop back / escalate), and only then author the next. Batch steps into one workflow *only* when they are independent siblings inspected together (parallel) or a short chain with no judgment between them.

## The workflow runtime — what to internalize

- **Authoring.** Call the `Workflow` tool with a `script`. Every script begins with a mandatory `export const meta = {...}` literal (`name`, `description`, optional `phases`), then a body using `agent()`, `parallel()`, `pipeline()`, `phase()`, `log()`. Pass the script inline via the tool's `script` parameter — do not write it to a file first.
- **Execution is asynchronous.** The `Workflow` tool returns immediately with a run id; a `<task-notification>` arrives when the run finishes. The orchestrator does not poll — it is re-invoked on completion. Then it reads the output files and judges.
- **`agent(prompt, opts)` runs one skill.** It spawns one subagent, runs it to completion, and returns its final text. With `{schema}` it returns a validated object instead. It returns `null` if the agent dies after the runtime's retries — `.filter(Boolean)` when collecting results.
- **Files are the only data channel.** The skill writes its artifact to the path named in the prompt; the `agent()` return value is a debug/summary channel, not the data. After the run, `Read` the artifact and judge it — never route on the return string alone.
- **Concurrency is managed for you.** The runtime caps concurrent agents at `min(16, cores−2)` and queues the rest. Pass *every* item to `parallel()`/`pipeline()`; do not hand-limit concurrency.

## One skill invocation — `agent()`

The default. A single skill, run and inspected on its own:

```javascript
export const meta = {
  name: 'run-analyze',
  description: 'Run the ANALYZE skill over the target files',
}
await agent('/ANALYZE.md Read the target files under src/. Write the analysis to ANALYZE.md.')
```

After the completion notification, `Read ANALYZE.md` and form judgment.

**The prompt is the skill invocation. The rules:**

- **Name the skill and have the subagent run it.** Begin the prompt with `/SKILL_NAME.md ` and the instructions on the same line. Inside a delegated `agent()` prompt, treat that reference as an explicit instruction to the subagent to **invoke that skill through its `Skill` tool**; don't rely on the leading slash auto-expanding the way it does as the first turn of a top-level session. If the skills ship as part of a plugin and a bare name doesn't resolve, the subagent invokes the plugin-qualified skill id.
- **Enumerate the read set explicitly.** Never tell the agent to "figure out what it needs". State exactly: "Read DOMAIN.md and ARCHITECTURE.md. Write INTERFACES.md."
- **Don't instruct the skill on its procedure; provision it instead.** Skills define their own quality bars. The prompt routes — names the skill, the read set, the output path, and passes through user-imposed constraints or facts the skill cannot derive. Inlined orchestrator analysis either contradicts the skill subtly or duplicates its work.
- **Specify the output path precisely.** The agent writes to exactly one file.
- **Do not route on the return value.** Check whether the expected output file exists and read it; the return string is a debug channel, not the data channel.

## Parallel siblings — `parallel()`

For independent invocations inspected together. `parallel()` is a barrier — it awaits all thunks and returns their results; a thunk that throws resolves to `null`, so `.filter(Boolean)` before using them.

```javascript
export const meta = {
  name: 'fan-out',
  description: 'Run the same skill over several independent items in parallel',
  phases: [{ title: 'Fan out' }],
}
const items = [
  { skill: 'REPORT', out: 'reports/a.md', reads: 'a/' },
  { skill: 'REPORT', out: 'reports/b.md', reads: 'b/' },
]
const results = await parallel(items.map(it => () =>
  agent(`/${it.skill}.md Read ${it.reads}. Write the report to ${it.out}.`, { label: it.out })
))
return results.filter(Boolean)
```

After the run, read each output file and judge them together.

## Multi-stage across items — `pipeline()`

When several items each flow through the same ordered stages with no barrier between them, `pipeline()` runs each item through all stages independently — item A can be at stage 2 while item B is still at stage 1. Each stage callback receives `(prevResult, originalItem, index)`. A stage that throws drops that item to `null` and skips its remaining stages:

```javascript
const items = ['x', 'y']
await pipeline(items,
  i => agent(`/FIRST_SKILL.md ... ${i} ...`, { label: `first:${i}`, phase: 'S1' }),
  (_, i) => agent(`/SECOND_SKILL.md ... ${i} ...`, { label: `second:${i}`, phase: 'S2' }),
)
```

Use `pipeline()` only for a genuinely gate-free stretch you've deliberately chosen not to inspect between. Wherever a decision sits between two stages, drive the items as **one `parallel()` workflow per stage** — fan out stage 1, inspect all outputs, then fan out stage 2 — so no item passes a gate unseen.

## Codify mechanical stretches in one workflow

The judgment loop stays with the orchestrator, but where a stretch has a **mechanical pass/fail** (not judgment), encode it inside a single workflow — this is where workflows are strongest:

- **Loop until a check passes.** Run a checker, fix what failed, repeat until it passes or two rounds make no progress.

```javascript
let round = 0
while (round < 5) {
  const check = await agent('Run the test suite. Report failures as {failures: [...]}.', { schema: CHECK, phase: 'check' })
  if (!check.failures.length) break
  await agent(`Fix these failures: ${JSON.stringify(check.failures)}`, { phase: 'fix' })
  round++
}
```

- **Fan out then adversarially verify.** Produce findings with one agent per item, then have independent agents verify each finding before it's reported. Moving these into the script gives a more trustworthy result than a single pass.

Reserve these for stretches whose success criteria are checkable without the orchestrator's reasoning. Anything requiring the orchestrator to read a document and decide stays a separate workflow with inspection between.

## Prompt construction

The prompt is the API call — be explicit about the output contract:

- **Specify the exact output file path.** The agent writes; the orchestrator reads.
- **Reference files by path rather than embedding**, since the subagent has filesystem access. Embed inline only when the content is short (< ~50 lines), is feedback that doesn't exist as a file, or is a specific excerpt.
- **Use XML tags** (`<issues>`, `<reference>`) to fence inlined content on the same line as the skill invocation.
- **Pass a `{schema}`** when you want a structured status back (e.g. a verdict or a findings list) — but the artifact file remains the source of truth.

For a feedback loop, inline the failure context on the same line as the skill reference:

```javascript
await agent(`/IMPLEMENT.md Read the plan from PLAN.md. A review found these issues: <issues>${findings}</issues> Fix them and write IMPLEMENT.md.`)
```

## Regeneration and resume

- **Feedback by regeneration is just a fresh `agent()`.** To re-run a skill with corrected context, author a new workflow whose `agent()` prompt inlines the failure context and points at the same output path. Subagents are always fresh.
- **Resuming an interrupted run.** If a workflow is stopped or lost mid-run, relaunch it — agents that already finished return cached results and only the rest re-run. Resume holds within the same session; if the session ends, the next run starts the workflow fresh. To iterate on a script, the `Workflow` tool result gives its saved path — edit that file and relaunch with `{scriptPath}`.

## Failure handling

- An `agent()` that fails after the runtime's retries returns `null` (a `pipeline()` stage drops the item to `null`). `.filter(Boolean)` the results and treat a missing artifact file as the failure signal — then retry that one invocation in a fresh workflow.
- The runtime retries transient errors before giving up; a `null` means retries were exhausted. Re-author that single `agent()` with a sharper output-path instruction.
- If the artifact file is absent after a run, the skill did not complete — re-author that single `agent()`.
- If the run finished but wrote no skill output, the subagent likely treated the `/SKILL.md` reference as literal text instead of invoking the skill. Re-author the prompt to explicitly instruct the subagent to invoke the named skill via its `Skill` tool (plugin-qualified id if a bare name doesn't resolve).
