# Implementation Specification

This is the detailed technical spec for every module. Claude Code should use this as the primary reference when writing code.

---

## 1. config_loader.py

### Purpose
Load and validate YAML configuration files.

### Functions

```python
def load_scoring_config(path: str = "config/scoring.yaml") -> dict:
    """Load scoring weights. Return default weights if file missing."""

def load_county_configs(directory: str = "config/counties", county_filter: str = None) -> list[dict]:
    """
    Load all .yaml files from directory.
    If county_filter provided, load only that file (e.g., 'lubbock_tx' loads 'lubbock_tx.yaml').
    Validate each config has required fields.
    Raise ValueError if a config is missing required fields.
    """

REQUIRED_COUNTY_FIELDS = [
    "county", "state", "clerk_portal_url", "lead_types"
]

OPTIONAL_COUNTY_FIELDS = {
    "appraiser_url": None,
    "lookback_days": 7,
    "appraiser_file_format": "dbf",
    "appraiser_columns": {}  # column name mapping overrides
}
```

### County Config Schema (`config/counties/*.yaml`)

```yaml
county: "Lubbock"
state: "TX"
clerk_portal_url: "https://example.com/clerk"
appraiser_url: "https://example.com/appraiser"
lookback_days: 7
appraiser_file_format: "dbf"   # "dbf" or "csv"

# Override default column names if this county uses different ones
appraiser_columns:
  owner: "OWN1"          # default: "OWNER"
  site_addr: "SITEADDR"  # default: "SITE_ADDR"
  site_city: "SITE_CITY"
  site_zip: "SITE_ZIP"
  mail_addr: "MAILADR1"  # default: "ADDR_1"
  mail_city: "MAILCITY"  # default: "CITY"
  mail_state: "STATE"
  mail_zip: "MAILZIP"    # default: "ZIP"

lead_types:
  - code: "LP"
    label: "Lis Pendens"
    flags: ["Lis pendens"]
  - code: "NOFC"
    label: "Notice of Foreclosure"
    flags: ["Pre-foreclosure"]
  - code: "TAXDEED"
    label: "Tax Deed"
    flags: ["Tax lien"]
  - code: "JUD"
    label: "Judgment"
    flags: ["Judgment lien"]
  - code: "CCJ"
    label: "Certified Judgment"
    flags: ["Judgment lien"]
  - code: "DRJUD"
    label: "Domestic Judgment"
    flags: ["Judgment lien"]
  - code: "LNCORPTX"
    label: "Corporate Tax Lien"
    flags: ["Tax lien", "LLC / corp owner"]
  - code: "LNIRS"
    label: "IRS Lien"
    flags: ["Tax lien"]
  - code: "LNFED"
    label: "Federal Lien"
    flags: ["Tax lien"]
  - code: "LN"
    label: "Lien"
    flags: []
  - code: "LNMECH"
    label: "Mechanic Lien"
    flags: ["Mechanic lien"]
  - code: "LNHOA"
    label: "HOA Lien"
    flags: []
  - code: "MEDLN"
    label: "Medicaid Lien"
    flags: []
  - code: "PRO"
    label: "Probate"
    flags: ["Probate / estate"]
  - code: "NOC"
    label: "Notice of Commencement"
    flags: []
  - code: "RELLP"
    label: "Release Lis Pendens"
    flags: []
```

### Scoring Config Schema (`config/scoring.yaml`)

```yaml
base_score: 30
per_flag_bonus: 10
combo_bonus:
  types: ["LP", "NOFC"]   # lis pendens + foreclosure on same owner
  points: 20
amount_thresholds:
  - min: 100000
    points: 15
  - min: 50000
    points: 10
recency_bonus:
  days: 7
  points: 5
address_bonus: 5
max_score: 100
```

---

## 2. clerk.py

### Purpose
Scrape county clerk portal using Playwright (async).

### Functions

