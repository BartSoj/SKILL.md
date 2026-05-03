---
name: VIEW_SPEC.md
description: Produce a deep per-view design specification for one view in a rich terminal UI application — view shell, per-panel layout, mode-specific layouts, keybinding detail with panel dispatch, focus mechanics, status-bar composition, state-specific layouts, terminal-capability degradation, mouse behavior, and external program invocation. Use when asked to create a VIEW_SPEC.md, deeply specify a TUI view, extend a TUI_IA blueprint into a per-view spec, design panel layout and focus mechanics for a flagship TUI view, or produce a VIEW_SPEC.md.
---

# Task: Create VIEW_SPEC.md for a Single TUI View

## Objective

Produce a VIEW_SPEC.md that serves as the deep, per-view design specification for one view in a rich, full-screen terminal UI application (lazygit / k9s / spotify-tui style): the view shell (total grid, frame characters, status-bar position, title bar, help-hint footer); per-panel layout (position, dimensions, resize behavior, border treatment, title, content rendering, scrolling, selection model, in-panel states); mode-specific layouts; per-key behavior in each panel including conflict resolution; focus mechanics; status-bar composition; state-specific layouts; terminal-capability degradation across color depth, Unicode, and minimum-size tiers; mouse behavior; and external editor / shell-out invocation. The defining claim — and the definition of "done" — is that an implementing agent reading TUI_IA.md's blueprint for the view + DESIGN.md's terminal style tokens + this VIEW_SPEC.md composes the view exactly as designed across every supported terminal capability and mode, without inventing layout, keybinding, focus, or degradation decisions of its own.

VIEW_SPEC is a Tier 2, opt-in artifact. Most views stay with their lightweight per-view blueprint inline in TUI_IA.md. A view earns a dedicated VIEW_SPEC only when it meets the selection criteria: composition would exceed roughly thirty inline lines; multi-panel layout with non-trivial panel-resize or focus behavior; multiple modes whose layouts diverge materially; keybinding conflicts that need explicit resolution rules; complex terminal-capability degradation across multiple distinct fallback tiers; non-trivial mouse interactions; multiple work units touch the same view; or the view is the application's flagship surface. Views not meeting any of these stay with inline blueprints in TUI_IA.md.

The commonest violation is overlapping concerns with TUI_IA — redefining panels, modes, keybindings, primary action, data dependencies, or traceability. Those belong in TUI_IA. VIEW_SPEC owns *composition, mode-specific layouts, focus mechanics, status-bar composition, state-specific layouts, terminal-capability degradation, mouse behavior, and external-program invocation* — and nothing else from TUI_IA's surface.

---

## Inputs

1. **View identifier** (required) — the stable view ID from TUI_IA.md (e.g., `StatusView`, `LogView`, `DashboardView`). This is the primary key for the spec and the value of the `view_id` frontmatter field.
2. **TUI_IA.md** (required) — provides the view's lightweight blueprint to extend: layout reference, panel inventory, modes available, keybindings, default focus, primary action, data dependencies, full state matrix, entry/exit points. Read the blueprint entry for the named view end-to-end before drafting; also read the Layout Model, Mode Model, Global Keybinding Matrix, Focus Model, and Terminal Capability Strategy sections — they bound the design space for this VIEW_SPEC.
3. **DESIGN.md** (required if present) — terminal-style tokens. Cite by stable name: color tokens for each color-depth tier including monochrome fallback; glyph tokens with Unicode-rich and ASCII fallback; frame-character set (single, double, rounded, ASCII fallback); status-bar segment style; motion / animation tokens for terminals (spinner frame counts, cursor blink rules); density tokens. If DESIGN.md is absent, surface the missing-token need in section 12 — do not invent tokens here.
4. **Sibling VIEW_SPEC.md files** (optional, auto-discovered) — discovery mechanism: walk the entry points and exit points in this view's TUI_IA blueprint and load any sibling VIEW_SPEC at `tui-views/<sibling-id>/VIEW_SPEC.md`. Used only to align focus-restoration contracts and external-editor re-entry behavior at boundaries.
5. **Prior `tui-views/<view-id>/VIEW_SPEC.md`** (optional, auto-discovered — only if refining) — read fully; preserve every resolved open question and every panel-internal naming choice. Re-run citations against the current TUI_IA and DESIGN.md — do not assume a stale token name or a stale panel name from TUI_IA is still valid.

