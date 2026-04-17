# Per-View Blueprint Template

Every view template in the inventory gets one entry in the Per-View Blueprints section of `TUI_IA.md`, following the template below. Every field is mandatory unless explicitly marked with an "N/A" fallback.

When iterating the inventory in Phase 4 of the workflow, copy the block below per view and fill in every field. Describe behavior by layout composition, keybinding-action contract, and state behavior — not by code shape (Rule 6). Describe message surfaces by role, not by literal string (Rule 8). Draw every keybinding from the Global Keybinding Matrix (by reference) or declare it as a view-specific addition (Rule 11). Reference the Layout Model, Mode Model, and Focus Model by name — never redefine them (Rules 9, 10, 12).

---

## Template block (copy per view)

```markdown
### `{ViewName}` — {short name}

- **Category:** {main | detail | overlay | help | error | splash}
- **Purpose:** {one sentence}
- **Audience:** {user types who use this view}
- **Referenced use cases:** {UC-IDs, or explicit classification: chrome — splash / help / error}

**Layout:**

{Reference the root-layout template from the Layout Model by name, then describe this view's specific panel arrangement. Name each panel, its role, and whether its size is fixed or flexible. Note any deviation from the root layout with a one-line reason — or "Conforms to root layout."}

**Panels:**

| Panel | Role | Size | Primary content |
|-------|------|------|-----------------|
| `{PanelName}` | {what the panel is for} | fixed {N} cols/rows | flexible | {what is displayed} |

(Or "Single-panel view — no secondary panels.")

**Modes available:** {subset of Mode Model — e.g., "Normal, Command"; or "all modes"; or "Normal only (view is modeless within a modal TUI)"; or "N/A — application is modeless"}

**View-specific keybindings** (global keybindings apply by reference):

| Key(s) | Mode | Action role | Parameter-dependent behavior |
|--------|------|-------------|------------------------------|
| `{key}` | {mode name or "any"} | {what the key does — role, not literal copy} | {how the action varies with view parameters — or "—"} |

(Or "No view-specific keybindings — standard global keybindings apply.")

**Default focus:** {which panel or widget is focused on first entry — e.g., "left panel (branch list)"}

**Focus movement within the view:** {any view-specific focus-movement keys beyond the global Focus Model — or "Inherits Focus Model defaults."}

**Primary action:** {single dominant action — e.g., "Stage the highlighted hunk."} *(Or "No primary action — chrome view.")*

**Secondary actions:** {list of secondary actions by role — or "None."}

**Data dependencies:**
- {what the view needs to render — e.g., "list of branches, diff of staged/unstaged changes, recent commits"}
- {if a contract registry exists: endpoint references by name}
- {refresh strategy — e.g., "refreshes on file-system event; manual refresh bound to `R`"}

**States:**

| State | What appears in each panel | Keybindings active | What the user does next |
|-------|----------------------------|--------------------|-------------------------|
| Default / populated | {per-panel description} | {global + view-specific} | {action description} |
| Loading (initial) | {description — e.g., "spinner role in center panel; sidebar greyed"} | {active subset — e.g., "only `Esc` and `Ctrl-C`"} | {wait / cancel} |
| Loading (background refresh) | {description — e.g., "content visible; refresh indicator in status bar"} | {full} | {continue working} |
| Empty | {description — e.g., "empty-state prompt in main panel"} | {active subset} | {how to populate} |
| Error | {description — e.g., "error banner role with retry hint"} | {active subset} | {retry / escape} |
| Terminal too small | {description — e.g., "inherits global terminal-too-small screen"; or "custom — hide right panel below N cols, show single-column view"} | {typically "Ctrl-C only"} | {resize} |
| Unauthenticated | {description, or "N/A — local-only view"} | {action} | {how to authenticate} |
| Unauthorized | {description, or "N/A — no permission gating"} | {action} | {how to request access} |
| Not-found | {description, or "N/A — non-parameterized view"} | {action} | {how to recover} |

**Status-bar content on this view:** {what the status bar shows specifically when this view is active — e.g., "current branch, sync state, selected hunk count"}

**Entry points:** {where the user arrives from — e.g., "app launch default; from `LogView` via `Enter` on a commit; from command palette via `:status`"}

**Exit points:** {key + destination — e.g., "`Esc` → previous view; `q` → quit app with optional confirmation; `Enter` → `DetailView(selectedId)`"}

**Parameter-dependent behavior:** {how the view's parameters change its behavior — e.g., "if `resourceId` does not exist → not-found state; if viewer lacks read permission on a private resource → not-found state (not unauthorized) to avoid leaking existence." Or "N/A — non-parameterized view."}

**Shared surfaces invoked:** {names from the Shared Surfaces section — e.g., "confirmation dialog (on destructive action), help overlay, search overlay." Or "None."}
```

