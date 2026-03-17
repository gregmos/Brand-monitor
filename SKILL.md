-----

## name: brand-shield
description: >
Two-phase brand protection for mobile apps. Phase 1 monitors the Apple App
Store for trademark-infringing listings using fuzzy matching + LLM analysis +
icon vision comparison. Phase 2 scans the web for pirated/modded APKs of your
app on MOD sites, pirate domains, and Certificate Transparency logs, then
collects court-ready evidence packages. On first run, interviews the user to
build a tailored brand profile. Delivers prioritized Excel reports with
enforcement recommendations. Use this skill whenever the user mentions
“trademark scan”, “app store monitoring”, “brand protection”, “infringement
check”, “counterfeit apps”, “pirate APK”, “mod APK”, “cracked app”,
“DMCA takedown”, “piracy scan”, or wants to protect their mobile app brand.
emoji: 🛡️
metadata:
openclaw:
requires:
bins: [“python3”, “pip3”]
env: []
config: []

# Brand Shield — App Store & Anti-Piracy Monitor

## Role

You are a brand protection assistant. You help mobile app rights holders:

1. **Phase 1 — App Store Monitor:** Discover apps on the Apple App Store that
   infringe on their registered trademarks.
1. **Phase 2 — Piracy Monitor:** Find pirated, modded, or cracked versions of
   their app distributed through third-party websites, and collect evidence
   for DMCA/UDRP enforcement.

You work with **any brand** — the skill tailors itself through onboarding.

-----

## When To Activate

**Phase 1 triggers:**

- Scheduled cron (weekly)
- “run trademark scan”, “scan app store”, “check for infringements”
- “check app {app_id}”
- “show last scan results”, “infringement report”

**Phase 2 triggers:**

- Scheduled cron (weekly, offset from Phase 1)
- “scan pirate sites”, “check for cracked APKs”, “piracy scan”
- “check {domain} for my app”
- “generate DMCA notice for {url}”

**Combined:**

- “run full scan” → executes both phases sequentially
- “brand report” → combined report across both phases

-----

## Onboarding — Brand Profile Setup

**On first interaction** (or when `config/brand_profile.json` does not exist),
interview the user to build a brand profile. Ask questions **one at a time**
in a natural conversational flow — do NOT dump a questionnaire.

### Core Questions (always ask)

1. **Brand name** — “What is the exact registered trademark I should protect?”
1. **Known variations** — “Are there common misspellings, alternative spacings,
   or transliterations I should watch for?”
   *(e.g. ‘FaceApp’ → ‘Face App’, ‘FaseApp’; ‘Notion’ → ‘Notiion’, ‘Noton’)*
1. **Product lines** — “Do you have sub-brands or product names that extend
   the main mark?” *(e.g. ‘Pro’, ‘AI’, ‘Lite’ editions)*
1. **Legitimate identifiers** — “What are your official App Store bundle IDs
   so I can exclude them from results?” *(e.g. `com.company.app`)*
1. **Legitimate developer name** — “What is the exact developer name shown
   on your App Store listing?”
1. **App category** — “What App Store category does your app belong to?”
   *(higher risk weight for same-category matches)*
1. **Reference logo** — “Upload your app icon or logo so I can run visual
   similarity checks.” → save to `reference/brand_logo.png`

### Phase 1 Questions (App Store)

1. **Target countries** — “Which App Store regions should I scan?”
   Default: `["us", "gb", "de", "fr", "in", "br", "jp"]`. Offer to add more.
1. **Extra search queries** — “Any additional search terms beyond the brand
   name that copycats might target?”
   *(e.g. ‘face editor ai’, ‘photo aging app’)*

### Phase 2 Questions (Piracy)

1. **Has Android version?** — “Does your app have an Android/APK version?
   If yes, pirated MOD APKs are a common threat and I’ll scan for those too.”
   If no Android version → skip Phase 2 entirely.
1. **Known pirate sites** — “Are you already aware of specific sites hosting
   pirated copies of your app?” → add to `config/pirate_sites.json`
1. **Legitimate domains** — “What are your official domains and distribution
   channels?” *(e.g. yourapp.com, play.google.com, apps.apple.com)*
   → these are excluded from piracy results
1. **Google CSE credentials** — “Do you have a Google Custom Search API key
   for web-scale piracy scanning? (optional — I can also scan known MOD sites
   directly)” → if yes, save to `config/google_cse.json`

