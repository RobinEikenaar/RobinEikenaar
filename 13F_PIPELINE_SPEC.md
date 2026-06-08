# 13F Holdings Pipeline — Implementation Spec (for Meridian)

> **Purpose of this doc:** a self-contained brief an agent can execute *unattended
> overnight* to build a pipeline that ingests SEC Form 13F filings, maps holdings to
> stocks, and powers a company-screen view (top holders + accumulating vs. selling).
>
> **This spec was written from a session that could NOT see Meridian's code.** Step 0
> exists precisely for that reason: discover the real stack/schema first, then adapt.

---

## 0. Pre-flight — discover before you build

Do this **first** and let it drive every later choice. Do not assume.

1. **Identify the stack.** Inspect the repo: `package.json` → Node/TS; `requirements.txt` /
   `pyproject.toml` / `*.py` → Python; anything else → adapt. GitHub reports Meridian's
   dominant language as HTML, so the backend language is unconfirmed — find it.
   - If there is **already a backend** (Node, Python, etc.), implement the pipeline **in
     that language** and reuse its HTTP client, config, and DB-access patterns.
   - If there is **no backend yet** (static HTML only), default to **Python** (stdlib
     `sqlite3` + `requests` + `lxml`), in a new `pipeline/` (or `13f/`) directory.
2. **Find the SQLite database.** Locate the `.db`/`.sqlite` file and read the existing
   schema (`.schema` / `sqlite_master`). Match its conventions: naming (snake_case vs
   camelCase), id strategy (autoincrement vs natural keys), timestamp columns, migrations
   tooling (if any).
3. **Find how data currently gets written.** If there's an existing ingestion/repository
   layer, route new writes through it rather than opening raw connections.
4. **Record findings** in a short `pipeline/NOTES.md` (stack, db path, schema style,
   integration points) so the decisions are auditable in the morning.

**Hard constraint:** integrate with what exists. Do not introduce a second ORM/DB layer or
a parallel database. Add tables to the existing SQLite file.

---

## 1. Goal & scope

**In scope**
- Pull 13F holdings data from SEC EDGAR via API/bulk datasets.
- Map each holding (identified by CUSIP) to a stock **ticker**.
- Store filings + holdings in SQLite with quarter-over-quarter history.
- Power a **company screen**: for a given stock, show top institutional holders, total
  shares accumulated, and whether holders are net buying or selling.

**Out of scope (note as limitations, do not try to fix)**
- 13F only covers **long** U.S.-listed equity & options positions of managers with >$100M
  in 13(f) securities. No shorts, no cash, no most non-U.S. holdings.
- Data is **45 days lagged** (filing deadline is 45 days after quarter-end). It is a
  snapshot, not real-time.

---

## 2. Data sources (SEC EDGAR)

### 2.0 Mandatory request etiquette
- **Every** request to `sec.gov` / `data.sec.gov` MUST send a `User-Agent` header
  identifying the app and a contact email, e.g.
  `User-Agent: Meridian 13F Pipeline robin.eikenaar93@gmail.com`.
  Missing/spoofed UA → blocked.
- Respect the rate limit: **≤ 10 requests/second**. Add a small sleep / token-bucket
  throttle. Back off on HTTP 429 with exponential delay.

### 2.1 Backbone: bulk quarterly 13F datasets (use this for "pull everything")
- SEC publishes **structured Form 13F data sets**, one package per calendar quarter, as a
  ZIP of tab-separated (`.tsv`) tables. Landing page: SEC "Form 13F data sets"
  (dera/data). The agent should locate the current download index programmatically rather
  than hardcoding a URL that may rotate.
- Key tables inside each quarter package:
  - **`SUBMISSION.tsv`** — one row per filing: `ACCESSION_NUMBER`, `CIK`, `FILING_DATE`,
    `PERIODOFREPORT` (the quarter), submission type (`13F-HR`, `13F-HR/A` = amendment).
  - **`COVERPAGE.tsv`** — filer name and metadata, keyed by `ACCESSION_NUMBER`.
  - **`INFOTABLE.tsv`** — the holdings, one row per position, keyed by
    `ACCESSION_NUMBER`. Columns of interest:
    - `NAMEOFISSUER`, `TITLEOFCLASS`
    - `CUSIP`  ← the security identifier (NOT a ticker)
    - `VALUE`  ← market value of the position (**see units gotcha below**)
    - `SSHPRNAMT` ← shares or principal amount
    - `SSHPRNAMTTYPE` ← `SH` (shares) or `PRN` (principal)
    - `PUTCALL` ← blank, `Put`, or `Call`
    - `INVESTMENTDISCRETION`, `VOTING_AUTH_*`
