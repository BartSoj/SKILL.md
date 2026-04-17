---
name: TUI_IA.md
description: Generate a terminal UI information architecture registry — layout model, mode model, keybinding matrix, focus model, per-view blueprints, flows, shared surfaces, terminal capability strategy, and use-case traceability for rich full-screen terminal applications. Use when asked to create a TUI IA, design the TUI view map, produce a keybinding matrix, define focus and mode rules for a terminal UI, or produce a TUI_IA.md.
---

# Task: Generate TUI_IA.md — Terminal UI Information Architecture Registry

## Objective

Produce a TUI_IA.md that serves as the single source of truth for a rich terminal UI application's information architecture: the layout model (panels, splits, fixed vs flexible regions), the mode model (normal / edit / command / visual / search, vim-style or bespoke — "modeless" is a valid answer), the keybinding matrix, the focus model, the full view-template inventory, per-view blueprint (purpose, layout, modes available, keybindings, default focus, primary action, data dependencies, full state matrix, entry/exit points), multi-view flows, shared surfaces (command palette, dialogs, help overlay, status bar, notifications, search overlay, external-editor invocation), terminal capability strategy (color depth, Unicode vs ASCII fallback, mouse, minimum terminal size, reserved keys, clean exit), and bidirectional traceability from product use cases to views. An agent or developer reading this document knows exactly what views exist, how panels compose into each view, which keys do what in which mode, which panel is focused, how the app degrades on small or monochrome terminals, and how every use case is realized on the TUI surface — without making any independent layout, keybinding, or mode decisions. TUI_IA sits between product requirements (proposal, use cases) and per-unit TUI SPECs, the same way WEB_IA sits between product requirements and per-unit web SPECs.

---

## Inputs

1. **Product / proposal document** (required) — product purpose, target users, and rationale. Sets the scope and audience frame for the IA.
2. **Use cases document** (required) — enumerates user interactions and jobs-to-be-done. Primary input. Every use case with a TUI surface must map to one or more views in the inventory.
3. **Architecture document** (optional but recommended) — TUI framework choice (bubbletea, textual, ratatui, blessed, tview, ncurses, prompt-toolkit, etc.), language, state management, event loop model, auth model. Informs layout conventions, keybinding style, and shared-surface choices. If absent, infer conservative defaults and flag non-obvious choices as open questions.
4. **Existing TUI codebase** (auto-discovered) — discover current views, panels, keybindings, and layout from source (e.g., bubbletea `tea.Model` implementations, textual `Screen` classes, ratatui `Widget` trees, blessed `screen.append` calls). Treat as reference only — TUI_IA is the target state, not a mirror of the current state.
5. **Decision documents** (optional) — architectural decisions that constrain the IA (e.g., "the app is modeless — no vim modes", "status bar always shows branch and sync state", "`q` quits from any view").
6. **Sibling IAs** (optional) — if `CLI_IA.md`, `WEB_IA.md`, `MOBILE_IA.md`, or `VOICE_IA.md` exist, use-case traceability must be consistent across them. A product that ships both a simple CLI (one-shot commands) and a TUI (interactive full-screen) gets both `CLI_IA.md` and `TUI_IA.md`; they describe disjoint surfaces and must not claim the same use case as primary on both without explicit justification.
7. **Contract registry** (optional) — if present, per-view data dependencies may reference endpoints by name; wire formats are owned by CONTRACT_REGISTRY.md, not redefined here.

The user may override scope (e.g., "only the main TUI, not the admin-mode TUI"). Support that by recording the narrowed scope in the Identity & Context section and excluding out-of-scope views from the inventory.

Scope note: this skill covers **rich terminal UI applications** — full-screen interactive terminal programs with multi-view navigation, panel-based layouts, and keybinding-driven interaction (lazygit, k9s, htop/btop, zellij, ranger, bubbletea apps, textual apps). Simple CLI tools that read args and print output belong in `CLI_IA.md`. Interactive REPLs with prompt-based interaction (no alt-screen, no multi-panel layout) are borderline — if the REPL has a persistent UI, multiple views, and keybindings beyond readline, document it here; otherwise, document it in `CLI_IA.md`.

---

## Workflow

TUI IA generation proceeds in seven phases: scope, centralized models (layout / mode / focus / globals), inventory, per-view blueprint, flows and shared surfaces, terminal capability and global states, traceability and validation. Phases are sequential — later phases depend on earlier decisions — but revisit earlier phases if a later phase reveals a gap.

