# amux v2 — From-Scratch Vision

**This supersedes the "make v1 prettier" mindset.** v1 (Wave 0–3) made amux *competent 2025*. v2 makes it *the best terminal multiplexer interface ever built*. Steve Jobs proud.

## The goal — only two things matter

1. **Switch between sessions and into a session in <100ms with zero friction.**
2. **Working in a session feels alive — agent output streams in real-time, your keystrokes land instantly, you never wait.**

Everything else (animations, colors, gestures) serves these two goals. Anti-goals: don't ape iOS for its own sake, don't add features that don't help speed or aliveness.

---

## The four 100x improvements

### 1. Live tmux via WebSocket pty stream

**Today**: amux polls `GET /api/sessions/{name}/peek` every 3 seconds. Returns a snapshot of stripped tmux output as text. Rendered into a `<div>`. So you're always watching a 1.5-second-old corpse.

**v2**: WebSocket endpoint that streams **raw pty bytes** from the tmux pane, character-by-character, with proper ANSI color codes preserved. Frontend xterm.js consumes the stream. You see:
- Each character the agent types, appearing in real time
- Your own keystrokes echo back from tmux (input forwarding via WS)
- Spinners actually spin
- Progress bars actually fill
- Cursor blinks
- Colors are real

This is the single biggest improvement. It transforms amux from "monitoring dashboard" into "actual terminal".

**Implementation sketch** (Python side, add to `amux-server.py`):

```python
# tmux pty streaming via WebSocket
import asyncio, json
from threading import Thread

class TmuxStreamer:
    """One per session. Reads tmux pane via pipe-pane, dispatches to subscribers."""
    _streamers = {}  # session_name -> TmuxStreamer

    def __init__(self, name):
        self.name = name
        self.subscribers = set()
        self.proc = None
        self._start_pipe()

    def _start_pipe(self):
        # tmux pipe-pane outputs every byte from the pane to a fifo
        fifo = f"/tmp/amux-pipe-{self.name}.fifo"
        os.makedirs(os.path.dirname(fifo), exist_ok=True)
        if not os.path.exists(fifo): os.mkfifo(fifo)
        subprocess.run(["tmux", "pipe-pane", "-t", tmux_target(self.name),
                        "-O", f"cat > {fifo}"])  # -O overwrites previous pipe
        # Thread to read from fifo
        Thread(target=self._read_loop, args=(fifo,), daemon=True).start()

    def _read_loop(self, fifo):
        with open(fifo, "rb") as f:
            while True:
                chunk = f.read(4096)
                if not chunk: continue
                # Dispatch to all subscribers
                for sub in list(self.subscribers):
                    try: sub.send(chunk)
                    except: self.subscribers.discard(sub)

    @classmethod
    def get(cls, name):
        if name not in cls._streamers:
            cls._streamers[name] = cls(name)
        return cls._streamers[name]

# WebSocket route — use a lightweight WS shim since BaseHTTPServer doesn't do WS
# Option A: add `websockets` Python lib (single CDN-ish dep — add to requirements)
# Option B: roll a tiny WS handshake inline (~100 LOC, no deps, matches single-file religion)

# Recommend B for the religion. Sketch:
class WSConnection:
    def __init__(self, sock):
        self.sock = sock
        self.alive = True
    def send(self, data):
        # write a binary frame
        self.sock.send(self._frame(data))
    def recv(self):
        # read a frame
        ...

def handle_ws_session(name, ws):
    """GET /ws/sessions/{name} with Upgrade headers."""
    streamer = TmuxStreamer.get(name)
    streamer.subscribers.add(ws)
    # Send replay buffer (last ~2KB of tmux output so user sees context)
    initial = capture_pane_bytes(name, lines=200)
    ws.send(initial)
    # Read inbound (user keystrokes)
    try:
        while ws.alive:
            msg = ws.recv()
            if msg["type"] == "input":
                # forward to tmux via send-keys
                subprocess.run(["tmux", "send-keys", "-t", tmux_target(name),
                                "-l", msg["data"]])
            elif msg["type"] == "key":
                subprocess.run(["tmux", "send-keys", "-t", tmux_target(name),
                                msg["data"]])  # e.g. C-c
    finally:
        streamer.subscribers.discard(ws)
```

**Frontend** (replace `peek-body` `<div>` rendering with xterm.js Terminal):