```python
async def scrape_clerk(config: dict) -> list[dict]:
    """
    Main entry point. Launches browser, scrapes all lead types, returns raw records.

    Args:
        config: County config dict from config_loader

    Returns:
        List of raw record dicts with keys:
        doc_num, doc_type, filed, owner, grantee, amount, legal, clerk_url
    """

async def search_lead_type(page, lead_type: dict, start_date: str, end_date: str) -> list[dict]:
    """
    Search clerk portal for a single lead type within date range.
    Handle pagination. Return list of raw record dicts.
    """

async def parse_result_row(row_element, lead_type: dict, base_url: str) -> dict | None:
    """
    Extract fields from a single result table row.
    Return None if row can't be parsed (log warning).
    """

async def retry_async(coro_func, max_attempts: int = 3, backoff: float = 2.0):
    """
    Retry an async function up to max_attempts times with exponential backoff.
    Log each retry attempt. Raise after final failure.
    """
```

### Behavior Details

- **Browser**: Chromium, headless mode, default viewport
- **Date range**: `(today - lookback_days)` to `today`, formatted as the portal expects
- **Pagination**: After extracting current page results, check for "Next" button/link. If exists, click and continue. If not, done.
- **Record parsing**: Each clerk portal is different. The code should be structured so the parsing logic (CSS selectors, table column indices) can be overridden per county config if needed. Start with a generic table parser.
- **Amount field**: Parse to integer cents or float dollars. Strip `$`, commas. If unparseable, set to `None`.
- **clerk_url**: Construct direct link to document. Pattern will vary by county — store URL template in config.
- **Timeout**: 30 seconds per page load. If timeout, retry.

### Error Handling
- Bad row → log WARNING with row content, return None, continue
- Page timeout → retry up to 3x, then log ERROR, move to next lead type
- Browser crash → log ERROR, attempt restart, if fails again skip county

---

## 3. appraiser.py

### Purpose
Download and parse property appraiser bulk parcel data. Build owner name lookup.

### Functions

```python
def download_appraiser_data(config: dict) -> Path:
    """
    Download bulk parcel file to temp directory.
    Handle __doPostBack form submissions if needed.
    Return path to downloaded file.
    Uses requests library (not Playwright).
    3 retries with backoff.
    """

def parse_parcel_file(file_path: Path, config: dict) -> list[dict]:
    """
    Parse DBF (or CSV) file into list of parcel dicts.
    Use config['appraiser_columns'] for column name mapping.
    Normalize all text to uppercase, strip whitespace.
    """

def build_name_lookup(parcels: list[dict]) -> dict[str, dict]:
    """
    Build lookup dict keyed by name variants.
    For each parcel:
      - Parse owner name into first/last
      - Create keys: "FIRST LAST", "LAST FIRST", "LAST, FIRST"
      - All keys uppercased
      - Value: parcel dict with address fields
    Handle edge cases:
      - Multiple owners (e.g., "SMITH JOHN & SMITH JANE") — index both
      - Corporate owners (e.g., "ABC HOLDINGS LLC") — index as-is
      - Trusts — index as-is
    Return lookup dict.
    """

def load_appraiser(config: dict) -> dict[str, dict]:
    """
    Convenience function: download → parse → build lookup.
    If appraiser_url is None, return empty dict (skip enrichment).
    """
```

### DBF Column Mapping Defaults

```python
DEFAULT_COLUMNS = {
    "owner": "OWNER",
    "site_addr": "SITE_ADDR",
    "site_city": "SITE_CITY",
    "site_zip": "SITE_ZIP",
    "mail_addr": "ADDR_1",
    "mail_city": "CITY",
    "mail_state": "STATE",
    "mail_zip": "ZIP",
}
```

Apply overrides from `config['appraiser_columns']` on top of defaults.

---

## 4. matcher.py

### Purpose
Fuzzy match clerk records against appraiser name lookup to attach addresses.

### Functions

```python
def enrich_records(records: list[dict], lookup: dict[str, dict]) -> list[dict]:
    """
    For each record, attempt to match owner name to appraiser lookup.
    Mutates records in place (adds address fields + match metadata).
    Returns the same list.
    """

def match_owner(owner_name: str, lookup: dict[str, dict]) -> tuple[dict | None, float]:
    """
    Generate name variants from owner_name.
    First try exact match (fast path).
    Then fuzzy match against all lookup keys using rapidfuzz.process.extractOne().
    Return (best_match_parcel, score).
    """

def generate_name_variants(name: str) -> list[str]:
    """
    From a raw name string, generate search variants:
    - Original (uppercased, stripped)
    - "FIRST LAST"
    - "LAST FIRST"
    - "LAST, FIRST"
    Handle: "LAST, FIRST M", "FIRST M LAST", "LAST FIRST MIDDLE"
    Return list of variant strings.
    """

MATCH_THRESHOLD_STRONG = 80   # auto-link
MATCH_THRESHOLD_WEAK = 60     # link but flag
```

