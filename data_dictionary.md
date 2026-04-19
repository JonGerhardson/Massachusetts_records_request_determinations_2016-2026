# Data Dictionary — `spr_appeals.db`

SQLite database of Massachusetts Public Records Division (PRD) appeals and their determination letters, scraped from the Secretary of State's [AppealsWeb](https://www.sec.state.ma.us/AppealsWeb) system plus the determination PDFs themselves.

Source of truth for dates and categorical fields is the **AppealsWeb** tracker (`appeals` + `case_details`). The `determinations` table adds full-text search and regex-extracted fields over the PDF letters.

**Case number scheme:** `YYYYNNNN` (year + 4-digit sequence), e.g. `20243445` = appeal #3445 filed in 2024 (= `SPR24/3445` in PDF text).

---

## Table: `appeals` (28,318 rows)
One row per appeal. Populated from the AppealsWeb search-results HTML.

| Column | Type | Notes |
|---|---|---|
| `case_no` | TEXT PK | `YYYYNNNN`. Joins to every other table. |
| `year` | INTEGER | Year portion of `case_no`. Re-derivable via `CAST(substr(case_no,1,4) AS INTEGER)`. |
| `opened` | TEXT | Date appeal was filed, **`MM-DD-YYYY`**. |
| `extended_deadline` | TEXT | `MM-DD-YYYY` if an extension was granted. |
| `closed` | TEXT | Date appeal was resolved, **`MM-DD-YYYY`**. Empty for open cases. |
| `type` | TEXT | `Appeal` (90%) \| `Fee Petition` (7%) \| `Time Petition` (3%). |
| `subtype` | TEXT | e.g. `Initial`, `Reconsideration`, `In Camera`. |
| `status` | TEXT | `Closed` (99.4%) \| `Open` (0.6%). |
| `requester` | TEXT | Requester name as entered in the tracker (may differ from the name printed on the PDF). |
| `custodian` | TEXT | Records-custodian agency name. |
| `determination_count` | INTEGER | Number of determination PDFs the tracker lists for this case. |
| `detail_url` | TEXT | AppealsWeb detail-page URL. |
| `parsed_at` | TIMESTAMP | When the row was scraped. |

**Quality notes**
- `appeals.opened` is the authoritative appeal-opened date (used to backfill `determinations.appeal_opened`).
- `appeals.closed` is the authoritative resolution date (used to backfill `determinations.closed_date`). Empty on the 156 currently-open cases.

---

## Table: `case_details` (28,318 rows)
One row per appeal, scraped from each appeal's detail page (richer than `appeals`).

| Column | Type | Notes |
|---|---|---|
| `case_no` | TEXT PK | FK → `appeals.case_no`. |
| `case_type`, `case_subtype`, `status` | TEXT | Same semantics as `appeals.type/subtype/status`. |
| `requester`, `custodian` | TEXT | Slightly cleaner than the search-results versions. Used to backfill `determinations.requester_name/custodian_agency`. |
| `date_request_submitted` | TEXT | `MM-DD-YYYY`. Date the **original public-records request** was made to the agency (precedes `opened`). Often empty. |
| `response_provided_date` | TEXT | `MM-DD-YYYY`. Date the agency responded to the original request. |
| `processing_fees_charged` | REAL | Dollar amount the agency quoted/charged. |
| `petitions_submitted_regarding_fees` | TEXT | `Yes`/`No`. |
| `determination` | TEXT | Free-text summary from the tracker. |
| `time_required_to_comply` | TEXT | Days the agency was given. |
| `date_initial_opened` / `date_initial_closed` / `initial_extension_deadline` | TEXT | Dates for the **initial** determination phase. |
| `date_reconsideration_opened` / `..._closed` / `..._extension_deadline` | TEXT | Dates for the **reconsideration** phase, if any. |
| `date_in_camera_opened` / `..._closed` / `..._extension_deadline` | TEXT | Dates for the **in-camera-review** phase, if any. |
| `did_request_go_to_court` | TEXT | `Yes`/`No`. |
| `scraped_at` | TIMESTAMP | When the detail page was scraped. |

**Quality notes**
- The `date_initial_*` / `date_reconsideration_*` / `date_in_camera_*` triples are the best source for per-stage timing — richer than `determinations.stage`/`stage_type` which are inferred from filenames + regex.
- Dates are `MM-DD-YYYY` strings; empty strings (not NULL) for missing values.

---

## Table: `pdf_downloads` (29,674 rows)
Tracks which PDFs were downloaded. One row per file.

| Column | Type | Notes |
|---|---|---|
| `case_no` | TEXT | FK → `appeals.case_no`. |
| `pdf_index` | INTEGER | 1-based position within the case. PK is `(case_no, pdf_index)`. |
| `filename` | TEXT | Relative path under `pdfs/`. Stem convention: `{case_no}` for index 1, `{case_no}_{N}` for N ≥ 2. |
| `downloaded_at` | TIMESTAMP | When the file was pulled. |