- **Units gotcha — read carefully:** historically `VALUE` was reported in **thousands of
  dollars**; under the rule change effective **2023** it is reported in **whole dollars**.
  Normalize to whole dollars at ingest and store a `value_unit_raw`/normalization note, or
  detect by `PERIODOFREPORT`. Verify against a known filing before trusting totals.

### 2.2 Incremental: per-filer submissions API (for fresh filings between bulk drops)
- `https://data.sec.gov/submissions/CIK##########.json` (CIK zero-padded to 10 digits) →
  recent filings list. Filter `form` for `13F-HR` / `13F-HR/A`.
- For a given filing, the **information table XML** lives in the filing's archive folder:
  `https://www.sec.gov/Archives/edgar/data/{cik}/{accession_no_no_dashes}/` — find the
  `*infotable*.xml` (a.k.a. `form13fInfoTable.xml`). Its `<infoTable>` elements carry the
  same fields as INFOTABLE.tsv (`nameOfIssuer`, `cusip`, `value`, `shrsOrPrnAmt/sshPrnamt`,
  `putCall`, etc.). Namespaces vary across years — parse defensively.
- **Amendments (`13F-HR/A`):** can be *restatements* (full replacement) or *new holdings*
  additions. Handle by treating the latest filing for a `(cik, period)` as authoritative;
  key on accession and prefer the most recent `FILING_DATE`.

### 2.3 Identifiers — do NOT confuse these
- `https://www.sec.gov/files/company_tickers.json` maps **CIK → ticker** for *filers/
  issuers that file with the SEC*. This is **not** a CUSIP→ticker map for arbitrary
  holdings and must not be used as one. (It can help label the *manager*, not the holding.)

---

## 3. The one hard part: CUSIP → ticker mapping

13F gives you **CUSIP + issuer name**, never a ticker. To "map it to stock" you must
resolve CUSIP → ticker. CUSIP↔ticker is licensed data; the practical free route:

- **OpenFIGI mapping API:** `POST https://api.openfigi.com/v3/mapping`
  body: `[{"idType":"ID_CUSIP","idValue":"<cusip>"}, ...]`
  - Without API key: ~25 jobs/request, ~25 requests/min.
  - With **free API key** (`X-OPENFIGI-APIKEY` header): ~100 jobs/request, higher
    req/min. **Get a free key** and store it in config/env, never hardcode.
  - Response yields FIGI, `ticker`, `exchCode`, `securityType`, `name`. Prefer the US
    composite/primary listing when multiple are returned.
- **Cache permanently.** CUSIPs are stable. Resolve each distinct CUSIP **once**, store in
  `securities`, and never re-query a resolved one. Only batch the *unmapped* CUSIPs.
- **Coverage:** expect ~95%+. Carry unmapped holdings by CUSIP (ticker NULL) — never drop
  them. Add a periodic retry for NULL-ticker rows (new listings get added over time).

---

## 4. SQLite schema (adapt names to Meridian's conventions)

Add these to the existing DB. Use the repo's id/naming style; the columns below are the
contract.

```sql
-- Institutional managers (13F filers)
CREATE TABLE IF NOT EXISTS filers (
  cik         TEXT PRIMARY KEY,        -- zero-padded 10-digit
  name        TEXT NOT NULL
);

-- Securities, resolved to ticker via OpenFIGI (ticker nullable until resolved)
CREATE TABLE IF NOT EXISTS securities (
  cusip        TEXT PRIMARY KEY,
  issuer_name  TEXT,
  ticker       TEXT,                   -- NULL = unresolved; retry later
  figi         TEXT,
  resolved_at  TEXT                    -- ISO timestamp; NULL = never attempted
);

-- One row per 13F filing (a manager × quarter)
CREATE TABLE IF NOT EXISTS filings (
  accession_no TEXT PRIMARY KEY,       -- natural key from EDGAR
  cik          TEXT NOT NULL REFERENCES filers(cik),
  period       TEXT NOT NULL,          -- quarter end, e.g. '2026-03-31'
  filing_date  TEXT NOT NULL,
  form_type    TEXT NOT NULL,          -- '13F-HR' | '13F-HR/A'
  is_amendment INTEGER NOT NULL DEFAULT 0
);

-- One row per position within a filing
CREATE TABLE IF NOT EXISTS holdings (
  accession_no TEXT NOT NULL REFERENCES filings(accession_no),
  cusip        TEXT NOT NULL REFERENCES securities(cusip),
  shares       INTEGER,               -- SSHPRNAMT when type = SH
  prn_type     TEXT,                  -- 'SH' | 'PRN'
  value_usd    INTEGER,               -- normalized to WHOLE dollars (see 2.1 gotcha)
  put_call     TEXT,                  -- '' | 'Put' | 'Call'
  PRIMARY KEY (accession_no, cusip, put_call)
);

-- Track ingestion progress for idempotent re-runs
CREATE TABLE IF NOT EXISTS ingest_log (
  source       TEXT NOT NULL,         -- 'bulk:2026Q1' | 'filer:CIK...'
  status       TEXT NOT NULL,         -- 'started' | 'done' | 'error'
  detail       TEXT,
  updated_at   TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_holdings_cusip   ON holdings(cusip);
CREATE INDEX IF NOT EXISTS idx_filings_period   ON filings(period);
CREATE INDEX IF NOT EXISTS idx_filings_cik_per  ON filings(cik, period);
```