### Match Result Fields Added to Record

```python
# On strong match (≥80%):
record["prop_address"] = parcel["site_addr"]
record["prop_city"] = parcel["site_city"]
record["prop_state"] = config["state"]
record["prop_zip"] = parcel["site_zip"]
record["mail_address"] = parcel["mail_addr"]
record["mail_city"] = parcel["mail_city"]
record["mail_state"] = parcel["mail_state"]
record["mail_zip"] = parcel["mail_zip"]
record["match_score"] = 85.5
record["needs_enrichment"] = False
record["needs_review"] = False

# On weak match (60-79%):
# Same fields, but:
record["needs_review"] = True
record["needs_enrichment"] = False

# On no match (<60%):
record["prop_address"] = None
# ... all address fields None
record["needs_enrichment"] = True
record["needs_review"] = False
```

---

## 5. scorer.py

### Purpose
Calculate seller score (0-100) for each record based on distress signals.

### Functions

```python
def score_records(records: list[dict], scoring_config: dict, lead_type_map: dict) -> list[dict]:
    """
    Score all records. Mutates in place.
    lead_type_map: {doc_type_code: lead_type_config_dict} for flag lookup.
    Returns records sorted by score descending.
    """

def calculate_score(record: dict, scoring_config: dict, lead_type_map: dict) -> tuple[int, list[str]]:
    """
    Calculate score for a single record.
    Returns (score, flags_list).
    """

def check_combo_bonus(owner: str, all_records: list[dict], combo_types: list[str]) -> bool:
    """
    Check if the same owner appears in records with multiple combo types.
    E.g., same owner has both LP and NOFC records.
    """
```

### Scoring Algorithm (step by step)

```
1. score = base_score (30)
2. flags = lead_type_map[record["doc_type"]]["flags"]  # from config
3. Check if owner name suggests LLC/corp → add "LLC / corp owner" flag
4. Check if filed date is within recency window → add "New this week" flag
5. score += len(flags) * per_flag_bonus (10)
6. If combo_bonus applies (owner has both LP + NOFC): score += combo_bonus (20)
7. If amount > 100000: score += 15
8. Elif amount > 50000: score += 10
9. If record has property address (not None): score += address_bonus (5)
10. score = min(score, max_score)  # cap at 100
11. record["score"] = score
12. record["flags"] = flags
```

---

## 6. exporter.py

### Purpose
Write output files: JSON (dual location) and CSV (CRM export).

### Functions

```python
def export_json(records: list[dict], county_config: dict, date_range: tuple[str, str]):
    """
    Build output wrapper, merge with existing data, write to both locations.
    """

def export_csv(records: list[dict], output_path: str = "data/export.csv"):
    """
    Write CRM-ready CSV. Split owner into first/last name.
    """

def merge_records(existing: list[dict], new: list[dict]) -> list[dict]:
    """
    Merge by doc_num. New records overwrite existing with same doc_num.
    Records not in new list are preserved.
    Return merged list sorted by score descending.
    """

def split_name(full_name: str) -> tuple[str, str]:
    """
    Best-effort split into (first_name, last_name).
    'SMITH, JOHN' → ('JOHN', 'SMITH')
    'JOHN SMITH' → ('JOHN', 'SMITH')
    'ABC HOLDINGS LLC' → ('ABC HOLDINGS', 'LLC')
    """
```

### JSON Output Structure

```python
{
    "fetched_at": "2026-04-10T07:00:00Z",     # ISO 8601 UTC
    "source": "{county} County Clerk",
    "date_range": "{start_date} to {end_date}",
    "total": len(records),
    "with_address": len([r for r in records if r.get("prop_address")]),
    "records": records
}
```

### CSV Columns (in order)

```
First Name, Last Name, Mailing Address, Mailing City, Mailing State,
Mailing Zip, Property Address, Property City, Property State, Property Zip,
Lead Type, Document Type, Date Filed, Document Number, Amount/Debt Owed,
Seller Score, Motivated Seller Flags, Source, Public Records URL
```

### CSV Field Mapping