~1,552 cases have multiple PDFs (reconsideration / in-camera-review / supplemental follow-ups).

---

## Table: `determinations` (29,674 rows)
One row **per PDF** (not per case). The main analysis table. Columns fall into three groups: identity, regex-extracted from PDF text, and backfilled from `appeals`/`case_details`.

### Identity
| Column | Type | Notes |
|---|---|---|
| `pdf_stem` | TEXT PK | Filename stem: `20241234` or `20241234_2`. Unique per PDF. |
| `case_no` | TEXT | Case ID. Many rows per case if multi-stage. FK → `appeals.case_no`. |
| `stage` | INTEGER | 1 for initial letter, 2 for second letter, etc. Parsed from filename suffix. |
| `stage_type` | TEXT | `NULL` for stage=1; one of `reconsideration` / `in_camera` / `supplemental` for stage ≥ 2. Inferred from first 2500 chars of text. Accuracy verified ~97% for `reconsideration`/`in_camera` after regex fix; `supplemental` is a real catch-all (re-orders, new developments, clarifications). |
| `pdf_path` | TEXT | Relative path to the PDF on disk. |
| `text` | TEXT | Full `pdftotext -layout` output. Scanned PDFs were OCR'd via `ocrmypdf` (608 of them, all 2016–2020). |
| `indexed_at` | TIMESTAMP | When the row was last (re)indexed. |

### Regex-extracted from PDF text
These reflect what was **printed in the letter**, not necessarily what is authoritative. Use for forensic detail; prefer backfilled fields for canonical values.

| Column | Type | Notes |
|---|---|---|
| `appeal_number` | TEXT | Same as `case_no` (extracted from SPR number in text). |
| `determination_date` | TEXT | `YYYY-MM-DD`. Date **printed** at top of letter. Often matches `closed_date` but ~5% of stage-1 letters diverge — mostly normal admin delay (letter dated days before case closed), plus a handful of typos in the source letters themselves (wrong year, missing digits). **Prefer `closed_date` for analysis.** 1,145 nulls (regex miss). |
| `requester_name` | TEXT | Backfilled from `case_details.requester` if the regex missed. 4 nulls. |
| `requester_affiliation` | TEXT | Org name when the text has `"...of/from/on behalf of <org>..."`. High miss rate — regex only fires when that phrasing is used. |
| `custodian_agency` | TEXT | Backfilled from `case_details.custodian` if the regex missed. 4 nulls. |
| `exemptions_claimed` | TEXT | Pipe-separated list, e.g. `Exemption (a)\|Exemption (c)`. Five regex patterns: (1) window-based scan after `Exemption(s)` anchors (tolerates `[E]xemption`), (2) `§/section/sec. 7(26)(x)` statute citations, (3) `cl./clause 26 (x), (y)...` boilerplate, (4) `Exemption A/C/F` capitalized-name format, (5) statute-inferred secondary pass. Validated against 126-row manual audit: 97/126 exact match, 0 false positives on "none" rows. 9,034 non-null rows (30.4%). |
| `dispute_type` | TEXT | Set only when `exemptions_claimed IS NULL`. Classifies **why** no exemption was extracted. Cascade: `fee_petition` / `time_petition` (from `appeals.type`) → `stage2_plus` (stage ≥ 2) → `nonresponse` (custodian never responded) → `no_records` / `no_duty` / `withdrawn` / `denial` / `fee_only` / `insufficient_petition` (procedural closures) → `substantive_no_exemption` (custodian responded substantively, no exemption formally claimed) → `stonewall` (custodian only acknowledged/extended time) → `resolved_after_appeal` (custodian provided records after appeal) → `other`. 20,640 non-null rows. |
| `statutes_cited` | TEXT | Pipe-separated `G.L.` / `M.G.L.` chapter/section refs with nested sub-clauses preserved, e.g. `G.L. c. 66, § 10(b)(viii)\|G.L. c. 4, § 7(26)(c)`. Text is normalized (newlines collapsed, OCR `§ 1O`/`§ lOA` → `§ 10`/`§ 10A`) before matching. Spacing and punctuation variants still produce near-duplicates (e.g., `"G. L. c. 66, § 10A"` vs `"G. L. c. 66 § 10A"`). |
| `regulations_cited` | TEXT | Pipe-separated C.M.R. refs with sub-clauses, e.g. `950 C.M.R. 32.08(2)(b)`. Periods in `C.M.R.` are optional (catches bare `"950 CMR 32.08"`). |
| `case_citations` | TEXT | Pipe-separated case law citations in full-form Bluebook style (`Party v. Party, VOL REPORTER PAGE (YEAR)`). Handles multi-word reporters (`Mass. App. Ct.`, `F. Supp.`, `U.S.`, `Fair Empl. Prac. Cas. (BNA)`), `&` in party names (`Rapo & Jepsen`), and missing-space OCR artifacts (`Flatley,419`). Case-name sides capped at 6 words each to avoid sentence-spanning false positives; occasional preamble text (e.g., `"Records Law is restricted. See Attorney Gen. v. ..."`) may appear — normalize downstream. **~45% recall** (13,460/29,674 rows non-null) — misses short-form citations (no `"v."`), docket-number format (Superior Court), `(PETA)` parenthetical aliases, and heavily OCR-garbled variants. The same case may appear in multiple party-name forms across rows — normalize downstream if aggregating. |
| `prior_appeals_referenced` | TEXT | Pipe-separated `SPRYY/NNNN` numbers mentioned in text (excluding self). |
| `fees_mentioned` | REAL | Largest dollar amount appearing in text (heuristic). |
| `outcome_guess` | TEXT | Heuristic classification of the Supervisor's ruling. One of: `ordered to provide` (14,833) \| `moot` (7,656) \| `upheld custodian` (2,315) \| `declined to opine` (1,068) \| `dismissed` (27) \| NULL (3,775). Matched against the last ~1500 chars of the letter (the ruling section). Spot-checked after regex overhaul — precision substantially improved over the initial keyword heuristic, but still a guess: treat as starting filter, not ground truth. `declined to opine` = Supervisor refused to rule because of pending litigation (950 C.M.R. 32.08(2)(b)). No `remanded` outcomes were found in the corpus. |

