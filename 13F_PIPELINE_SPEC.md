# Institutional Sponsorship (13F) — Spec for Meridian

> **What this is:** a self-contained brief an agent can execute *unattended overnight* to
> add an **institutional sponsorship panel** to Meridian's stock zoom-in, powered by SEC
> Form 13F data ingested into Meridian's SQLite DB.
>
> **What Meridian is (context that drives every decision below):** a screener/trading tool
> built on **relative strength (RS) + a breakout strategy**. The **company level** is a
> zoom-in on a single stock showing **technical + fundamental data** to help decide whether
> to buy. This feature adds a third lens to that decision: **is big money accumulating?**
>
> **This spec was written from a session that could NOT see Meridian's code.** Step 0
> exists for that reason — discover the real stack/schema first, then integrate.

---

## 1. The product idea (read this first — it shapes the build)

The 13F data is **not** a standalone screen. It is **one confirmation panel** that sits
next to the existing TA and fundamentals on the company zoom-in, feeding a buy / no-buy
decision on an RS + breakout setup.

**Why it belongs in a breakout strategy.** This is the "institutional sponsorship" idea
(the *I* in CAN SLIM). Breakouts hold when big money is accumulating — that demand pushes
price through resistance and keeps it there. So the question the panel answers is:

> **"Is institutional money building a position in this stock, or distributing it?"**

Rising sponsorship **confirms** an RS breakout; institutions exiting is a yellow flag even
on a clean chart.

**The limitation we MUST design around — do not fight it.** 13F is **quarterly** and
**lagged ~45 days** after quarter-end. For a strategy that times entries on price/RS, this
data is slow. Therefore institutional data is a **context / conviction layer, never a
timing trigger and never a hard filter**:

- ✅ Use it to *raise or lower confidence* in an otherwise-valid setup.
- ❌ Do not gate entries on it, and do not treat a single quarter's move as a signal.

Design the panel as a **multi-quarter trend** (quality over recency), not a one-quarter
snapshot.

