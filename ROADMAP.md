# Roadmap

Tracks the plan for this PWA and its planned native successor. Captured
here so future Claude sessions (desktop, mobile, Projects) inherit the
intent without re-deriving it.

Last updated: 2026-04-14 (Flutter repo name locked in:
`N2v3r/vault-budget-flutter`).

## Current state

PWA is **stable and live.** Nothing blocking on the web. Three hosts
(see `CLAUDE.md` → "Deployment & hosting"). Changes from here are
incremental polish, bug fixes, or new features for existing users.

## Next milestone: Flutter native rewrite for Play Store

**Decision:** full Android rewrite in Flutter (Dart), shipped as an
`.aab` to the Google Play Store.

**Why Flutter vs alternatives** (decided after weighing options):
- Not React Native — would inherit some React concepts but still a full
  rewrite, and Flutter's native feel / animations / perf are stronger
  for a chart-heavy finance app
- Not Kotlin native — Android-only with no iOS fallback, most verbose,
  and the whole app would need Jetpack Compose from scratch anyway
- Not Capacitor (web wrapper) — the user explicitly wants a truly
  native app, not "a web app in a shell"

**Scope v1.0:** **full feature parity** with the PWA — not a
slimmed-down MVP. That means all 13 charts, all 18+ overlays, the
three-tier system, multi-account, debts, split transactions, CSV
import, CSV + PDF export, onboarding, debug overlay, fold-aware
two-pane layout.

**Repo location:** a **separate third repo**,
**`N2v3r/vault-budget-flutter`** (not `Vault-Budget`, not
`cnc-dash-v123`). Not yet created — will be spun up at the start of
Phase 0. Flutter/Dart code never lands in this repo.

**Effort estimate:** 8–12 weeks of focused solo-dev work. At evening
pace (a few hours per day), realistically 4–6 months calendar time.
This is a commitment, not a weekend.

## Recommended tech stack

| Concern | Pick | Why |
|---|---|---|
| State management | **Riverpod** | Modern standard for solo Flutter projects of this size; avoids BLoC boilerplate |
| Navigation | **go_router** | Declarative, deep-linking-ready, official Flutter team recommendation |
| Persistent storage | **Hive** | NoSQL, matches the current `vb-v10` JSON blob shape — easier migration than normalising into sqflite schemas |
| Charts | **fl_chart** | Most mature Flutter charting library; covers all 13 current chart types |
| Fonts | **google_fonts** package | Loads `Outfit` + `DM Sans` cleanly, same as the PWA |
| Fold-aware layout | **dual_screen** or `MediaQuery.displayFeatures` | Matches the existing `@media (spanning: ...)` + ≥820px two-pane behaviour |
| PDF export | **pdf** + **printing** packages | Client-side PDF generation, no backend needed |
| CSV | **csv** package | Same scope as the current PWA's import/export |
| Google auth | **google_sign_in** | User's existing Android Google account — no new signup flow |
| Cloud backup | **googleapis** (Drive v3) + **googleapis_auth** | Writes `vb-v10` blob to the app-private Drive folder; survives phone wipes |

## Phased plan

| Phase | Scope | Effort |
|---|---|---|
| **0** | Dev env setup (Flutter SDK, Android Studio, emulator, JDK 17). Verify with `flutter doctor`. | 1 day |
| **1** | Project scaffolding — `flutter create`, Riverpod, go_router, Hive, theme, typography, glassmorphism widgets | 3–5 days |
| **2** | Data layer — typed Dart models from the `vb-v10` JSON shape, Hive adapters, migration import from PWA export, **Google Drive backup/restore wrapper** (see "Sync architecture" below) | ~1 week |
| **3** | Simple tier — dash, envelopes, add/edit tx, basic settings, theme toggle. First shippable alpha. | ~2 weeks |
| **4** | Standard tier — goals, recurring, transfers, search, calendar, health score, first 3–4 charts | 2–3 weeks |
| **5** | Power tier — multi-account, debts, split tx, remaining 9–10 charts, CSV import, CSV + PDF export, onboarding, debug overlay | 3–4 weeks |
| **6** | Polish + release prep — animations (implicit + Rive for hero), splash, adaptive icon, signed keystore, privacy policy page, screenshots in 5 device sizes, feature graphic, Play Console listing | 1–2 weeks |
| **7** | Play Store submission — internal testing → closed beta → open beta → production. Age rating + data safety declaration (mandatory for finance apps). | 1 week setup + 1–7 day review |

## Decisions

All four product questions resolved on 2026-04-14. Captured here with
reasoning so future sessions don't re-litigate.