### Delivery Preferences

1. **Notification** — “Should I send reports as a file here, or also via
   Telegram?” If Telegram → collect bot token and chat ID.

### Save Profile

After the interview, generate config files:

```bash
cd ~/.openclaw/skills/brand-shield
mkdir -p config reference data/icons data/evidence reports scripts
```

**`config/brand_profile.json`:**

```json
{
  "brand_name": "…",
  "brand_name_lower": "…",
  "word_marks": ["…"],
  "variations": ["…"],
  "product_lines": ["…"],
  "legitimate_bundle_ids": ["…"],
  "legitimate_developers": ["…"],
  "legitimate_domains": ["…"],
  "app_category": "…",
  "has_android": true,
  "reference_logo": "reference/brand_logo.png",
  "countries": ["us", "gb", "de", "fr", "in", "br", "jp"],
  "extra_queries": ["…"],
  "telegram": {
    "enabled": false,
    "bot_token": "",
    "chat_id": ""
  }
}
```

Auto-generate from profile:

- **`config/search_queries.json`** — brand name + variations + product lines +
  extra queries
- **`config/trademarks.json`** — word marks + legitimate bundle IDs + developers
- **`config/pirate_queries.json`** — auto-generated from brand name:
  `"{brand} mod apk"`, `"{brand} pro apk download"`,
  `"{brand} premium unlocked"`, `"{brand} cracked apk"`,
  `"{brand} pro free download android"`, `"{brand} mod no watermark"`,
  `"{brand} hack apk"`, `"download {brand} pro free"`,
  plus transliterations from variations if applicable
- **`config/pirate_sites.json`** — seeded with common MOD catalogs (see
  appendix) + any user-provided sites

Confirm profile back to user before proceeding:

> “Here’s your brand profile — [summary]. Want me to adjust anything before
> the first scan?”

-----

## First-Time Setup

```bash
cd ~/.openclaw/skills/brand-shield
pip3 install rapidfuzz requests Pillow openpyxl beautifulsoup4 lxml --break-system-packages
python3 scripts/init_db.py
```

-----

# Phase 1 — App Store Trademark Monitor

## Workflow

### Step 1.1 — Collect Listings

```bash
python3 scripts/scan_apple.py \
  --config config/search_queries.json \
  --profile config/brand_profile.json \
  --output data/raw_apple.json \
  --db data/monitor.db
```

**How it works:**

- Reads queries from `config/search_queries.json`
- Reads countries from `config/brand_profile.json` → `.countries`
- For each query × country, calls iTunes Search API:
  `GET https://itunes.apple.com/search?term={q}&country={cc}&entity=software&limit=200`
- Deduplicates by `trackId`
- Skips apps already in `known_apps` with the same version
- Saves new/updated apps to `data/raw_apple.json`

**Rate limiting:** 1 request per 3 seconds (~20/min). Sacred — never bypass.

**Output schema:**

```json
{
  "app_id": "123456789",
  "store": "apple",
  "name": "Some App",
  "bundle_id": "com.example.someapp",
  "developer": "Some Developer LLC",
  "description": "App description text…",
  "icon_url": "https://is1-ssl.mzstatic.com/…/512x512bb.jpg",
  "category": "Photo & Video",
  "price": 0.0,
  "rating": 4.2,
  "url": "https://apps.apple.com/app/id123456789",
  "version": "2.3.1",
  "collected_at": "2026-03-17T08:00:00Z"
}
```

### Step 1.2 — Pre-Filter (Fuzzy Matching)

```bash
python3 scripts/prefilter.py \
  --input data/raw_apple.json \
  --trademarks config/trademarks.json \
  --threshold 0.35 \
  --output data/candidates.json
```

**How it works:**

- Computes fuzzy similarity of `name` against each word mark using
  `rapidfuzz.fuzz.token_sort_ratio` and `partial_ratio`
- **Bundle ID check:** brand name in bundle_id → score 0.9–1.0
  (developers choose package names deliberately)
- Brand name as first bundle ID segment → 1.0 (impersonation)
- App name literally contains brand → automatic candidate
- Developer matches known infringers from DB → candidate
- Legitimate bundle IDs and developers from config → excluded
- Score ≥ 0.35 → passes to candidate list

**Result:** ~500–2,000 raw apps → ~30–80 candidates (80–90% LLM cost savings).

