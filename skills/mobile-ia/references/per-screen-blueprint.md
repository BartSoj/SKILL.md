# Per-Screen Blueprint Template

Every screen template in the inventory gets one entry in the Per-Screen Blueprints section of `MOBILE_IA.md`, following the template below. Every field is mandatory unless explicitly marked with an "N/A" fallback.

When iterating the inventory in Phase 4 of the workflow, copy the block below per screen and fill in every field. Describe behavior by section roles and contracts, not by code shape (Rule 6). Describe text by role, not by literal string (Rule 8). Reference the centralized Navigation Model, Deep-Link Strategy, Permissions Strategy, and Shared Surfaces sections by name rather than re-describing them (Rules 9–13). Document platform variations explicitly or declare "identical across platforms" (Rule 15).

---

## Template block (copy per screen)

```markdown
### `{ScreenName}` — {short name}

- **Screen identifier / route:** `{ScreenName}` or framework-agnostic route — e.g., `ResourceDetail(resourceId)`
- **Navigation position:** {root tab name / stack name; modal vs pushed; alert vs sheet} — references the Navigation Model section
- **Platform availability:** {iOS | Android | both — note any features that differ}
- **Deep link:** `{URL pattern}` — {auth-gate behavior by reference to Deep-Link Strategy}; or "Not deep-linkable."
- **Purpose:** {one sentence}
- **Audience:** {user types who reach this screen}
- **Auth requirement:** {public | authenticated | authenticated-with-permission: {permission name}}
- **Referenced use cases:** {UC-IDs, or explicit classification: chrome | nav-container | auth-gate | error | splash | onboarding-slide | informational}

**Sections / blocks** (ordered, top-to-bottom / outside-in):

1. **{section name}** — {role and required content}
2. **{section name}** — {role and required content}

**Primary CTA:** {single action — e.g., "Fork the resource"} *(or "No CTA — {chrome | nav-container | auth-gate | error | splash | onboarding-slide | informational}.")*

**Secondary actions:** {list, or "None."}

**Data dependencies:**
- {what the screen needs to render — e.g., "resource metadata, viewer permissions, child-item list"}
- {if a contract registry exists: endpoint references by name}

**Gestures supported:** {pull-to-refresh, swipe-to-delete, long-press, pinch, swipe-between-tabs, haptics triggered — or "None beyond platform defaults."}

**Orientation support:** {portrait-only | landscape-only | both — note exceptions}

**Permissions involved:** {names from the Permissions Strategy table — or "None."}

**Platform variations:**
- **iOS:** {describe divergent behavior — e.g., "overflow action sheet is a bottom sheet with system dividers"}
- **Android:** {describe divergent behavior — e.g., "overflow action sheet is a Material bottom sheet"}

*(Or "Identical across platforms.")*

**States:**

| State | What the user sees | Actions available |
|-------|--------------------|-------------------|
| Default / loaded with data | {description} | {actions} |
| Loading (cold start) | {description} | {actions} |
| Loading (warm start) | {description} | {actions} |
| Loading (subsequent refresh) | {description} | {actions} |
| Empty | {description} | {actions} |
| Error — transient network | {description} | {actions} |
| Error — hard failure | {description} | {actions} |
| Offline | {description — cached content? disabled actions? banner?} | {actions} |
| Unauthenticated | {description or "N/A — public screen"} | {actions} |
| Unauthorized | {description or "N/A — no permission gating"} | {actions} |
| Not found | {description or "N/A — not parameterized"} | {actions} |
| Permission-denied — {permission name} | {description per permission the screen depends on} | {actions — e.g., "Settings deep link"} |
| Backgrounded (mid-screen) | {state preservation — e.g., "scroll position preserved, in-progress upload continued via background task"} | {actions on resume} |

(Add a Permission-denied row for every permission the screen depends on. Omit rows that are genuinely N/A only by replacing their content with "N/A — {reason}" — do not drop the row.)

**Entry points:**
- {from parent screen — e.g., "tapped row in Resource List"}
- {from deep link — e.g., "`tool://resource/{id}` when unauthenticated routes through Login first"}
- {from push notification — which category, what payload}
- {from widget tap, Siri Shortcut / App Intent, share sheet, cold start}

*(Or "No entry from external surfaces — reachable only from in-app navigation.")*

**Exit points:**
- {primary CTA destination — which screen, in which stack}
- {secondary navigation — back, tab switch, modal dismiss}
- {app backgrounded — what persists}

**Lifecycle behavior:** {cold / warm / background / killed behavior if non-default — otherwise "Inherits global Lifecycle defaults."}

**Parameter-dependent behavior:** {how parameters change what is shown — e.g., "if `resourceId` points to a private resource the viewer cannot access → not-found screen with 'Request access' CTA; if the resource was deleted → tombstone variant". Or "N/A — no parameters."}