---

## 5. Ingestion pipeline (idempotent, resumable)

Order of operations for a single overnight run:

1. **Init/migrate** schema (CREATE TABLE IF NOT EXISTS — safe to re-run).
2. **Pick target quarters.** Default: ingest the most recent N quarters that aren't
   already complete in `ingest_log` (so the screen can compute diffs). Make N a config
   value (default 8 = two years).
3. **For each quarter (bulk path):**
   a. Download the quarter ZIP (skip if `ingest_log` shows `done`).
   b. Stream-parse `SUBMISSION.tsv` + `COVERPAGE.tsv` → upsert `filers`, `filings`.
   c. Stream-parse `INFOTABLE.tsv` → upsert `holdings`, normalizing `VALUE` units and
      collecting distinct new CUSIPs into `securities` (ticker NULL).
   d. Wrap each quarter in a transaction; mark `ingest_log` done on commit.
4. **Resolve CUSIPs:** batch all `securities` rows where `ticker IS NULL AND
   (resolved_at IS NULL OR resolved_at older than retry window)` through OpenFIGI; update
   `ticker`, `figi`, `resolved_at`. Respect OpenFIGI rate limits.
5. **(Optional) Incremental fresh filings:** for a watchlist of CIKs, poll the
   submissions API for `13F-HR` newer than what's stored; parse the info-table XML; upsert.
6. **Log a run summary** (rows ingested per quarter, CUSIPs resolved, unmapped count).

**Idempotency rules:** all writes are UPSERTs keyed on natural keys (`accession_no`,
`cusip`). Re-running a completed quarter is a no-op. A crashed run resumes from
`ingest_log`. Never duplicate holdings.

**Robustness for unattended runs:** retry transient HTTP errors (timeouts, 5xx, 429) with
exponential backoff; on a fatal per-quarter error, log it and continue to the next quarter
rather than aborting the whole run; everything resumable next night.

---

## 6. Company screen — the queries that matter

The screen is keyed on a **stock** (resolve ticker → CUSIP(s) via `securities`; one company
can have multiple CUSIPs across share classes — include all).

### 6.1 Top holders this quarter
```sql
SELECT f.name AS manager, h.shares, h.value_usd
FROM holdings h
JOIN filings f_fil ON f_fil.accession_no = h.accession_no
JOIN filers  f     ON f.cik = f_fil.cik
WHERE h.cusip = :cusip
  AND f_fil.period = :period
  AND COALESCE(h.put_call,'') = ''        -- exclude options for share counts
ORDER BY h.shares DESC;
```

### 6.2 Total accumulated + holder count
```sql
SELECT SUM(h.shares) AS total_shares,
       SUM(h.value_usd) AS total_value,
       COUNT(DISTINCT f_fil.cik) AS holder_count
FROM holdings h
JOIN filings f_fil ON f_fil.accession_no = h.accession_no
WHERE h.cusip = :cusip AND f_fil.period = :period
  AND COALESCE(h.put_call,'') = '';
```

