# amux Apple-grade Redesign — Design Spec

**Authoritative reference for all redesign subagents.** Any deviation requires explicit reason in commit message.

## Hard constraints (DO NOT VIOLATE)

1. **Single-file rule** — all CSS/JS stays inline in `amux-server.py`. No new external files for runtime assets. Adding CDN `<link>`/`<script>` tags is OK.
2. **Mobile-first** — 375px-wide must look first-class. 44×44 minimum touch targets. Use `env(safe-area-inset-*)` for screen edges.
3. **SSE realtime** — never break `_sse_events`, `connectSSE`, `_onClientResume`. UI changes must continue to receive live updates within ~2s.
4. **API contract** — never rename/remove existing endpoints. iOS app depends on them.
5. **Syntax check** — run `python3 -c "import ast; ast.parse(open('amux-server.py').read())"` after every edit.
6. **No build step** — vanilla JS, no JSX/TS/Svelte. Tailwind/Petite-Vue not allowed (we decided pure CSS + Motion One).

## Design philosophy

> "If Apple shipped amux as part of macOS, what would it look like?"

Concretely: iOS 18 / macOS Tahoe vibe. Liquid Glass materials. Spring-physics motion. Generous whitespace. Sharp typography. Subtle depth. Zero AI-slop (gradients, glassmorphism overuse, rainbow gradients, busy icons).

## Design tokens (CSS variables)

Add to `:root` of dashboard CSS:

```css
:root {
  /* ── Typography ── */
  --font-sans: -apple-system, BlinkMacSystemFont, "SF Pro Text", "SF Pro Display", "Segoe UI", system-ui, sans-serif;
  --font-mono: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
  --font-rounded: ui-rounded, "SF Pro Rounded", -apple-system, system-ui, sans-serif;

  /* iOS Dynamic Type — relative to root font-size (16px default) */
  --text-caption2: 0.6875rem;  /* 11 */
  --text-caption1: 0.75rem;    /* 12 */
  --text-footnote: 0.8125rem;  /* 13 */
  --text-subhead:  0.9375rem;  /* 15 */
  --text-callout:  1rem;       /* 16 */
  --text-body:     1.0625rem;  /* 17 — iOS default body */
  --text-headline: 1.0625rem;  /* 17 + bold */
  --text-title3:   1.25rem;    /* 20 */
  --text-title2:   1.375rem;   /* 22 */
  --text-title1:   1.75rem;    /* 28 */
  --text-large:    2.125rem;   /* 34 */

  --weight-regular: 400;
  --weight-medium:  500;
  --weight-semibold: 600;
  --weight-bold:    700;

  /* ── Spacing — 4pt grid ── */
  --s-1: 0.25rem; --s-2: 0.5rem; --s-3: 0.75rem;
  --s-4: 1rem;    --s-5: 1.25rem; --s-6: 1.5rem;
  --s-8: 2rem;    --s-10: 2.5rem; --s-12: 3rem;
  --s-16: 4rem;   --s-20: 5rem;

  /* ── Radii (iOS standards) ── */
  --r-xs: 6px;   /* tags, chips */
  --r-sm: 10px;  /* buttons */
  --r-md: 14px;  /* cards, inputs */
  --r-lg: 22px;  /* sheets, large surfaces */
  --r-xl: 28px;  /* tab bar pills */
  --r-full: 9999px;

  /* ── Color (iOS system colors, dark default) ── */
  /* Backgrounds — layered, like UIView */
  --bg-base:        #000000;          /* base — true black for OLED */
  --bg-layer-1:     #1c1c1e;          /* cards/surfaces */
  --bg-layer-2:     #2c2c2e;          /* nested surfaces */
  --bg-layer-3:     #3a3a3c;          /* tertiary */
  --bg-grouped:     #1c1c1e;          /* iOS grouped table bg */
  --bg-tinted:      rgba(120,120,128,0.16);  /* subtle fills */

  /* Materials — backdrop-filter blur, translucent */
  --mat-thick:  rgba(28,28,30,0.78);
  --mat-regular: rgba(28,28,30,0.6);
  --mat-thin:   rgba(28,28,30,0.4);
  --mat-ultra:  rgba(28,28,30,0.2);

  /* Text */
  --label-primary:    rgba(255,255,255,1.0);
  --label-secondary:  rgba(235,235,245,0.6);
  --label-tertiary:   rgba(235,235,245,0.3);
  --label-quaternary: rgba(235,235,245,0.16);

  /* Tints (iOS system tints) */
  --tint-blue:   #0a84ff;    /* primary */
  --tint-green:  #30d158;
  --tint-orange: #ff9f0a;
  --tint-red:    #ff453a;
  --tint-purple: #bf5af2;
  --tint-pink:   #ff375f;
  --tint-teal:   #64d2ff;
  --tint-yellow: #ffd60a;
  --tint-indigo: #5e5ce6;

  /* Separators */
  --sep-opaque:    rgba(56,56,58,1);
  --sep-non-opaque: rgba(84,84,88,0.6);  /* Apple UIColor separator dark — was 0.34, too faint */

  /* Focus ring (keyboard a11y) */
  --focus-ring: 0 0 0 4px color-mix(in srgb, var(--tint-blue) 30%, transparent);

  /* ── Motion (Motion One uses these) ── */
  /* Spring presets live in JS, NOT CSS — see window.AmuxSprings below. */

  --ease-emphasized: cubic-bezier(0.32, 0.72, 0, 1); /* Apple UIView default — NOT Material 3 */
  --ease-standard:   cubic-bezier(0.4, 0, 0.2, 1);

  --duration-instant: 100ms;
  --duration-fast:    200ms;
  --duration-medium:  300ms;
  --duration-slow:    500ms;

  /* ── Shadows (iOS-tasteful, never flashy) ── */
  --shadow-sm:  0 1px 2px rgba(0,0,0,0.3);
  --shadow-md:  0 4px 16px rgba(0,0,0,0.4);
  --shadow-lg:  0 12px 40px rgba(0,0,0,0.5);
  --shadow-sheet: 0 -8px 32px rgba(0,0,0,0.6);

  /* ── Z-layers ── */
  --z-base: 0;
  --z-elevated: 10;
  --z-sticky: 100;
  --z-modal-backdrop: 900;
  --z-modal: 1000;
  --z-toast: 1100;
  --z-tooltip: 1200;
}

/* Light theme is CLASS-BASED via body.light — NOT @media (prefers-color-scheme: light).
   The app has a manual theme toggle that persists in localStorage. Do not introduce
   @media (prefers-color-scheme: light) blocks — they will diverge from the user toggle. */
body.light {
  --bg-base: #ffffff;
  --bg-layer-1: #f2f2f7;
  --bg-layer-2: #ffffff;
  --bg-layer-3: #e5e5ea;
  --bg-grouped: #f2f2f7;
  --bg-tinted: rgba(120,120,128,0.12);

  --mat-thick:  rgba(255,255,255,0.78);
  --mat-regular: rgba(255,255,255,0.6);
  --mat-thin:   rgba(255,255,255,0.4);
  --mat-ultra:  rgba(255,255,255,0.2);

  --label-primary:   #000000;
  --label-secondary: rgba(60,60,67,0.6);
  --label-tertiary:  rgba(60,60,67,0.3);
  --label-quaternary: rgba(60,60,67,0.18);

  --sep-opaque:    rgba(198,198,200,1);
  --sep-non-opaque: rgba(60,60,67,0.29);

  /* Light-mode shadows are deliberately lighter (0.06–0.18 vs dark's 0.3–0.6)
     because shadow on light bg reads as dirt at dark-mode opacities. */
  --shadow-sm:    0 1px 2px rgba(0,0,0,0.06);
  --shadow-md:    0 4px 16px rgba(0,0,0,0.10);
  --shadow-lg:    0 12px 40px rgba(0,0,0,0.14);
  --shadow-sheet: 0 -8px 32px rgba(0,0,0,0.18);
}
```

## Critical: legacy theme coexistence

`amux-server.py` carries **two parallel CSS variable systems** during the migration:

| System | Vars | Used by |
|---|---|---|
| **Legacy** (`:root` at top of style) | `--bg`, `--card`, `--border`, `--text`, `--dim`, `--accent`, `--green`, `--red`, `--yellow`, `--cyan` | All pre-Wave-0 components (~hundreds of references) |
| **Apple-grade** (Wave 0 `:root`) | `--bg-base`, `--bg-layer-{1,2,3}`, `--bg-tinted`, `--mat-*`, `--label-*`, `--sep-*`, `--tint-*`, `--focus-ring`, `--shadow-*` | New redesigned views and primitives |

Both systems have their own `body.light` override block (also coexisting). Property
namespaces don't overlap, so there's no shadowing — but they ALSO don't talk to
each other. **Implication for redesign work:**