The output file is conventionally placed at `tui-views/<view-id>/VIEW_SPEC.md` in the consuming project, where `<view-id>` matches the `view_id` frontmatter field exactly.

---

## Rules

### Completeness rules

- Every section in the Output Format template is mandatory. Conditional sections (mouse, external editor, status bar) declare their inapplicability with the verbatim "not supported" / "not applicable" sentence specified in the template — they are never silently omitted.
- Every panel listed in TUI_IA's blueprint for this view appears in section 3 by the same name, in the same order, with no panels added and no panels dropped.
- Every mode listed in TUI_IA's blueprint for this view appears in section 4 with its own layout description. If the view is single-mode, section 4 reads "Single mode (normal); no mode-specific layouts." verbatim.
- Every state from TUI_IA's state matrix for this view that materially changes layout appears in section 8. States whose layout matches "default / populated" are listed as "`{StateName}` — Inherits default layout — no panel changes." rather than dropped silently.
- Every capability tier defined in DESIGN.md is documented in section 9. If DESIGN.md defines tiers `truecolor`, `256-color`, `16-color`, `monochrome`, `unicode-rich`, `unicode-basic`, `ascii-only`, every one appears with a per-view rule.
- The Open Questions section (§ 12) must be empty unless genuinely ambiguous. If you cannot resolve a question from TUI_IA + DESIGN.md, surface it rather than inventing.

### Precision rules

- Use exact names — TUI_IA panel names verbatim, TUI_IA mode names verbatim, DESIGN.md token names verbatim, TUI_IA keybinding notation verbatim. Do not paraphrase a panel name from TUI_IA into a "more descriptive" variant.
- Cite keybindings using a consistent notation: `Ctrl-x` for Ctrl modifier, `Ctrl-Shift-X` for Ctrl-Shift modifier, `gG` for chord sequences, `<Esc>` for non-printable special keys, `Tab` and `Shift-Tab` written out. Match TUI_IA's notation when it differs.
- State exact values for every measurable: panel widths in columns or percentages, panel heights in rows or percentages, minimum overall terminal size in `cols × rows`, hard-refusal threshold in `cols × rows`, minimum and maximum panel dimensions, scrollbar width if any.
- Distinguish "instant transition" from "progressive transition" explicitly when describing state changes. TUI transitions are almost always instant; if a transition is progressive (e.g., a fade-in over N animation frames), name the duration and the motion token from DESIGN.md.

### Scope rules

- Section 1 Identity & Cross-references must list every TUI_IA cross-reference (view ID, mode availability, layout family, sibling view IDs) and every DESIGN.md token cited in the body.
- Do not redefine panels, modes, keybindings, primary action, data dependencies, or use-case traceability — those are TUI_IA's. Reference them; never restate them.
- Do not define new tokens, glyphs, or colors — those are DESIGN.md's. Cite by stable name.
- Do not pick implementation libraries (no `tui-rs`, `bubbletea`, `textual`, `ratatui`, `blessed`, `prompt-toolkit`). VIEW_SPEC is library-agnostic.
- Do not write final user-facing strings — TUI_IA owns labels, button text, and message copy. Describe message *roles* if a string is referenced.

### Cross-reference rules

VIEW_SPEC is the bridge between TUI_IA's lightweight blueprint and per-unit code-level SPECs. Every cross-reference must resolve to a stable identifier:

- **TUI_IA panel names** must match TUI_IA's blueprint for this view exactly. A panel renamed in VIEW_SPEC means TUI_IA is stale — update TUI_IA first.
- **TUI_IA mode names** must match TUI_IA's Mode Model exactly (`Normal`, `Command`, `Insert`, `Visual`, `Search`, or the project's named set). A new mode means TUI_IA's Mode Model is incomplete — extend TUI_IA first.
- **TUI_IA keybindings** in section 5 must extend (never redefine) TUI_IA's Global Keybinding Matrix and the view's view-specific keybindings. Any shadowing must be documented with the escape-hatch rule from TUI_IA.
- **DESIGN.md tokens** are cited by stable name (e.g., `color-accent`, `glyph-bullet`, `frame-double`, `frame-ascii`). Wrap token names in backticks. Token values are not inlined — they are owned by DESIGN.md.
- **DESIGN.md capability tiers** are cited by name in section 9 (e.g., `truecolor`, `256-color`, `16-color`, `monochrome`, `unicode-rich`, `unicode-basic`, `ascii-only`). Tier names match DESIGN.md exactly.
- **Sibling view IDs** in section 11 (external editor re-entry) and elsewhere match TUI_IA's view inventory exactly.

### Single YAML frontmatter block

- Exactly one YAML frontmatter block at the top of the VIEW_SPEC.md output, never two — even when refining an existing VIEW_SPEC, merge into a single block. The block carries every field enumerated in the Output Format template (`skill`, `date`, `status`, `view_id`, `modes_supported`, `panels_specified`, `keybindings_detailed`, `states_specified`, `capability_tiers_specified`, `open_questions`).
- Every count in the frontmatter matches the body exactly: `modes_supported` is the number of modes documented in § 4 (1 if the view is single-mode); `panels_specified` is the number of panel entries in § 3; `keybindings_detailed` is the number of rows in § 5's per-key behavior table; `states_specified` is the number of state entries in § 8; `capability_tiers_specified` is the number of capability rules in § 9; `open_questions` is the number of unresolved questions in § 12 (zero when § 12 reads "All questions resolved.").
- `status` is `complete` only when § 12 reads "All questions resolved."; `has_open_questions` when one or more questions remain unresolved; `blocked` only when a missing input (an absent DESIGN.md, a stale TUI_IA blueprint that lacks the panels needed to spec) prevents authoring and the gap is explicit in § 12.

### Iteration discipline

- Each table row enumerates one element — one panel, one keybinding, one state, one capability tier. Do not collapse "all panels" into a single row.
- For tables, use markdown tables; for ordered behaviors, use ordered lists. Avoid mixing.
- Empty-state language is verbatim where prescribed by the template — "Single mode (normal); no mode-specific layouts.", "Mouse not supported; keyboard-only.", "No external programs invoked from this view.", "No status bar; help-hint footer only.", "All questions resolved." — so frontmatter status detection is mechanical.

### No-code rule

- VIEW_SPEC must not contain code, pseudocode, or framework-shaped descriptions (no `Model` structs, `Command` enums, `View::draw()` signatures, `KeyEvent` matchers, reducer signatures). Describe composition, behavior, and contracts in prose.

---

## Output Format

The VIEW_SPEC.md file must follow this exact structure. Every section is mandatory. Conditional sections use the verbatim "not supported" / "not applicable" sentences specified in the template — never silent omission.

