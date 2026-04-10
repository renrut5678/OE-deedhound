# Architecture

## System Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    CONFIG LAYER                          │
│  config/counties/*.yaml    config/scoring.yaml          │
└──────────────┬──────────────────────┬───────────────────┘
               │                      │
               ▼                      ▼
┌──────────────────────┐  ┌─────────────────────────┐
│   CLERK SCRAPER      │  │   APPRAISER LOADER      │
│   clerk.py           │  │   appraiser.py          │
│                      │  │                         │
│  • Playwright async  │  │  • Download DBF file    │
│  • Search by doc type│  │  • Parse with dbfread   │
│  • Extract records   │  │  • Build name lookup    │
│  • Retry 3x         │  │    (3 variants/owner)   │
│  • Paginate          │  │  • Cache in memory      │
└──────────┬───────────┘  └────────────┬────────────┘
           │                           │
           ▼                           ▼
┌──────────────────────────────────────────────────────────┐
│                    MATCHER (matcher.py)                   │
│                                                          │
│  For each clerk record:                                  │
│   1. Generate name variants from owner field             │
│   2. rapidfuzz.process.extractOne() against appraiser lookup│
│   3. ≥80% → attach property + mailing address            │
│   4. 60-79% → attach but flag needs_review               │
│   5. <60% → no match, flag needs_enrichment              │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                    SCORER (scorer.py)                     │
│                                                          │
│  Base: 30                                                │
│  +10 per distress flag                                   │
│  +20 lis pendens + foreclosure combo                     │
│  +15 amount > $100k                                      │
│  +10 amount > $50k                                       │
│  +5  new this week                                       │
│  +5  has property address                                │
│  Cap: 100                                                │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                   EXPORTER (exporter.py)                  │
│                                                          │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐  │
│  │ records.json  │  │ records.json   │  │ export.csv   │  │
│  │ (data/)       │  │ (dashboard/)   │  │ (data/)      │  │
│  └──────┬───────┘  └───────┬────────┘  └──────┬───────┘  │
└─────────┼──────────────────┼───────────────────┼─────────┘
          │                  │                   │
          ▼                  ▼                   ▼
    Backup/archive     GitHub Pages         CRM Import
                       Dashboard            (manual CSV
                       (index.html)          upload)
```

## Data Flow (per county, per run)

```
1. Load county config (YAML)
2. Launch Playwright browser
3. Navigate to clerk portal
4. For each lead type (LP, NOFC, TAXDEED, etc.):
   a. Search clerk portal for date range (today - LOOKBACK_DAYS → today)
   b. Extract all result rows
   c. For each row: parse into raw record dict
   d. Handle pagination (next page until no more)
5. Close browser
6. Download appraiser DBF file (requests + __doPostBack if needed)
7. Parse DBF → build owner name lookup dict
8. For each raw record:
   a. Fuzzy match owner name → appraiser lookup
   b. Attach address fields if match found
   c. Assign distress flags based on doc_type
   d. Calculate seller score
9. Load existing records.json
10. Merge new records (deduplicate by doc_num)
11. Write updated records.json to data/ and dashboard/
12. Write export.csv to data/
13. Log summary: total scraped, matched, scored, written
```

## Component Responsibilities

### config_loader.py
- Load all YAML files from `config/counties/`
- Load `config/scoring.yaml`
- Validate required fields exist
- Return list of county configs + scoring config
- Support `--county` CLI arg to filter to single county

### clerk.py
- Accept a county config dict
- Launch async Playwright browser (Chromium, headless)
- Navigate to clerk portal URL
- For each configured lead type:
  - Perform search with date range
  - Parse result table rows into record dicts
  - Handle pagination
- Return list of raw record dicts
- All network calls: 3 retries with exponential backoff
- Never raise on bad records — log and skip

### appraiser.py
- Accept a county config dict
- Download bulk parcel data file (DBF format)
- Handle `__doPostBack` form submissions if needed
- Parse with `dbfread`
- Build lookup dict keyed by name variants:
  - `"FIRST LAST"` → parcel record
  - `"LAST FIRST"` → same parcel record
  - `"LAST, FIRST"` → same parcel record
- Return lookup dict

### matcher.py
- Accept a list of raw records + appraiser lookup dict
- For each record:
  - Extract owner name
  - Generate search variants
  - Use `rapidfuzz.fuzz.ratio()` to find best match
  - If ≥80%: attach all address fields
  - If 60-79%: attach but set `needs_review: true`
  - If <60%: set `needs_enrichment: true`
- Return enriched records

### scorer.py
- Accept a list of enriched records + scoring config
- For each record:
  - Start at base score (default 30)
  - Map doc_type → flag labels
  - Add points per flag
  - Check for combo bonuses (LP + foreclosure)
  - Check amount thresholds
  - Check recency (filed within current week)
  - Check if address was found
  - Cap at 100
- Return scored records

### exporter.py
- Accept scored records + county metadata
- **JSON export:**
  - Load existing `data/records.json` if it exists
  - Merge new records (deduplicate by `doc_num`)
  - Write wrapper: `{fetched_at, source, date_range, total, with_address, records}`
  - Write to both `data/records.json` and `dashboard/records.json`
- **CSV export:**
  - Write CRM-ready CSV to `data/export.csv`
  - Split owner name into First Name / Last Name
  - Map all fields per BLUEPRINT.md Section 13

### fetch.py (main entry point)
- Parse CLI args (`--county`, `--dry-run`)
- Load configs via config_loader
- For each county config:
  - Call clerk.scrape()
  - Call appraiser.load()
  - Call matcher.enrich()
  - Call scorer.score()
  - Call exporter.export()
- Log summary
- Exit 0 on success, 1 on failure

## Error Handling Strategy

```
Level 1 — Record-level:   try/except per record, log WARNING, skip, continue
Level 2 — Page-level:     retry 3x with backoff, then log ERROR and move to next lead type
Level 3 — County-level:   try/except per county, log ERROR, continue to next county
Level 4 — Run-level:      top-level catch in fetch.py, exit code 1, GitHub Actions logs failure
```

## Concurrency Model

- Playwright runs in async mode (single browser, sequential page navigation)
- Counties are processed sequentially (one at a time)
- No threading or multiprocessing needed at this scale
- Future: could parallelize counties with asyncio.gather()