| CSV Column | Record Field | Transform |
|---|---|---|
| First Name | owner | split_name()[0] |
| Last Name | owner | split_name()[1] |
| Mailing Address | mail_address | as-is |
| Mailing City | mail_city | as-is |
| Mailing State | mail_state | as-is |
| Mailing Zip | mail_zip | as-is |
| Property Address | prop_address | as-is |
| Property City | prop_city | as-is |
| Property State | prop_state | as-is |
| Property Zip | prop_zip | as-is |
| Lead Type | cat_label | as-is |
| Document Type | doc_type | as-is |
| Date Filed | filed | as-is |
| Document Number | doc_num | as-is |
| Amount/Debt Owed | amount | format as dollar string or empty |
| Seller Score | score | as-is |
| Motivated Seller Flags | flags | join with "; " |
| Source | — | "{county} County Clerk" |
| Public Records URL | clerk_url | as-is |

---

## 7. fetch.py (main entry point)

### Purpose
Orchestrate the full pipeline. This is what gets executed.

### CLI Arguments

```
python scraper/fetch.py [options]

--county NAME     Run only this county (matches yaml filename without extension)
--dry-run         Validate configs and test connectivity, don't write output
--log-level LEVEL Set logging level (DEBUG, INFO, WARNING, ERROR). Default: INFO
```

### Main Flow

```python
import asyncio, argparse, logging, sys
from config_loader import load_county_configs, load_scoring_config
from clerk import scrape_clerk
from appraiser import load_appraiser
from matcher import enrich_records
from scorer import score_records
from exporter import export_json, export_csv

async def run_county(config, scoring_config):
    """Process a single county. Returns (success: bool, record_count: int)."""
    raw_records = await scrape_clerk(config)
    lookup = load_appraiser(config)
    enriched = enrich_records(raw_records, lookup)
    lead_type_map = {lt["code"]: lt for lt in config["lead_types"]}
    scored = score_records(enriched, scoring_config, lead_type_map)

    start = (today - timedelta(days=config.get("lookback_days", 7))).isoformat()
    end = today.isoformat()
    export_json(scored, config, (start, end))
    export_csv(scored)

    return True, len(scored)

async def main():
    args = parse_args()
    setup_logging(args.log_level)

    scoring = load_scoring_config()
    counties = load_county_configs(county_filter=args.county)

    results = {}
    for config in counties:
        name = f"{config['county']}, {config['state']}"
        try:
            success, count = await run_county(config, scoring)
            results[name] = f"OK — {count} records"
        except Exception as e:
            logging.error(f"County {name} failed: {e}")
            results[name] = f"FAILED — {e}"

    # Print summary
    for name, status in results.items():
        logging.info(f"  {name}: {status}")

    if any("FAILED" in s for s in results.values()):
        sys.exit(1)

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 8. dashboard/index.html

### Purpose
Static HTML scoreboard. No build step, no framework. Vanilla HTML + JS + CSS.

### Behavior

1. On page load, fetch `records.json` (relative path, same directory)
2. Parse JSON, extract `records` array
3. Render HTML table with all fields
4. Color-code rows:
   - `score >= 70` → red background (`#ffcccc`)
   - `score >= 40` → orange background (`#ffe0b2`)
   - `score < 40` → white background
5. Default sort: score descending
6. Show header stats: total records, records with addresses, last fetched date, date range
7. Clickable `doc_num` links to `clerk_url`
8. Column headers clickable for sorting

### Table Columns

```
Score | Flags | Owner | Doc Type | Filed | Amount | Property Address | City | Doc # (linked) | Status
```

Status column shows enrichment status:
- Matched → green checkmark
- Needs Review → yellow warning
- Needs Enrichment → red X

---

## 9. .github/workflows/scrape.yml

### Spec

```yaml
name: County Scraper — Auto Run
on:
  schedule:
    - cron: '0 7 * * *'      # Daily at 7:00 AM UTC
  workflow_dispatch:           # Manual trigger

jobs:
  scrape-and-enrich:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r scraper/requirements.txt

      - name: Install Playwright browsers
        run: python -m playwright install --with-deps chromium

      - name: Run scraper
        run: python scraper/fetch.py

      - name: Commit updated leads
        run: |
          git config user.name 'github-actions'
          git config user.email 'actions@github.com'
          git add data/ dashboard/
          git diff --quiet || git commit -m 'Auto: leads refreshed'
          git push

      - name: Log failure
        if: failure()
        run: echo 'Scraper failed — check logs for details'
```