```markdown
---
skill: VIEW_SPEC.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
view_id: {view-id-from-TUI_IA}
modes_supported: {N}
panels_specified: {N}
keybindings_detailed: {N}
states_specified: {N}
capability_tiers_specified: {N}
open_questions: {N}
---

# VIEW_SPEC: `{ViewName}` — {short name}

## 1. Identity & Cross-references

- **View ID:** `{ViewName}` (matches TUI_IA inventory)
- **Mode availability:** {comma-separated list from TUI_IA's blueprint — e.g., "Normal, Command"; or "Modeless"}
- **Layout family:** {split-pane | single-pane | overlay | modal-overlay} (cite the root layout from TUI_IA's Layout Model — e.g., "split-pane — three-column under TUI_IA root layout `split-screen`")
- **DESIGN.md tokens cited:** {comma-separated list of every token referenced anywhere in this document — e.g., `` `color-accent`, `color-error`, `glyph-bullet`, `frame-double`, `frame-ascii`, `status-segment-divider` ``}
- **Sibling views referenced:** {TUI_IA view IDs cited in this spec — e.g., `LogView`, `DetailView` — or "None."}

---

## 2. View Shell

- **Total grid layout:** {column / row count if fixed-grid; flex shares if flexible — e.g., "three columns: left fixed at 24 cols, center flexible, right fixed at 32 cols; one fixed row at the top for title bar; one fixed row at the bottom for status bar"}
- **Minimum overall terminal size:** {`cols × rows` — e.g., "80 × 24"} (must be ≥ TUI_IA's project-wide minimum)
- **Hard-refusal threshold:** {`cols × rows` below which the view defers to TUI_IA's global terminal-too-small screen — e.g., "60 × 16"}
- **Frame characters by region boundary:**

| Region boundary | Unicode-rich token | ASCII fallback token |
|-----------------|---------------------|----------------------|
| {boundary, e.g., "left panel ↔ center panel"} | `{token}` | `{token}` |

(Repeat per boundary; refer to DESIGN.md tokens by name only.)

- **Status bar position:** {top | bottom | both | none}
- **Title bar:** {present? content role — e.g., "present; shows view name + repo identity role"; or "None."}
- **Help-hint footer:** {present? what is shown — e.g., "present; shows top three view-specific keybindings as `key role` triples"; or "None."}

---

## 3. Per-Panel Layout

For every panel listed in TUI_IA's blueprint for this view (preserve names verbatim, in the same order):

### `{PanelName}`

- **Position and dimensions:** {absolute or percentage; min/max widths and heights — e.g., "left column; fixed 24 cols wide; min 18 cols, max 32 cols; full height between title bar and status bar"}
- **Resize behavior:** {what happens when the terminal resizes — e.g., "preserves fixed width; if total width drops below the minimum, the panel hides per the resize cascade in section 9"}
- **Border treatment:**
  - Focused: {frame style + color token — e.g., "`frame-double` in `color-accent`"}
  - Unfocused: {frame style + color token — e.g., "`frame-single` in `color-text-muted`"}
- **Title:** {content role + alignment + glyph prefix — e.g., "shows panel name; left-aligned with leading `glyph-bullet`"; or "None."}
- **Content rendering:** {list | table | text | form | tree | log | preview | empty surface — name the kind exactly}
- **Scrolling:** {vertical | horizontal | wrap | none}; {scrollbar style or absence — e.g., "vertical only; scrollbar shown as 1-col gutter on right edge using `glyph-scrollbar-thumb`"; or "none — content paginated by `n` / `p`"}
- **Selection model:** {single | multi | range | none}; {how the selection is rendered — e.g., "single; selected row inverted via `color-selection-bg`"}
- **Empty state:** {what the panel shows when its data set is empty — e.g., "centered empty-state prompt role using `color-text-muted`"}
- **Loading state:** {what the panel shows during initial load and during background refresh — e.g., "initial: spinner role centered; background refresh: subtle indicator in title using `glyph-refresh-active`"}
- **Error state:** {what the panel shows when its data fetch fails — e.g., "error banner role replacing content; retry hint at bottom"}

(Repeat per panel.)

---

## 4. Mode-Specific Layouts

For each mode supported by this view (per TUI_IA's blueprint):

### `{ModeName}`

- **What changes between this mode and Normal:** {borders, prompt bar, content area — e.g., "Command mode adds a prompt row above the status bar; center panel grays out; left-panel borders dim from `color-accent` to `color-text-muted`"}
- **Mode indicator:** {where it shows + color + label — e.g., "left segment of status bar; `color-mode-command` background; label `:`"}
- **Status-bar contents in this mode:** {described in terms of section 7's segments — e.g., "left segment shows mode label; center segment shows the active prompt buffer; right segment is hidden"}
- **Help hints in this mode:** {what the help-hint footer shows — e.g., "shows `<Enter> submit | <Esc> cancel`"}

(Repeat per mode. If the view is single-mode: "Single mode (normal); no mode-specific layouts." — verbatim, and skip the rest of this section.)

---

## 5. Keybinding Detail

Beyond TUI_IA's keybinding matrix — this section documents per-panel dispatch, conflict resolution, mnemonic structure, reserved keys, and chord/repeat behavior. Do not re-list keys defined in TUI_IA's Global Keybinding Matrix; reference them by name.

### Per-key behavior

| Key(s) | Mode | Receiving panel | Action role | Source |
|--------|------|-----------------|-------------|--------|
| `{key}` | `{mode}` | `{panel name or "view-level"}` | {what the key does in this panel} | TUI_IA / view-specific |

(Repeat per key. Every key from TUI_IA's view-specific keybindings table for this view appears here with its panel dispatch resolved.)

### Conflict resolution

When the same key has different meanings in different panels, the resolution rule:

| Key | Resolution rule |
|-----|-----------------|
| `{key}` | {rule — e.g., "dispatched to the focused panel; if focus is on the status bar, dispatched to the previously focused content panel"} |

(Or "No conflicts in this view.")

### Mnemonic structure

- **Chord prefixes:** {list — e.g., "`g` is the go-to chord prefix: `gg` top, `gG` bottom, `gd` definition"; or "No chord prefixes."}
- **Leader keys:** {list — e.g., "`<Space>` is the leader key for view-scoped commands: `<Space>r` refresh, `<Space>f` filter"; or "No leader keys."}
- **Vim-style g-prefix sequences:** {list, or "None."}

### Reserved keys

| Key | Why reserved |
|-----|--------------|
| `Ctrl-C` | force-quit per TUI_IA Global Keybinding Matrix |
| `Ctrl-D` | {reason — e.g., "owned by terminal; never bound"} |
| `Ctrl-Z` | {reason — e.g., "suspends process per shell convention"} |

(Repeat for every key this view never binds. If only the global reserved set applies: "Inherits TUI_IA reserved-keys list; no view-specific additions.")

### Hold / repeat behavior

| Key(s) | Behavior |
|--------|----------|
| `j` / `k` | auto-repeat with terminal repeat rate; no debounce |
| `{key}` | single-shot only; subsequent presses within {N}ms ignored |

(Or "All keys follow terminal default repeat behavior; no view-specific hold/repeat rules.")

---

## 6. Focus Behavior

- **Default focus on entry:** {panel name from section 3 — e.g., "`{PanelName}`"} (must match TUI_IA's blueprint default-focus declaration)
- **Focus order on `Tab` / `Shift-Tab`:** {sequence — e.g., "left panel → center panel → right panel → left panel (cycles)"; or "left panel → center panel → right panel (stops at right; `Shift-Tab` reverses)"}
- **Focus indicator:** {how focus is shown — e.g., "focused panel border switches to `frame-double` in `color-accent`; focused panel title gets `glyph-focus-marker` prefix; cursor visible in focused panel only"}
- **Focus restoration on view re-entry:** {behavior — e.g., "restores last focus; if the previously focused panel is hidden under the current viewport size, falls back to default focus"}
- **Focus loss when focused panel becomes empty or unavailable:** {behavior — e.g., "focus moves to the next panel in the focus order that is non-empty; if all are empty, focus moves to the panel that owns the primary action"}

---

## 7. Status Bar Composition

(If TUI_IA declares no status bar for this view: "No status bar; help-hint footer only." or "No persistent chrome." — verbatim, and skip the rest of this section.)

- **Left segment:**
  - Content role: {e.g., "current mode label"}
  - Format: {e.g., "uppercase mode name in `color-mode-{mode}-fg` on `color-mode-{mode}-bg`"}
  - Max width: {N cols}
  - Truncation rule: {e.g., "truncated with `glyph-ellipsis` from the right when the segment exceeds max width"}
- **Center segment:**
  - Content role: {e.g., "active context — repo identity for repo views, resource path for resource views"}
  - Alignment: {left | center | right within its segment}
  - Truncation rule: {e.g., "truncated with `glyph-ellipsis` from the left to preserve trailing identity"}
- **Right segment:**
  - Content role: {e.g., "ambient state — sync status, branch dirty marker, notification counter"}
  - Format: {e.g., "space-separated chips, each `glyph-chip-prefix` `name`"}
- **Mode indicator placement:** {left | center | right segment}
- **Notification overlay:** {how transient messages appear and dismiss — e.g., "appears as an inline overlay above the status bar; auto-dismisses after 4 seconds; manual dismissal with `<Esc>` or `Ctrl-G`"}

---

## 8. State-Specific Layouts

For every state from TUI_IA's state matrix for this view that materially changes layout. States that inherit default layout list as a single line: "`{StateName}` — Inherits default layout — no panel changes."

### `{StateName}`

- **What changes per panel:** {per-panel description — e.g., "left panel: dimmed; center panel: replaced by error banner role with retry hint; right panel: hidden below 80 cols"}
- **Status-bar changes:** {what changes in the status bar — or "Status bar unchanged."}
- **Active keybindings:** {full | subset — e.g., "subset: only `r` retry, `<Esc>` exit, and global `Ctrl-C`"}
- **Transition behavior:** {instant | progressive — e.g., "instant"; if progressive, name the duration and the motion token from DESIGN.md}

(Repeat per material state.)

---

## 9. Terminal Capability Degradation

For each capability tier defined in DESIGN.md (cite tier names verbatim).

### Color depth

| Tier | Palette mapping for this view |
|------|-------------------------------|
| `truecolor` | {tokens used at full fidelity — e.g., "all `color-*` tokens render their truecolor values"} |
| `256-color` | {mapping rule — e.g., "`color-accent` → DESIGN.md's `color-accent-256` mapping"} |
| `16-color` | {mapping rule — e.g., "all accent colors collapse to `color-accent-16`; selection uses inverse video"} |
| `monochrome` | {emphasis fallback — e.g., "no color is used; focus = bold border via `frame-double`; selection = inverse video; error = blinking attribute on banner"} |

### Unicode glyphs

| Tier | Glyph mapping for this view |
|------|-----------------------------|
| `unicode-rich` | {full glyph set used — e.g., "all `glyph-*` tokens render their Unicode-rich values"} |
| `unicode-basic` | {fallback rule — e.g., "`glyph-bullet` → `•`; `glyph-arrow` → `→`; spinner uses 4-frame block sequence"} |
| `ascii-only` | {ASCII fallback — e.g., "frame characters → `frame-ascii` (`+` / `-` / `|`); spinner → 4-frame slash sequence; `glyph-bullet` → `*`"} |

### Minimum terminal size and reflow cascade

- **Minimum overall terminal size:** {`cols × rows` — must match section 2}
- **Hard-refusal threshold:** {`cols × rows` — below this the view defers to TUI_IA's global terminal-too-small screen}
- **Reflow cascade:** ordered list of what hides or collapses as columns shrink — e.g.,
  1. Below 120 cols: right panel hides; center panel takes the freed columns.
  2. Below 90 cols: title bar shrinks to view name only (drops ambient identity).
  3. Below 80 cols: defer to global terminal-too-small screen.

### Mouse support tier

- {whether mouse is supported here; cite section 10 — e.g., "Mouse supported per section 10."; or "Mouse not supported; keyboard-only."}

---

## 10. Mouse Behavior

(If mouse is not supported in this view: "Mouse not supported; keyboard-only." — verbatim, and skip the rest of this section.)

- **Click targets per panel:**

| Panel | Click action |
|-------|-------------|
| `{PanelName}` | {action — e.g., "click on a row selects it and focuses the panel"} |

(Repeat per panel.)

- **Drag behavior:** {selection | resize | scroll | none — e.g., "drag on the panel boundary resizes the split between adjacent panels; minimum panel widths from section 3 are honored"}
- **Wheel:** {scroll | zoom-equivalent | none — e.g., "scrolls the focused panel; horizontal-wheel scrolls horizontally where supported"}
- **Right-click:** {context menu | focus | disabled — e.g., "context menu role on selected row in `{PanelName}`; disabled in other panels"}
- **Modifier-click:** {Shift-click range select | Ctrl-click multi-select | none — e.g., "Shift-click extends range selection in `{PanelName}`; Ctrl-click toggles individual selection"}

---

## 11. External Editor / External Tool Invocation

(If this view spawns no external programs: "No external programs invoked from this view." — verbatim, and skip the rest of this section.)

For each external invocation point:

### {Invocation name — e.g., "Edit commit message"}

- **Trigger:** {keybinding from section 5, or action from section 4 — e.g., "`e` from `{PanelName}` in Normal mode"}
- **Command:** {exact program plus arguments — e.g., "`$EDITOR` with the temp-file path as the sole positional argument"}
- **`$EDITOR` unset behavior:** {fallback — e.g., "fall back to `vi`; if `vi` is missing, surface an editor-missing error role and abort the action"}
- **Working directory:** {where the external program runs — e.g., "the project root, resolved at view-entry time"}
- **TUI suspension:** {whether the alt-screen is released before the external program runs — e.g., "alt-screen released; mouse mode reset; cursor restored; standard terminal returns"}
- **Re-entry behavior:** {how the view is restored — e.g., "alt-screen reattached; full redraw of all panels; focus returns to the panel that triggered the invocation; data refreshed if the external program modified the source file"}
- **Error handling:** {when the external command fails — e.g., "non-zero exit code surfaces an error banner role on re-entry; partial output is discarded; keybinding remains available for retry"}

(Repeat per invocation point.)

---

## 12. Open Questions

This section must be EMPTY before implementation begins. If any questions remain unresolved, the spec is not ready. The user resolves open questions after reviewing the spec — do not ask during generation.

When drafting the spec, you will encounter decisions where multiple approaches are defensible and the input documents (TUI_IA, DESIGN.md, sibling VIEW_SPECs) do not clearly favor one. Do NOT silently pick an answer to keep this section empty. Instead:

- If a question has one obviously correct answer given the input documents, resolve it and write the decision into the appropriate section.
- If a question has no clear answer — multiple valid approaches exist, or the input documents are ambiguous or silent — list it here.

Format for open questions:

- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {if you lean one way, say so and why — but leave the decision to the user}

Once the user resolves all questions, move each resolution into the appropriate section and replace this section with:

"All questions resolved."
```