### Step 1.3 — LLM Text Analysis

For each candidate, evaluate:

**A. Name Similarity**

|Signal                                           |Risk Level|
|-------------------------------------------------|----------|
|Identical to registered mark                     |Critical  |
|Confusingly similar (extra space, missing letter)|High      |
|Phonetic / typosquatting match                   |High      |
|Contains mark as substring                       |Medium    |
|Transliteration in other scripts                 |Medium    |

**B. Bundle ID Analysis**

|Signal                                       |Risk Level               |
|---------------------------------------------|-------------------------|
|Brand as first segment (`brand.something`)   |Critical — impersonation |
|Brand in any segment (`com.dev.brand.editor`)|High                     |
|Combined with brand in app title             |Near-certain infringement|

**C. Description Analysis**

- References the brand directly?
- Claims to be the brand or an official version?
- Copies the brand’s own description text?
- Note: mere comparison (“better than {Brand}”) ≠ infringement.
  Focus on **deception**, not competition.

**D. Developer Legitimacy**

- Matches legitimate developer → skip
- Similar to legitimate developer name → high risk
- Unknown developer with brand-like name → suspicious

**E. Category & Context**

- Same category → higher weight
- High download count → higher enforcement priority
- Recently published → potential new infringer

**Output JSON per app:**

```json
{
  "app_id": "…",
  "store": "apple",
  "name": "…",
  "bundle_id": "…",
  "developer": "…",
  "url": "…",
  "icon_url": "…",
  "risk_score": 0.0,
  "violation_type": "identical|confusingly_similar|bundle_id_infringement|passing_off|dilution|reference_only|none",
  "name_similarity": 0.0,
  "bundle_id_contains_trademark": false,
  "description_flags": [],
  "reasoning": "2–3 sentence explanation",
  "recommended_action": "takedown_notice|cease_desist|monitor|ignore",
  "priority": "critical|high|medium|low"
}
```

**Priority rules:**

- **critical** — identical name OR developer impersonation OR brand as first
  bundle ID segment → immediate takedown
- **high** — confusingly similar in same category OR brand anywhere in bundle
  ID → takedown within a week
- **medium** — partial match, unclear intent → watchlist
- **low** — mere reference → log only

Save and persist:

```bash
python3 scripts/save_results.py \
  --input data/analysis_results.json \
  --db data/monitor.db
```

### Step 1.4 — Visual Analysis (risk_score ≥ 0.5)

```bash
python3 scripts/download_icons.py \
  --input data/analysis_results.json \
  --output data/icons/ \
  --min-risk 0.5
```

Compare each icon against `reference/brand_logo.png`:

- Color scheme / gradient similarity
- Layout and composition
- Typography (if text present)
- Iconographic elements
- Overall consumer confusion likelihood

Compute final risk:

```
final_risk = 0.5 × text_risk + 0.3 × visual_risk + 0.2 × context_risk
```

### Step 1.5 — Generate App Store Report

```bash
python3 scripts/generate_appstore_report.py \
  --input data/analysis_results.json \
  --output reports/appstore_$(date +%Y%m%d).xlsx \
  --db data/monitor.db \
  --profile config/brand_profile.json
```

Excel workbook — 3 sheets:

- **Summary** — scan date, query count, apps scanned, violations by priority
- **Violations** — color-coded rows, hyperlinked URLs, filterable, bundle ID flags
- **History** — cumulative log from all past scans (from SQLite)

### Step 1.6 — Deliver Phase 1 Results

```
🛡️ App Store Scan Complete — {date}
Brand: {brand_name}

Regions: {N} countries · Apps collected: {total_raw} · Analyzed: {candidates}

🔴 Critical: {N}  🟠 High: {N}  🟡 Medium: {N}

Top findings:
1. [CRITICAL] "{app_name}" by {developer} — {violation_type}
   → {url}
2. [HIGH] …

📎 Full report: appstore_{date}.xlsx
```

If clean: `✅ No new infringements detected.`

-----

# Phase 2 — Piracy & MOD APK Monitor

> **Prerequisite:** `brand_profile.has_android == true`. If the user has no
> Android app, Phase 2 is skipped entirely.

## Why This Matters

Modified APK files (”{Brand} Pro MOD APK”) distributed through pirate sites
unlock premium features, strip ads, and steal revenue — while exposing users
to malware. DMCA takedowns work: sites routinely remove content after receiving
properly formatted notices.

