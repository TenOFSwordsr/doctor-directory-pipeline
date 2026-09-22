# Doctoreto / Nobat Scraper and Directory Import Pipeline

The local, Windows-side half of the visital.ir doctor-directory build: scrapers for the two
Iranian doctor sites (doctoreto.com and nobat.ir), a dedupe/merge layer, and importers that push
the result into the clinic directory API (see the `clinic-import-api` repo). It also carries
the API client, an offline mock of the whole API for testing, and per-city run state. The current
shape is documented in `AHURA_PIPELINE.md` and `SCRAPER_GUIDE.md`.

**Suggested repo name:** `doctor-directory-pipeline`
**Stack:** Python 3 (`.venv`), stdlib `urllib`/`ssl`, `concurrent.futures`, pandas + openpyxl for the Excel read/write, Selenium (legacy scripts only)
**Status:** active
**Last modified:** 2026-09-13

## What it does

**Scraping.** `scrape_license_specialty.py` is the current entry point: it drives both sites for
one city or all twelve and exports only نام پزشک / شماره نظام پزشکی / تخصص / تخصص‌های پزشک /
شهر / منبع / لینک. The two engines underneath it:

- `doctoreto_scraper.py` - parses doctoreto's `__NEXT_DATA__` (`medicalNumber`, full
  `specialities` array with fellowships) with an HTML fallback; `--live` skips the cached
  server listings.
- `nobat_scraper.py` - no browser: reads JSON-LD `identifier.IR-MedLicense` from
  `/find/city-N/c-PAGE/` listings and profiles, multithreaded with resumable checkpoints.
- `scraper_comments_server.py` - patient reviews («نظرات») per city from saved profile HTML,
  3 s polite delay, checkpoint every 25 doctors; `--test` runs offline against `profile_dump.html`.
- `scraper.py` / `scraper_comments.py` - the older single-city Selenium version, kept but superseded.

**Merging.** `merge_cities.py` joins doctoreto and nobat per city into `merged/<city>_combined.xlsx`
with a منبع tab (doctoreto / nobat / هر دو) and a «جفت‌های تکراری» sheet proving which key matched
each pair. Matching uses شماره نظام پزشکی when present, else normalized full name (strips «دکتر»,
spaces, ZWNJ) - because doctoreto never exposes the council number. Doctoreto wins on ties; the
nobat rating and link survive in meta. `build_isfahan_excel.py`, `build_both_sites_doctors.py` and
`build_category_mapping.py` are the Isfahan-specific audits that produced that strategy.

**Importing.** `ahura_api.py` wraps every `/wp-json/ahura/v1` endpoint and maps Excel rows to
payloads. `import_server_cities.py` (server exports → doctors + offices + thumbnails),
`import_nobat.py` (merged Excel, source-aware `external_id`s, nobat reviews as comments), and
`import_to_ahura.py` (comments Excel). City-specific one-shot drivers: `import_isfahan_live.py`,
`import_qom_live.py`, `import_qom_full.py`, `import_shiraz_fast.py`, `import_shiraz_full.py`,
`tehran10_import.py`, `qom_full_scrape.py`. `mock_ahura_api.py` runs the same API on port 8090 so
imports can be exercised without touching production.

## Layout

```
scrape_license_specialty.py   unified two-site license/specialty scraper (entry point)
doctoreto_scraper.py          __NEXT_DATA__ parser      nobat_scraper.py    JSON-LD parser
scraper_comments_server.py    per-city review scraper   scraper*.py         legacy Selenium
merge_cities.py               dedupe + combined Excel   ahura_api.py        API client
import_server_cities.py       server exports -> API     import_nobat.py     merged -> API
mock_ahura_api.py             offline API mock :8090    fix_author_names.py bulk comment-author rename
AHURA_PIPELINE.md             full build log, run order, verified counts
SCRAPER_GUIDE.md              Persian usage guide for the license/specialty scrapers
server-exports/<city>/        input: 12-city Excel + images scraped on the server
exports_license/ exports_nobat/ merged/ exports/   generated output, not source
```

## Running it

Everything runs through the local venv. Full run order for one city (from `AHURA_PIPELINE.md`):

```bash
.venv\Scripts\python scrape_license_specialty.py --city qom --limit 10     # quick test
.venv\Scripts\python scrape_license_specialty.py --all-cities --threads 12
.venv\Scripts\python nobat_scraper.py --city tabriz
.venv\Scripts\python merge_cities.py --city tabriz
.venv\Scripts\python mock_ahura_api.py                                     # terminal 1
.venv\Scripts\python import_nobat.py --city tabriz --mock                  # verify offline
.venv\Scripts\python import_nobat.py --city tabriz                         # go live
```

`--dry` / `--mock` / `--limit` are supported on the importers. Every import is an idempotent
upsert (`doctoreto-<profile-hash>` / `nobat-<id>`), so reruns are safe; `*_import_state.json`
files track progress.

## Notes

- **`_ahura_key.txt` at this root holds the live visital.ir API key.** Delete it before making
  the repo public and rotate via `POST /key/rotate`. Most importers also read the absolute path
  `C:\Users\Administrator\Documents\Projects\_ahura_key.txt`, so the key must be supplied another
  way once published.
- `requirements.txt` is **empty** despite pandas/openpyxl/Selenium being imported. Pin the real
  dependencies before publishing. No `.gitignore` either - `.venv/`, `__pycache__/`,
  `chrome_profile*/`, `*.log`, `*.tgz` and the generated `merged/`, `exports*/` trees are all
  currently trackable.
- 37 `_*`-prefixed files plus scattered `*.log`s at the root are debugging residue from live runs
  (`_tehran_*.json`, `_isfahan_delete_payload*.json`, `_pma_last_count.html`); `bahr2.html`,
  `card_dump.html`, `ms.html`, `profile_dump.html` are saved pages used as offline fixtures.
- `fix_author_names.py` bypasses the REST API and rewrites comment authors directly in MySQL
  through a cPanel phpMyAdmin session - it needs browser cookies and is not reproducible offline.
- Upstream producer of `server-exports/` is the sibling project in
  `Documents/Projects/doctoreto-server` (`scrape_tabriz.py`, run at `/root/doctoreto-scraper` on
  the author's VPS). Verified end state there: 2,580 unique Tabriz doctors, re-run produced zero
  duplicates.
- `AHURA-API.md` recommends the `requests` library; the client actually uses `urllib.request`,
  so no third-party HTTP dependency is needed.