```js
// Per focus mode entry: create xterm Terminal, wire WS
import { Terminal } from 'https://cdn.jsdelivr.net/npm/xterm@5.5.0/+esm';
import { FitAddon } from 'https://cdn.jsdelivr.net/npm/@xterm/addon-fit@0.10.0/+esm';

class LiveTerminal {
  constructor(container, sessionName) {
    this.term = new Terminal({
      cursorBlink: true,
      fontFamily: 'ui-monospace, SF Mono, Menlo, monospace',
      fontSize: 13,
      theme: {
        background: 'transparent',
        foreground: 'var(--label-primary)',
        // ... full iOS-tinted ANSI palette
      },
      allowTransparency: true,
      scrollback: 5000,
    });
    this.fit = new FitAddon();
    this.term.loadAddon(this.fit);
    this.term.open(container);
    this.fit.fit();
    this.connect(sessionName);

    // Forward typing
    this.term.onData(data => {
      this.ws.send(JSON.stringify({ type: 'input', data }));
    });
  }

  connect(name) {
    const wsUrl = location.origin.replace('http', 'ws') + '/ws/sessions/' + name + '?_token=' + AUTH_TOKEN;
    this.ws = new WebSocket(wsUrl);
    this.ws.binaryType = 'arraybuffer';
    this.ws.onmessage = (ev) => {
      if (typeof ev.data === 'string') this.term.write(ev.data);
      else this.term.write(new Uint8Array(ev.data));
    };
    this.ws.onclose = () => setTimeout(() => this.connect(name), 1000);
  }
}
```

**Result**: text appears character-by-character, ANSI colors live, you can type and tmux echoes back. **This alone is the 100x.**

---

### 2. Overview cards = live mini-terminals + magnetic hover

**Today**: session cards show `name + path + last activity + tags`. Static. Boring. You have to click to see what's actually happening.

**v2**: each card is a **live mini-terminal showing the last 8-12 lines of tmux output**, updating in real-time via the same WebSocket. The card *is* the session, not a label for it.

Behavior:
- **Idle state** (mouse far away): card shows mini-terminal text + status dot. Subtle.
- **Hover** (mouse over card): card scales 1.04 with spring physics (`AmuxSprings.snappy`). The mini-terminal grows. A subtle "type here →" hint fades in.
- **Hover + start typing**: input is captured by the hovered card directly. You don't need to click. Cursor appears inline at the bottom of the mini-terminal. Enter sends to that session. Esc releases focus.
- **Click**: enters full focus mode (View Transitions API morph — the card grows to fill the right column on desktop, or to full-screen on mobile).
- **Agent waiting for input**: the card border pulses with `--tint-blue` (subtle, ~2s cycle). When agent finishes a turn and shows `❯ `, the border lights up. When you start typing, the pulse stops.
- **Agent error/rate-limit**: border switches to `--tint-orange` solid (no pulse — calmer, says "look here").
- **Long-running task** (agent running): top-right corner shows a tiny spinner with elapsed time.

Layout:
- **Desktop overview**: grid of cards, ~320×180px each, ≥3 per row at ≥1440px wide
- **Desktop focus mode**: 320px sidebar of stacked mini-cards (compact, 80px tall) + main panel with live terminal
- **Mobile overview**: full-width cards stacked, ~140px tall each, swipe gestures available
- **Mobile focus**: full-screen, mini-cards collapse to a horizontal scroll strip at top showing other running sessions (tap to switch)

**Implementation sketch**:

```html
<div class="session-card" data-session="gel-astro" data-state="idle">
  <header class="card-header">
    <span class="card-name">gel-astro</span>
    <span class="card-status-dot" data-status="waiting"></span>
    <span class="card-elapsed" hidden>2m 14s</span>
  </header>
  <div class="card-mini-term" data-mini-term="gel-astro">
    <!-- xterm.js writes here, ~10 rows, 60 cols truncated -->
  </div>
  <div class="card-hint">type to send →</div>
</div>
```

```css
.session-card {
  background: var(--bg-layer-1);
  border-radius: var(--r-md);
  border: 1px solid var(--sep-non-opaque);
  padding: var(--s-3);
  transition: transform var(--duration-fast) var(--ease-emphasized),
              box-shadow var(--duration-fast) var(--ease-standard);
  cursor: pointer;
}
.session-card:hover {
  transform: scale(1.04);
  box-shadow: var(--shadow-md);
  z-index: 1;
}
.session-card[data-state="hover-typing"] {
  outline: 2px solid var(--tint-blue);
  outline-offset: 2px;
}
.card-status-dot[data-status="waiting"] {
  animation: pulse-attention 2s ease-in-out infinite;
  background: var(--tint-blue);
}
@keyframes pulse-attention {
  0%, 100% { opacity: 0.4; transform: scale(1); }
  50% { opacity: 1; transform: scale(1.3); }
}
.session-card[data-attention="waiting"] {
  border-color: color-mix(in srgb, var(--tint-blue) 60%, transparent);
  animation: card-pulse 2.5s ease-in-out infinite;
}
@keyframes card-pulse {
  0%, 100% { box-shadow: 0 0 0 0 color-mix(in srgb, var(--tint-blue) 0%, transparent); }
  50% { box-shadow: 0 0 0 6px color-mix(in srgb, var(--tint-blue) 20%, transparent); }
}
.card-mini-term {
  height: 140px;
  background: var(--bg-base);
  border-radius: var(--r-sm);
  padding: var(--s-2);
  font: 11px var(--font-mono);
  color: var(--label-secondary);
  overflow: hidden;
  position: relative;
}
.card-hint {
  position: absolute;
  bottom: var(--s-2);
  right: var(--s-3);
  font: var(--text-caption2) var(--font-sans);
  color: var(--label-tertiary);
  opacity: 0;
  transition: opacity var(--duration-fast);
}
.session-card:hover .card-hint { opacity: 1; }
```