- When you redesign a view, switch its CSS to the new tokens (`var(--bg-layer-1)`,
  `var(--label-primary)`, etc.). Do NOT introduce new uses of `var(--bg)`/`var(--card)`/etc.
- Do NOT delete or rewrite the legacy `:root` or legacy `body.light` block — other
  views still depend on them. Eventual removal will be a dedicated commit.
- If your view mixes legacy chrome with new content and the layered surfaces don't
  match across the seam, that's a known migration artifact — flag it in your commit
  message but don't fix the legacy chrome.

## Required CDN additions

CDN versions are pinned — never use `@latest`. If you regenerate the script tag, copy these exact lines:

```html
<!-- Motion One — spring physics, ~3.8KB -->
<script src="https://cdn.jsdelivr.net/npm/motion@10.18.0/dist/motion.min.js"></script>

<!-- Lucide icons — clean, modern (only load icons we use, not the whole pack) -->
<script src="https://unpkg.com/lucide@0.460.0/dist/umd/lucide.min.js"></script>

<!-- Foundation: alias Motion lowercase + expose AmuxSprings preset object -->
<script>
  // Motion One UMD exposes window.Motion (capital M); alias to lowercase for convenience.
  // If window.motion is ever undefined, window.Motion is the canonical source.
  if (typeof window.motion === 'undefined' && typeof window.Motion !== 'undefined') {
    window.motion = window.Motion;
  }
  // Spring presets — JS object, NOT CSS variables. CSS vars containing JSON strings
  // are not cleanly parseable (the CSS string delimiters fight JSON.parse).
  window.AmuxSprings = {
    snappy:  { type: 'spring', stiffness: 700, damping: 35 },
    bouncy:  { type: 'spring', stiffness: 400, damping: 22 },
    smooth:  { type: 'spring', stiffness: 220, damping: 28 },
    gentle:  { type: 'spring', stiffness: 120, damping: 20 }
  };
</script>
```

**Canonical motion pattern** — subagents use `window.AmuxSprings` directly:

```js
motion.animate(el, { scale: 1.05 }, window.AmuxSprings.bouncy);
motion.animate(el, { opacity: [0, 1] }, window.AmuxSprings.smooth);

// Honor prefers-reduced-motion in Motion One (it ignores the CSS media query):
const prefersReducedMotion = matchMedia('(prefers-reduced-motion: reduce)').matches;
motion.animate(el, { opacity: [0, 1] },
  prefersReducedMotion ? { duration: 0.01 } : window.AmuxSprings.smooth);
```

## Component primitives (build once in Wave 0, reuse everywhere)

**Naming:** the new primitives are namespaced `-ios` (e.g. `.btn-ios`, `.input-ios`, `.chip-ios`).
This avoids collision with the legacy `.btn`, `.input`, `.chip` classes still used by old call sites.
`.surface` stays un-suffixed (no legacy conflict). When you write a redesigned view, always use
the `-ios` classes; the bare versions are reserved for legacy components until they're migrated.

### `.surface` — card / sheet base
```css
.surface {
  background: var(--bg-layer-1);
  border-radius: var(--r-md);
  border: 1px solid var(--sep-non-opaque);
}
.surface-elevated { background: var(--bg-layer-2); box-shadow: var(--shadow-md); }
.surface-material { background: var(--mat-regular); backdrop-filter: blur(40px) saturate(180%);
                    -webkit-backdrop-filter: blur(40px) saturate(180%); }
```

### `.btn-ios` — tap-friendly
```css
.btn-ios {
  min-height: 44px;
  padding: 0 var(--s-4);
  border-radius: var(--r-sm);
  font: var(--weight-semibold) var(--text-callout)/1 var(--font-sans);
  display: inline-flex; align-items: center; gap: var(--s-2);
  background: var(--bg-tinted); color: var(--label-primary);
  border: none; cursor: pointer; user-select: none;
  -webkit-tap-highlight-color: transparent;  /* kills iOS Safari grey flash */
  transition: background var(--duration-fast) var(--ease-standard),
              transform var(--duration-instant) var(--ease-standard);
}
.btn-ios:hover { background: var(--bg-layer-3); }
.btn-ios:active { transform: scale(0.96); }
.btn-ios:focus-visible { outline: none; box-shadow: var(--focus-ring); }
.btn-ios-primary { background: var(--tint-blue); color: white; }
.btn-ios-primary:hover { background: color-mix(in srgb, var(--tint-blue) 88%, white); }
.btn-ios-destructive { color: var(--tint-red); }
.btn-ios-ghost { background: transparent; }
```

