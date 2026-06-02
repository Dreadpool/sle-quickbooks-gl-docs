# QuickBooks GL in BigQuery — Documentation

This is the working reference for the QuickBooks general-ledger data mirrored into
BigQuery (project `jovial-root-443516-a7`, dataset `quickbooks_gl`). It covers the
table shape, how to query it, why the data looks the way it does, and how to
verify the pipeline is healthy.

> **For the agent reading this:** this is a living document. If you learn
> something new about the data, find an error here, or change how the pipeline
> works, **update this file in place** and keep it accurate. It is meant to be
> the single source of truth a future agent reads before touching this data.
>
> **Note (public copy):** specific dollar figures — vendor-spend totals and the
> overall ledger scale — are intentionally omitted here. The query patterns
> remain, so you can compute any figure directly against the warehouse.

---

## Overview

`quickbooks_gl` mirrors every journal line booked in QuickBooks across both legal
entities — **Salt Lake Express** (`salt_lake_express`) and **NWSBW LLC**
(`nwsbw_llc`). Use it for vendor spend, expense breakdowns, accounts receivable,
and any analysis sourced from accounting rather than ticketing.

### Tables

| Table | Purpose |
|-------|---------|
| `gl_transactions` | The journal lines themselves. Daily-partitioned on `txn_date`, clustered on `account_number, txn_type`. ~615K rows, 2021-01-01 to current. Loaded daily by the QuickBooks export pipeline. |
| `pipeline_runs` | Loader audit log — one row per ingest run: `run_id`, `run_ts`, `status`, `from_date`, `to_date`, `rows_received`, `rows_loaded`, `rows_deleted`, `duration_sec`, `error_message`, `source_ip`, `company`. |
| `gl_transactions_backup_20260329` | One-time snapshot taken before the March 2026 reload. Do not query for analysis. |

### `gl_transactions` schema

| Column | Type | Notes |
|--------|------|-------|
| `txn_date` | DATE | Journal entry date. REQUIRED. Partition column. |
| `txn_type` | STRING | QuickBooks transaction type, e.g. `General Journal`, `Deposit`, `Bill`, `Bill Payment`, `Paycheck`. REQUIRED. Cluster column. |
| `txn_number` | STRING | QuickBooks transaction number. Free-text; used as a logical marker (e.g., the recurring monthly Transcor settlement is always `NBTA`). |
| `account_number` | STRING | Chart-of-accounts number, e.g. `40100` (revenue), `67000` (Website Development Costs), `61211` (Reservationist Payroll). REQUIRED. Cluster column. |
| `account_name` | STRING | Display name of the account. REQUIRED. |
| `name` | STRING | Customer/vendor/employee name on the line, if any. Often NULL on general journal entries; populated on paychecks with the employee name. |
| `memo` | STRING | Free-text memo. Bookkeeper-authored; **do not rely on memo text for filtering** — wording changes over time. |
| `debit` | NUMERIC(12,2) | Debit amount, or NULL. |
| `credit` | NUMERIC(12,2) | Credit amount, or NULL. |
| `amount` | NUMERIC(12,2) | Signed line amount (debit positive, credit negative). REQUIRED. |
| `source_file` | STRING | Loader provenance: `QB_GL_Full_Export.csv` (historical one-time export, through 2026-02-26) or `daily_pipeline` (ongoing daily loads, 2026-02-27 onward). |
| `loaded_at` | TIMESTAMP | When the row was ingested. REQUIRED. |
| `company` | STRING | Legal entity: `salt_lake_express` or `nwsbw_llc`. Splits SLE from NWS even inside a journal entry that spans both. |

### Companies

- `salt_lake_express` — ~599K lines, 2021-01-01 → current.
- `nwsbw_llc` — ~11K lines, **2026-02-27 → current only.** NWS QuickBooks data did
  not start landing in BigQuery until late February 2026.

**Always filter by `company`** for entity-scoped analysis. Summing
`account_number = '40100'` (revenue) without a company filter combines both
entities.

---

## The shape of this data — read before any expense or "are these duplicates?" question

