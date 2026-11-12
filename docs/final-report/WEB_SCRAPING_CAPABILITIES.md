# FinalSay Web Scraping Capabilities

**Assessment date:** 12 November 2026. Counts below derive from `backend/finalsay/adapters/live.py`, source onboarding code, and current SQLite state.

## Exact source/domain counts

| Measure | Count | Evidence/meaning |
|---|---:|---|
| Supported source adapter types | 6 | Thapar HTML, NITJ JSON API, IITB HTML, VTU examination catalog, Delhi examination catalog, Mumbai circulars |
| Explicit source profiles in current database | 6 | Three selected active profiles plus three older disabled profiles |
| Currently enabled profiles | 3 | VTU, University of Delhi Examination, University of Mumbai |
| Distinct configured institution portals (registrable domains) | 6 | `thapar.edu`, `nitj.ac.in`, `iitb.ac.in`, `vtu.ac.in`, `du.ac.in`, `mu.ac.in` |
| Exact allowlisted hostnames | 10 | See host list below; multiple hosts may belong to one institution domain |
| Domains with nonzero notice rows | 5 | Thapar, IITB, VTU, Delhi, Mumbai; NITJ has no notices in this DB |
| Profiles with scrape audit history | 6 | Every current DB source profile has at least one `scrape_run` row |
| Profiles with fully `ok` run(s) in current DB | 1 | VTU has 49 `ok` rows; other profiles have only partial/error records |

These numbers answer different questions. Adapter support is not proof every portal is currently reachable. A DB profile/run record is not a current successful poll. Five portals have notice rows; only VTU has fully successful `ok` poll records in the observed run table. All six adapter types have unit-level parser/client tests using controlled responses.

### Allowlisted hostnames

- Thapar: `www.thapar.edu`
- NIT Jalandhar: `nitj.ac.in`, `www.nitj.ac.in`, `v1.nitj.ac.in`
- IIT Bombay: `acad.iitb.ac.in`
- VTU: `vtu.ac.in`, `www.vtu.ac.in`
- University of Delhi Examination: `exam.du.ac.in`
- University of Mumbai: `mu.ac.in`, `www.mu.ac.in`

The configured URLs are explicit source entry points, not arbitrary crawler roots. Admins can create/update institution records only using one of the six supported scrape types, and each type validates the URL hostname against its fixed allowlist. There is no unlimited arbitrary-URL scraper. Thus there is a fixed current supported domain set, even though individual portal listings can expose many pages and notices.

Source trust is based on operator-selected portals plus host restrictions and transport/robots checks. There is no automated institutional trust score, signed-notice verification, or third-party fact-checking rank. An allowlisted URL provides provenance about where content was fetched; it does not itself certify that every linked notice is authentic or endorsed.

## Fetch and parse behavior

The live client uses Python standard-library HTTPS/urllib primitives, custom HTML parsers (`HTMLParser`), and JSON parsing. There is no BeautifulSoup, Selenium, Playwright, headless browser, RSS crawler, or general search-engine discovery subsystem. JavaScript-rendered pages requiring client-side execution are not supported by the existing fetch path.

Adapter behavior is specific to each portal:

- Thapar: parses the configured Notice list and follows selected PDF links.
- NITJ: reads the official JSON academic-notices feed and handles feed document links.
- IIT Bombay: parses academic circular listings and same-host PDFs.
- VTU: parses examination category/post pages and associated attachments; comment-anchor normalization avoids duplicate post identities.
- Delhi Examination: parses cards/links and supports offset pagination/backfill; third-party document URLs may remain listing metadata because external hosts are not fetched.
- Mumbai: parses circular rows/code/date and handles same-host PDF/JPG/JPEG/PNG attachments; OCR is supported for image scans; suspicious or unsupported uploads are skipped.

PDF discovery/download is implemented for supported adapter links. HTML listing metadata can be stored when a PDF cannot be safely fetched/extracted. External host attachments are skipped or represented as catalog metadata according to adapter logic. There is no guarantee that every notice file is reachable or text-extractable.

## Pagination, bounds, and cadence

The scheduler runs a startup poll and is configured for `FINALSAY_SCRAPE_INTERVAL_SECONDS=7200` (two hours) by default. Admins can also trigger manual polling. Manual and scheduled polls share a local lock; PostgreSQL uses an advisory lock for cross-process coordination. Backfills persist per-institution cursors and are bounded. A source can process at most 250 entries in the generic adapter path per poll; adapters may use smaller windows (for example, 25 recent plus a bounded historical page). Requests are sequential within a source client, with a minimum 0.5-second interval per host.

Safety/availability bounds include: HTTPS only; exact hostname allowlist; public-address DNS checks; robots policy checking; same-site redirect constraints; no embedded URL credentials or nonstandard ports; three request attempts for selected transient errors; 45-second overall response-read bound; and 20 MB caps for HTML/PDF (3 MB JSON). DNS is retried at most three times with bounded lookup waits. These limits improve safety but do not establish source permission/completeness or prevent upstream failures.

## Current database and run evidence

There are 2,284 official notice rows total (2,164 current, 120 historical). Selected sources contain 2,125 current rows: VTU 720, Delhi 415, Mumbai 990. Legacy current records remain for Thapar 24 and IIT Bombay 15; NITJ has none. Selected-source quality flags are: 1,338 extracted, 269 listing-only, 1 unextractable, 517 unknown. All selected current rows have publication-date values; no duplicate current source keys were found.

The scrape-run table contains 269 events: 49 `ok`, 145 `partial`, and 75 `error`. By portal: VTU has 49 ok/24 partial/22 error; Delhi 52 partial/26 error; Mumbai 40 partial/20 error. Thapar and IITB have historic partial/error events; NITJ has an error event. The newest selected-source records are errors; examples include DNS resolution failure and bad file descriptor. Scheduled polling code and tests exist, but a poll cycle can fail and still be audited without changing records.

## Duplicate and revision behavior

Each adapter supplies a stable external identifier when available, otherwise a normalized source URL becomes the key. A partial unique database index enforces one *current* official record per `(institution, source_key)`. Repeated unchanged polling refreshes retrieval metadata without adding duplicate rows. Changed bytes/content for the same source key can create a new version linked by `previous_version_id`; the old row remains historical.

If the institution posts a revised notice at a new URL with a new source key, the scraper stores it as a separate notice. Student candidate retrieval can find semantically related current notices and the comparison model can classify the old submitted content against it; however, no universal cross-URL version link is automatically created. This distinction is material for institutional portals where each revision receives a new listing URL.

## Tested versus theoretically supported

- **Unit-tested:** adapter parsers, retries, pagination, file limits, allowlists, robots behavior, DNS checks, source-key normalization, and selected PDF/image handling, with mocked/fixture responses.
- **Live database evidence:** 6 configured profiles and run history; 5 profiles have notice rows; 49 fully `ok` runs are currently recorded for VTU. Current selected source cycles show errors/partial results.
- **Not established:** an exhaustive crawl of any portal, real-world completeness percentage, all linked PDFs being available, successful processing of every language/scan, or full backfill of every site's archive.
- **Theoretical ceiling:** no universal crawl count can be inferred. The architecture supports six explicit adapters and their allowlisted hosts; it does not support arbitrary institutions merely because their pages are public.
