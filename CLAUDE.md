# CLAUDE.md

Guidance for Claude Code working in this repo.

## What this is

**tarmac** — a personal developer dashboard. The entire app is ONE self-contained
file, `index.html` (inline CSS + JS, vanilla, no framework). It must run when opened
directly via `file://` as a Chrome homepage / pinned tab. `README.md` is user docs;
`tarmac-config.example.json` is an importable example backup.

## Golden rules (do not break these)

- **Keep it one file.** Do not split `index.html` into separate `.js`/`.css`, and do
  not add a build step, bundler, npm deps, or `<script type="module">`. ES modules and
  most `fetch()` calls fail under `file://` — that's why everything is inline.
- **No external network / CDN.** Only bookmark links may hit the network. No analytics,
  web fonts, or CDN scripts. The app must work fully offline.
- **The localStorage key is `tarmac:state`.** Don't rename it — it orphans user data.
- **Schema changes:** update `normalizeState()` (and `normalizeDaily()` for day-scoped
  data) so old/partial JSON still loads with sane defaults, and bump `STATE_VERSION`.
  A bad import must never wipe state (see the `looksLikeState` guard).
- **All state changes go through `mutate(fn, opts)`** — it saves (debounced) and
  re-renders. Never touch `localStorage` outside `saveNow()`.
- **Escape everything user-supplied** with `escapeHtml()` inside template strings, and
  run URLs through `safeUrl()`. There is no framework auto-escaping here.
- **Renders must be focus-safe and idempotent.** `render()` can fire at any time
  (mutations, the 1s tick, midnight rollover). Build persistent inputs once in
  `mount()`; in `render()` only refresh dynamic regions and never clobber an input the
  user is typing in (see the standup applet's `document.activeElement` guard). When
  saving on keystroke, use `mutate(fn, { render: false })`.
- **Match the surrounding style:** small functions, clear names, comments at the existing
  density. No golf.

## Navigating the file for surgical edits

`index.html` is large but greppable. **Prefer `Grep` over reading the whole file.**
Start from the `FILE MAP` comment at the top of the `<script>`, then jump with anchors:

- **Edit an applet** → grep its id, e.g. `id: "deepwork"` (jumps to its
  `registerApplet({...})`). Ids: `clock quota todo deepwork tally capacity workday
  standup blockers snippets decisions bookmarks`.
- **A CSS rule** → grep the class (e.g. `.timer`, `.bm-tile`, `.quota`, `.spark7`).
- **Applet helpers** by name: `computeQuota`, `editQuota`, `moveTodo`, `wireTodoDrag`,
  `timerStart`, `openStandupCopy`, `editSnippet`, `addDecision`, `bookmarkTile`,
  `linkFields`.
- **Core seams**: `const CONFIG`, `normalizeState`, `mutate`, `registerApplet`,
  `renderGrid`, `day(`, `openDialog`, `btn-export`, `file-import`, `Boot + timers`.

Line numbers drift with edits — anchor on these strings, not on line numbers.

## The applet contract

Every card is registered as:

```js
registerApplet({
  id, title, wide?,      // wide:true spans two grid columns
  mount(card) { … },     // build body ONCE + wire delegated events (persistent)
  render(card) { … },    // refresh dynamic region from state (focus-safe, idempotent)
  tick?(card) { … }      // OPTIONAL, ~1×/sec (only clock + deepwork use it)
});
```

`parts(card)` returns `{head, sub, body}`. Delegate events on `head`/`body` (which
persist) so `render()` can replace inner HTML freely without losing listeners.

### Add a new applet

1. If it stores data: add a slice to `CONFIG` and handle it in `normalizeState()`. For
   day-scoped data, add a factory to `DAILY_DEFAULTS` and read it via `day("<id>")`.
2. Call `registerApplet({...})` in the APPLET DEFINITIONS section.
3. Add its id to `CONFIG.applets.order`, and to the applet table in `README.md`.

## Day-scoped state

Standup notes, deep-work counts, and the meeting/focus tally live under
`state.daily["YYYY-MM-DD"].<id>`. Get today's slice with `day("<id>")`. Keys use the
**local** date (`dateKey`, not UTC), roll over at local midnight (checked in the 1s
loop), and are pruned after 30 days.

## Validate before finishing

There's no test suite. At minimum, syntax-check the inline JS (works in the Bash tool):

```bash
node -e 'const fs=require("fs");const m=fs.readFileSync("index.html","utf8").match(/<script>([\s\S]*?)<\/script>/);fs.writeFileSync(process.env.TEMP+"/t.js",m[1])' \
  && node --check "$TEMP/t.js" && echo OK
```

For behavioral changes, also open `index.html` in a browser: confirm the affected card
renders, and that **Export → Import** round-trips cleanly.

## Conventions

- Work on a branch, never commit straight to `main`.
- End commit messages with the `Co-Authored-By:` trailer (see `git log`).
