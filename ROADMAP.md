# Roadmap

Tracks the plan for this PWA and its planned native successor. Captured
here so future Claude sessions (desktop, mobile, Projects) inherit the
intent without re-deriving it.

Last updated: 2026-04-14.

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

**Repo location:** a **separate third repo** (not `Vault-Budget`, not
`cnc-dash-v123`). Name TBD when created. Flutter/Dart code never lands
in this repo.

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
- **`<future>`** — the Flutter rewrite, to be created before Phase 0.
  Add its URL here once it exists.