### Phase 1: Scope & Audience

Derive from the product doc and use cases document the target TUI surface (single-view full-screen app, multi-view navigator, dashboard, editor-style app) and the audiences the IA serves (end user, operator, contributor). Record explicitly what is *not* in scope for this TUI_IA document — GUI wrappers, web surfaces, simple CLI binaries, embedded terminal widgets in other IDEs. Keep audience treatment thin — named user types and their 3–7 top tasks per audience. No deep persona work.

### Phase 2: Layout, Mode, Focus, and Global Keybindings

Decide the centralized models before enumerating views. This is the analogue of Phase 2 in `CLI_IA.md` (command grammar and globals) and Phase 2 in `WEB_IA.md` (URL strategy and global navigation): per-view drift is prevented by stating the cross-cutting rules once and enforcing them across every view.

Decide, in order:

**Layout Model**
- **Root layout** — single-view / split-screen / tabbed / overlay-based / wizard-linear
- **Panel arrangement conventions** — horizontal splits, vertical splits, grids, free-floating overlays
- **Flexible vs fixed regions** — which regions have fixed width/height in characters, which flex to remaining space
- **Minimum terminal size** — rows × columns below which the app refuses to start or displays a "terminal too small" view
- **Resize behavior** — reflow rules; what collapses first when space is scarce (hide sidebar, collapse status bar, hide secondary panels)
- **Border / frame conventions** — the *fact* that borders are drawn and that ASCII fallback exists, not the specific glyphs

**Mode Model**
- **Modes in use** — named list (e.g., Normal, Command, Insert, Visual, Search). If the TUI is modeless, state "modeless — Normal mode is implicit and universal" and move on.
- **Mode indicator** — how the current mode is communicated to the user (status bar label, cursor shape)
- **Mode transitions** — which keys/actions enter and exit each mode
- **Mode-specific keybinding scopes** — which mode owns which keybindings

**Focus Model**
- **Focus units** — panels, widgets, or both
- **Default focus per view** — stated in each view's blueprint; the *rule* for where default focus starts is stated here
- **Focus movement keys** — Tab / Shift-Tab, arrow keys, explicit keys (`H/J/K/L`, `Ctrl-W` + direction)
- **Focus indication** — how focus is communicated (border emphasis, label highlight, cursor position) — the *fact* of indication, not the specific visuals
- **Focus preservation** — what happens on re-entry to a view: restore last focus or reset to default

**Global Keybindings**
- Keys available on every view and (unless mode says otherwise) every mode — typical set: `?` help, `q` quit, `Ctrl-C` force-quit, `:` command mode, `/` search, `Esc` return-to-normal, `Ctrl-L` redraw
- **Modifier conventions** — when to use `Ctrl-X` vs chord sequences (`g g`) vs plain letters
- **Chord sequences** — multi-key sequences and chord timeout rules
- **Conflict rules** — when a view's local keybinding shadows a global, the global must still have an escape hatch (e.g., `Esc` always returns to normal mode; `Ctrl-C` always force-quits)
- **Mouse support** — whether mouse is bound, what actions (click to focus, scroll, drag to resize split, right-click menu)

Use architecture-doc hints where available; fall back to conservative defaults. Surface non-obvious choices as open questions rather than guessing.

### Phase 3: View Inventory

From use cases, architecture, and existing codebase discovery, enumerate the full set of **view templates** (not view instances — see Rule 1). Classify each view by category: main / detail / overlay / help / error / splash. Build a single inventory table with one row per view template: name, category, layout reference, referenced use cases, one-line purpose.

For a new product, derive views from use cases: every use case with a TUI surface implies at least one view where the user accomplishes or progresses that use case. For an existing codebase, discover current views and reconcile against use cases — adding missing views, flagging orphan views.

### Phase 4: Per-View Blueprint

Iterate every view template in the inventory and produce its full entry. This is the bulk of the output. Each entry must include every mandatory field listed in `references/per-view-blueprint.md`.

Read `references/per-view-blueprint.md` before writing the first entry; then copy the block per view and fill in every field. Work through views in the order they appear in the inventory, grouped by category via subsection headings.

When writing keybindings, list only **view-specific** keys — global keybindings are inherited from the Global Keybinding Matrix and must not be re-listed (Rule 11). When writing layout, reference the Layout Model by name for the root-layout template and describe only this view's specific panel arrangement (Rule 9).

### Phase 5: Flows & Shared Surfaces

