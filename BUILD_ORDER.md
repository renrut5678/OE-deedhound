# Build Order

Follow this sequence exactly. Each step depends on the ones before it.

---

## Step 1: Config Layer

**Build:** `scraper/config_loader.py`, `config/scoring.yaml`, `config/counties/example_county.yaml`

**Why first:** Everything else depends on config. Build and test the loader in isolation.

**Acceptance criteria:**
- `load_scoring_config()` returns valid scoring dict
- `load_county_configs()` loads all YAML files from directory
- `load_county_configs(county_filter="example_county")` loads only one
- Missing required fields raise `ValueError` with clear message
- Optional fields have sensible defaults

**Test:** Write a temporary test script that loads the example config and prints it.

---

## Step 2: Appraiser Loader

**Build:** `scraper/appraiser.py`

**Why second:** Name lookup is needed by the matcher, and it's simpler than the clerk scraper (no browser automation). Good warmup.

**Acceptance criteria:**
- `download_appraiser_data()` downloads file to temp dir (stub with local file for testing)
- `parse_parcel_file()` reads DBF, returns list of dicts
- `build_name_lookup()` generates 3 variants per owner, all uppercased
- Handles corporate names, multiple owners, edge cases
- Returns empty dict if `appraiser_url` is None

**Test:** Download a sample DBF (or create a test fixture), verify lookup dict keys.

---

## Step 3: Fuzzy Matcher

**Build:** `scraper/matcher.py`

**Why third:** Depends on appraiser lookup dict. Pure logic, no I/O.

**Acceptance criteria:**
- `generate_name_variants("SMITH, JOHN M")` returns correct variants
- `match_owner()` returns strong match at ≥80%, weak at 60-79%, None below 60%
- `enrich_records()` attaches address fields correctly
- Sets `needs_enrichment` / `needs_review` flags appropriately
- Handles None/empty owner names without crashing

**Test:** Create mock lookup, run against test records, verify address attachment and flags.

---

## Step 4: Scoring Engine

**Build:** `scraper/scorer.py`

**Why fourth:** Depends on flag data from config and enrichment results from matcher.

**Acceptance criteria:**
- Base score is 30
- Each flag adds 10 points
- LP + NOFC combo on same owner adds 20
- Amount thresholds work correctly (>$100k = +15, >$50k = +10, not both)
- Recency and address bonuses apply correctly
- Score caps at 100
- Records returned sorted by score descending
- LLC/corp detection works ("LLC", "INC", "CORP", "TRUST" in owner name)

**Test:** Create test records with known flags/amounts, verify exact scores.

---

## Step 5: Exporter

**Build:** `scraper/exporter.py`

**Why fifth:** Depends on scored records from scorer.

**Acceptance criteria:**
- `export_json()` writes valid JSON with correct wrapper structure
- Writes to both `data/records.json` and `dashboard/records.json`
- `merge_records()` deduplicates by doc_num, preserves old records
- `export_csv()` writes correct columns in correct order
- `split_name()` handles "LAST, FIRST", "FIRST LAST", corporate names
- Empty fields become empty strings in CSV (not "None")

**Test:** Export sample data, manually verify JSON structure and CSV columns.

---

## Step 6: Clerk Scraper

**Build:** `scraper/clerk.py`

**Why sixth:** Most complex component. Requires Playwright, async, portal-specific logic. All other components should be solid before tackling this.

**Acceptance criteria:**
- Launches headless Chromium via Playwright
- Navigates to clerk portal URL
- Searches each lead type with correct date range
- Extracts all fields from result rows
- Handles pagination (detects and clicks "Next")
- 3 retries with exponential backoff on failures
- Never crashes on bad rows — logs and skips
- Returns list of raw record dicts

**Test:** Run against actual clerk portal URL (once configured). Verify records are extracted. Use `--dry-run` to validate connectivity.

**Note:** This module will need county-specific CSS selectors. The initial build should implement a generic table parser. If a specific county's portal has a unique layout, add selector overrides in the county YAML config.

---

## Step 7: Main Entry Point

**Build:** `scraper/fetch.py`

**Why seventh:** Orchestrates all components built in steps 1-6.

**Acceptance criteria:**
- Parses `--county`, `--dry-run`, `--log-level` CLI args
- Processes each county sequentially
- Catches per-county exceptions without stopping other counties
- Prints summary at end
- Exit code 0 if all counties succeeded, 1 if any failed
- `--dry-run` validates configs and tests connectivity only

**Test:** Run full pipeline with example county config. Verify output files exist and contain valid data.

---

## Step 8: Dashboard

**Build:** `dashboard/index.html`

**Why eighth:** The scraper pipeline must work first so there's data to display.

**Acceptance criteria:**
- Fetches and parses `records.json` on load
- Renders table with all columns
- Color-codes rows by score (red ≥70, orange ≥40, white <40)
- Sorts by score descending by default
- Shows header stats (total, with address, date range)
- Doc number links to clerk URL
- Works when opened as local file AND served from GitHub Pages
- Responsive (usable on mobile)

**Test:** Open in browser with sample records.json. Verify colors, sorting, links.

---

## Step 9: GitHub Actions Workflow

**Build:** `.github/workflows/scrape.yml`

**Why ninth:** Automation layer on top of working system.

**Acceptance criteria:**
- Runs daily at 7:00 AM UTC
- Can be triggered manually (workflow_dispatch)
- Installs all dependencies including Playwright + Chromium
- Runs `scraper/fetch.py`
- Commits and pushes updated data/dashboard files
- Logs failure on error

**Test:** Push to GitHub, trigger manually from Actions tab, verify commit appears.

---

## Step 10: Requirements File

**Build:** `scraper/requirements.txt`

**Note:** Create this in Step 1 alongside config_loader, but finalize it here after all dependencies are confirmed.

**Final contents:**
```
requests
beautifulsoup4
lxml
dbfread
playwright
rapidfuzz
pyyaml
```

---

## Post-Build Verification Checklist

- [ ] `python scraper/fetch.py --dry-run` completes without error
- [ ] `python scraper/fetch.py` produces valid `data/records.json`
- [ ] `data/records.json` and `dashboard/records.json` are identical
- [ ] `data/export.csv` has correct columns and data
- [ ] `dashboard/index.html` renders correctly in browser
- [ ] Scores are calculated correctly (spot-check 5 records)
- [ ] Address enrichment is working (check `with_address` count)
- [ ] Running twice doesn't duplicate records (check total count)
- [ ] GitHub Actions workflow runs successfully
- [ ] GitHub Pages serves dashboard
