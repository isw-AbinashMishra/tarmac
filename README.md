# tarmac

A personal developer dashboard — a single static HTML file you open as your Chrome
homepage / pinned tab. No backend, no build step, no framework. Vanilla JS + inline
CSS. Works opened directly via `file://`.

> Built to the "baseplate" spec. The app, repo, and localStorage key all use the name
> **tarmac**; treat "baseplate" as the spec codename.

---

## Files

| File                          | What it is                                                        |
| ----------------------------- | ---------------------------------------------------------------- |
| `index.html`                  | The whole app (inline CSS + JS). This is all you need to run.     |
| `README.md`                   | This file.                                                       |
| `tarmac-config.example.json`  | A filled-in example you can **Import** to see the dashboard populated, or use as a template for your own backup JSON. |

Open `index.html` in Chrome. That's it.

---

## Quick start

1. Open `index.html`.
2. Fill in your real bookmark URLs and first quota — see **"Edit these lines"** below,
   or just do it in-app (hover a bookmark tile → pencil; **+ Add** on the Quota card).
3. Optionally **Export** a backup once you've set things up.

### Set as your Chrome homepage / startup page

1. Chrome → `⋮` → **Settings** → **On startup** → "Open a specific page or set of pages".
2. **Add a new page** and paste the file address, e.g. `file:///D:/source/tarmac/index.html`.
3. Home button: Settings → **Appearance** → enable **Show home button** → same address.
4. Right-click the tab → **Pin** so it survives restarts.

---

## Applets

Ten cards, all pure local state:

| id          | Applet                     | Notes                                                            |
| ----------- | -------------------------- | --------------------------------------------------------------- |
| `clock`     | Clock + date               | Live time, full date, days remaining in the calendar month.     |
| `quota`     | Monthly quota burn-down    | `{label, monthlyLimit, used, resetDay}`; pace = used% vs elapsed%.|
| `deepwork`  | Deep-work timer            | Pomodoro (default 50/10). Counts focus blocks + interruptions per day. |
| `tally`     | Meeting / focus tally      | Two ± counters in 0.5 steps, ratio, rolling 7-day sparkline.     |
| `todo`      | To-do list                 | Add/complete/delete, drag-to-reorder, optional due date, open badge. |
| `standup`   | Standup notes buffer       | Free text per day, day-picker for history, "Copy as standup".    |
| `blockers`  | Waiting-on / blocked       | `{item, who, since}`; shows age in days, sorted oldest first.    |
| `snippets`  | Snippet / command stash    | `{label, snippet, tag}`; click-to-copy, text filter.            |
| `decisions` | Decision log (light ADR)   | `{date, decision, why}`; newest first, searchable.              |
| `bookmarks` | Bookmarks                  | Grouped links with optional emoji/icon; editable in-app + config.|

---

## Data, backup & safety

- **One source of truth:** a single `state` object, persisted (debounced) to
  `localStorage` under the key **`tarmac:state`**. localStorage is per-browser and
  per-profile, so it is *not* a real backup.
- **Real backup = Export / Import JSON.** Export downloads `tarmac-backup-*.json`.
  Import validates the file's shape, then **replaces** state. A bad file shows an error
  toast and leaves your data untouched (no wipe).
- **Storage-quota guard:** if a save fails (storage full/blocked), you get a
  non-blocking toast and your in-memory state is preserved until you close the tab.
- **No telemetry, no external calls** except opening bookmark links. If offline,
  everything still works (no CDN dependency).

### Day-scoped data

Standup notes, deep-work counts, and the meeting/focus tally are stored per day under
`state.daily["YYYY-MM-DD"].<applet>`. Today's key is computed on load and every second,
so buffers **roll over automatically at local midnight**. Past days are kept for history
and pruned after **30 days**.

---

## Configuration

All defaults live in the `CONFIG` object at the top of the `<script>` in `index.html`.
**CONFIG only seeds a fresh install** (empty storage). Once you have saved state, change
things in-app, or Export → edit the JSON → Import.

### Enable / disable / reorder applets

Config-driven via `state.applets`:

```js
applets: {
  order:    ["clock", "quota", "deepwork", "tally", "todo",
             "standup", "blockers", "snippets", "decisions", "bookmarks"],
  disabled: []            // e.g. ["decisions"] to hide the decision log
}
```

Edit `CONFIG.applets` for a fresh install, or edit the exported JSON and re-import.

### Add a bookmark group

- **In-app:** **+ Group**, then **+ Link** inside it. Hover a tile → pencil to edit;
  clear a link's label to delete it.
- **In code:** add to `CONFIG.bookmarkGroups`. Each group is
  `{ label, links: [{ label, url, icon }] }`.

### Add a quota instance

- **In-app:** **+ Add** on the Quota card.
- **In code:** add to `CONFIG.quotas`:
  ```js
  { label: "OpenAI", unit: "$", monthlyLimit: 120, used: 0, resetDay: 1, source: "manual" }
  ```
  The **pace** badge compares your usage against a straight-line expectation for how far
  you are into the reset cycle (`over pace` = used% > elapsed%).

  *Future live data:* `computeQuota(q)` is pure — it derives everything from
  `{monthlyLimit, used, resetDay}`. To wire a real source later, add an adapter keyed on
  `q.source` (e.g. `"github"`) that sets `q.used` from a `fetch()` before render. Nothing
  else needs to change. (Live fetching is intentionally out of scope for a `file://` app —
  it needs external APIs + secrets.)

### Add a new applet

No need to touch unrelated code:

1. If it stores data, add a slice to `CONFIG` + `normalizeState()` (or, for a day-scoped
   applet, add a factory to `DAILY_DEFAULTS` and use `day("myid")`).
2. Call `registerApplet({ id, title, wide?, mount(card), render(card), tick? })` in the
   **APPLET DEFINITIONS** section.
3. Add its id to `CONFIG.applets.order`.

The framework builds the card, mounts it once, and renders it from `state`.
See the big header comment in `index.html` for the full contract.

---

## Edit these lines (in `index.html`)

Your real bookmark URLs and first quota, by line number:

| Line | Change                                                                     |
| ---- | ------------------------------------------------------------------------- |
| 591  | `{ label: "Jira",   url: "#REPLACE-ME", ... }` → your Jira URL            |
| 592  | `{ label: "GitHub", url: "#REPLACE-ME", ... }` → your GitHub URL          |
| 593  | Google (already `https://www.google.com`) — change if you like            |
| 594  | Claude (already `https://claude.ai`) — change if you like                 |
| 607  | `label: "GitHub credits"` — rename your first quota                       |
| 609  | `monthlyLimit: 1000` — your allowance                                     |
| 611  | `resetDay: 1` — the day-of-month it resets (1–28)                         |

> ⚠️ Editing `CONFIG` only affects a **fresh install**. If you've already opened the app
> (state is saved), either run `tarmac.reset()` in the console first, or just edit those
> bookmarks/quotas in-app.

---

## Keyboard & debugging

- Press **`/`** to jump to the add-task field.
- Theme button cycles **system → light → dark** (respects `prefers-color-scheme`).
- Console: `tarmac.state` (inspect), `tarmac.export()`, `tarmac.reset()`.