A full integrity audit (March–June 2026) answered the recurring question "are
there true duplicates in here, or just debit/credit sides?" — see the audit
results below. The short version:

- **The grain is one journal line, and there is no unique row ID.** Each
  QuickBooks transaction (a paycheck, a bill, a journal entry) explodes into many
  lines. You cannot dedup by a key column because none exists.
- **The same dollar amount legitimately appears on many lines — that is not
  duplication.** Two reasons, both correct accounting:
  1. *Double-entry.* Every transaction posts a debit on one account and a matching
     credit on another, so each amount shows up at least twice by design. The
     `debit` and `credit` columns never both populate on one line; `amount` =
     `debit − credit` on every row (verified, zero exceptions).
  2. *Per-employee payroll lines.* A single `Paycheck` pay run posts one line per
     employee per payroll item, so a 300-person payroll creates 300 identical
     lines (often $0 wash lines on Payroll Liabilities or the bank clearing
     account, or identical small benefit deductions). These are distinct employee
     postings, not copies.
- **Never strip "duplicate" lines before summing.** Removing them understates
  payroll and benefits. Sum the lines as-is, grouped by account. The pipeline
  mirrors QuickBooks faithfully — if a line repeats, QuickBooks repeated it.
- **There are zero true ingestion duplicates.** No line has ever been loaded twice
  across runs (verified). The daily loader uses delete-then-insert over its date
  window, so re-running a day replaces rather than accumulates.
- **Trap:** `loaded_at` is stamped per-row in microseconds, so identical lines in
  one batch get slightly different timestamps. Counting `DISTINCT loaded_at`
  falsely flags re-ingestion. Use `DISTINCT DATE(loaded_at)` to test for cross-run
  duplicates.

---

## How to query expenses

- An expense lands as a `debit` on an expense account (`account_number` in the
  6xxxx range). **Sum `debit`** for the account, never `amount` (which nets debits
  against any offsetting credits).
- **Always filter `company`** (`'salt_lake_express'` or `'nwsbw_llc'`). NWS only
  exists from 2026-02-27 onward.
- Filter on `account_number` + `txn_type`, **not on `memo`** — bookkeeper memo
  wording drifts over time.
- Payroll is split by role into named accounts (e.g. `61211` Reservationist
  Payroll, `61209` Dispatch Payroll, `61206` Accounting Payroll, `51xxx` Driver
  Payroll by route). "CSR" maps to Reservationist Payroll (`61211`). There is no
  department/role column — the account name is the only role signal.

```sql
-- Annual spend on a given expense account, one company
SELECT EXTRACT(YEAR FROM txn_date) AS year, ROUND(SUM(debit), 2) AS spend
FROM `jovial-root-443516-a7.quickbooks_gl.gl_transactions`
WHERE account_number = '67000'            -- the expense account you want
  AND company = 'salt_lake_express'
GROUP BY year ORDER BY year;
```

To see what accounts exist:
`SELECT DISTINCT account_number, account_name FROM ... WHERE company = '...' ORDER BY account_number`.

### Pattern: Transcor (TDS) vendor fees

SLE's monthly payment to Transcor Data Services (TDS, the booking-platform vendor)
is booked through a recurring multi-line general journal entry. The structure
changed twice since 2021, so the only stable filter is account + txn_type +
txn_number:

```sql
-- Annual Transcor (TDS) payments, SLE only
SELECT EXTRACT(YEAR FROM txn_date) AS year, ROUND(SUM(debit), 2) AS sle_paid_transcor
FROM `jovial-root-443516-a7.quickbooks_gl.gl_transactions`
WHERE account_number = '67000'          -- "Website Development Costs"
  AND txn_type = 'General Journal'
  AND txn_number = 'NBTA'               -- monthly Transcor settlement marker
  AND company = 'salt_lake_express'
GROUP BY year ORDER BY year;
```

Running the query above returns the verified annual totals by year. (Actual
dollar amounts omitted from this public copy — run the query against the
warehouse to get them.)

**Caveats:**
1. **Do not filter on memo text.** The memo wording changed in Aug 2025; a
   `memo LIKE '%transcor%'` filter silently drops everything after.