**Explicitly out of scope for this use case** (drop them — they're noise for a breakout
decision): per-fund detail tables, options/derivative positions in the headline view,
13F "cloning"/copy-trading features. Keep the data (don't drop rows), just don't surface it.

---

## 2. What the panel shows (the functionality)

A compact panel on the company zoom-in, glanceable next to TA/fundamentals:

1. **Verdict badge** — one of `Accumulating ▲` / `Neutral –` / `Distributing ▼`, derived
   from the trends below. This is the at-a-glance read.
2. **Sponsorship trend (sparkline)** — number of institutional holders over the last ~8
   quarters. *Rising = healthy demand.* The single most useful line for a breakout trader.
3. **Net accumulation** — total institutional shares held and **% change QoQ**, plus
   **net buyers vs. sellers** count this quarter.
4. **Fresh sponsorship (optional secondary line)** — count of **new positions initiated**
   this quarter. New buyers are a stronger signal than existing holders trimming.
5. **As-of stamp** — the report period and filing lag, so the user knows the data's age
   (e.g. "as of 2026-03-31 · filed through 2026-05-15"). Critical given the 45-day lag.

### 2.1 Verdict logic (the badge)
Compute from the most recent completed quarter vs. the prior 1–3 quarters. Tune thresholds
in config; sensible defaults:

```
Inputs (latest quarter vs prior):
  d_holders   = % change in number of institutional holders
  d_shares    = % change in total institutional shares held
  buyers      = count of managers ADD/NEW
  sellers     = count of managers TRIM/EXIT
  trend_up    = holders rising in >= 2 of last 3 quarters

Verdict:
  Accumulating ▲  if (d_holders >= +3%  OR  trend_up)  AND  d_shares >= 0  AND buyers >= sellers
  Distributing ▼  if (d_holders <= -3%)  OR  (d_shares <= -5%  AND sellers > buyers)
  Neutral –       otherwise
```
> These are starting heuristics, not gospel — expose the thresholds and let the user tune.

### 2.2 Scoring integration
Institutional accumulation **both informs the user (visual) AND feeds Meridian's scoring**,
as a *confidence modifier* — never a hard filter (per §1):

- Map the verdict to a small bounded adjustment to the existing setup score/rank, e.g.
  `Accumulating ▲ → +X`, `Neutral → 0`, `Distributing ▼ → −X`, where X is a configurable
  weight that is **small relative to the RS/breakout core score** (it nudges ranking, it
  doesn't dominate it).
- Make the weight configurable, default conservative, and **fail safe**: if a stock has no
  13F data (e.g. unmapped CUSIP, brand-new listing), the modifier is **0** — never penalize
  a setup for missing institutional data.
- During Step 0, find how Meridian computes its score/rank and wire the modifier in at that
  point; if the scoring path is unclear or risky to touch, **ship the panel as a visual
  first and leave the scoring hook as a documented TODO** rather than guessing.

---

## 3. Pre-flight — discover before you build (do this FIRST)

Do not assume; let findings drive every later choice.

1. **Identify the stack.** Inspect the repo: `package.json` → Node/TS; `requirements.txt` /
   `pyproject.toml` / `*.py` → Python; etc. GitHub reports Meridian's dominant language as
   HTML, so the backend language is unconfirmed — find it. Implement the pipeline in the
   **existing backend language**; if there's no backend yet, default to **Python**
   (stdlib `sqlite3` + `requests` + `lxml`) in a new `pipeline/` dir.
2. **Find the SQLite DB** and read its schema. Match conventions (naming, id strategy,
   timestamps, migrations). **Add tables to the existing DB — never create a parallel one.**
3. **Find the existing stock zoom-in / company view** (where TA + fundamentals render) and
   how it queries data — that's where the panel attaches.
4. **Find how RS/breakout scoring is computed** — that's where §2.2's modifier hooks in.
5. **Record findings** in `pipeline/NOTES.md` (stack, db path, schema style, where the
   company view + scoring live, integration points) so morning review is auditable.

---

## 4. Data source (SEC EDGAR)

### 4.0 Request etiquette (mandatory)
- Every request to `sec.gov` / `data.sec.gov` MUST send a `User-Agent` with app + contact,
  e.g. `User-Agent: Meridian 13F robin.eikenaar93@gmail.com`. Missing UA → blocked.
- Rate limit **≤ 10 req/s**; throttle and back off on HTTP 429.

### 4.1 Backbone: bulk quarterly 13F datasets ("pull everything")
- SEC publishes structured **Form 13F data sets**, one ZIP of TSV tables per quarter.
  Locate the current download index programmatically (URLs rotate).
- Key tables (keyed by `ACCESSION_NUMBER`):
  - **`SUBMISSION.tsv`** — `ACCESSION_NUMBER`, `CIK`, `FILING_DATE`, `PERIODOFREPORT`
    (the quarter), submission type (`13F-HR`, `13F-HR/A` = amendment).
  - **`COVERPAGE.tsv`** — filer name + metadata.
  - **`INFOTABLE.tsv`** — the holdings, one row per position:
    `NAMEOFISSUER`, `TITLEOFCLASS`, **`CUSIP`** (security id, NOT a ticker), `VALUE`,
    `SSHPRNAMT` (shares/principal), `SSHPRNAMTTYPE` (`SH`/`PRN`), `PUTCALL` (''/Put/Call).
- **Units gotcha:** historically `VALUE` was in **thousands of dollars**; under the **2023**
  rule change it's **whole dollars**. Normalize at ingest; verify against a known filing so
  totals aren't 1000× off.

### 4.2 Incremental: per-filer submissions API (fresh filings between bulk drops)
- `https://data.sec.gov/submissions/CIK##########.json` (CIK zero-padded to 10 digits) →
  recent filings; filter `13F-HR` / `13F-HR/A`.
- The **information table XML** lives in the filing's archive folder:
  `https://www.sec.gov/Archives/edgar/data/{cik}/{accession_no_no_dashes}/` → find
  `*infotable*.xml`. Same fields as INFOTABLE.tsv; **namespaces vary by year — parse
  defensively.**
- **Amendments (`13F-HR/A`):** treat the latest filing per `(cik, period)` as
  authoritative (prefer most recent `FILING_DATE`).
- For this product, the bulk path covers everything; the incremental path is optional
  (e.g. to refresh a watchlist sooner).

### 4.3 Identifier trap — do NOT confuse these
`https://www.sec.gov/files/company_tickers.json` maps **CIK → ticker** for *filers/issuers*,
**not** CUSIP → ticker for holdings. Do not use it as a holdings map.

---

## 5. The one hard part: CUSIP → ticker (so the panel can find a stock)

13F gives **CUSIP + issuer name**, never a ticker. To join 13F holdings to a Meridian
stock you must resolve CUSIP → ticker:

- **OpenFIGI mapping API:** `POST https://api.openfigi.com/v3/mapping`,
  body `[{"idType":"ID_CUSIP","idValue":"<cusip>"}, ...]`.
  - Get a **free API key** (`X-OPENFIGI-APIKEY` header) for higher limits; store in
    config/env, never hardcode.
  - Response yields FIGI, `ticker`, `exchCode`, `securityType`, `name`. Prefer the US
    composite/primary listing.
- **Cache permanently** in `securities` — CUSIPs are stable; only batch *unmapped* ones.
- **Coverage** ~95%+. Carry unmapped holdings by CUSIP (ticker NULL), never drop them;
  periodically retry NULLs (new listings get added over time).
- **Then join to Meridian's own ticker universe** so the panel keys off the same symbol the
  rest of the company view uses. One company can have **multiple CUSIPs** (share classes) —
  aggregate them all under the ticker.

---

## 6. SQLite schema (adapt names to Meridian's conventions)

Add to the existing DB; columns below are the contract.

```sql
CREATE TABLE IF NOT EXISTS filers (
  cik   TEXT PRIMARY KEY,             -- zero-padded 10-digit
  name  TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS securities (
  cusip        TEXT PRIMARY KEY,
  issuer_name  TEXT,
  ticker       TEXT,                  -- NULL = unresolved; retry later
  figi         TEXT,
  resolved_at  TEXT                   -- ISO ts; NULL = never attempted
);

CREATE TABLE IF NOT EXISTS filings (
  accession_no TEXT PRIMARY KEY,
  cik          TEXT NOT NULL REFERENCES filers(cik),
  period       TEXT NOT NULL,         -- quarter end, e.g. '2026-03-31'
  filing_date  TEXT NOT NULL,
  form_type    TEXT NOT NULL,         -- '13F-HR' | '13F-HR/A'
  is_amendment INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS holdings (
  accession_no TEXT NOT NULL REFERENCES filings(accession_no),
  cusip        TEXT NOT NULL REFERENCES securities(cusip),
  shares       INTEGER,               -- SSHPRNAMT when type = SH
  prn_type     TEXT,                  -- 'SH' | 'PRN'
  value_usd    INTEGER,               -- normalized to WHOLE dollars (see 4.1)
  put_call     TEXT,                  -- '' | 'Put' | 'Call'
  PRIMARY KEY (accession_no, cusip, put_call)
);

-- Precomputed per-stock-per-quarter rollup that powers the panel cheaply
CREATE TABLE IF NOT EXISTS sponsorship_quarterly (
  ticker        TEXT NOT NULL,
  period        TEXT NOT NULL,        -- quarter end
  holder_count  INTEGER NOT NULL,     -- # institutions holding (shares, excl options)
  total_shares  INTEGER NOT NULL,
  total_value   INTEGER NOT NULL,
  new_count     INTEGER NOT NULL,     -- managers initiating this quarter
  exit_count    INTEGER NOT NULL,
  add_count     INTEGER NOT NULL,
  trim_count    INTEGER NOT NULL,
  PRIMARY KEY (ticker, period)
);

CREATE TABLE IF NOT EXISTS ingest_log (
  source     TEXT NOT NULL,          -- 'bulk:2026Q1' | 'rollup:2026Q1'
  status     TEXT NOT NULL,          -- 'started' | 'done' | 'error'
  detail     TEXT,
  updated_at TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_holdings_cusip  ON holdings(cusip);
CREATE INDEX IF NOT EXISTS idx_filings_period  ON filings(period);
CREATE INDEX IF NOT EXISTS idx_filings_cik_per ON filings(cik, period);
CREATE INDEX IF NOT EXISTS idx_spq_ticker      ON sponsorship_quarterly(ticker);
```

> `sponsorship_quarterly` is the key design choice: the panel reads **pre-aggregated** rows
> (fast, simple UI query) instead of crunching millions of holdings on every page view.

---

## 7. Pipeline (idempotent, resumable, cron-friendly)

One overnight run does end-to-end:

1. **Init/migrate** schema (`CREATE TABLE IF NOT EXISTS` — safe to re-run).
2. **Pick target quarters.** Default: most recent **8** quarters not yet `done` in
   `ingest_log` (8 = two years, enough for the sparkline + trend). Config value.
3. **Ingest each quarter (bulk path):**
   a. Download quarter ZIP (skip if `ingest_log` = done).
   b. Stream `SUBMISSION.tsv` + `COVERPAGE.tsv` → upsert `filers`, `filings`.
   c. Stream `INFOTABLE.tsv` → upsert `holdings`, normalizing `VALUE` units; collect new
      CUSIPs into `securities` (ticker NULL).
   d. One transaction per quarter; mark `ingest_log` done on commit.
4. **Resolve CUSIPs** with NULL ticker via OpenFIGI; update `ticker/figi/resolved_at`.
5. **Build `sponsorship_quarterly` rollups** per ticker per quarter from `holdings`
   (shares only, `put_call=''`), including QoQ add/trim/new/exit counts (see §8 logic).
6. **(Optional) incremental** fresh filings for a watchlist via the submissions API.
7. **Log a run summary**: rows ingested, CUSIPs resolved, unmapped count, tickers rolled up.

**Idempotency:** all writes UPSERT on natural keys (`accession_no`, `cusip`,
`(ticker,period)`). Re-running a done quarter is a no-op; a crashed run resumes from
`ingest_log`. **Robustness:** retry transient HTTP (timeout/5xx/429) with backoff; on a
per-quarter fatal error, log and continue to the next quarter rather than aborting.

---

## 8. Computing the rollups & verdict

**QoQ classification** (per ticker, per manager) drives `new/exit/add/trim` counts: compare
a manager's total shares in `period` vs `prev_period`:
`NEW` (prev 0 → cur >0), `EXIT` (cur 0), `ADD` (cur>prev), `TRIM` (cur<prev), else `HOLD`.

**Panel read query** (cheap — straight from rollups):
```sql
SELECT period, holder_count, total_shares, total_value,
       new_count, exit_count, add_count, trim_count
FROM sponsorship_quarterly
WHERE ticker = :ticker
ORDER BY period DESC
LIMIT 8;   -- newest = headline numbers; full set = sparkline
```
From those rows compute `d_holders`, `d_shares`, buyers/sellers, `trend_up`, then the
**verdict** per §2.1, and the **score modifier** per §2.2.

> SQLite `FULL OUTER JOIN` (for the QoQ manager diff) needs 3.39+. If the bundled SQLite is
> older, emulate with `LEFT JOIN ... UNION ... LEFT JOIN`.

---

## 9. Scheduling

- Single entrypoint (e.g. `python -m pipeline.run` or `npm run ingest:13f`) runs §7
  end-to-end and exits non-zero on fatal error. Log to a timestamped file.
- Cron / launchd / Task Scheduler on the home machine triggers it nightly. Document the
  exact command in `pipeline/NOTES.md`.
- New 13F data only lands ~45 days after quarter-end and trickles in; nightly is plenty —
  most nights it no-ops cheaply.

---

## 10. Acceptance criteria (verify before declaring done)

1. Schema added to the **existing** Meridian SQLite DB; no parallel DB.
2. ≥ 4 recent quarters ingested (enough for trend + sparkline).
3. `holdings` spot-checked against a known filing (a famous fund's top position looks right).
4. `VALUE` unit normalization verified (plausible whole-dollar totals, not 1000× off).
5. CUSIP resolution ≥ ~90% have a ticker; unmapped rows retained, not dropped; joined to
   Meridian's ticker universe.
6. `sponsorship_quarterly` populated; panel query returns correct holder_count trend and
   buy/sell counts for a hand-checked ticker.
7. Verdict badge + sparkline + net-accumulation render on the company zoom-in next to
   TA/fundamentals; as-of/lag stamp shown.
8. Score modifier wired in (small, bounded, fail-safe to 0 when no data) **or** documented
   TODO if the scoring path was too risky to touch.
9. Re-running the entrypoint is a **no-op** (idempotency holds).
10. `pipeline/NOTES.md` documents stack, db path, run command, integration points, deviations.

---

## 11. Decisions left to the overnight agent (use judgement, record in NOTES.md)

- Language/dir layout (per §3 discovery).
- Quarters of history to backfill (default 8).
- Exact verdict thresholds and score-modifier weight (defaults in §2.1/§2.2; keep
  conservative and configurable).
- Whether to fully wire the score modifier now or ship panel-first with a TODO (per §2.2) —
  prefer shipping the visual panel and a documented hook over a risky scoring change.
- Optional watchlist of CIKs for the incremental path.

If a decision is genuinely ambiguous and high-impact, document it in NOTES.md with a
recommendation rather than making an irreversible choice.

---

## 12. Gotchas that bite

| Gotcha | Handling |
|---|---|
| 13F is quarterly + ~45-day lagged | Context/conviction layer only — never a timing trigger or hard filter; show as-of stamp |
| No ticker in 13F (CUSIP only) | OpenFIGI map + cache in `securities`; join to Meridian's ticker universe |
| `VALUE` units changed in 2023 (thousands → whole $) | Normalize at ingest; verify |
| Amendments (`13F-HR/A`) | Latest filing per (cik, period) is authoritative |
| Multiple CUSIPs per company (share classes) | Aggregate all under the ticker |
| Options rows (Put/Call) | Exclude from share counts/sponsorship; out of headline scope |
| Stock with no 13F data | Verdict = none, score modifier = 0 — never penalize |
| SEC blocks missing User-Agent | Always send UA + contact email |
| Rate limits (SEC 10/s, OpenFIGI per-min) | Throttle + backoff on 429 |
| Heavy per-view computation | Pre-aggregate into `sponsorship_quarterly`; panel reads that |