---

## Scope

### In scope

- View shell composition — total grid layout, minimum and hard-refusal terminal sizes, frame characters by region boundary, status-bar position, title bar, help-hint footer
- Per-panel layout — position, dimensions, resize behavior, focused/unfocused border treatment, title, content rendering kind, scrolling, selection model, empty/loading/error in-panel states
- Mode-specific layouts for every mode the view supports per TUI_IA
- Per-key behavior with panel dispatch, conflict resolution rules, mnemonic structure (chord prefixes, leader keys, g-prefix sequences), reserved keys, hold/repeat behavior
- Focus mechanics — default focus, focus order on `Tab` / `Shift-Tab`, focus indicator, focus restoration on re-entry, focus-loss handling
- Status bar segment composition — left/center/right content roles, formats, max widths, truncation rules, mode indicator placement, notification overlay
- State-specific layout deltas for every TUI_IA state that materially changes layout
- Terminal capability degradation across color depth, Unicode, and minimum-size tiers — palette mapping, glyph fallback, reflow cascade
- Mouse behavior when supported — click targets, drag, wheel, right-click, modifier-click
- External editor and shell-out invocation when applicable — trigger, command, fallbacks, working directory, suspension, re-entry, error handling

### Out of scope

- View identity — the fact a view exists, its category, primary action, data dependencies, modes available, default focus, full state matrix, entry/exit, parameter-dependent behavior, traceability to use cases — owned by `TUI_IA.md`'s per-view blueprint
- Token definitions — color values, glyph values, frame-character values, motion durations — owned by `DESIGN.md`
- Final user-facing copy — labels, button text, error messages — owned by TUI_IA (message roles) and the UX-writing phase (literal strings)
- Implementation libraries and language idioms (`tui-rs`, `bubbletea`, `textual`, `ratatui`, `blessed`, `prompt-toolkit`, model struct fields, event-loop internals, reducer signatures) — owned by per-unit `SPEC.md`
- Backend wire formats and endpoint shapes — owned by `INTERFACES.md`; data dependencies cite endpoint names from TUI_IA where applicable
- Database schema and aggregate boundaries — owned by `DATA.md`
- Cross-view flows that sequence multiple views — owned by `TUI_IA.md`'s Flows section
- Use-case traceability — owned by `TUI_IA.md`'s Traceability Matrix