Identify multi-view journeys — `review → stage → commit → push` in a git TUI, `list clusters → cluster detail → pod list → log stream` in a k8s TUI, `browse library → playlist detail → play` in a music TUI. For each, write a flow entry: ordered steps pointing to views in the inventory, entry conditions, success exit, abort behavior (usually bound to `Esc` or `q`), and the use case(s) the flow implements. Flows do not introduce new views (Rule 13); they sequence existing ones.

Identify shared surfaces that cross views: command palette, confirmation dialogs, modal input dialogs, help overlay, status-bar content conventions, notifications/toasts, search overlay, error banners, version-mismatch banners, broken-connection banners. Consolidate these in the dedicated Shared Surfaces section (Rule 14). Per-view entries may list which shared surfaces they can invoke, but must not re-describe them.

### Phase 6: Terminal Capability, Global States, and External Editor

Document terminal capability strategy at the IA level — this is the TUI analogue of CLI_IA's terminal capability section, but more demanding because a TUI drives the full alt-screen:

- **Color depth** — truecolor / 256-color / 16-color / monochrome; graceful degradation rules (what falls back to what)
- **Unicode vs ASCII fallback** — substitutions for line-drawing glyphs, spinners, icons, progress characters
- **Mouse** — optional or required; what actions degrade when mouse is absent
- **Minimum terminal size** — rows × columns; what the app shows below the minimum
- **Reserved keys** — keys the terminal emulator typically captures (`Ctrl-S`, `Ctrl-Q`, `Ctrl-Z`) and the alternative the app provides
- **Clean exit** — restoring alt-screen, cursor visibility, mouse mode on normal exit and on SIGINT / SIGTERM / terminal disconnect

Document global states independent of any single view: uncaught error / crash behavior, quit confirmation (if any), version-mismatch with backend, broken-connection banner, session-expired re-auth prompt, terminal-too-small screen.

Document external editor and shell integration **if the TUI uses them** — otherwise omit the section entirely (Rule 15):
- Which flows invoke `$EDITOR` (commit messages, free-form input, multi-line search queries)
- Behavior when `$EDITOR` is unset — fallback editor, inline mini-editor, or abort with a clear message role
- Shell-out conventions — suspend TUI on shell-out, restore alt-screen on return

### Phase 7: Traceability & Validation

Build the bidirectional use-case ↔ view matrix. Every use case in the input use cases document maps to at least one view, or is explicitly listed as having no TUI surface with a one-line reason. Every view in the inventory is referenced by at least one use case, or is explicitly classified (chrome — splash / help / error).

If sibling IAs are present (`CLI_IA.md`, `WEB_IA.md`), cross-reference use cases that surface on both — and call out any use case claimed as primary on both the CLI and TUI surfaces, since in a product with both surfaces each use case should have one primary channel unless explicitly justified.

Verify every rule holds: every view has a full blueprint, state matrix complete per view, primary action declared (or chrome classification), layout model / mode model / keybinding matrix / focus model each appear exactly once and per-view entries reference them, every view's keybindings extend rather than redefine globals, terminal capability strategy respected across all views, no implementation detail, no visual specifics, conditional sections either complete or absent. Update frontmatter counts to reflect the final document.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. View templates, not view instances

A parameterized view like `ResourceLogView(resourceId)` is one view template, not N views. Document each view once. Note the parameters and enumerate parameter-dependent behavior (e.g., "if the resource is not found → not-found state; if the viewer lacks permission → unauthorized state distinguishable from not-found to avoid information leak"). Do not enumerate concrete view instances.

### 2. Primary action is mandatory per non-chrome view

Every non-chrome view must declare **exactly one** primary action — the single dominant thing a user does there (e.g., "stage changes", "review a PR", "explore logs", "play the selected track"). Views with two or more legitimate primary actions must either be split or explicitly justify the pairing. Chrome views (splash, help overlay, fatal-error view) carry no primary action and must be explicitly classified.

### 3. State matrix is mandatory per view

Every per-view blueprint must enumerate, at minimum: default / populated; loading (initial load and background refresh); empty (no items to show); error (data fetch or operation failed); terminal too small (viewport below minimum — what the view shows); unauthenticated (if the TUI connects to an authed backend; often N/A for local-only TUIs); unauthorized (authenticated but lacks permission); not-found (for parameterized views). For each state describe **what appears in each panel, which keybindings remain active, and what the user does next**. Do not describe visual appearance — specific colors, exact glyphs, and spinner frame choices are out of scope.

### 4. Every view maps to at least one use case or is explicitly classified