2. **Do not sum every line in the NBTA entry.** Each monthly entry has multiple
   lines (revenue credit, receivable debit, dues, terminal rent, and the Transcor
   fee). Only the line on account `67000` is the Transcor fee.
3. **From Feb 2026, NBTA entries combine both entities.** SLE and NWS lines share
   one journal entry — split by the `company` column, not by memo or account.
4. **No dedup risk between the two `source_file` loaders.** The historical CSV ends
   2026-02-26 and `daily_pipeline` starts 2026-02-27 — non-overlapping ranges.

---

## Integrity audit — verified results (June 2026)

| Check | Result |
|---|---|
| `amount = debit − credit` | True on all ~615K rows, 0 exceptions |
| Lines with both debit and credit populated | 0 |
| Repeated-line groups | 71,896 (all legitimate: double-entry + payroll) |
| Repeated groups loaded across >1 calendar date | **0** (no cross-run re-ingestion) |
| Repeated groups spanning both source files | 0 (CSV and daily loader are date-disjoint) |
| Successful pipeline runs with `rows_received == rows_loaded` | 71 / 71 |
| Failed runs | 18, all on bootstrap day 2026-03-28, **loaded 0 rows** |
| Ledger imbalance (debits − credits) | balances to within rounding noise (~0.0004%) — debits ≈ credits |

The daily loader uses **delete-then-insert** over its date window, so re-running a
day replaces rather than accumulates rows. That is why ~440K cumulative loaded
rows leave only ~41K net `daily_pipeline` rows.

### Re-run the audit

```sql
-- 1. Double-entry integrity + ledger balance, by company
SELECT
  company,
  COUNT(*) AS lines,
  ROUND(SUM(debit) - SUM(credit), 2) AS debit_minus_credit,  -- ~0 = balanced
  COUNTIF(amount <> COALESCE(debit,0) - COALESCE(credit,0)) AS amount_mismatch_rows,  -- want 0
  COUNTIF(debit IS NOT NULL AND credit IS NOT NULL AND debit<>0 AND credit<>0) AS both_sides_rows  -- want 0
FROM `jovial-root-443516-a7.quickbooks_gl.gl_transactions`
GROUP BY company;

-- 2. True re-ingestion test: do identical lines load on >1 calendar date?
--    Any nonzero result = the delete-then-insert left stale copies. Want 0.
WITH grp AS (
  SELECT COUNT(*) AS copies, COUNT(DISTINCT DATE(loaded_at)) AS load_dates
  FROM `jovial-root-443516-a7.quickbooks_gl.gl_transactions`
  GROUP BY txn_date, txn_type, txn_number, account_number, account_name,
           name, memo, debit, credit, amount, company, source_file
  HAVING copies > 1
)
SELECT COUNT(*) AS dup_groups,
       COUNTIF(load_dates > 1) AS groups_loaded_on_multiple_dates  -- want 0
FROM grp;

-- 3. Pipeline run health: received vs loaded, failures
SELECT status, COUNT(*) AS runs,
       SUM(rows_received) AS recv, SUM(rows_loaded) AS loaded, SUM(rows_deleted) AS deleted,
       MIN(run_ts) AS first_run, MAX(run_ts) AS last_run
FROM `jovial-root-443516-a7.quickbooks_gl.pipeline_runs`
GROUP BY status ORDER BY runs DESC;
```

If query 2 returns a nonzero `groups_loaded_on_multiple_dates`, or query 3 shows a
run where `rows_received <> rows_loaded`, the pipeline has started accumulating
real duplicates — investigate the loader's delete step. Otherwise the data is
sound and you can sum lines by account without deduping.

---

## Monitoring (automated)

Staleness alerting is **live**: a daily cloud job emails the SLE team when the GL
pipeline goes **7+ days** without a `status='success'` row in
`quickbooks_gl.pipeline_runs`. The loader pulls a rolling ~30-day window and
delete-then-inserts, so the data self-heals once it runs again — the alert just
flags when it has stopped firing. If the export pipeline changes, update this
section.