---

## Quality Checklist

Before considering a VIEW_SPEC.md complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `view_id`, `modes_supported`, `panels_specified`, `keybindings_detailed`, `states_specified`, `capability_tiers_specified`, `open_questions`)
- [ ] Frontmatter `view_id` matches a view ID present in TUI_IA's view inventory
- [ ] Frontmatter counts match the body exactly (`modes_supported` = modes in § 4 with 1 for single-mode, `panels_specified` = panel entries in § 3, `keybindings_detailed` = rows in § 5's per-key behavior table, `states_specified` = state entries in § 8, `capability_tiers_specified` = tier rules in § 9, `open_questions` = unresolved items in § 12)
- [ ] `status` is `complete` only when § 12 reads "All questions resolved."; `has_open_questions` when one or more questions remain; `blocked` only when a missing input prevents authoring and § 12 makes the gap explicit
- [ ] § 1 lists view ID, mode availability, layout family, every cited DESIGN.md token, and every sibling view ID referenced in this spec
- [ ] § 2 declares total grid layout, minimum terminal size, hard-refusal threshold, frame characters per region boundary (with Unicode-rich and ASCII-fallback tokens), status bar position, title bar, and help-hint footer
- [ ] Every panel from TUI_IA's blueprint for this view appears in § 3 by name, in TUI_IA's order; no panels added, no panels dropped
- [ ] Each panel entry covers position, dimensions, resize behavior, focused/unfocused border treatment, title, content rendering kind, scrolling, selection model, empty/loading/error in-panel states
- [ ] Every mode from TUI_IA's blueprint appears in § 4 with its own layout description; single-mode views state "Single mode (normal); no mode-specific layouts." verbatim
- [ ] § 5's per-key behavior table covers every view-specific keybinding from TUI_IA's blueprint plus any reserved keys; conflicts have explicit resolution rules; chord/leader/g-prefix structure is named
- [ ] § 5 does not re-list keys defined in TUI_IA's Global Keybinding Matrix — references them by name instead
- [ ] § 6 declares default focus matching TUI_IA's blueprint, focus order on `Tab` / `Shift-Tab`, focus indicator, focus-restoration rule, focus-loss handling
- [ ] § 7 declares left/center/right segment content, format, max width, truncation, mode indicator placement, and notification overlay; or states the absent-status-bar sentence verbatim
- [ ] § 8 covers every state from TUI_IA's state matrix that materially changes layout; states that inherit default layout list as "`{StateName}` — Inherits default layout — no panel changes."
- [ ] § 9 documents every capability tier defined in DESIGN.md (color depth tiers, Unicode tiers); reflow cascade is ordered by viewport reduction; minimum terminal size and hard-refusal threshold match § 2
- [ ] § 10 either declares mouse interactions or states "Mouse not supported; keyboard-only." verbatim
- [ ] § 11 either declares external invocations with command / fallback / working-directory / suspension / re-entry / error handling, or states "No external programs invoked from this view." verbatim
- [ ] No DESIGN.md tokens are inlined as values — only cited by name in backticks
- [ ] No new tokens, glyphs, colors, or final user-facing strings are introduced
- [ ] No implementation library names appear (`tui-rs`, `bubbletea`, `textual`, `ratatui`, `blessed`, `prompt-toolkit`)
- [ ] No code, pseudocode, or framework-shaped descriptions (`Model` structs, `Command` enums, `KeyEvent` matchers, reducer signatures)
- [ ] No overlapping concerns with TUI_IA — panels, modes, keybindings, primary action, data dependencies, traceability are cited, never restated
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] § 12 Open Questions is empty (rendered as "All questions resolved.") or contains genuine ambiguities only with options, tradeoffs, and a recommendation
- [ ] Output is self-contained — readable by an implementing agent without needing to open TUI_IA or DESIGN.md to know what this view's panels, modes, keybindings, focus, status bar, states, capability degradation, mouse behavior, and external invocations are