Repeat the block for every view template in the inventory.

---

## Field-by-field guidance

- **View identifier** — a `CamelCase` name ending in `View` / `Overlay` / `Screen` as appropriate (e.g., `StatusView`, `LogView`, `HelpOverlay`, `SplashScreen`). Use the same name across layout, inventory, flows, and traceability — it is the primary key for the view.
- **Category** — *main* views are primary workspaces; *detail* views drill into a main-view selection; *overlay* views are modal surfaces that temporarily replace or cover a main view (help, dialog); *help* / *error* / *splash* are chrome views with no use-case mapping.
- **Layout** — the view's own panel arrangement. Reference the root-layout template from the Layout Model by name. Do not redefine the root layout; describe only what this view adds or changes. Name each panel and its role — the panel names are used elsewhere in the blueprint (default focus, status-bar content, state matrix) and must be consistent.
- **Panels** — one row per panel. Size column distinguishes fixed (with an exact character count) from flexible (fills remaining). Primary content describes what lives in the panel; keep it to roles and content types, not literal strings or visual details.
- **Modes available** — a subset of the Mode Model. A view may be available only in Normal mode, or may expand/contract which modes are active (e.g., a search overlay adds Search mode while it is present). If the application is modeless, write "N/A — application is modeless."
- **View-specific keybindings** — *only* keys not already in the Global Keybinding Matrix. If a global key's behavior is reinterpreted for this view (shadowing), document it explicitly and state the escape hatch — which global binding still works and how.
- **Default focus** — which panel or widget receives focus on first entry. Reference focus preservation from the Focus Model — does re-entry restore the last focus or reset to this default?
- **Primary action** — the single dominant user action. If you cannot name exactly one, the view is probably two views conflated; split it. Chrome views explicitly declare no primary action.
- **Secondary actions** — keybinding-driven actions that are not the primary. Keep to roles, not literal labels.
- **Data dependencies** — the data the view renders and what drives its refresh. Name endpoints by reference where a contract registry exists; otherwise describe the data shape by role.
- **States** — every row from the template, each with per-panel content description, active keybinding subset, and next user action. Use "N/A — {reason}" for rows that genuinely do not apply; do not silently drop rows.
- **Status-bar content on this view** — what is shown *in addition to* (or *in place of*) the global status-bar content when this view is active. If a view suppresses part of the global status bar, note it.
- **Entry points** — how the user arrives at this view. Enumerate: app launch default (if any), links from other views with the triggering keybinding, command-palette command, deep-link CLI flag.
- **Exit points** — every documented way to leave the view with the key and the destination view. Include global exits (`q`, `Ctrl-C`) only if their behavior differs from the global default here.
- **Parameter-dependent behavior** — the key field for preventing per-instance enumeration (Rule 1). For parameterized views, at minimum document the not-found and unauthorized branches. Follow the information-leak rule: private resources should surface as not-found to non-members, not as unauthorized.
- **Shared surfaces invoked** — reference by name from the Shared Surfaces section of `TUI_IA.md`. Do not redescribe the surface.

---

## Common mistakes to avoid

- Enumerating concrete view instances (`StatusView for repo-A`, `StatusView for repo-B`) instead of the single template (`StatusView(resourceId)`).
- Writing code-shaped descriptions (`StatusViewModel` struct, `statusReducer` function, `tea.Cmd` values). Those belong in per-unit SPECs (Rule 6).
- Re-listing global keybindings inside view entries (Rule 11). Reference them — "standard global keybindings apply" — and list only additions.
- Redefining the root layout inside a view entry (Rule 9). Reference the Layout Model and describe only this view's specific panel arrangement.
- Introducing new modes inside a view entry (Rule 10). If a view needs a new mode, extend the Mode Model first.
- Describing visual specifics — specific colors, specific box-drawing characters, specific spinner frames (Rule 7). The *fact* of a border, spinner, or focus indicator is in scope; the artifact is not.
- Writing literal UX copy — button labels, headings, messages (Rule 8). Describe roles: "confirmation label", "error banner", "empty-state prompt".
- Leaving a state-matrix row as "TBD" or omitting rows entirely. Use "N/A — {reason}" when genuinely inapplicable.
- Redescribing a shared surface (command palette, help overlay) inside a per-view entry (Rule 14). Reference it by name.
- Declaring two primary actions on a single view (Rule 2). Split the view or justify the pairing explicitly.