### 6.3 Accumulating vs. selling (quarter-over-quarter, the key feature)
Compare each manager's position in `:period` vs the prior period. Classify and net it:
```sql
WITH cur AS (
  SELECT f_fil.cik, SUM(h.shares) shares
  FROM holdings h JOIN filings f_fil ON f_fil.accession_no=h.accession_no
  WHERE h.cusip=:cusip AND f_fil.period=:period AND COALESCE(h.put_call,'')=''
  GROUP BY f_fil.cik
),
prev AS (
  SELECT f_fil.cik, SUM(h.shares) shares
  FROM holdings h JOIN filings f_fil ON f_fil.accession_no=h.accession_no
  WHERE h.cusip=:cusip AND f_fil.period=:prev_period AND COALESCE(h.put_call,'')=''
  GROUP BY f_fil.cik
)
SELECT
  COALESCE(cur.cik, prev.cik) AS cik,
  COALESCE(cur.shares,0)  AS cur_shares,
  COALESCE(prev.shares,0) AS prev_shares,
  COALESCE(cur.shares,0) - COALESCE(prev.shares,0) AS delta,
  CASE
    WHEN prev.shares IS NULL AND cur.shares > 0 THEN 'NEW'
    WHEN cur.shares  IS NULL OR cur.shares = 0  THEN 'EXIT'
    WHEN cur.shares > prev.shares THEN 'ADD'
    WHEN cur.shares < prev.shares THEN 'TRIM'
    ELSE 'HOLD'
  END AS action
FROM cur FULL OUTER JOIN prev ON cur.cik = prev.cik;
```
> SQLite supports `FULL OUTER JOIN` in modern versions (3.39+). If the bundled SQLite is
> older, emulate with `LEFT JOIN ... UNION ... LEFT JOIN`.

**Screen rollups to surface in the UI:**
- Net share delta across all managers (positive = net accumulation, negative = net selling).
- Count of `ADD`+`NEW` (buyers) vs `TRIM`+`EXIT` (sellers).
- Biggest individual buyers/sellers by `delta`.

---

## 7. Scheduling the overnight run

- Provide a single entrypoint (e.g. `python -m pipeline.run` or `npm run ingest:13f`) that
  does Section 5 end-to-end and exits non-zero on fatal error.
- Make it cron-friendly; log to a file with timestamps. (A nightly cron / launchd / Task
  Scheduler entry on the home machine triggers it.) Document the exact command in
  `pipeline/NOTES.md`.
- New 13F data only appears ~45 days after each quarter-end and trickles in; a nightly run
  is plenty — most nights it will find nothing new and no-op cheaply.

---

## 8. Acceptance criteria (verify before declaring done)

1. Schema created in the **existing** Meridian SQLite DB; no parallel DB introduced.
2. At least the **2 most recent available quarters** ingested so diffs work.
3. `holdings` row counts sanity-check against a known filing (spot-check one large manager,
   e.g. confirm a famous fund's top position looks right).
4. `VALUE` unit normalization verified (totals are plausible whole-dollar amounts, not off
   by 1000×).
5. CUSIP resolution ≥ ~90% of rows have a ticker; unmapped rows retained, not dropped.
6. Company-screen queries (6.1–6.3) return correct top-holders and buy/sell classification
   for a hand-checked ticker.
7. Re-running the entrypoint is a **no-op** (idempotency holds).
8. `pipeline/NOTES.md` documents stack found, DB path, run command, and any deviations.

---

## 9. Decisions left to the overnight agent (use judgement, then record)

- Exact language/dir layout (per Step 0 discovery).
- How many quarters of history to backfill (default 8).
- Whether to wire the company-screen queries into Meridian's existing HTML/UI now, or just
  expose them as a query module/endpoint for a follow-up. If the UI integration is large or
  ambiguous, **stop at the data + query layer and leave a clear TODO** rather than guessing
  at frontend structure.
- Watchlist of CIKs for the incremental path (optional; bulk path covers everything).

If a decision is genuinely ambiguous and high-impact, leave it documented in NOTES.md with
a recommendation rather than making an irreversible choice.

---

## 10. Quick reference — gotchas that bite

| Gotcha | Handling |
|---|---|
| No ticker in 13F (CUSIP only) | OpenFIGI map + cache in `securities` |
| `VALUE` units changed in 2023 (thousands → whole $) | Normalize at ingest; verify |
| Amendments (`13F-HR/A`) | Latest filing per (cik, period) is authoritative |
| Multiple CUSIPs per company (share classes) | Aggregate all CUSIPs for a ticker |
| Options rows (`Put`/`Call`) | Exclude from share counts; surface separately if wanted |
| SEC blocks missing User-Agent | Always send UA + contact email |
| Rate limits (SEC 10/s, OpenFIGI per-min) | Throttle + backoff on 429 |
| Unmapped CUSIPs | Keep them (ticker NULL); periodic retry |
| 45-day data lag | It's a snapshot; document, don't "fix" |