### `.input-ios` / `.textarea-ios`
```css
.input-ios, .textarea-ios {
  width: 100%; min-height: 44px;
  padding: var(--s-3) var(--s-4);
  background: var(--bg-layer-1);
  border: 1px solid var(--sep-non-opaque);
  border-radius: var(--r-md);
  font: var(--text-body) var(--font-sans);
  color: var(--label-primary);
  transition: border-color var(--duration-fast) var(--ease-standard),
              background var(--duration-fast) var(--ease-standard);
}
.input-ios:focus, .textarea-ios:focus {
  outline: none;
  border-color: var(--tint-blue);
  background: var(--bg-layer-2);
  box-shadow: 0 0 0 4px color-mix(in srgb, var(--tint-blue) 20%, transparent);
}
```

### `.chip-ios` — tags, status badges
```css
.chip-ios {
  padding: 2px var(--s-2);
  border-radius: var(--r-xs);
  font: var(--weight-medium) var(--text-caption1) var(--font-sans);
  background: var(--bg-tinted); color: var(--label-secondary);
  text-transform: none;  /* never UPPERCASE — un-Apple */
}
.chip-ios-success { background: color-mix(in srgb, var(--tint-green) 18%, transparent); color: var(--tint-green); }
.chip-ios-warning { background: color-mix(in srgb, var(--tint-orange) 18%, transparent); color: var(--tint-orange); }
.chip-ios-info    { background: color-mix(in srgb, var(--tint-blue) 18%, transparent);   color: var(--tint-blue); }
.chip-ios-danger  { background: color-mix(in srgb, var(--tint-red) 18%, transparent);    color: var(--tint-red); }
```

### `.sheet` — iOS modal sheet
- Presented from bottom (mobile) / centered (desktop)
- Backdrop: `rgba(0,0,0,0.5)` + `backdrop-filter: blur(20px)`
- Sheet itself: `--mat-thick` + `--r-lg` top corners, drag handle indicator
- Animations: spring-bouncy entrance, spring-snappy dismiss
- Swipe-to-dismiss on mobile

### `.nav-tab-bar` — bottom tab bar (mobile) / top tabs (desktop)
- Mobile: fixed bottom, `--mat-thick` background, safe-area aware
- Desktop: top, no fixed positioning, pill-style active state
- Active tab: `--tint-blue` + spring scale 1.05

## Anti-patterns (the critic will flag these)

Anti-patterns apply to **NEW** code you write. Do not rewrite existing legacy CSS
unless your view explicitly touches it — scope creep is its own critic flag.

- ❌ UPPERCASE labels (unless logo-ish acronyms)
- ❌ `border-radius: 4px` everywhere — use semantic radii
- ❌ `transition: all` — explicit properties only
- ❌ Generic shadows (`0 2px 4px rgba(0,0,0,0.1)`) — use design tokens
- ❌ Emojis as icons — use Lucide
- ❌ Two-color gradients without purpose — flat is better
- ❌ Rainbow indicators / linear-gradient cycling — instant AI-slop tell
- ❌ Heavy `font-weight: 800/900` — caps at 700
- ❌ `min-width` smaller than touch target on mobile
- ❌ Dropdown `<select>` without custom styling on mobile
- ❌ `text-shadow` on body text — only allow for hero display titles
- ❌ `cursor: pointer` on non-interactive elements (divs, cards) — pointer is for real links/buttons only
- ❌ Lucide default 2px stroke — always set 1.5px to match SF Symbols (foundation sets this globally)
- ❌ `@media (prefers-color-scheme: light)` — use `body.light` selector (manual toggle)
- ❌ Hardcoded spring `{ stiffness, damping }` in `motion.animate(...)` — use `window.AmuxSprings.*`

## Critic scoring (0-10 per view)

Reviewer scores each redesigned view across 6 dimensions, each 0-10. Mean ≥ 9 to merge.

| Dim | What it measures |
|---|---|
| Visual hierarchy | Eye flow, primary action obvious, no flat sea of text |
| Spacing rhythm | 4pt grid respected, consistent gutters, breathable |
| Motion | Spring physics where appropriate, 60fps, no jitter |
| Type | SF stack, proper weights, line-height correct |
| Material | Glass/blur used purposefully, depth obvious |
| Touch | 44pt targets, safe areas respected, gestures responsive |