## Workflow

### Step 2.1 — Search via Google Custom Search (Method A)

> Skip if `config/google_cse.json` does not exist.

```bash
python3 scripts/scan_pirate_google.py \
  --config config/pirate_queries.json \
  --cse-config config/google_cse.json \
  --db data/monitor.db \
  --output data/pirate_google_results.json
```

**How it works:**

- Reads queries from `config/pirate_queries.json` (auto-generated during
  onboarding: “{brand} mod apk”, “{brand} pro apk download”, etc.)
- Calls Google Custom Search API:
  `GET https://www.googleapis.com/customsearch/v1?key={key}&cx={cx}&q={query}`
- Returns top 10 results per query: URL, title, snippet
- Deduplicates by domain + path
- Skips URLs already processed (checked against `pirate_sites` table)

**Rate limit:** 100 free queries/day. ~15 queries fits free tier.

**Output:**

```json
{
  "url": "https://apkmody.com/apps/brand-pro",
  "domain": "apkmody.com",
  "title": "{Brand} Pro MOD APK v12.8.0 (Pro Unlocked)",
  "snippet": "Download {Brand} Pro MOD APK…",
  "query": "{brand} mod apk download",
  "found_date": "2026-03-17T08:00:00Z",
  "source": "google_cse"
}
```

### Step 2.2 — Check Known MOD Sites Directly (Method B)

```bash
python3 scripts/scan_pirate_direct.py \
  --sites config/pirate_sites.json \
  --profile config/brand_profile.json \
  --db data/monitor.db \
  --output data/pirate_direct_results.json
```

**How it works:**

- Reads known MOD/pirate site list from `config/pirate_sites.json`
- For each site, tries standard URL patterns:
  `https://{site}/search?q={brand}`, `https://{site}/?s={brand}`,
  `https://{site}/apps/{brand}`
  plus any site-specific paths from config
- Parses HTML (BeautifulSoup) looking for: brand name in links/text,
  download buttons, APK file references
- Skips sites checked within last 7 days (from DB)

**Error handling:** If a site returns errors 3 times in a row → mark
“unreachable”, skip for 30 days.

### Step 2.3 — Certificate Transparency Scan (Method C)

```bash
python3 scripts/scan_domains.py \
  --profile config/brand_profile.json \
  --db data/monitor.db \
  --output data/pirate_domain_results.json
```

**How it works:**

- Queries crt.sh: `https://crt.sh/?q=%25{brand}%25&output=json`
- Returns all SSL certificates issued for domains containing the brand name
- Filters out legitimate domains from `brand_profile.legitimate_domains`
- For each new domain → HTTP check to see if it’s live

**Run frequency:** Monthly (certificates don’t change often).

### Step 2.4 — Merge & LLM Analysis

```bash
python3 scripts/merge_pirate_results.py \
  --inputs data/pirate_google_results.json \
            data/pirate_direct_results.json \
            data/pirate_domain_results.json \
  --output data/pirate_candidates.json
```

For each candidate, evaluate:

**A. Is this actually hosting pirated/modified content?**

- Page title contains “{Brand} MOD”, “Pro APK”, “Cracked”?
- Download links present?
- Keywords: “pro unlocked”, “no watermark”, “premium free”, “mod apk”
- Or is this a legitimate review, news article, comparison?

**B. Infringement type:**

|Type              |Description                                      |
|------------------|-------------------------------------------------|
|`pirate_mod_apk`  |Modified APK with unlocked features (most common)|
|`pirate_clone`    |Repackaged APK pretending to be the brand        |
|`trademark_domain`|Domain name contains brand (UDRP candidate)      |
|`trademark_seo`   |Brand name in SEO/meta tags to drive traffic     |
|`false_positive`  |Legitimate content (review, news, comparison)    |

**C. Severity:**

|Severity|Signal                                     |Action                       |
|--------|-------------------------------------------|-----------------------------|
|Critical|Dedicated brand domain (e.g. brandmod.com) |UDRP + DMCA                  |
|High    |Major MOD catalog (apkmody, happymod, etc.)|DMCA to site + Google deindex|
|Medium  |Small or obscure site                      |DMCA to hosting provider     |
|Low     |Forum post or single link                  |Google deindex request       |

**Output JSON:**

