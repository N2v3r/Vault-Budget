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
Enforced by `reconcileAccounts(data)` at line 381. Call it after any batch
of tx mutations that could drift balances.

**Monthly rollover** (around line 520): On first load in a new month,
unspent envelope budget cascades to the next month (or expires, depending
on envelope config). Rolled envelopes are counted and a `fl()` toast
fires once per rollover event.

**Undo:** `undoStack.current` keeps the last N snapshots. Toast from `fl(m)`
exposes an Undo button for ~4 seconds. `fl(m, true)` suppresses the undo
button (use for non-reversible or informational messages).

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
modal/overlay as `{t: "typeName", ...payload}`. Setting `ov` opens an overlay;
setting `ov` to `null` closes with animation. Closing animation is driven
by a separate `ovClosing` flag plus a `transitionend` listener. Modal types
include `addTx`, `editTx`, `splitTx`, `search`, `fill`, `addEnv`, `addGoal`,
`fundGoal`, `addRec`, `xfer`, `import`, `calendar`, `addDebt`, `payDebt`,
`set` (settings), `graph`, `onboard`, `debug`.

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

Activated by `?debug=1` in URL or 5-tap on the settings cog in Power tier.
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

## Roadmap (separate project)

A Flutter rewrite targeting feature parity with this PWA is planned for
Google Play Store submission. It will live in a **different repo** — do
not mix Flutter/Dart code into this one.