### Backfilled from `appeals` / `case_details`
These are the **canonical date and party fields** for analysis.

| Column | Type | Notes |
|---|---|---|
| `appeal_opened` | TEXT | `YYYY-MM-DD`. Sourced from `appeals.opened`. 0 nulls. |
| `closed_date` | TEXT | `YYYY-MM-DD`. Sourced from `appeals.closed`. **Use this, not `determination_date`, as the canonical letter-date signal.** 40 nulls (case still open). |

---

## FTS index: `determinations_fts`
FTS5 virtual table over `determinations.text`. Query via:

```sql
SELECT d.case_no, d.closed_date, snippet(determinations_fts, 2, '<b>', '</b>', '…', 20)
FROM determinations_fts f
JOIN determinations d ON d.rowid = f.rowid
WHERE determinations_fts MATCH 'scientific study NEAR/5 confidential'
ORDER BY bm25(determinations_fts) LIMIT 20;
```

The FTS rowid maps 1:1 to `determinations.rowid`. Triggers keep it in sync on insert/update/delete.

---

## Joining the tables

```sql
-- All letters for a case, ordered by stage:
SELECT pdf_stem, stage, stage_type, closed_date
FROM determinations
WHERE case_no = '20243445'
ORDER BY stage;

-- Full picture of a case (appeals metadata + each letter):
SELECT a.case_no, a.opened, a.closed, a.custodian, a.requester,
       d.pdf_stem, d.stage, d.stage_type
FROM appeals a
LEFT JOIN determinations d ON d.case_no = a.case_no
WHERE a.case_no = '20243445'
ORDER BY d.stage;

-- Per-stage timing (cleaner than inferring from determinations):
SELECT case_no,
       date_initial_opened, date_initial_closed,
       date_reconsideration_opened, date_reconsideration_closed,
       date_in_camera_opened, date_in_camera_closed
FROM case_details
WHERE case_no = '20243445';
```

---

## Known data-quality caveats

1. **Typos in source letters.** `determination_date` faithfully reproduces whatever date is printed — including documented typos (e.g., "January 8, 2024" on a 2024 SPR case actually resolved Jan 2025). Use `closed_date` for filtering/sorting.
2. **Stage-1 mis-tags.** A handful of cases have their initial-letter PDF missing from the scrape; the earliest surviving PDF gets `stage=1` even when its opening line is *"I write in connection with..."* rather than *"I have received the petition..."*. Spot-check `stage=1` rows where `closed_date - appeal_opened > 90 days` if stage accuracy matters.
3. **`outcome_guess` is a heuristic**, matched against the last ~1500 chars of the letter. Known blind spots: "partial" outcomes (some items ordered, others upheld) collapse into whichever wins the ordered/upheld race; fee-petition rulings and time-extension petitions use different phrasings and may land in NULL.
4. **Regex fields can have false positives.** `fees_mentioned` picks the largest `$` number in the text, which may include an unrelated number. `case_citations` has ~45% row-level recall; dominant misses are short-form ("Flatley, 419 Mass. at 511"), docket-number format ("Suffolk Sup. No. 2284CV02061-C"), WL citations, and `(PETA)` parenthetical aliases. A minority of citations carry a short preamble (e.g., `"Records Law is restricted. See Attorney Gen. v. Collector of Lynn..."`). `statutes_cited` can include both `"G. L. c. 4, § 7"` and `"G. L. c. 4, § 7(26)"` as separate pipe entries when both forms appear in the text; the latter is authoritative.
5. **Empty `requester_affiliation`** is common — the regex only fires when the letter uses "of/from/on behalf of" phrasing. Many letters omit the affiliation entirely.
6. **OCR quality.** 608 pre-2021 PDFs were OCR'd via tesseract; text may contain minor recognition errors (e.g., "lO(b)" for "10(b)", "ih" for "th").