A view in the inventory either appears in the traceability matrix under at least one use case, or carries an explicit classification: chrome — splash / help / error. Silent orphan views are not allowed.

### 5. Every use case maps to at least one view or is explicitly noted

Every use case in the input use cases document appears in the traceability matrix mapped to at least one view, or is listed as "no TUI surface" with a one-line reason (e.g., "one-shot CLI only", "background job with no UI"). Silent drops are not allowed.

### 6. No implementation detail

Do not write widget library types, model struct fields, event-loop internals, reducer signatures, or state-machine variant names. Those belong in per-unit SPECs. Describe views by **layout composition, keybinding-action contract, and state behavior**, not by code shape.

**Worked example — the IA-vs-SPEC line.** This is the single most important boundary in the document; internalize it.

> **In scope (TUI_IA):** "The Status view shows a three-panel layout: left panel lists branches (narrow, fixed width); center panel shows the staged/unstaged diff (flexible, occupies remaining width); right panel shows the commit log (fixed narrow). The status bar at the bottom shows current branch and sync state. The view supports Normal mode (navigate) and Command mode (`:` prefix). Pressing `j/k` moves within the focused panel; `Tab` rotates focus across panels; `Enter` opens the selected item in detail; `s` stages the highlighted hunk; `?` opens the help overlay. Respects terminal resize — panels reflow; below 80 columns the right panel is hidden."
>
> **Out of scope (TUI_IA, belongs in SPEC):** "`StatusViewModel` holds `selectedPanel: PanelId` and `branches: [Branch]`; the `s` key dispatches `Action.StageHunk(hunkId)` reduced by `statusReducer`…"

Prefer layout-and-keybinding descriptions over code-level descriptions throughout the output.

### 7. No visual decisions

No specific colors, specific box-drawing characters, specific spinner frames, specific icon glyphs, specific ANSI escape choices. Terminal-capability degradation rules (which palette is used when) are in scope; the *specific* visual artifacts are not — they are owned downstream of this document.

### 8. No final copy

Describe message roles ("empty-state prompt", "error banner", "confirmation label") rather than literal strings. Short disambiguating placeholders are acceptable — no more.

### 9. Layout model stated once

The Layout Model section is authoritative. Every per-view blueprint references the root-layout template by name and describes only its own view-specific panel arrangement. If a view deviates from the root layout (e.g., a modal overlay that replaces the normal pane structure), note the exception inline in the view entry and explain why.

### 10. Mode model stated once

The Mode Model section is authoritative. "Modeless — Normal is implicit and universal" is a valid answer. Per-view entries reference modes by name from the Mode Model and list which subset of modes applies; they do not redefine modes or invent new ones. A view that needs a new mode means the Mode Model is incomplete — extend it.

### 11. Keybinding matrix is authoritative

The Global Keybinding Matrix section is authoritative. Per-view blueprint entries reference global keybindings by name — "standard global keybindings apply" — and list only view-specific additions. A view may not redefine a global keybinding; if a view-local meaning is truly required, document the shadowing explicitly in the view entry with the escape hatch back to the global behavior (e.g., "`q` is rebound to 'close detail panel' here; `Ctrl-C` still force-quits").

### 12. Focus model stated once

The Focus Model section is authoritative. Per-view entries state the default focus on entry and any view-specific focus-movement keys that extend the global focus rules. The *convention* for how focus moves between panels is stated once in the Focus Model.

### 13. Flows do not introduce new views

Every flow step references an existing view template in the inventory. If a flow needs a step that has no corresponding view, add that view to the inventory first (with a full blueprint), then reference it from the flow.

### 14. Shared surfaces are consolidated

Command palette, confirmation dialogs, modal input dialogs, help overlay, notifications, search overlay, and global banners live in the Shared Surfaces section — not scattered across per-view entries. Per-view entries may list which shared surfaces they invoke by name; they must not redefine them.

### 15. Conditional sections must be omitted, not left empty

If External Editor & Shell Integration, Mouse Support, Plugin / Extension Surface, Accessibility, Configuration / Theming, Internationalization, or Telemetry sections are not applicable to the project, omit them entirely. Do not leave them present with "N/A" content. Tight output is the house style.

### 16. Terminal capability strategy stated once, enforced across views

The Terminal Capability Strategy section is authoritative. Every view's terminal-too-small state and every view's fallback behavior for color / Unicode / mouse must conform to the rules in that section. A view whose degradation rules differ from the strategy means the strategy is incomplete — extend it.