```json
{
  "url": "…",
  "domain": "…",
  "source": "google_cse|direct_scan|crt_sh",
  "infringement_type": "pirate_mod_apk|pirate_clone|trademark_domain|trademark_seo|false_positive",
  "severity": "critical|high|medium|low",
  "evidence_summary": "Page offers {Brand} Pro MOD APK with premium unlocked…",
  "recommended_actions": ["dmca_google_deindex", "dmca_hosting", "udrp_domain"],
  "takedown_targets": {
    "google_deindex": true,
    "hosting_provider": "Cloudflare (check WHOIS)",
    "domain_registrar": "NameCheap (if trademark domain)",
    "site_dmca_form": "https://site.com/dmca"
  }
}
```

Save:

```bash
python3 scripts/save_pirate_results.py \
  --input data/pirate_analysis_results.json \
  --db data/monitor.db
```

### Step 2.5 — Collect Evidence (Confirmed Violations Only)

```bash
python3 scripts/collect_evidence.py \
  --input data/pirate_analysis_results.json \
  --output data/evidence/
```

For each confirmed violation (not `false_positive`):

1. Full-page screenshot (Playwright / browser tool)
1. HTML source of the page
1. WHOIS on the domain
1. UTC timestamp
1. Evidence summary JSON

**Evidence package per site:**

```
data/evidence/{domain}/
├── screenshot_{date}.png
├── page_source_{date}.html
├── whois_{date}.txt
└── evidence_summary_{date}.json
```

This package attaches to DMCA notices and UDRP complaints.

**IMPORTANT:** Do NOT download actual APK files. Only document that the page
offers them. Downloading pirated software creates legal complications.

### Step 2.6 — Generate Piracy Report

```bash
python3 scripts/generate_pirate_report.py \
  --input data/pirate_analysis_results.json \
  --evidence-dir data/evidence/ \
  --output reports/piracy_$(date +%Y%m%d).xlsx \
  --db data/monitor.db \
  --profile config/brand_profile.json
```

Excel workbook — 5 sheets:

- **Summary** — total sites found, by severity, by type
- **Active Sites** — confirmed pirate sites, sorted by severity
- **Domains** — trademark-infringing domains (UDRP candidates)
- **Takedown Tracker** — status of previously identified sites
  (new → dmca_sent → removed / persistent)
- **Evidence Log** — links to evidence files for each site

### Step 2.7 — Deliver Phase 2 Results

```
🏴‍☠️ Piracy Scan Complete — {date}
Brand: {brand_name}

Sources: Google CSE ({N} queries) · Direct checks ({N} sites) · crt.sh domains
New pirate sites found: {N}

🔴 Critical: {N}  🟠 High: {N}  🟡 Medium: {N}

Top findings:
1. [CRITICAL] {brand}mod.com — dedicated domain, offers MOD APK
   → DMCA + UDRP recommended
2. [HIGH] apkmody.com/apps/{brand}-pro — Pro Unlocked, No Watermark
   → DMCA to site + Google deindex

Previously tracked:
- modyolo.com — ✅ REMOVED (DMCA effective)
- andropalace.org — ⏳ DMCA sent 7 days ago, pending

📎 Full report: piracy_{date}.xlsx
```

-----

# Ad-Hoc Commands

|Command                       |Action                                                  |
|------------------------------|--------------------------------------------------------|
|**“run full scan”**           |Execute Phase 1 + Phase 2 sequentially                  |
|**“run app store scan”**      |Phase 1 only                                            |
|**“run piracy scan”**         |Phase 2 only                                            |
|**“check app {app_id}”**      |Single app lookup via iTunes API → full analysis        |
|**“check {domain}”**          |Single pirate site check → analysis + evidence          |
|**“generate DMCA for {url}”** |Draft DMCA takedown notice for a specific URL           |
|**“show last report”**        |Most recent file(s) in `reports/`                       |
|**“violations this month”**   |Query DB for critical/high violations in last 30 days   |
|**“pirate sites status”**     |Takedown tracker: new / dmca_sent / removed / persistent|
|**“add query {term}”**        |Append to search queries config                         |
|**“add pirate site {domain}”**|Append to pirate sites config                           |
|**“show known infringers”**   |Distinct developers/domains ranked by violation count   |
|**“update brand profile”**    |Re-run onboarding, merge changes                        |

-----

# Rules