---

## 10. Cross-Cutting Concerns

### 10.1 The `cat` Field

The `cat` field in output records is set equal to `doc_type`. It exists for downstream CRM compatibility. In the exporter, set it explicitly:

```python
record["cat"] = record["doc_type"]
record["cat_label"] = lead_type_map[record["doc_type"]]["label"]
```

### 10.2 Internal vs Output Fields

These fields appear in final JSON output (records.json):
- All fields from Section 4.1 of BLUEPRINT.md
- `needs_enrichment` (bool) — included so dashboard can show status
- `needs_review` (bool) — included so dashboard can show status

These fields are **internal only** (NOT in output):
- `match_score` (float) — fuzzy match percentage, used for debugging only

### 10.3 CSV Null Handling

All `None`/`null` values in CSV output must be written as empty strings, never the literal text `"None"`. Use:
```python
value if value is not None else ""
```

### 10.4 Logging Specification

Every module should use:
```python
import logging
logger = logging.getLogger(__name__)
```

The `setup_logging()` function in fetch.py:
```python
def setup_logging(level: str = "INFO"):
    logging.basicConfig(
        level=getattr(logging, level.upper(), logging.INFO),
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
    )
```

Log level guidelines:
- `DEBUG` — raw HTML snippets, full match details, parsed row data
- `INFO` — county start/finish, lead type counts, total records, export paths
- `WARNING` — skipped records (bad parse), weak matches, missing fields
- `ERROR` — failed lead types, county failures, file I/O errors

### 10.5 Retry Strategy

All network requests (Playwright page loads and requests.get/post) use this pattern:

```python
async def retry_async(coro_func, max_attempts=3, base_delay=2.0):
    """
    Exponential backoff: attempt 1 → wait 2s → attempt 2 → wait 4s → attempt 3 → raise.
    Each attempt gets its own timeout (30s for Playwright pages, 60s for file downloads).
    """
    for attempt in range(1, max_attempts + 1):
        try:
            return await coro_func()
        except Exception as e:
            if attempt == max_attempts:
                raise
            delay = base_delay * (2 ** (attempt - 1))
            logger.warning(f"Attempt {attempt}/{max_attempts} failed: {e}. Retrying in {delay}s...")
            await asyncio.sleep(delay)
```

For synchronous requests (appraiser download), use equivalent sync version with `time.sleep()`.

### 10.6 Error Handling Matrix

| Scope | Failure | Action | Continue? |
|-------|---------|--------|-----------|
| Single record | Bad parse / missing fields | Log WARNING, skip record | Yes — next record |
| Single page | Timeout after 3 retries | Log ERROR, stop this lead type | Yes — next lead type |
| Single lead type | All pages failed | Log ERROR | Yes — next lead type |
| Appraiser download | File unavailable after 3 retries | Log ERROR, skip enrichment (all records get needs_enrichment=true) | Yes — scoring still runs |
| Single county | Unhandled exception | Log ERROR, catch in fetch.py | Yes — next county |
| Entire run | All counties failed | Log ERROR, exit code 1 | No — exit |
| Partial success | Some records scraped, some failed | Write what we have | Yes — export partial results |

Key rule: **Always write output for whatever we successfully scraped**, even if some lead types or enrichment failed. Partial data is better than no data.

### 10.7 County Config File Naming

County config filenames must be lowercase, format: `{county}_{state}.yaml`
- Example: `lubbock_tx.yaml`, `miami_dade_fl.yaml`
- The `--county` CLI arg matches the filename without `.yaml` extension
- Matching is case-insensitive

### 10.8 Appraiser Format Switching

In `appraiser.py`, `parse_parcel_file()` checks `config["appraiser_file_format"]`:
```python
if config.get("appraiser_file_format", "dbf") == "csv":
    # use csv.DictReader
else:
    # use dbfread.DBF
```

### 10.9 Git Workflow Commit Logic

The GitHub Actions workflow uses `git diff --quiet || git commit` which means:
- `git diff --quiet` exits 0 if no changes → skip commit (no new data)
- `git diff --quiet` exits 1 if changes exist → run commit and push
- This prevents empty commits when no new records are found