```js
// Wire hover-to-type: when mouse over card, capture global keystrokes
let _hoveredCard = null;
document.addEventListener('mouseover', (e) => {
  const card = e.target.closest('.session-card');
  if (card !== _hoveredCard) {
    _hoveredCard?.removeAttribute('data-state');
    _hoveredCard = card;
    if (card) card.dataset.state = 'hover';
  }
});
document.addEventListener('keydown', (e) => {
  if (!_hoveredCard) return;
  // ignore modifier keystrokes alone, IME composition, etc.
  if (e.ctrlKey || e.metaKey || e.altKey) return;
  if (e.key.length > 1 && e.key !== 'Backspace' && e.key !== 'Enter') return;
  _hoveredCard.dataset.state = 'hover-typing';
  // Route to that session's WS
  const name = _hoveredCard.dataset.session;
  const ws = _liveTerms[name]?.ws;
  if (!ws) return;
  if (e.key === 'Enter') ws.send(JSON.stringify({ type: 'key', data: 'Enter' }));
  else if (e.key === 'Backspace') ws.send(JSON.stringify({ type: 'key', data: 'BSpace' }));
  else ws.send(JSON.stringify({ type: 'input', data: e.key }));
  e.preventDefault();
});
```

**Result**: zero-click "scan all my sessions" with the most powerful visual signal possible (the actual output). Type without clicking. Pulse pulls your eye to whatever needs you.

---

### 3. Cmd+K command palette + keyboard nav

**Today**: switching sessions requires moving the mouse to a card and clicking.

**v2**: keyboard-first navigation, Raycast/Linear style.

- **Cmd+K** (Ctrl+K on Windows) anywhere → fuzzy switcher palette
  - Lists all sessions: running on top, stopped below
  - Type to filter (sub-string match across name, path)
  - Arrow up/down to navigate, Enter to switch, Esc to dismiss
  - Recently used sessions ranked first (à la VSCode/Cmd+P)