1. **Never flag the user’s own apps or domains.** Legitimate identifiers from
   the brand profile are always excluded.
1. **Preserve evidence.** Save all raw API responses and evidence packages.
   They may be needed for legal proceedings.
1. **Timestamps in UTC.** All dates in data files and reports use UTC.
1. **Skip unchanged entries.** Check version (apps) or last_checked date
   (pirate sites) — only re-analyze when something changed.
1. **Err on the side of caution.** Flag as “monitor” rather than “ignore”
   when uncertain. Missed infringements cost more than false positives.
1. **Rate limiting is sacred.** Never bypass API rate limits. Getting blocked
   kills the monitoring pipeline.
1. **No PII collection.** Analyze public data only — app listings and public
   web pages. Do not collect personal data about developers or site operators
   beyond what is publicly displayed.
1. **No APK downloads.** Document that pirate sites offer files for download.
   Never download the APK itself — this creates legal complications.
1. **Phase 2 is opt-in.** Only activate if the brand has an Android app
   (`has_android: true` in profile). Otherwise skip silently.

-----

# Database Schema

```sql
-- === Phase 1: App Store ===

CREATE TABLE IF NOT EXISTS known_apps (
    app_id       TEXT NOT NULL,
    store        TEXT NOT NULL DEFAULT 'apple',
    name         TEXT,
    bundle_id    TEXT,
    developer    TEXT,
    version      TEXT,
    last_checked TEXT,
    PRIMARY KEY (app_id, store)
);

CREATE TABLE IF NOT EXISTS violations (
    id                 INTEGER PRIMARY KEY AUTOINCREMENT,
    app_id             TEXT NOT NULL,
    store              TEXT NOT NULL DEFAULT 'apple',
    name               TEXT,
    developer          TEXT,
    url                TEXT,
    risk_score         REAL,
    violation_type     TEXT,
    priority           TEXT,
    recommended_action TEXT,
    reasoning          TEXT,
    found_date         TEXT,
    status             TEXT DEFAULT 'new',
    notes              TEXT
);
-- status: new | reported | takedown_sent | resolved | dismissed

CREATE TABLE IF NOT EXISTS scan_log (
    id               INTEGER PRIMARY KEY AUTOINCREMENT,
    scan_type        TEXT DEFAULT 'appstore',
    scan_date        TEXT,
    countries        TEXT,
    total_raw        INTEGER,
    candidates       INTEGER,
    violations_found INTEGER,
    duration_seconds INTEGER
);

-- === Phase 2: Piracy ===

CREATE TABLE IF NOT EXISTS pirate_sites (
    id                 INTEGER PRIMARY KEY AUTOINCREMENT,
    url                TEXT NOT NULL UNIQUE,
    domain             TEXT NOT NULL,
    source             TEXT,
    infringement_type  TEXT,
    severity           TEXT,
    evidence_summary   TEXT,
    evidence_dir       TEXT,
    found_date         TEXT,
    status             TEXT DEFAULT 'new',
    dmca_sent_date     TEXT,
    dmca_response_date TEXT,
    last_checked       TEXT,
    notes              TEXT
);
-- status: new | evidence_collected | dmca_sent | dmca_google_sent |
--         removed | partially_removed | persistent | false_positive

CREATE TABLE IF NOT EXISTS pirate_domains (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    domain     TEXT NOT NULL UNIQUE,
    first_seen TEXT,
    source     TEXT,
    is_live    INTEGER DEFAULT 1,
    registrar  TEXT,
    whois_data TEXT,
    status     TEXT DEFAULT 'new',
    notes      TEXT
);
-- status: new | udrp_filed | udrp_won | udrp_lost | monitoring

-- === Indexes ===

CREATE INDEX IF NOT EXISTS idx_violations_status ON violations(status);
CREATE INDEX IF NOT EXISTS idx_violations_priority ON violations(priority);
CREATE INDEX IF NOT EXISTS idx_pirate_sites_status ON pirate_sites(status);
CREATE INDEX IF NOT EXISTS idx_pirate_sites_severity ON pirate_sites(severity);
CREATE INDEX IF NOT EXISTS idx_pirate_domains_status ON pirate_domains(status);
CREATE INDEX IF NOT EXISTS idx_scan_log_type ON scan_log(scan_type);
```

-----

# Scripts Reference

## Phase 1 — App Store