### 17. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc." Use exact view names, panel names, keybindings, mode names, and use-case references. If you cannot be exact, flag it as an open question rather than hiding behind a placeholder word.

---

## Output Format

```markdown
---
skill: TUI_IA.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
views_documented: {N}
modes_documented: {N}             # 1 for modeless
global_keybindings: {N}
shared_surfaces: {N}
flows_documented: {N}
use_cases_covered: {N}
use_cases_total: {N}
min_terminal_rows: {N}
min_terminal_cols: {N}
open_questions: {N}
---

# TUI_IA

> Single source of truth for the terminal UI application's information architecture.
> Every view template, layout rule, mode, keybinding, focus rule, flow, and shared
> surface is authoritative — per-unit TUI SPECs must reference these entries.

## Identity & Context

{One paragraph: product name, the binary or mode this document covers, and
 the audience frame. Reference the product proposal by path. State explicitly
 what surfaces are out of scope — GUI wrappers, web surfaces, simple CLI binaries,
 admin-mode TUIs, etc.}

## Audience & Top Tasks

| Audience | Top tasks in the TUI |
|----------|----------------------|
| {audience name} | {3–7 comma-separated top tasks} |

(Repeat for each audience identified in the product doc.)

## Layout Model

- **Root layout:** {single-view | split-screen | tabbed | overlay-based | wizard-linear — with one-sentence rationale}
- **Panel arrangement conventions:** {horizontal / vertical / grid / free-floating — when each is used}
- **Flexible vs fixed regions:** {rule — e.g., "sidebars are fixed at N columns; main content flexes to remaining width; status bar is fixed at 1 row"}
- **Minimum terminal size:** {rows × columns — e.g., "24 rows × 80 columns"}
- **Resize behavior:** {reflow rules — what collapses first when space is scarce}
- **Border / frame conventions:** {fact that borders are drawn; ASCII fallback exists — no specific glyphs}
- **Exceptions:** {any views that deviate from the root layout, with reason — or "None."}

## Mode Model

- **Modes in use:** {named list — e.g., "Normal, Command, Insert, Visual, Search"; or "Modeless — Normal is implicit and universal"}
- **Mode indicator:** {how the current mode is communicated — e.g., "status-bar label"}
- **Mode transitions:**

| From → To | Trigger |
|-----------|---------|
| `{mode A}` → `{mode B}` | `{key or action}` |

(Repeat for each transition. If modeless: "Modeless — no transitions.")

- **Mode-specific keybinding scopes:** {rule — which mode owns which keys, or "All keybindings live in Normal mode."}

## Global Keybinding Matrix

Keys available on every view. Unless a mode explicitly restricts them, they work in every mode.

| Key(s) | Mode | Action role |
|--------|------|-------------|
| `?` | any | {open help overlay} |
| `q` | normal | {quit or close current view — one of the two per product convention} |
| `Ctrl-C` | any | {force-quit with unsaved-work warning or direct kill — per product convention} |
| `:` | normal | {enter command mode} |
| `/` | normal | {enter search mode — or "N/A"} |
| `Esc` | any non-normal | {return to normal mode} |

(Repeat for every global key. Extend or trim based on product decisions.)

### Modifier conventions

{When `Ctrl-X` vs chord sequences (`g g`) vs plain letters are used.}

### Chord sequences

{List of chords and chord-timeout rule — e.g., "`g g` go-to-top, `G` go-to-bottom; chord timeout 500 ms." Or "No chord sequences."}

### Conflict and escape rules

- {E.g., "`Esc` always returns to Normal mode, even when shadowed by a view-local binding."}
- {E.g., "`Ctrl-C` always force-quits, regardless of view."}

### Mouse support

- **Enabled:** {yes | no | optional — toggled via flag or config key}
- **Bindings:** {what click / scroll / drag does — or "N/A"}

## Focus Model

- **Focus units:** {panels | widgets | both}
- **Default-focus rule:** {where focus starts on first entry to any view — per-view defaults are declared in each blueprint}
- **Focus-movement keys:** {`Tab` / `Shift-Tab` / arrows / explicit keys like `H J K L`}
- **Focus indication:** {fact of indication — e.g., "focused panel is highlighted via border emphasis"; no specific color}
- **Focus preservation on re-entry:** {restore last focus | reset to default | per-view setting}

## View Inventory

| View | Category | Layout | Use cases | Purpose |
|------|----------|--------|-----------|---------|
| `{ViewName}` | main / detail / overlay / help / error / splash | {root-layout reference or "custom — see blueprint"} | {UC-IDs or classification} | {one line} |

(Group by category via subsection headings. Repeat for every view template.)

---

## Per-View Blueprints

Every view template in the inventory gets a full blueprint entry here. The
blueprint template — with every required field, its purpose, and common
mistakes to avoid — lives in
[`references/per-view-blueprint.md`](references/per-view-blueprint.md).
Read that file before writing the first entry; then copy the block per view
and fill in every field.

Entries in the output appear in the order they appear in the View Inventory,
grouped by the same category headings.

### `{ViewName}` — {short name}

{Full blueprint block per `references/per-view-blueprint.md`.}

---

(Repeat the blueprint block for every view template in the inventory.)

---

## Flows

### {Flow name}

- **Purpose:** {one sentence}
- **Use cases implemented:** {UC-IDs}
- **Entry condition:** {what triggers the flow — e.g., "user launches the TUI with `--review <pr>`"}
- **Steps:**
  1. `{ViewName}` — {what the user does here; what state is produced for the next step}
  2. `{ViewName}` — {what the user does here}
- **Success exit:** {terminal state — e.g., "commit created, pushed to remote, Status view reflects clean working tree"}
- **Abort behavior:** {key binding and result — e.g., "`Esc` returns to the previous view; partial selections are discarded"}
- **Resume on re-entry:** {what happens if the user leaves mid-flow and returns}

(Repeat for each flow. If none: "No multi-view flows in scope.")

---

## Shared Surfaces

| Surface | Kind | Trigger | Dismissal | Invoked by views |
|---------|------|---------|-----------|------------------|
| {name} | {command palette / confirmation dialog / modal input / help overlay / notification / search overlay / global banner} | {key or event that opens it} | {key that closes it} | {view names or "global"} |

(Repeat for every shared surface. If none: "No shared surfaces in scope.")

### Status Bar Content

{What the status bar shows globally vs per-view — e.g., "Always: current mode, app version. Per-view: see each blueprint's status-bar content field."}

---

## Terminal Capability Strategy

- **Color depth:** {supported tiers — e.g., "truecolor | 256 | 16 | monochrome"} and degradation rules (what falls back to what as the terminal reports fewer colors)
- **Unicode vs ASCII fallback:** {detection rule; what substitutes for line-drawing, spinners, icons when ASCII-only}
- **Mouse:** {required | optional; which interactions degrade without mouse — e.g., "resize splits requires keyboard `Ctrl-W <` / `Ctrl-W >` when mouse is unavailable"}
- **Minimum terminal size:** {rows × columns}; below the minimum, the app shows the terminal-too-small state and waits for resize
- **Reserved keys:** {keys the terminal typically captures — `Ctrl-S`, `Ctrl-Q`, `Ctrl-Z` — and the alternative keybinding the app provides for each}
- **Clean exit:** {restoration contract — "alt-screen restored, cursor visibility restored, mouse mode reset on normal exit, SIGINT, SIGTERM, and terminal-disconnect (SIGHUP)"}

---

## External Editor & Shell Integration

(Omit this section entirely if the TUI never invokes an external editor or shells out.)

- **Flows invoking `$EDITOR`:** {list — e.g., "commit message entry, free-form note input"}
- **`$EDITOR` unset behavior:** {fallback rule — e.g., "fall back to `vi`; if `vi` missing, abort with an editor-missing error role"}
- **Shell-out conventions:** {suspend TUI → restore alt-screen on return; which commands invoke a shell}

---

## Global States

| Condition | What the user sees | Recovery path |
|-----------|-------------------|---------------|
| Uncaught error / crash | {description — e.g., "alt-screen restored, stack trace role printed to stderr, non-zero exit"} | {action} |
| Quit confirmation | {description, or "N/A — quit is immediate"} | {action} |
| Version mismatch with backend | {description, or "N/A — local-only"} | {action} |
| Broken connection | {description — e.g., "broken-connection banner in status bar, views show last-known data in a stale state"} | {action} |
| Session expired | {description, or "N/A — no auth"} | {action} |
| Terminal too small | {description — e.g., "terminal-too-small screen with required size; app waits for resize"} | {action} |

---

## Mouse Support

(Omit this section entirely if mouse strategy is fully covered by the Terminal Capability Strategy minimums and there is nothing extra to say.)

- **Beyond defaults:** {extended mouse interactions — e.g., "right-click opens context menu on list items", "drag resizes splits beyond `Ctrl-W <` / `Ctrl-W >`"}
- **Modes honoring mouse:** {which modes accept mouse input}

---

## Plugin / Extension Surface

(Omit this section entirely if the TUI has no plugin system.)

- **Extension points:** {what plugins can register — custom views, custom keybindings, custom commands, custom status-bar segments}
- **Registration model:** {high-level — e.g., "plugins are declared in a config file; loaded at startup"}
- **Plugin-view contract:** {how plugin views participate in the layout, mode, and focus models}

---

## Accessibility

(Omit this section entirely if accessibility is out of scope for this phase. TUI screen-reader support is inherently limited; include only if explicitly in scope.)

- **Screen-reader friendliness:** {rules — e.g., "every view announces its role in a text status line alongside any visual indicator"}
- **Color-contrast fallback:** {monochrome-safe rule — "no state is conveyed by color alone"}
- **Motion:** {rule for reduced-motion — e.g., "spinners degrade to static text when `PROMPT_NO_ANIMATION=1`"}

---

## Configuration / Theming

(Omit this section entirely if users cannot remap keys or theme the TUI.)

- **User-configurable:** {what can be changed — key remapping, color theme, panel defaults}
- **Fixed:** {what cannot be changed — layout structure, mode model, global kill key}
- **Config file location and precedence:** {where user config lives; precedence vs built-in defaults}

---

## Internationalization

(Omit this section entirely if single-locale.)

- **Locale detection:** {rule — e.g., `LC_ALL` / `LANG` / `TOOL_LOCALE` precedence}
- **Message catalog:** {framework; file layout}
- **RTL considerations:** {rule for panel ordering and text rendering, or "No RTL locales in scope."}

---

## Telemetry

(Omit this section entirely if no telemetry.)

- **Opt-in vs opt-out:** {rule}
- **What is collected:** {high-level — e.g., view transitions, command names, session duration. Never argument values or content.}
- **Disable mechanism:** {env var or config key}
- **Privacy contract:** {what the TUI promises not to send}

---

## Traceability Matrix

### Use cases → views

| Use case | View(s) | Notes |
|----------|---------|-------|
| {UC-ID} | `{ViewName}`, `{ViewName}` | {brief note, or "Primary surface."} |
| {UC-ID} | — | No TUI surface — {one-line reason}. |

(Every use case from the input document appears here. If sibling IAs exist, note
 in the third column when a use case also surfaces on web, CLI, mobile, or voice.)

### Views → use cases

| View | Use case(s) | Classification if none |
|------|-------------|------------------------|
| `{ViewName}` | {UC-IDs} | — |
| `{ViewName}` | — | chrome — splash / help / error |

(Every view from the inventory appears here.)

### Cross-channel consistency (if sibling IAs exist)

| Use case | Primary surface | Secondary surfaces | Notes |
|----------|-----------------|--------------------|-------|
| {UC-ID} | TUI / CLI / web / mobile / voice | {list or "—"} | {one-line reason for the primary-surface choice} |

(Only present if `CLI_IA.md`, `WEB_IA.md`, `MOBILE_IA.md`, or `VOICE_IA.md` is an input.
 A use case claimed as primary on multiple surfaces must be explicitly justified here.)

---

## Relationship to Other Documents

- **Product proposal and use cases:** This document realizes use cases on the TUI surface. Use-case IDs in the traceability matrix match the source use cases document.
- **Per-unit SPECs:** SPECs for TUI units must reference the TUI_IA entry for the view(s) or shared surface(s) they implement, and use view names, panel roles, keybindings, and mode names from this document verbatim.
- **CONTRACT_REGISTRY.md** (if present): per-view network calls may reference endpoint names; wire-format shapes are owned by the contract registry, not redefined here.
- **CLI_IA.md** (if present): the simple CLI surface is a parallel document. The TUI and CLI describe disjoint surfaces. The Traceability Matrix's cross-channel table calls out any use case claimed as primary on both.
- **WEB_IA.md / MOBILE_IA.md / VOICE_IA.md** (if present): parallel documents for other channels; the Traceability Matrix's cross-channel table keeps use-case coverage consistent across channels.
- **Visual / UX execution:** specific colors, box-drawing character choices, spinner frames, and final literal copy are owned downstream of this document, not here.

---

## Open Questions

- [ ] {Question — e.g., "Should `q` close the current panel (lazygit convention) or quit the whole app (vim convention)?"}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Full view-template inventory with category classification (main / detail / overlay / help / error / splash)
- Per-view blueprint: purpose, layout, modes available, keybindings, default focus, primary action, data dependencies, full state matrix (incl. terminal-too-small), status-bar content, entry/exit, parameter-dependent behavior
- Layout Model, Mode Model, Global Keybinding Matrix, Focus Model — each stated once as authoritative
- Multi-view flow descriptions that sequence views from the inventory
- Shared surfaces inventory (command palette, dialogs, help overlay, status bar, notifications, search overlay, global banners)
- Terminal capability strategy (color depth, Unicode fallback, mouse, minimum size, reserved keys, clean exit)
- Global states (crash, quit confirmation, version mismatch, broken connection, session expired, terminal too small)
- External editor and shell integration when applicable
- Bidirectional traceability between use cases and views, cross-channel when sibling IAs exist
- Conditional sections (mouse, plugins, accessibility, theming, i18n, telemetry) when applicable to the product

### Out of scope

- Implementation details — framework specifics (bubbletea/textual/ratatui idioms), event-loop internals, widget-library usage, state-machine variant names, reducer signatures — owned by per-unit `SPEC.md`
- Visual design — specific colors, box-drawing character choices, spinner frames, icon glyphs, ANSI escape details — owned downstream of this document (visual / UX execution)
- Final UX copy — message strings, labels, button text — owned by a UX-writing phase
- Simple CLI (one-shot commands that read args and print output) — owned by `CLI_IA.md`
- Web information architecture — owned by `WEB_IA.md`
- Mobile and voice information architecture — owned by sibling IAs (`MOBILE_IA.md`, `VOICE_IA.md`)
- Backend and API wire formats — owned by `CONTRACT_REGISTRY.md`
- Database schema, business logic, algorithm design — owned by architecture and unit SPECs
- Packaging, distribution, signing, release pipeline — owned by DEPLOYMENT docs
- Persona research and user research — owned by a separate research phase

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `views_documented`, `modes_documented`, `global_keybindings`, `shared_surfaces`, `flows_documented`, `use_cases_covered`, `use_cases_total`, `min_terminal_rows`, `min_terminal_cols`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] Every view template in the inventory has a full per-view blueprint entry below
- [ ] Every blueprint declares view identifier, category, purpose, audience, referenced use cases (or classification), layout, modes available, view-specific keybindings, default focus, primary action (or chrome classification), secondary actions, data dependencies, full state matrix (incl. terminal-too-small), status-bar content, entry points, exit points, parameter-dependent behavior, and shared surfaces invoked
- [ ] Every view with parameters documents parameter-dependent behavior (at minimum: not-found and unauthorized branches)
- [ ] Layout Model, Mode Model, Global Keybinding Matrix, and Focus Model each appear exactly once and are referenced by per-view entries
- [ ] Every view's mode field is a subset of Mode Model; "modeless" is acceptable if declared in Mode Model
- [ ] Every view's keybindings extend (never redefine) globals; any shadowing is documented inline with the escape-hatch rule
- [ ] Every use case in the input use cases document maps to at least one view in the traceability matrix, or is explicitly listed as having no TUI surface with a one-line reason
- [ ] Every view in the inventory is referenced by at least one use case in the traceability matrix, or is explicitly classified (chrome — splash / help / error)
- [ ] Terminal Capability Strategy section is present; every view's fallback behavior for color / Unicode / mouse conforms to it
- [ ] Minimum terminal size is declared; every view's terminal-too-small state is documented
- [ ] Every flow step references an existing view template from the inventory (no flow introduces new views)
- [ ] Shared surfaces appear only in the dedicated Shared Surfaces section, not scattered in per-view entries
- [ ] Conditional sections (External Editor, Mouse Support, Plugin Surface, Accessibility, Configuration/Theming, Internationalization, Telemetry) are either fully completed or fully omitted — never present-and-empty
- [ ] No implementation detail (widget-library types, model struct fields, event-loop internals, reducer signatures) appears anywhere in the document
- [ ] No specific visual choices (colors, box-drawing glyphs, spinner frames, icon glyphs, ANSI escapes) appear anywhere in the document
- [ ] No final literal UX copy — message roles are described, not literal strings
- [ ] Frontmatter `use_cases_covered` and `use_cases_total` are present; any gap is explained in the Traceability Matrix
- [ ] If `CLI_IA.md`, `WEB_IA.md`, `MOBILE_IA.md`, or `VOICE_IA.md` is an input, the cross-channel consistency table is present and no use case is claimed as primary on multiple surfaces without explicit justification
