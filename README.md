# Vault Budget

A personal budget management PWA with a three-tier feature system, 13 charts,
multi-account support, debts tracking, and CSV import/export.

## Live

- Production: `https://<username>.github.io/vault-budget/` (set after enabling GitHub Pages)
- Legacy URL `https://cnc-dash.web.app/vault-budget.html` redirects here

## Tech stack

- Single-file HTML app — React 18 + Babel loaded from CDN
- `localStorage` persistence (key: `vb-v10`)
- Self-contained PWA: manifest, icons (192/512px), and service worker all
  generated inline via Blob URLs at runtime — no external PWA assets
- Currency: ZAR, locale: en-ZA
- Google Fonts: Outfit + DM Sans

## Deployment

Pushes to `main` auto-deploy via GitHub Pages. No build step — `index.html`
is served as-is.

## Local development

Clone the repo, then serve the folder with any static server:

```bash
python3 -m http.server 8000
# open http://localhost:8000/
```

All state lives in browser `localStorage` under the `vb-v10` key.

## Tier system

- **Simple** 🌱 — dashboard, envelopes, transactions, basic settings
- **Standard** 📊 — adds goals, recurring, transfers, search, health score
- **Power** ⚡ — adds multi-account, debts, split transactions, CSV import,
  all 13 charts, CSV + PDF export

## Roadmap

A full native Android rewrite in Flutter is planned as a separate project
targeting feature parity with this PWA. See project notes.

## License

TBD
