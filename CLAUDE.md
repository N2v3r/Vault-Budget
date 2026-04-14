# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working with this repository.

## What this project is

A personal budget management PWA. **One file** (`index.html`, ~2,700 lines)
loads React 18 + Babel from CDN, runs entirely client-side, stores data in
`localStorage`. There is **no build step, no package.json, no test suite,
no linter**. The deployed app and the source file are identical.

**Do not suggest bundlers, npm scripts, TypeScript migration, or framework
upgrades unless explicitly asked.** Those are non-goals — the file-level
simplicity is the whole point.

## Development commands

The only command you need:

```bash
python3 -m http.server 8000
# open http://localhost:8000/
```

There is nothing else to run. No `npm install`, no `npm test`, no
`npm run build`.

## Deployment

Pushes to `main` auto-deploy via GitHub Pages within 1–2 minutes. **Every
commit on `main` is a release.** Use branches only for experiments you're
not confident about. No PR/review gate — this is a solo-dev project.

Live URL: `https://n2v3r.github.io/Vault-Budget/`

Legacy URL `https://cnc-dash.web.app/vault-budget.html` is a redirect to
the live URL (maintained by the separate `cnc-dash-v123` repo).

## Data architecture

**State shape (`D` object):**

```
{
  envs:       [{id, name, color, budget, annual, subcats, ...}]   // envelopes
  txs:        [{id, type, amt, desc, payee, eid, accountId, ts}]  // transactions
  goals:      [{id, name, target, saved}]
  recurring:  [{id, payee, amt, eid, freq, nextTs, lastTs}]
  accounts:   [{id, name, balance, openingBalance, ...}]
  debts:      [{id, name, balance, minPayment, ...}]
  unal:       number    // unallocated balance
  mode:       "simple" | "standard" | "power"
  theme:      "dark" | "light"
  ...
}
```

**The `sv()` function (~line 480) is the SINGLE write path.** It serializes
`D` to `localStorage` under key `vb-v10`, captures an undo snapshot, and
emits a `__vb_track('save', 'sv', {...})` debug event. **Never write to
`localStorage` directly — always go through `sv()`.**

**Account balance invariant:** For any account with `openingBalance` set,
`balance === openingBalance + sum(txs where accountId === acc.id)`.
Enforced by `reconcileAccounts(data)` at line 381. You don't need to call
it manually — `sv()` runs it on every save (line 476), so any save path
that goes through `sv()` gets reconciliation for free.

**Legacy data migration:** `migrateOpeningBalances()` at line 383 runs once
on load to backfill `openingBalance` for accounts saved before that field
existed (derives it from current balance minus summed txs). Relevant if
you're touching account code or debugging balance drift from old saves.

**Monthly rollover** (line 498 useEffect): On first load in a new month,
unspent envelope budget cascades forward. Cascade loops through every
missed month, so if the app wasn't opened for 3 months it still settles
correctly. The rollover write uses `sv(next, true)` — the `true` flag
skips the undo stack so rollover can't be accidentally "undone".

**Recurring auto-post** (line 496 useEffect): Each render, walks
`D.recurring` and posts any entries with `nextTs <= now` as transactions.
Two safeguards:
- **Dedup:** skips posting if a tx matching the same payee already exists
  within 10% of the interval window (`Math.abs(t.ts - rec.nextTs) < iv*.1`)
- **Safety cap:** max 12 posts per recurring per render, to stop runaway
  posting if `nextTs` is far in the past (e.g. old sample data)

Both matter if you touch that effect — removing either can spam the
transaction list.

**Undo:** `undoStack.current` keeps up to **20 snapshots** (impl:
`[prev, ...undoStack.current.slice(0, 19)]`). Toast from `fl(m)` exposes
an Undo button for ~4 seconds. `fl(m, true)` suppresses the undo button
(use for non-reversible or informational messages).