### 1. iOS / App Store? — **Android only**
Flutter codebase could still be built for iOS later if priorities
change, but App Store listing, Apple Developer account ($99/yr), and
App Store review are **deferred indefinitely**. No Cupertino widgets,
no iOS-specific permission flows.

### 2. Sync / multi-device? — **Google Drive cloud backup**
Not a real-time multi-device sync — a **Google-Drive-backed
periodic backup / restore** using the user's existing Android Google
account.

See the "Sync architecture" section below for the full design.

### 3. PWA sunset? — **Runs alongside, forever**
Zero marginal cost — already on free hosting. Iters on non-Android
devices (iOS, desktops, Fold outer screen, tablets without Play
Services) keep the PWA as their option. No in-app banner, no URL
retirement.

### 4. Data migration from PWA to native app? — **JSON export / import**
- **PWA side: already done.** `SetOv` at line 2620 of `index.html`
  has `📤 Export Backup` and `📥 Import Backup` buttons. Export
  produces `vault-backup-YYYY-MM-DD.json` (pretty-printed `D` blob).
  Import parses, validates `envs` + `txs` presence, runs
  `migrateOpeningBalances` + `reconcileAccounts`, saves with
  `sv(restored, true)`. No work needed.
- **Native side:** Flutter app gains an "Import from PWA" flow that
  reads the same `vault-backup-YYYY-MM-DD.json` format. Must handle
  the same validation + reconciliation. Shipped in Phase 3.

## Sync architecture (Google Drive)

Driving principle: the user's budget file lives on the device, **not**
in the cloud. Drive is a pull-anywhere **backup mirror**, not the
source of truth. This avoids the complexity of real-time sync while
solving the actual problem (phone wipe / new-phone migration).

**Storage location:** `appDataFolder` scope — Google's
app-private-on-your-own-Drive area, invisible in the user's Drive UI,
only readable by this app. No permission to browse the user's other
files.

**What gets backed up:** A single file, `vault-budget-v10.json`, which
is the JSON-serialised `vb-v10` blob (the whole `D` state object).

**Cadence:** Three triggers, debounced:
- After any `sv()` save — debounced to at most one Drive write per 60s
- On app foreground after > 5 min background — ensures a recent copy
- Manual "Back up now" button in Settings

**Restore flow:** On first launch post-install (no Hive data present):
- Offer "Sign in with Google to restore backup" card
- If accepted → fetch `vault-budget-v10.json` from appDataFolder →
  import into Hive → done
- If declined or no backup exists → start fresh, user can turn on
  backup later in Settings

**Conflict policy:** last-write-wins. Each Drive upload overwrites the
previous. No diff, no merge. Acceptable because the realistic user is
one person editing on one device.

**Optional enhancement (v1.1):** keep N versioned copies
(`vault-budget-v10.2026-04-14.json`, etc.) with a 30-day retention, so
a user who accidentally deletes everything can roll back. Not in v1.0.

**Auth scope required:**
`https://www.googleapis.com/auth/drive.appdata` — app folder only.
**Not** full Drive access; not files in the user's main Drive.

**Privacy policy impact:** must now disclose:
- App authenticates with Google Sign-In
- App reads/writes a single JSON file in its private Drive folder
- File contains financial transactions, envelope balances, goals

**Play Store "Data safety" declaration:** must declare financial data
is transferred to "Other service" (Google Drive) for the purpose of
"Account management / backup". Not collected by us directly.

## SA-budget adaptation (shipped 2026-06-16, branch `claude/budget-app-8lcrir`)

Adapted the PWA to the owner's real South-African finances (full build spec
in session history). All single-file / offline / localStorage / CDN
constraints preserved. Key model + UI additions:

- **Internal transfers** — new `xfer` transaction type written as a linked
  pair (`xferGroup`). Account balances include it; income/expense totals
  and all spending charts exclude it (`isExpTx` / `isIncTx` / `gByMonth`).
  New **Move-Money** overlay (`AcctXferOv`). This is the §1 keystone — a
  sweep between own accounts is never counted as spending or income.
- **Run-rate exclusions** — `tx.kind` of `oneoff` (capital events),
  `lending_out`/`lending_repay` (recoverable) and `sinking` are kept out of
  the monthly run-rate (`EXP_EXCLUDE` / `INC_EXCLUDE`).
- **Category groups** — envelopes carry `group` (Income · Home & Family ·
  Health · Everyday · Lifestyle · Lending). New **Budget Table** overlay
  (`BudgetOv`): grouped Actual vs editable Target vs Diff, subtotals, and a
  prominent Income − Spending bottom line. Income rows come from income
  sources and can be paused.