**Shared surfaces invoked:** {names from the Shared Surfaces section — or "None."}
```

Repeat the block for every screen template in the inventory.

---

## Field-by-field guidance

- **Screen identifier / route** — use a screen name the team would recognize (`ResourceDetail`, `OnboardingStep2`), not an abstract placeholder. For parameterized screens, include the parameters in the identifier (`ResourceDetail(resourceId)`).
- **Navigation position** — state the tab or stack the screen lives in, and whether it is pushed, presented modally, shown as a sheet, or shown as a system alert. Reference the Navigation Model section — do not invent new containers.
- **Platform availability** — "iOS", "Android", or "both". If both but with materially different behavior, say "both (features differ — see Platform variations)".
- **Deep link** — the URL pattern if the screen is deep-linkable, plus a reference to how the Deep-Link Strategy handles the auth-gate case. If not deep-linkable, write "Not deep-linkable." — silent omission is not allowed.
- **Auth requirement** — "public" means no credentials needed; "authenticated" requires a signed-in user; "authenticated-with-permission" requires a specific scope or role — name it.
- **Sections / blocks** — describe each visible region by role and content, in the order it appears. Do not name the component, do not describe its visual appearance. Pull header / content / footer regions into separate entries when they serve separate roles.
- **Primary CTA** — the single action the screen is designed to invite. If the screen has no primary CTA (chrome, nav-container, auth-gate, error, splash, onboarding-slide, informational), classify it explicitly — do not leave blank.
- **Data dependencies** — what the screen needs to render. High-level only — "resource metadata, viewer permissions, child-item list". If a contract registry exists, cite endpoints by name.
- **Gestures supported** — the mobile-specific gesture contract. Pull-to-refresh, swipe-to-delete, long-press, pinch-to-zoom, swipe-between-tabs, haptic feedback triggered. Platform-default gestures (tap, nav-bar back) are assumed — list only screen-specific or non-default behaviors.
- **Orientation support** — portrait-only, landscape-only, or both. If the screen honors the app-level default without deviation, write "Follows app-level default." and do not restate the default.
- **Permissions involved** — by name from the Permissions Strategy table. If the screen depends on camera, write "camera". Do not re-describe the permission's prompting or denial behavior — that lives in the Permissions Strategy section.
- **Platform variations** — either enumerate divergences by platform, or write "Identical across platforms." Silent equivalence is not allowed (Rule 15). In-scope variations are behavioral (iOS sheet vs Android bottom sheet, swipe-back vs system back). Out-of-scope variations are aesthetic (specific system colors, typography tokens, animation timings).
- **States** — every row from the template. For every permission listed in the "Permissions involved" field, add a matching "Permission-denied — {permission name}" row. Use "N/A — {reason}" for rows that genuinely do not apply; do not drop rows.
- **Entry points** — enumerate the ways a user arrives at this screen: parent-screen navigation, deep link, push-notification tap, widget tap, Siri Shortcut / App Intent invocation, share-sheet target, cold start onto this route. This is how you prove the screen is reachable.
- **Exit points** — where the primary CTA lands, where back / swipe-back lands (which stack, which screen), and what happens when the app is backgrounded from this screen.
- **Lifecycle behavior** — per-screen deltas from the global Lifecycle Behavior section. Common deltas: upload continues via background task, biometric re-auth required on resume, timer cancelled on background. If the screen inherits the defaults, write "Inherits global Lifecycle defaults."
- **Parameter-dependent behavior** — the key field for preventing per-instance enumeration (Rule 1). Describe how each parameter changes behavior beyond trivial input variation. At minimum, screens targeting a named resource must document the not-found and unauthorized branches.
- **Shared surfaces invoked** — reference by name from the Shared Surfaces section of `MOBILE_IA.md`. Do not redescribe the surface.

---

## Common mistakes to avoid

- Enumerating concrete instances (`ResourceDetail for repo-A`, `ResourceDetail for repo-B`) instead of the single template (`ResourceDetail(resourceId)`).
- Writing code-shaped descriptions (`ResourceDetailViewModel` exposes `state: StateFlow<…>`). Those belong in per-unit SPECs (Rule 6).
- Describing visual design (colors, typography, spacing, animation curves, specific icons) anywhere in the blueprint (Rule 7).
- Re-describing the Permissions Strategy inline in a screen entry — only reference permissions by name (Rule 11).
- Re-describing a shared surface (e.g., in-app review prompt) inline — only reference it by name from Shared Surfaces (Rule 13).
- Leaving "Platform variations" blank or missing — every screen declares either divergences or "Identical across platforms" (Rule 15).
- Omitting Permission-denied rows for permissions the screen depends on, or dropping state-matrix rows entirely — use "N/A — {reason}" instead of silent omission (Rule 3).
- Writing literal UX copy ("Fork Repository", "Please enable camera access") instead of role descriptions (Rule 8).
- Introducing a screen inside a flow without adding it to the inventory first (Rule 12).
- Inventing a navigation container (a new tab, a new modal family) in a per-screen entry instead of referencing the Navigation Model section (Rule 9).