- **Cmd+1..9** → jump to nth running session
- **Cmd+]** / **Cmd+[** → next/prev session (cycle through running)
- **Cmd+W** → back to overview from focus mode
- **Cmd+Enter** → send (when input is focused)
- **/** in any input → slash command menu (already designed in v1, polish in v2)
- **Esc** → close any sheet/menu/palette

The palette is the killer feature. Apple devs live in Cmd+K.

```html
<div id="cmd-palette" class="cmd-palette" hidden>
  <div class="palette-backdrop"></div>
  <div class="palette-panel surface-material">
    <input class="palette-input input-ios" placeholder="Switch session…" autofocus>
    <ul class="palette-results"></ul>
    <footer class="palette-footer">
      <span><kbd>↑↓</kbd> navigate</span>
      <span><kbd>↵</kbd> open</span>
      <span><kbd>esc</kbd> close</span>
    </footer>
  </div>
</div>
```

Spring-physics entrance (`AmuxSprings.snappy`), backdrop fade, focus auto-grabbed. Width 600px on desktop, full-width-with-margins on mobile. Position fixed top: 20vh.

---

### 4. Mobile focus mode = drag-detent sheet

**Today** (v1): focus mode is full-screen takeover on mobile. Good, but you lose context of the session list.

**v2**: focus mode on mobile is a **drag-detent sheet** (Apple Maps / Apple Music Now Playing pattern):
- **Peek** (default, 30vh): handle + session header + 4 lines of terminal. Always visible. Quick context.
- **Half** (60vh): more terminal + input dock visible.
- **Full** (100vh): standard focus mode. Drag down = collapse one detent.
- Velocity-based snap: drag fast down = collapse 2 detents.

Behind the sheet: the session list stays visible (dimmed slightly). Tap a different card = sheet animates to new session (no full-screen flash).

When user starts typing in the input, sheet auto-promotes to full.

This is the "feels like a real iOS app" detail.

Use the `~120 LOC starter` from `research/ios-panel-libraries.md` directly. Already designed.

---

## What we keep from v1

- All design tokens, primitives (.btn-ios, .input-ios, .chip-ios, .surface-*)
- `window.AmuxSprings`, Motion One, Lucide icons
- The new top header / pill tabs (works well on overview)
- The /paste page (separate, already polished)
- The focus-shell HTML/CSS scaffolding (will be enhanced not replaced)
- Slash menu, action sheets, edge-swipe (foundations laid in v1)

## What v2 replaces wholesale

- `#peek-body` `<div>` → xterm.js LiveTerminal instance
- 3s polling → WebSocket stream
- Static metadata cards → live mini-terminal cards
- Click-only interaction → hover-to-type + Cmd+K + keyboard shortcuts
- Mobile full-screen takeover → drag-detent sheet

## Migration plan

Sequential, can't be parallelized (single-file constraint + shared WS infrastructure):

| # | Phase | Risk | Time | Subagent or self |
|---|---|---|---|---|
| A | WS backend (server.py) | Medium — touches Python core | 60-90 min | Self (I want to verify carefully) |
| B | LiveTerminal frontend (xterm.js wire-up) | Low | 30-45 min | Self or subagent |
| C | Replace peek-body with LiveTerminal in focus-shell | Low | 20 min | Subagent |
| D | Mini-terminal in overview cards (per-card LiveTerminal) | Medium — perf concerns with many WS | 45-60 min | Subagent |
| E | Hover-to-type behavior | Low | 30 min | Subagent |
| F | Pulse-when-waiting (parse tmux output for `❯ ` prompt detection) | Low | 30 min | Subagent |
| G | Cmd+K palette | Low | 45 min | Subagent |
| H | Keyboard shortcuts (Cmd+1..9, Cmd+], etc.) | Low | 30 min | Subagent |
| I | Mobile drag-detent sheet (replace full-screen takeover) | Medium — gesture math | 60 min | Subagent |
| J | Final polish + recording walkthrough | — | 30 min | Self |

Total ~6-8 hours of subagent work, possible to parallelize phases C onward after A+B land.

## Anti-patterns we're explicitly avoiding

- **Full-screen takeover on desktop** — kills "see other sessions" context. Use split or drag-sheet.
- **Bottom tab bar in focus mode** — wastes space. Hide entirely.
- **Animated tab content transitions** — Apple never does this. Chrome animates, content swaps instantly.
- **Glassmorphism on content** — only on chrome (header, sheets, dock).
- **3-line headers** — single 44pt row, always.
- **iOS-styling for desktop browser** — desktop gets desktop UX (sidebar, Cmd+K, hover). Mobile gets iOS UX. They're different products served from the same code.
- **Notifications/badges everywhere** — only pulse what truly needs attention (agent waiting).
- **Auto-scroll terminal to bottom unconditionally** — preserve user scroll position when they're reading scrollback.

## Self-test (post-implementation acceptance)

- [ ] Type a character in any focused session → appears in tmux within 50ms
- [ ] Switch sessions via Cmd+K → fully rendered in <100ms
- [ ] Hover a card in overview → see live preview update in real-time
- [ ] Hover + type → text goes to that session, no click needed
- [ ] Agent finishes a turn → card border pulses blue
- [ ] On iPhone, drag focus sheet handle → snaps to peek/half/full
- [ ] On iPhone, keyboard appears → sheet stays above keyboard, terminal still visible
- [ ] Cmd+W returns to overview with view-transition animation
- [ ] Esc dismisses palette/menu without dismissing focus
- [ ] WS reconnects automatically if dropped
- [ ] No JS errors in console during normal flow
- [ ] CPU/memory stable with 6+ live mini-terminals
- [ ] Page load to first session interactive: <500ms on cable

If all 12 pass: ship.

## Open questions for the user (defer answering until needed)

1. Mini-terminal rendering: full xterm.js per card (~1MB JS, heavy with 10 cards) vs lightweight ANSI-to-HTML render? Vote: **xterm.js for focus mode, lightweight for cards** — best of both.
2. Hover-to-type: should it require a brief hover dwell (300ms) to avoid accidental keystrokes? Vote: **yes, 200ms dwell**.
3. Pulse: should it be sound-enabled (subtle ping when agent finishes)? Default off, opt-in.
4. Drag-detent on iPad — same as iPhone, or use desktop split layout? Vote: **iPad in portrait = drag detent, landscape = split.**

---

## TL;DR

Make tmux LIVE (WS + xterm.js), make cards LIVE (mini-terminal + hover-type + pulse), add Cmd+K, add drag-detents on mobile. Everything else is in service of these four.

This is the spec. After 18:00 reset, dispatch subagents per the migration plan. Until then, build phase A (WS backend) myself.