- **Income sources** — modelled as `recurring` rows with `isIncome`,
  `paused`, `endTs`. Auto-post effect honours all three (the stopped
  investment-interest case is seeded `paused`).
- **Payee rules** — `D.rules` (`{match,kind,eid}`); `classifyImport()` applies
  them then Capitec description patterns then sign/keyword. **Rules** overlay
  (`RulesOv`) to manage; import can save a rule per payee.
- **Capitec import** — `ImportOv` rewritten: paste text / CSV / PDF (pdf.js
  from CDN, on-device), auto-categorise (transfer/subscription/fee/lending/
  person), per-row type override + skip + save-rule, hardened dedup
  (amount + payee + account).
- **Subscriptions & fees** — `SubsOv`: detects recurring subs, flags spikes
  (latest > 1.25× baseline), totals fee leakage; low-balance warning when
  Main < `account.minBuffer`.
- **Lending / receivables** — `D.receivables` + `LendingOv`: track money
  lent as recoverable, match repayments (reduce balance, not counted as
  income), pausable.
- **Sinking funds** — goals carry `sinking:true`; one-off capital events
  tagged `oneoff` so they don't distort the monthly view.
- **Seed** — demo data replaced with the owner's real budget (7 accounts,
  the 6 groups with ballpark targets, income sources, payee rules, a
  receivable, 6 months of representative history). Default mode = `power`.
  Settings → Reset restores this template (transactional data cleared).

New tier-feature keys: `budgetTable`, `acctXfer` (Standard); `payeeRules`,
`subscriptions`, `lending` (Power).

Verification: code-reviewed + delimiter-balance checked; live verification is
via the Netlify branch-preview URL (mobile workflow). Not yet merged to `main`.

### Follow-up fixes + polish pass (2026-06-16, same branch)

**Functional fixes (Part A):**
- **§1 hero** now shows TRUE money-on-hand — the combined balance across the 7
  accounts (real net worth), clearly labelled "TOTAL BALANCE · money on hand · N
  accounts". Budget-remaining is surfaced separately as a labelled "BUDGET LEFT"
  metric. The two are no longer conflated. (`netWorth`/`acctTotal` in `App`,
  threaded into `Dash` + `Sidebar`.)
- **§6 subscription spikes** — `analyzeSubs()`/`topSubSpike()` now scan EVERY month
  of each sub's history against a robust median baseline and flag any month >1.4×
  baseline (and ≥R100 over). The past AI-tool spike now fires, is surfaced as a
  dashboard banner, and shown in the Subs overlay with per-month sparklines (spike
  month glows amber).
- **§4 split** — over-budget envelopes now WARN (amber, "· allowed") instead of
  blocking, so the pharmacy→Medicine split works when Medicine is over. Added a
  second mode that re-buckets a **merchant's monthly total** across categories
  (`merchantSplit` txs, reachable from the new ✂ Split dashboard tool). Empty split
  lines are ignored on save.
- **Seed re-baselined** to the spec's reference shape: current month nets ≈ -R7,578
  (income ~R41k, run-rate ~R48.6k); current month kept exact, history mildly varied
  so the over-spend trend is visible. All real structure preserved + editable.
- **Backup** made prominent — a 💾 button in the dashboard header, sidebar footer
  and tools row opens a dedicated Backup & Restore overlay (export/import). Per-
  account **minimum buffer** is now user-editable in Settings → Accounts.

**Visual / UX polish (Part B):**
- `prefers-reduced-motion` honoured (CSS media query neutralises animations;
  `useCountUp` jumps to value).
- Count-up animation on the dashboard + budget-table "Income − Spending" net.
- Hero sheen sweep on load; gradient + glow envelope progress bars.
- Envelopes tab grouped by category group with colour-coded headers and per-group
  subtotals.

Verified headless (Playwright, mobile + desktop + light + reduced-motion):
**zero console errors**, all four must-fixes confirmed, persistence survives reload.

## Backlog (PWA-only, nice-to-have)

Not blocking the Flutter rewrite, but tracked here so they're not lost:

- `netlify.toml` for custom security headers (CSP, Permissions-Policy)
  if we ever want to harden
- Automated visual-regression snapshot (Playwright against Netlify
  preview URL) — currently every deploy is eyeballed

## Related repos

- **`N2v3r/Vault-Budget`** (this one) — the PWA source
- **`N2v3r/https-cnc-dash.web.app.`** — CNC work tool; hosts the
  legacy `/vault-budget.html` redirect to `n2v3r.github.io/Vault-Budget`
- **`N2v3r/vault-budget-flutter`** — the Flutter native rewrite. Not
  yet created; will be spun up at the start of Phase 0. Add its URL
  here once it exists.