**`sv(d, skipUndo)` second param:** passing `true` as the second arg skips
the undo snapshot. Used by the rollover effect and by the undo action
itself (so undoing doesn't create a new undo entry). Don't drop this
flag if you modify either path.

## Tier system (feature gating)

Three tiers define which UI features are visible:

- `SIMPLE_FEATURES` — dash, envList, envDetail, addTx, editTx, themeToggle,
  reset, onboard, dailyBudget, accentPicker
- `STANDARD_FEATURES` — SIMPLE + healthScore, goals, recurring, calendar,
  fill, transfer, search, notes, monthlyTrend, incVsExp, donut, accounts, etc.
- `POWER_FEATURES` — STANDARD + multiAccount, debts, splitTx, deepReports,
  csvExport/import, pdfExport, subcatMgmt, fabRadial, netWorth, debug, etc.

Defined at lines 221–223. Gating via `canSee(mode, feat)` at line 224.

**⚠️ Gotcha: when adding a new feature that should be tier-gated, you MUST
add its key to the appropriate `*_FEATURES` Set.** Forgetting means the
feature silently stays hidden in all tiers (or worse, appears in unintended
tiers). Grep for `canSee("<feat>")` to find all gates for a feature.

## UI patterns

**Overlay state machine (`ov`):** A single piece of state holds the active
modal/overlay as `{t: "typeName", ...payload}`. Setting `ov` opens an
overlay. `closeOv()` at line 465 is the close path: sets `ovClosing = true`,
then after a **250ms `setTimeout`** clears `ov` back to `null` and resets
`ovClosing`. There is **no `transitionend` listener** — the 250ms timer
must match the CSS close-transition duration; changing one without the
other breaks the close animation.

Modal types (ov.t values): `addTx`, `editTx`, `splitTx`, `search`, `fill`,
`addEnv`, `addGoal`, `fundGoal`, `addRecurring`, `transfer`, `import`,
`calendar`, `addDebt`, `payDebt`, `settings`, `graph`.

`onboard` and `debug` are **not** overlay types — onboarding is a separate
top-level state variable, and `DebugOverlay` renders unconditionally when
`__VB_DEBUG` is truthy (not via `ov`).

**Modal behaviour changes on wide viewports (≥820px):** centered card style,
no drag-to-dismiss, no bottom-sheet entrance. `centered` flag on `Morph`/`Ov`
components controls this.

**`p` props bundle:** Most screens receive a single props bag `p` containing
the data, setters, helpers, and current viewport flags. Read from `p.D`,
`p.fl`, `p.setOv`, `p.xwide`, etc. instead of threading individual props
through.

**13 chart components (lines 1242–1744):** Each chart takes `{D, G, ...}`
where `G` is shared layout/style helpers. Charts use inline SVG (no chart
library). When touching a chart, preserve the `tip`/`setTip` hover pattern
for tooltip state.

## Debug system

Activated by any of:
- `?debug=1` in URL
- 5-tap on the settings cog in Power tier
- `localStorage.setItem('vb_debug', '1')` from DevTools (persists across reloads)
- `window.__vb.enable()` console helper (uses the localStorage key internally)

Disable with `window.__vb.disable()` or `localStorage.removeItem('vb_debug')`.

Exposes:

- `window.__vb` — `{ log, dump, clear, track }` console API
- `__vb_track(category, name, payload)` — the event bus. Categories:
  `life`, `err`, `tap`, `input`, `net`, `gesture`, `action`, `toast`,
  `modal`, `state`, `save`, `sw`.

Debug overlay (`DebugOverlay` component) shows a live feed with category
filters, copy-to-clipboard, state snapshot.

**Adding a new tracked event:** Use an existing category where it fits.
Only create a new category if the event doesn't cleanly map to any existing
one. Category strings are not enum-enforced — typos silently create ghost
categories.

## PWA setup

v2 is **self-contained**: no external manifest, icons, or service worker
files.

- **Manifest** built inline as a JS object, serialized to a `Blob`, assigned
  to `<link rel="manifest">` via `URL.createObjectURL` (lines 43–60).
- **Icons** drawn via `<canvas>` at 192px and 512px on each page load
  (lines 16–38). Dark rounded square with a green money-bag emoji and
  "VAULT" wordmark.
- **Service worker** registered from a Blob URL with a pass-through fetch
  handler (lines 61–69). Offline fallback returns a 503 "Offline" response.

**Do not add external `vault-manifest.json`, icon PNGs, or `vault-sw.js`
files** — the inline generation is intentional so the repo stays one file.

## Locale, fonts, aesthetics

- Currency: **ZAR** (`R1,234.56` format), locale `en-ZA`
- Fonts: `Outfit` (display) + `DM Sans` (body) from Google Fonts
- Colours: Dark radial-gradient background (`#0a0b0f` → `#0d1b2a`), neon
  accents (teal `#00E5A0`, cyan, amber, red). CSS variables at `:root`
  define the neon glow presets.
- Font-family constants: `F = "'Outfit', sans-serif"`, `F2 = "'DM Sans', sans-serif"`

## Viewport / fold-aware layout

| Width | Layout |
|---|---|
| `<600px` (phones) | Single column, bottom nav, FAB |
| `600–819px` | Single column 720px cap, bottom nav, FAB |
| `≥820px` (Fold 7 inner, tablets, desktop) | **Sidebar (280px) + content (580px)**, no bottom nav, no FAB |
| `@media (spanning: single-fold-vertical)` | Fold span mode when device is unfolded |

Two-pane layout on wide viewports treats the sidebar as the canonical nav.
Tab swipe gestures are disabled when wide; sidebar pills are the only
navigation. `xwide` flag on `p` tells components the layout mode.

## What to watch out for

- **`vb-v10` is load-bearing.** Changing the localStorage key loses every
  existing user's data. Migrate via a versioned load step instead.
- **`sv()` is the only write path.** Tempting one-line `localStorage.setItem`
  shortcuts break undo, tracking, and the invariant guarantees.
- **`canSee()` gates can be silently wrong.** When adding a feature, grep
  `*_FEATURES` and make sure the new key is in the right Set. Missing from
  all three = feature permanently hidden.
- **Don't refactor into multiple files.** The single-file structure is the
  simplest possible story for Claude mobile / Projects and for deployment.
  Splitting would require introducing a bundler.
- **Don't install npm packages.** There is no package.json on purpose.
  CDN-loaded React + Babel is the runtime. Any new dep must come from a
  CDN `<script>` tag in `<head>`.
- **GitHub Pages serves `/` from `index.html`.** Renaming the file breaks
  the deployed URL. If you want multi-page routing, use hash routes, not
  separate `.html` files.

## Git workflow

- `main` is the deploy branch — **every commit on `main` goes live in
  1–2 minutes**.
- Single dev, no PR required. Direct commits to `main` are the expected
  workflow.
- For experimental work: `git switch -c try/<thing>`, iterate, then
  `git switch main && git merge --squash try/<thing> && git commit`.
- Recovery path: `git revert <bad-sha>` → push. GitHub Pages redeploys
  within ~1 minute.

## Roadmap

### Flutter rewrite (separate project)

A Flutter rewrite targeting feature parity with this PWA is planned for
Google Play Store submission. It will live in a **different repo** — do
not mix Flutter/Dart code into this one.

### Netlify as secondary host (queued, not started)

GitHub Pages is fine for production but has no preview-per-branch and no
one-click rollback UI. When convenient, add Netlify on top of the same
repo as a second deploy target:

1. Sign into `netlify.com` with GitHub, click "Add new site → Import
   existing project", select `N2v3r/Vault-Budget`.
2. Build command: leave blank. Publish directory: `/` (root). Production
   branch: `main`.
3. No `netlify.toml` required — single-file static site, Netlify
   auto-detects. Only add one if you want custom headers or a
   `/old-url → /new-url` redirect.
4. Every branch + PR then gets an automatic preview URL (`<slug>.netlify.app`).
   Production stays on GH Pages; Netlify becomes the staging/preview env.
5. If you later want Netlify as primary: point a custom domain at it
   and disable GH Pages in repo settings. Don't do this during active
   use — installed PWAs would need to re-register under the new origin.

**Why not do it now:** adds a second deploy every push, a second dashboard
to keep track of, and right now there are no branches other than `main`
so the preview feature has nothing to preview. Worth the setup the first
time a risky change is in flight.