Critic also flags:
- API breakage
- SSE breakage
- Mobile overflow at 375px
- Light-mode contrast failures (WCAG AA)

## Per-view checklist (apply when redesigning each)

- [ ] Loads with no JS errors (devtools console clean)
- [ ] Mobile @ 375×667 — no horizontal scroll, all targets ≥44pt
- [ ] Desktop @ 1440×900 — uses generous whitespace, doesn't feel cramped
- [ ] Dark + light mode both work
- [ ] Empty state designed (not just "no items")
- [ ] Loading state designed (skeletons or spinners with `motion`)
- [ ] Error state designed (subtle, dismissible)
- [ ] Keyboard nav works (Tab order, Esc closes modals)
- [ ] SSE updates animate in (not pop)
- [ ] First paint <500ms on cable

## Subagent task template

Every designer subagent gets prompted with:

```
You are redesigning the {VIEW_NAME} view of amux to Apple-grade quality.

Context: amux is a single-file Python web app (amux-server.py, ~37k lines).
The {VIEW_NAME} view is defined around lines {LINE_RANGE}.

REFERENCE FILES (read these first):
- /Users/sandervm/amux-redesign/DESIGN_SPEC.md — the spec, follow exactly
- /Users/sandervm/amux-redesign/.claude/rules/*.md — hard constraints
- /Users/sandervm/amux-redesign/CLAUDE.md — project conventions

DESIGN TOKENS: all needed CSS variables exist in :root already (Wave 0).
Use var(--*-*) — never hardcode colors/spacing/radii.

WORKFLOW:
1. Read the current view's HTML+CSS+JS in amux-server.py (use grep to find by ID)
2. Take a baseline screenshot via chrome-devtools MCP (already configured)
3. Plan the redesign — list 5-10 concrete changes
4. Apply changes (Edit tool, surgical)
5. python3 -c "import ast; ast.parse(open('amux-server.py').read())"  — must pass
6. Reload page, screenshot, compare
7. Self-critique against the 6 critic dimensions
8. Iterate until self-score ≥9
9. Commit with message: "redesign({view}): {one-line summary}"
10. Push to origin/main

YOU MAY NOT:
- Touch any view other than {VIEW_NAME}
- Change API endpoints
- Break SSE (don't modify _sse_events, connectSSE, _onClientResume)
- Add files outside amux-server.py
- Use Tailwind, React, Vue, etc.

OUTPUT: a final summary with score, list of changes, before/after screenshot paths.
```

## File map (views → approx line range in amux-server.py)

**Line numbers are approximate and drift after every Wave commit.** Always grep for the
anchor string (e.g. `<div id="board-view"`) to locate the current position before editing.
The "Approx lines" column is for orientation only.

| View | Anchor in HTML | Approx lines |
|---|---|---|
| session-view | `<div id="session-view">` | 11651–11680 |
| board-view | `<div id="board-view"` | 11682–11700 |
| calendar-view | `<div id="calendar-view"` | 11702–11705 |
| scheduler-view | `<div id="scheduler-view"` | 11706–11718 |
| files-view | `<div id="files-view"` | 11719–11831 |
| logs-view | `<div id="logs-view"` | 11832–11868 |
| notes-view | `<div id="notes-view"` | 11869–11927 |
| crm-view | `<div id="crm-view"` | 11929–11990 |
| map-view | `<div id="map-view"` | 11991–12070 |
| metrics-view | `<div id="metrics-view"` | 12101–12117 |
| torrents-view | `<div id="torrents-view"` | 12118–12136 |
| terminal-view | `<div id="terminal-view"` | 12137–12162 |
| browser-view | `<div id="browser-view"` | 12163–12182 |
| graph-view | `<div id="graph-view"` | 12183–12237 |
| journal-view | `<div id="journal-view"` | 12238–12307 |
| habits-view | `<div id="habits-view"` | 12334–12500 |
| top nav / chrome | `<header>` and `.nav-tab-bar` | search for "tab-bar"/header |
| modals (global) | search `class="modal"` and `id="*-modal"` | scattered |
| /paste page | search `GET /paste` route in Python | ~31435 |

CSS lives in `<style>` blocks between roughly lines 7770 and 11400. Tokens go at top of `<style>`.

## Subagent coordination

- **One designer at a time** holds the file lock (because single-file).
- **Critics + research subagents** run in parallel (read-only operations).
- **Orchestrator** (Claude in main session) tracks queue, picks next designer when current commits.
- Concurrency: 1 designer + 4+ critics/researchers = 5+ active.
