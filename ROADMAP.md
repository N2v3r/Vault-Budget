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

## Phased plan

| Phase | Scope | Effort |
|---|---|---|
| **0** | Dev env setup (Flutter SDK, Android Studio, emulator, JDK 17). Verify with `flutter doctor`. | 1 day |
| **1** | Project scaffolding — `flutter create`, Riverpod, go_router, Hive, theme, typography, glassmorphism widgets | 3–5 days |
| **2** | Data layer — typed Dart models from the `vb-v10` JSON shape, Hive adapters, migration import from PWA export | 3–5 days |
| **3** | Simple tier — dash, envelopes, add/edit tx, basic settings, theme toggle. First shippable alpha. | ~2 weeks |
| **4** | Standard tier — goals, recurring, transfers, search, calendar, health score, first 3–4 charts | 2–3 weeks |
| **5** | Power tier — multi-account, debts, split tx, remaining 9–10 charts, CSV import, CSV + PDF export, onboarding, debug overlay | 3–4 weeks |
| **6** | Polish + release prep — animations (implicit + Rive for hero), splash, adaptive icon, signed keystore, privacy policy page, screenshots in 5 device sizes, feature graphic, Play Console listing | 1–2 weeks |
| **7** | Play Store submission — internal testing → closed beta → open beta → production. Age rating + data safety declaration (mandatory for finance apps). | 1 week setup + 1–7 day review |

## Open questions (decide before the relevant phase)

These are intentionally unresolved — each changes the implementation
meaningfully. Defaults are listed if you never answer.

### 1. iOS / App Store?
- **Status:** undecided
- **Default if unanswered:** Android-only. Flutter codebase still
  compiles for iOS, but App Store listing, Apple Developer account
  ($99/yr), and App Store review are deferred indefinitely.
- **Deadline to resolve:** before Phase 7 (Play submission). iOS-aware
  code (e.g. Cupertino widgets, iOS-specific permissions) is easier to
  add from the start than retrofit.

### 2. Sync / multi-device?
- **Status:** undecided
- **Default if unanswered:** **offline-only, single device** (same as
  PWA). User installs app → data lives on that device → if phone is
  wiped, data is gone unless they exported first.
- **If you want sync:** options are Firebase (easiest, same ecosystem
  as the CNC project), Supabase (open-source alternative), or iCloud
  (iOS-only, doesn't help Android). Each adds ~1–2 weeks to the
  timeline and requires a privacy policy update.
- **Deadline to resolve:** before Phase 2 (data layer). Hive vs
  cloud-backed state is an architectural fork.

### 3. PWA sunset?
- **Status:** undecided
- **Default if unanswered:** PWA **runs alongside** the native app
  indefinitely. Zero marginal cost — it's already deployed on free
  hosting. Users on non-Android devices (iOS until q1, desktops, Fold
  outer screen) keep it as their option.
- **If you decide to sunset:** add an in-app banner on the PWA
  pointing to the Play Store listing, then eventually 410 Gone the
  hosted URLs. Not urgent.

### 4. Data migration from PWA to native app?
- **Status:** undecided
- **Default if unanswered:** **JSON export / import.** PWA gains a
  "Download your data" button (writes the `vb-v10` localStorage blob
  to a `.json` file). Native app gains an "Import from PWA" flow that
  reads the file. Cleanest, most auditable.
- **Alternatives:** QR code containing a compressed data URL (nice for
  same-device transfer), or fresh-start (user accepts data loss).
- **Deadline to resolve:** before Phase 3 (first shippable tier).

## Backlog (PWA-only, nice-to-have)

Not blocking the Flutter rewrite, but tracked here so they're not lost:

- Explicit "export my data" button in settings (would also serve
  question 4 above)
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
