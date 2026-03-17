# 🛡️ Brand Shield

AI agent skill that monitors the Apple App Store for trademark-infringing apps and scans the web for pirated/modded APKs of your product.

Built for [OpenClaw](https://openclaw.ai). Works with any brand — configures itself through an onboarding interview on first run.

## What It Does

**Phase 1 — App Store Monitor**

- Queries iTunes Search API across multiple countries
- Fuzzy pre-filter (rapidfuzz) scores app names and bundle IDs against your trademarks (~90% noise reduction)
- LLM analysis assigns risk scores and recommended actions
- Icon vision comparison against your reference logo
- Outputs a prioritized Excel report

**Phase 2 — Piracy Monitor**

- Searches for “{brand} mod apk”, “{brand} cracked”, etc. via Google Custom Search API
- Direct checks on known MOD sites (apkmody, happymod, etc.)
- Certificate Transparency scan (crt.sh) for new domains containing your brand
- Automated evidence collection: screenshots, HTML source, WHOIS
- Outputs a separate Excel report with takedown tracker

## How It Works

The skill is a single `SKILL.md` file — a structured prompt that tells the AI agent what to do, step by step.

On first run, it asks you:

- Your registered trademark and known misspellings
- Legitimate bundle IDs and developer names (excluded from results)
- Target App Store countries
- Whether you have an Android app (enables piracy scanning)
- Optional: Google CSE key, Telegram bot for notifications

Everything else — search queries, fuzzy matching config, pirate site list — auto-generates from your answers.

## Quick Start

1. Install the skill in OpenClaw
1. Say “run trademark scan” or “run full scan”
1. On first run, answer the onboarding questions
1. The agent handles the rest

## Requirements

- Python 3 with `rapidfuzz`, `requests`, `Pillow`, `openpyxl`, `beautifulsoup4`, `lxml`
- Apple iTunes Search API (free, no key needed)
- Google Custom Search API key (optional, for piracy web scanning)

## Adapting the Skill

This is a markdown file, not a codebase. Want to change something?

- **Add Google Play scanning** — drop `SKILL.md` into Claude and ask
- **Change report format** — same
- **Plug in a different search API** — same
- **Set up for Claude Code or Cowork** — ask Claude to adapt the workflow for your environment

No reverse-engineering required.

## File Structure

```
brand-shield/
├── SKILL.md                 ← the skill itself
├── config/
│   ├── brand_profile.json   ← generated during onboarding
│   ├── search_queries.json  ← auto-generated from profile
│   ├── trademarks.json      ← auto-generated from profile
│   ├── pirate_queries.json  ← auto-generated from profile
│   ├── pirate_sites.json    ← seeded with known MOD catalogs
│   └── google_cse.json      ← optional, for web-scale piracy scanning
├── scripts/                 ← Python scripts for data collection & reports
├── data/                    ← raw listings, candidates, analysis results
├── reports/                 ← Excel reports
└── reference/
    └── brand_logo.png       ← your app icon for visual comparison
```

## License

MIT