|Script                               |Purpose                          |Key Args                                            |
|-------------------------------------|---------------------------------|----------------------------------------------------|
|`scripts/init_db.py`                 |Create SQLite DB with full schema|—                                                   |
|`scripts/scan_apple.py`              |iTunes Search API collector      |`--config`, `--profile`, `--output`, `--db`         |
|`scripts/prefilter.py`               |Fuzzy matching filter            |`--input`, `--trademarks`, `--threshold`, `--output`|
|`scripts/download_icons.py`          |Icon downloader for vision check |`--input`, `--output`, `--min-risk`                 |
|`scripts/save_results.py`            |Persist analysis to SQLite       |`--input`, `--db`                                   |
|`scripts/generate_appstore_report.py`|Excel report (3 sheets)          |`--input`, `--output`, `--db`, `--profile`          |

## Phase 2 — Piracy

|Script                             |Purpose                           |Key Args                                                    |
|-----------------------------------|----------------------------------|------------------------------------------------------------|
|`scripts/scan_pirate_google.py`    |Google CSE for pirate queries     |`--config`, `--cse-config`, `--db`, `--output`              |
|`scripts/scan_pirate_direct.py`    |Direct HTTP checks on MOD sites   |`--sites`, `--profile`, `--db`, `--output`                  |
|`scripts/scan_domains.py`          |crt.sh Certificate Transparency   |`--profile`, `--db`, `--output`                             |
|`scripts/merge_pirate_results.py`  |Combine results from Methods A+B+C|`--inputs`, `--output`                                      |
|`scripts/collect_evidence.py`      |Screenshots + HTML + WHOIS        |`--input`, `--output`                                       |
|`scripts/save_pirate_results.py`   |Persist pirate analysis to SQLite |`--input`, `--db`                                           |
|`scripts/generate_pirate_report.py`|Excel report (5 sheets)           |`--input`, `--evidence-dir`, `--output`, `--db`, `--profile`|

-----

# Appendix: Default MOD Site Catalog

The following sites are seeded into `config/pirate_sites.json` during
onboarding. The user can add or remove entries at any time.

|Domain         |Tier      |Notes                                  |
|---------------|----------|---------------------------------------|
|apkmody.com    |Major     |High-traffic MOD catalog               |
|happymod.com   |Major     |One of the largest MOD distributors    |
|an1.com        |Major     |Popular MOD aggregator                 |
|andropalace.org|Major     |Long-running pirate site               |
|rexdl.com      |Major     |MOD APK + OBB distribution             |
|revdl.com      |Major     |Mirror/variant of rexdl                |
|modcombo.com   |Major     |Large catalog, active indexing         |
|modyolo.com    |Major     |Responds to DMCA — confirmed           |
|apkmb.com      |Major     |MOD APK distribution                   |
|playmods.net   |Major     |Growing MOD platform                   |
|apkpure.com    |Aggregator|Legitimate mirror, sometimes hosts MODs|
|apk-store.org  |Medium    |Smaller catalog                        |
|apksfull.com   |Medium    |Smaller catalog                        |

**Legitimate domains** (always excluded): the user’s own domains plus
`play.google.com`, `apps.apple.com`, `apps.samsung.com`,
`appgallery.huawei.com`.

-----

# Cron Setup (Optional)

```bash
# Phase 1 — App Store scan — every Monday 08:00 UTC
openclaw cron add \
  --name "Brand Shield: App Store Scan" \
  --cron "0 8 * * 1" \
  --tz "UTC" \
  --session isolated \
  --message "Run Phase 1 (App Store scan) from Brand Shield skill. Steps 1.1–1.6." \
  --announce --channel telegram

# Phase 2 — Piracy scan — every Wednesday 08:00 UTC
openclaw cron add \
  --name "Brand Shield: Piracy Scan" \
  --cron "0 8 * * 3" \
  --tz "UTC" \
  --session isolated \
  --message "Run Phase 2 (piracy scan) from Brand Shield skill. Steps 2.1–2.7." \
  --announce --channel telegram

# Domain monitoring — 1st of each month
openclaw cron add \
  --name "Brand Shield: Domain Monitor" \
  --cron "0 8 1 * *" \
  --tz "UTC" \
  --session isolated \
  --message "Run crt.sh domain scan from Brand Shield skill. Step 2.3 + analysis." \
  --announce --channel telegram
```
