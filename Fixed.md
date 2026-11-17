# Developer Guide: Fixed Deposit (FD) Outstanding Dashboard Revamp

## Evolution to ILMS Reporting — React & Python ETL Modernization

---

## Document Metadata

| Attribute | Details |
|---|---|
| **Project** | ILMA (Intraday Liquidity Management & Analytics) |
| **Module** | Fixed Deposit (FD) Outstanding Volume Dashboard |
| **Version** | v1.1 (supersedes v1.0) |
| **Confluence Reference** | Page ID: `817672332` |
| **Legacy Tech Stack** | Tableau Server (Workbook: `FD-Outstanding Volume`, site `SIT_SANDBOX` / TLM) + Legacy Python ETL Script (`fixed_deposit.py`) |
| **Target Tech Stack** | **Frontend:** React 18 + TypeScript + MUI + AG Grid + Apache ECharts<br>**Backend Query Service:** `ilms-rpt-data-query-service` (FastAPI / SQLAlchemy; DuckDB optional)<br>**ETL Service:** `tableaudataloadpy` (Python 3.10+ / Pandas / DriveX / MariaDB)<br>**Database:** MariaDB (`ILMA` schema, Table: `tbl_txn_fixed_deposit`) |
| **Primary Business Users** | Treasury & Liquidity Management (TLM SG Desk) and regional TLM / country treasury teams |
| **Author / Team** | ILMA Core Engineering Team |
| **Target Audience** | Full-Stack Engineers, Data Engineers, DevOps, QA, and Business Analysts |

### Changelog (v1.0 → v1.1)

| # | Area | Change |
|---|---|---|
| 1 | ETL | Report date is now derived from the **DriveX file name** (business-day safe) instead of `today − 2 calendar days`, which failed on Mondays / after public holidays. |
| 2 | ETL | Column mapping extended to **all** `tbl_txn_fixed_deposit` columns (v1.0 mapped only 16; the legacy `to_sql` loaded every Excel column, so v1.0 would have regressed `interco`, LCR and flag fields to NULL). |
| 3 | ETL | Added `source_file` metadata, skip-if-already-loaded check for the hourly schedule, latest-timestamp file selection, and pre-load validation rules. |
| 4 | ETL | Housekeeping query aligned to the stated rule (**3 latest dates + 11 previous month-ends**); v1.0 SQL could retain 12–13 month-ends. |
| 5 | DB | `exchange_rate DECIMAL(10,6)` widened — loses precision for IDR / VND / KRW (see §3.3). Indexes added on `report_date`. |
| 6 | API | Maturity bucket `CASE` fixed: matured / maturing-today (`Δ ≤ 0`) and NULL maturity dates previously fell into `> 1 year`. |
| 7 | API | Weighted tenor formulas floor negative remaining days at 0. |
| 8 | Scope | Added **Phase 1 Enhancements** (§6): maturity ladder, maturity calendar heatmap, country × bucket heatmap, counterparty concentration, trend view, intragroup and restricted-currency toggles. All buildable from existing data. |
| 9 | Docs | Migration drivers reworded to platform-evolution framing; "intraday" wording corrected (dataset is a daily T-2 snapshot). |

---

## 1. Executive Summary & Business Context

### 1.1 Business Objective

The **Fixed Deposit Outstanding Dashboard** gives Treasury & Liquidity Management (TLM) users a daily (T-2) and historical breakdown of the amounts owed to counterparties under fixed deposits, by maturity date, currency, country, counterparty and BIS entity type.

TLM users must maintain a strict liquidity buffer and threshold. They assess available cash flows to determine how much can be safely placed or replaced in fixed deposits without compromising short-term liquidity, and stagger FD maturities to keep cash inflows consistent and predictable and to mitigate liquidity-crunch risk.

### 1.2 Migration Drivers

- **Unified ILMS Reporting Portal:** Consolidates the FD view into the ILMS Reporting web portal alongside HQLA, BWS, WSF and CBLF dashboards, with consistent look-and-feel, access control and lower cross-service latency.
- **Interactive User Experience:** Upgrades static views to responsive filtering, reactive KPI cards, dynamic tenor bucketing, deep transactional drill-down, and new liquidity-oriented visuals (maturity ladder, calendar heatmap, concentration, trend).
- **ETL Resilience & Performance:** Modernizes the legacy Python ETL script into an idempotent, observable, container-ready data pipeline with robust DriveX file handling.
- **Automated Regression Testing:** Every release is reconciled against the legacy outputs by automated checks (§8), not manual eyeballing.

### 1.3 Delivery Phasing

| Phase | Scope |
|---|---|
| **Phase 0 — Parity** | Rebuild the 3 legacy tabs (Overview, Maturity by Currency, Details) with exact numeric parity. Gate for UAT sign-off. |
| **Phase 1 — Enhancements** | New visuals and toggles built on data **already in `tbl_txn_fixed_deposit`** (§6). |
| **Phase 2 — Future** | Items needing new data: rollover linkage, FD pricing vs benchmark, holiday-adjusted calendar, behavioural/LCR runoff view validation, early-break history (§10). |

---

## 2. Architecture & System Flow

```
┌───────────────────────────────────────────────────────────────┐
│ 1. Upstream Data Sources (MOT / RMG / Finacle / iMAX)          │
└───────────────────────────────┬───────────────────────────────┘
                                │ Pushes daily ~09:15 SGT
                                ▼
┌───────────────────────────────────────────────────────────────┐
│ 2. ILMS DriveX Storage                                         │
│    Folder: reports/dltransaction/DBSSSGSG/                     │
│    File:   DL_Transaction_<REPORT_DATE>_<YYYYMMDDHHMMSS>.xlsx  │
└───────────────────────────────┬───────────────────────────────┘
                                │ Polled hourly (skip if already loaded)
                                ▼
┌───────────────────────────────────────────────────────────────┐
│ 3. Revamped Python ETL Job (tableaudataloadpy/fixed_deposit)   │
│    • Resolve report date(s) from file names                    │
│    • Filter CONTRACT_TYPE == 'L_FD'                            │
│    • Validate schema & clean types                             │
│    • Atomic DELETE + bulk INSERT per report_date               │
│    • Housekeeping (3 latest dates + 11 prior month-ends)       │
│    • Audit row + structured logs                               │
└───────────────────────────────┬───────────────────────────────┘
                                │ Persists into MariaDB
                                ▼
┌───────────────────────────────────────────────────────────────┐
│ 4. Database Layer (MariaDB — ILMA schema)                      │
│    tbl_txn_fixed_deposit, tbl_etl_job_audit                    │
└───────────────────────────────┬───────────────────────────────┘
                                │ SQL (SQLAlchemy)
                                ▼
┌───────────────────────────────────────────────────────────────┐
│ 5. Backend REST API (ilms-rpt-data-query-service — FastAPI)    │
│    Parity:   /filter-options, /overview, /maturity-by-currency,│
│              /details, /export-excel                           │
│    Phase 1:  /maturity-ladder, /maturity-calendar,             │
│              /country-bucket-matrix, /concentration, /trend    │
└───────────────────────────────┬───────────────────────────────┘
                                │ JSON / streamed XLSX
                                ▼
┌───────────────────────────────────────────────────────────────┐
│ 6. Frontend Web Application (ilms-rpt-web — React 18 + TS)     │
│    Tab 1: Overview (KPIs + 3 stacked bars + maturity ladder)   │
│    Tab 2: Maturity by Currency (pivot + calendar heatmap)      │
│    Tab 3: Details (AG Grid + Excel export)                     │
│    Tab 4: Trend & Concentration (Phase 1)                      │
└───────────────────────────────────────────────────────────────┘
```

> **Data freshness:** the dataset is a **daily T-2 snapshot**. Despite the ILMA project name, this dashboard is not intraday. Show "Report Date" and "Last Loaded" (`MAX(load_time)`) in the page header.

> **Verify:** the DriveX folder name `DBSSSGSG` (three S's) is copied from v1.0. Confirm it is not a typo of `DBSSGSG` before deployment.

---

## 3. Data Model

### 3.1 Table: `ILMA.tbl_txn_fixed_deposit`

One row per FD contract per `report_date`. Retention: 3 latest report dates + 11 prior month-ends (~14 dates, ~14,000 rows max).

| Group | Columns |
|---|---|
| Keys & metadata | `id` (PK, auto-inc), `partition_key`, `report_date`, `load_time`, `load_count` (rows loaded in that run), `source_file`, `source_table`, `table_name`, `import_source` |
| Contract | `contract_reference`, `contract_type` (always `L_FD` after ETL filter), `liq_product_type`, `asset_liability`, `book_code`, `company_code`, `ccy_code` |
| Counterparty | `counterparty`, `cpty_desc`, `bis_entity_type`, `interco`, `interco_type` |
| Dates | `value_date`, `value_date_status`, `maturity_date`, `maturity_date_status`, `begin_date`, `end_date` |
| Amounts | `outstanding`, `nominal`, `exchange_rate`, `principal_paid`, `principal_received`, `interest_paid`, `interest_received` |
| Liquidity / LCR attributes | `lcr_amount_outflow`, `is_operational`, `is_protected`, `run_off_sme_retail_pref`, `legal_withdraw`, `liq_performing_status`, `eligibility_level`, `committed`, `self_issued`, `is_double_counted` |
| Generic | `attribute_2`, `attribute_3`, `attribute_7`, `attribute_9`, `customer_field10` (meaning to be confirmed, see §11) |

### 3.2 Key Fields & Business Definitions

| Column | Business Usage | Mapping / Derivation |
|---|---|---|
| `contract_reference` | Unique deal identifier | From MOT/RMG export. Used in Details view. |
| `company_code` | Operating entity / country | Filter + Y-axis in Country chart (e.g. `DBSSG_SG`, `DBSTW_TW`). |
| `ccy_code` | Currency code | Filter + Y-axis in Currency chart. |
| `bis_entity_type` | Counterparty classification | Colour legend: `BANK`, `CORP`, `FINA`, `INDIV`, `PSE`, `SOV`. |
| `counterparty` | Counterparty entity name | Filter + Y-axis in Counterparty chart. Literal string `'Null'` appears for `INDIV` rows and is displayed as **"Individuals (masked)"**. |
| `interco` / `interco_type` | Intragroup flag | Drives the Phase 1 **Intragroup toggle** (§6.2). |
| `value_date` | Deposit inception date | `Original Tenor = DATEDIFF(maturity_date, value_date)` |
| `maturity_date` | Deposit maturity date | `Remaining Tenor = DATEDIFF(maturity_date, report_date)` |
| `outstanding` | Outstanding balance (deal CCY) | Raw balance in transaction currency; negative for liabilities. |
| `exchange_rate` | FX rate to SGD | **Multiplier** (SGD per 1 unit of deal CCY). |
| *Outstanding Amount (SGD Mn)* | Calculated metric | `outstanding × exchange_rate / 1,000,000`, displayed as `($X.XX)` because liabilities are credit/outflow balances. |

### 3.3 Schema Changes (DDL)

`exchange_rate DECIMAL(10,6)` keeps only 6 decimals. As an SGD multiplier, low-value currencies lose material precision:

| CCY | Approx. SGD per unit | Stored as | Relative error |
|---|---|---|---|
| VND | 0.0000526 | 0.000053 | ~0.8% |
| IDR | 0.0000815 | 0.000082 | ~0.6% |
| KRW | 0.000935 | 0.000935 | ~0.05% |

Both Tableau and React read the same stored value, so parity checks will still pass. The error is against the true SGD equivalent. Confirm the precision in the DL_Transaction source file, then apply:

```sql
ALTER TABLE ILMA.tbl_txn_fixed_deposit
    MODIFY COLUMN exchange_rate DECIMAL(24,12) NULL DEFAULT NULL,
    ADD INDEX idx_fd_report_date (report_date),
    ADD INDEX idx_fd_rd_company_ccy (report_date, company_code, ccy_code);

CREATE TABLE IF NOT EXISTS ILMA.tbl_etl_job_audit (
    id            INT NOT NULL AUTO_INCREMENT,
    job_name      VARCHAR(100) NOT NULL,
    report_date   DATE NULL,
    source_file   VARCHAR(500) NULL,
    status        VARCHAR(20) NOT NULL,      -- SUCCESS | SKIPPED | FAILED
    rows_read     INT NULL,
    rows_loaded   INT NULL,
    total_sgd_mn  DECIMAL(20,6) NULL,
    message       VARCHAR(2000) NULL,
    started_at    DATETIME NOT NULL,
    finished_at   DATETIME NULL,
    PRIMARY KEY (id),
    INDEX idx_audit_job_date (job_name, report_date)
) COLLATE = 'utf8mb4_general_ci';
```

> **Note:** `lcr_amount_outflow` and all flag columns are `VARCHAR(500)`. Cast them in SQL (`CAST(lcr_amount_outflow AS DECIMAL(20,6))`) and normalise flags (`'Y'/'N'`, `'1'/'0'`, `'TRUE'/'FALSE'`) in a single SQL view or helper, not ad hoc in each endpoint. Retyping these columns is deferred to Phase 2.
>
> **Collation:** table is `utf8mb3`. Chinese counterparty names work, but 4-byte characters will fail. Migrate to `utf8mb4` at the next maintenance window.

---

## 4. Python ETL Job Revamp (`fixed_deposit.py`)

### 4.1 Current State vs. Revamped State

| Feature | Legacy Script (`fixed_deposit.py`) | Revamped Job |
|---|---|---|
| **Report date** | `today − 2 calendar days`. Fails on Mondays and after holidays (no file for Saturday). | Report date parsed from DriveX file names. Catches up any unloaded dates in a look-back window. |
| **DriveX file matching** | Exact hard-coded timestamp (`091500.xlsx`). Fails if the file arrives at `091502.xlsx`. | Regex `^DL_Transaction_(\d{8})_(\d{14})\.xlsx$` via DriveX listing API with retry and exponential back-off. Latest timestamp wins if re-pushed. |
| **Database insertion** | `df.to_sql(..., if_exists='append')`. Slow, row-by-row, no type mapping. | SQLAlchemy Core bulk insert (or `LOAD DATA LOCAL INFILE`) inside one transaction. |
| **Column normalisation** | Relies on Excel header case matching the DB exactly. | Explicit `COLUMN_MAPPING` covering **all** table columns. Unknown headers logged; missing required headers fail the job. |
| **Idempotency** | Appends; reruns duplicate data. | `DELETE` + `INSERT` for the report date in one transaction. Hourly reruns skip if `source_file` already loaded. |
| **Housekeeping** | Python loop (`for i in range(0, 10)`), one query per month. | Single set-based SQL statement (§4.2). |
| **Environment decoupling** | Hard-coded `/apps/ilmaapps/...` paths. | Environment variables / 12-factor config. |
| **Observability** | Basic file logger; `load_count` was NULL. | Structured JSON logs to Logstash, `tbl_etl_job_audit` row per run, alert on failure or on no file by 11:00 SGT. |

### 4.2 Optimised Housekeeping Strategy

**Retention rule:** the **3 latest report dates** plus the **last report date of each of the 11 preceding calendar months**.

```sql
DELETE FROM ILMA.tbl_txn_fixed_deposit
WHERE DATE(report_date) NOT IN (
    -- 3 latest report dates
    SELECT d FROM (
        SELECT DISTINCT DATE(report_date) AS d
        FROM ILMA.tbl_txn_fixed_deposit
        ORDER BY d DESC
        LIMIT 3
    ) AS recent_days
)
AND DATE(report_date) NOT IN (
    -- last available report date in each of the 11 previous calendar months
    SELECT d FROM (
        SELECT MAX(DATE(report_date)) AS d
        FROM ILMA.tbl_txn_fixed_deposit
        WHERE report_date >= DATE_FORMAT(CURDATE() - INTERVAL 11 MONTH, '%Y-%m-01')
          AND report_date <  DATE_FORMAT(CURDATE(), '%Y-%m-01')
        GROUP BY YEAR(report_date), MONTH(report_date)
    ) AS month_ends
);
```

- The derived-table wrappers (`SELECT d FROM (...) AS x`) are required. MariaDB rejects `LIMIT` inside `IN` subqueries and rejects subqueries on the table being deleted from, unless they are materialised.
- v1.0's separate "older than 1 year" `DELETE` is no longer needed, because the statement above already covers it.
- Housekeeping runs **only after a successful load**, so a failed run never shrinks history.

### 4.3 Revamped ETL Script

Path: `tableaudataloadpy/fixed_deposit/prj/fixed_deposit_revamped.py`

```python
"""
Fixed Deposit ETL Pipeline (Revamped)
Source:      ILMS DriveX (DL_Transaction reports)
Destination: MariaDB (ILMA.tbl_txn_fixed_deposit)
"""
import os
import re
import sys
import logging
import argparse
from datetime import datetime, timedelta

import pandas as pd
from sqlalchemy import create_engine, text
from logstash_async.handler import AsynchronousLogstashHandler

# ---------------------------------------------------------------------------
# 1. Configuration (12-factor)
# ---------------------------------------------------------------------------
DB_URL          = os.environ["ILMA_DB_URL"]               # mysql+pymysql://...
DRIVEX_FOLDER   = os.getenv("FD_DRIVEX_FOLDER", "reports/dltransaction/DBSSSGSG/")
LOOKBACK_DAYS   = int(os.getenv("FD_LOOKBACK_DAYS", "7"))
TMP_DIR         = os.getenv("FD_TMP_DIR", "/tmp/fd_etl")
JOB_NAME        = "FIXED_DEPOSIT_ETL"
TABLE           = "ILMA.tbl_txn_fixed_deposit"
FILE_RE         = re.compile(r"^DL_Transaction_(\d{8})_(\d{14})\.xlsx$")

# ---------------------------------------------------------------------------
# 2. Explicit column mapping: Excel UPPERCASE -> DB lowercase (ALL columns)
# ---------------------------------------------------------------------------
COLUMN_MAPPING = {
    "PARTITION_KEY": "partition_key",       "ASSET_LIABILITY": "asset_liability",
    "ATTRIBUTE_2": "attribute_2",           "ATTRIBUTE_3": "attribute_3",
    "ATTRIBUTE_7": "attribute_7",           "ATTRIBUTE_9": "attribute_9",
    "BIS_ENTITY_TYPE": "bis_entity_type",   "BOOK_CODE": "book_code",
    "CCY_CODE": "ccy_code",                 "COMMITTED": "committed",
    "COMPANY_CODE": "company_code",         "CONTRACT_REFERENCE": "contract_reference",
    "CONTRACT_TYPE": "contract_type",       "COUNTERPARTY": "counterparty",
    "CPTY_DESC": "cpty_desc",               "ELIGIBILITY_LEVEL": "eligibility_level",
    "EXCHANGE_RATE": "exchange_rate",       "IMPORT_SOURCE": "import_source",
    "INTERCO": "interco",                   "INTERCO_TYPE": "interco_type",
    "IS_DOUBLE_COUNTED": "is_double_counted", "IS_OPERATIONAL": "is_operational",
    "IS_PROTECTED": "is_protected",         "LCR_AMOUNT_OUTFLOW": "lcr_amount_outflow",
    "LEGAL_WITHDRAW": "legal_withdraw",     "LIQ_PERFORMING_STATUS": "liq_performing_status",
    "LIQ_PRODUCT_TYPE": "liq_product_type", "MATURITY_DATE": "maturity_date",
    "MATURITY_DATE_STATUS": "maturity_date_status", "OUTSTANDING": "outstanding",
    "RUN_OFF_SME_RETAIL_PREF": "run_off_sme_retail_pref", "SELF_ISSUED": "self_issued",
    "SOURCE_TABLE": "source_table",         "TABLE_NAME": "table_name",
    "VALUE_DATE": "value_date",             "VALUE_DATE_STATUS": "value_date_status",
    "NOMINAL": "nominal",                   "CUSTOMER_FIELD10": "customer_field10",
    "BEGIN_DATE": "begin_date",             "END_DATE": "end_date",
    "PRINCIPAL_PAID": "principal_paid",     "PRINCIPAL_RECEIVED": "principal_received",
    "INTEREST_PAID": "interest_paid",       "INTEREST_RECEIVED": "interest_received",
}
REQUIRED = ["CONTRACT_REFERENCE", "CONTRACT_TYPE", "COMPANY_CODE", "CCY_CODE",
            "BIS_ENTITY_TYPE", "OUTSTANDING", "EXCHANGE_RATE",
            "VALUE_DATE", "MATURITY_DATE"]
DATE_COLS    = ["value_date", "maturity_date", "begin_date", "end_date"]
NUMERIC_COLS = ["outstanding", "nominal", "exchange_rate", "principal_paid",
                "principal_received", "interest_paid", "interest_received"]

log = logging.getLogger(JOB_NAME)


def list_candidate_files(drivex, today: datetime) -> dict:
    """Return {report_date: latest_file_name} within the look-back window."""
    cutoff = (today - timedelta(days=LOOKBACK_DAYS)).strftime("%Y%m%d")
    latest = {}
    for name in drivex.list_files(DRIVEX_FOLDER):          # retry/backoff inside client
        m = FILE_RE.match(name)
        if not m or m.group(1) < cutoff:
            continue
        rd, ts = m.group(1), m.group(2)
        if rd not in latest or ts > FILE_RE.match(latest[rd]).group(2):
            latest[rd] = name
    return latest


def already_loaded(conn, report_date: str, source_file: str) -> bool:
    q = text(f"SELECT 1 FROM {TABLE} WHERE report_date = :rd "
             "AND source_file = :sf LIMIT 1")
    return conn.execute(q, {"rd": report_date, "sf": source_file}).first() is not None


def transform(raw: pd.DataFrame, report_date: str, source_file: str) -> pd.DataFrame:
    raw.columns = [c.strip().upper() for c in raw.columns]
    missing = [c for c in REQUIRED if c not in raw.columns]
    if missing:
        raise ValueError(f"Missing required columns: {missing}")
    unknown = set(raw.columns) - set(COLUMN_MAPPING)
    if unknown:
        log.warning("Unmapped source columns ignored: %s", sorted(unknown))

    df = raw[raw["CONTRACT_TYPE"] == "L_FD"].copy()
    df = df[[c for c in COLUMN_MAPPING if c in df.columns]].rename(columns=COLUMN_MAPPING)

    for c in DATE_COLS:
        if c in df:
            df[c] = pd.to_datetime(df[c], errors="coerce")
    for c in NUMERIC_COLS:
        if c in df:
            df[c] = pd.to_numeric(df[c], errors="coerce")

    # Validation rules (fail fast)
    if df["contract_reference"].duplicated().any():
        raise ValueError("Duplicate contract_reference within report date")
    if (df["exchange_rate"].isna() | (df["exchange_rate"] <= 0)).any():
        raise ValueError("Null or non-positive exchange_rate found")
    bad_dates = df["maturity_date"] < df["value_date"]
    if bad_dates.any():
        log.warning("%d rows with maturity_date < value_date", int(bad_dates.sum()))

    df["report_date"] = pd.to_datetime(report_date)
    df["load_time"]   = datetime.now()
    df["load_count"]  = len(df)
    df["source_file"] = source_file
    return df


def load(engine, df: pd.DataFrame, report_date: str) -> None:
    """Atomic replace of one report date."""
    records = df.astype(object).where(pd.notnull(df), None).to_dict("records")
    cols = list(df.columns)
    insert_sql = text(f"INSERT INTO {TABLE} ({', '.join(cols)}) "
                      f"VALUES ({', '.join(':' + c for c in cols)})")
    with engine.begin() as conn:                              # single transaction
        conn.execute(text(f"DELETE FROM {TABLE} WHERE report_date = :rd"),
                     {"rd": report_date})
        conn.execute(insert_sql, records)                     # executemany batch


def housekeeping(engine) -> None:
    with engine.begin() as conn:
        conn.execute(text(open(os.path.join(os.path.dirname(__file__),
                                            "sql/housekeeping.sql")).read()))


def execute_etl(today: datetime, drivex) -> int:
    engine = create_engine(DB_URL, pool_pre_ping=True)
    loaded = 0
    for rd_str, fname in sorted(list_candidate_files(drivex, today).items()):
        rd = datetime.strptime(rd_str, "%Y%m%d").strftime("%Y-%m-%d")
        with engine.connect() as conn:
            if already_loaded(conn, rd, fname):
                log.info("Skip %s (already loaded from %s)", rd, fname)
                continue
        path = drivex.download(DRIVEX_FOLDER + fname, TMP_DIR)
        raw = pd.read_excel(path, engine="openpyxl", dtype=str)
        df = transform(raw, rd, fname)
        load(engine, df, rd)
        total = (df["outstanding"] * df["exchange_rate"]).sum() / 1e6
        log.info("Loaded %s: %d rows, total SGD mn %.2f", rd, len(df), total)
        # write SUCCESS row to tbl_etl_job_audit here
        loaded += 1
    if loaded:
        housekeeping(engine)
    return loaded


if __name__ == "__main__":
    p = argparse.ArgumentParser(description="Fixed Deposit Data Load ETL")
    p.add_argument("--days_offset", type=int, default=0,
                   help="Shift 'today' back N days for backfill/testing")
    args = p.parse_args()
    try:
        from drivex_client import DriveXClient                # internal client
        n = execute_etl(datetime.today() - timedelta(days=args.days_offset),
                        DriveXClient.from_env())
        log.info("ETL completed. Report dates loaded: %d", n)
    except Exception:
        log.exception("ETL failed")                           # audit FAILED + alert
        sys.exit(1)
```

> **Note on `dtype=str`:** reading every column as string, then casting explicitly, avoids pandas silently turning contract references into floats or dropping leading zeros.

---

## 5. Backend REST API — Parity Endpoints (`ilms-rpt-data-query-service`)

FastAPI endpoints using SQLAlchemy connections to MariaDB. At ~1,000 rows per report date, every aggregate is a single indexed query. DuckDB is optional and only worthwhile if the service later caches multi-date extracts in memory.

### 5.1 Common Request Body

All `POST` endpoints accept:

```json
{
  "report_date": "2025-02-18",
  "filters": {
    "countries": [], "book_codes": [], "currencies": [],
    "bis_entity_types": [], "counterparties": [],
    "value_date_from": null, "value_date_to": null,
    "maturity_date_from": null, "maturity_date_to": null
  },
  "intragroup": "all",          // all | exclude | only   (Phase 1)
  "restricted_ccy_only": false  // Phase 1
}
```

An empty list means *All*. The `report_date` default is the latest available date.

### 5.2 Base Query (shared CTE)

```sql
WITH fd AS (
    SELECT t.*,
           t.outstanding * t.exchange_rate / 1000000          AS outstanding_sgd_mn,
           DATEDIFF(t.maturity_date, t.report_date)           AS rem_days,
           DATEDIFF(t.maturity_date, t.value_date)            AS orig_days
    FROM ILMA.tbl_txn_fixed_deposit t
    WHERE t.report_date = :report_date
      AND t.contract_type = 'L_FD'
      /* + dynamic filter predicates, always bound parameters, never string-formatted */
)
```

### 5.3 Business Formulas & Calculation Logic

**A. Outstanding Volume (SGD Mn)**

$$\text{Outstanding SGD Mn} = \sum \frac{\text{outstanding} \times \text{exchange\_rate}}{1{,}000{,}000}$$

Displayed in accounting notation, e.g. `($44,058.58)`, to denote liability balances.

**B. Weighted Remaining Tenor (Days)**

$$\text{WRT} = \frac{\sum \big(\max(0,\ \text{rem\_days}) \times |\text{outstanding\_sgd}|\big)}{\sum |\text{outstanding\_sgd}|}$$

**C. Weighted Original Tenor (Days)**

$$\text{WOT} = \frac{\sum \big(\text{orig\_days} \times |\text{outstanding\_sgd}|\big)}{\sum |\text{outstanding\_sgd}|}$$

Rows with NULL `maturity_date` are excluded from both tenor numerators **and** denominators. Display rounded to whole days.

**D. Remaining Maturity Tenor Buckets** (Δ = `rem_days`)

```sql
CASE
    WHEN maturity_date IS NULL           THEN 'Open / No Maturity'
    WHEN rem_days <= 0                   THEN 'Matured / Today'
    WHEN rem_days = 1                    THEN '1 day'
    WHEN rem_days = 2                    THEN '2 days'
    WHEN rem_days = 3                    THEN '3 days'
    WHEN rem_days BETWEEN 4   AND 6      THEN '4-6 days'
    WHEN rem_days = 7                    THEN '7 days'
    WHEN rem_days BETWEEN 8   AND 14     THEN '1-2 weeks'
    WHEN rem_days BETWEEN 15  AND 28     THEN '2-4 weeks'
    WHEN rem_days BETWEEN 29  AND 60     THEN '1-2 mths'
    WHEN rem_days BETWEEN 61  AND 90     THEN '2-3 mths'
    WHEN rem_days BETWEEN 91  AND 180    THEN '3-6 mths'
    WHEN rem_days BETWEEN 181 AND 365    THEN '6-12 mths'
    ELSE '> 1 year'
END AS remaining_bucket,
CASE  /* sort key for UI column order */
    WHEN maturity_date IS NULL THEN 99 WHEN rem_days <= 0 THEN 0
    WHEN rem_days <= 3 THEN rem_days WHEN rem_days <= 6 THEN 4 WHEN rem_days = 7 THEN 5
    WHEN rem_days <= 14 THEN 6 WHEN rem_days <= 28 THEN 7 WHEN rem_days <= 60 THEN 8
    WHEN rem_days <= 90 THEN 9 WHEN rem_days <= 180 THEN 10 WHEN rem_days <= 365 THEN 11
    ELSE 12
END AS bucket_order
```

> **Fixes vs v1.0:** in v1.0, Δ ≤ 0 and NULL maturity fell into `ELSE '> 1 year'`, overstating long-dated funding. The legacy Tableau label `6 days` is displayed as `4-6 days`. If the Tableau calculated field shares the v1.0 bug, reconciliation for those cells is documented as an **accepted, intentional difference** (§8.1), not a defect.

### 5.4 API Specifications

#### 1. Filter Options — `GET /api/v1/fixed-deposit/filter-options`

- **Query params:** `report_date: Optional[str]`
- **Response:**

```json
{
  "report_dates": [
    {"date": "2025-02-18", "type": "daily"},
    {"date": "2025-02-17", "type": "daily"},
    {"date": "2025-02-14", "type": "daily"},
    {"date": "2025-01-31", "type": "month_end"}
  ],
  "latest_load_time": "2025-02-20T09:42:11",
  "countries": ["DBSSG_SG", "DBSSG_IN", "DBSTW_TW", "DBSSG_AU", "DBSSG_HK"],
  "book_codes": ["DBSSG_SG_BK", "DBSSG_IN_BK"],
  "currencies": ["USD", "TWD", "SGD", "CNY", "AUD", "INR", "EUR", "HKD", "GBP"],
  "bis_entity_types": ["BANK", "CORP", "FINA", "INDIV", "PSE", "SOV"],
  "counterparties": ["DBS BANK LTD", "LINKING ALPHA LIMITED", "DHMA LIMITED"]
}
```

#### 2. Overview — `POST /api/v1/fixed-deposit/overview`

- **Body:** common request (§5.1)
- **Response** (illustrative values):

```json
{
  "kpis": {
    "outstanding_volume_sgd_mn": -44058.58,
    "weighted_remaining_tenor_days": 190,
    "weighted_original_tenor_days": 241
  },
  "holdings_by_country": [
    {"country": "DBSSG_SG", "bis_entity_type": "CORP", "amount_sgd_mn": -1650.20}
  ],
  "holdings_by_currency": [
    {"currency": "USD", "bis_entity_type": "BANK", "amount_sgd_mn": -5210.33}
  ],
  "holdings_by_counterparty": [
    {"counterparty": "DBS BANK LTD", "bis_entity_type": "BANK", "amount_sgd_mn": -6932.17}
  ]
}
```

`holdings_by_counterparty` returns the top 20 counterparties by `|amount|`. The rest are rolled into `"Others"` only when requested via `include_others=true`.

#### 3. Maturity by Currency — `POST /api/v1/fixed-deposit/maturity-by-currency`

- **Response:** pivot rows `{currency, total, buckets: {<bucket>: amount}}`, plus a `grand_total` row and an ordered `bucket_columns` array (from `bucket_order`), so the UI never hard-codes bucket order.

#### 4. Details — `POST /api/v1/fixed-deposit/details`

- **Params:** `page: int = 1`, `page_size: int = 100` (max 1000), `sort_col: str`, `sort_dir: asc|desc`, `search: Optional[str]`
- **Response:** `{rows: [...], total_rows, grand_total_sgd_mn}` where each row has `contract_reference, country, book_code, currency, bis_entity_type, counterparty, value_date, maturity_date, outstanding_amount_sgd_mn`.
- `sort_col` is validated against a whitelist.

#### 5. Export — `POST /api/v1/fixed-deposit/export-excel`

- Same body as Details (no paging). Streams an XLSX with a header block showing report date, applied filters and export timestamp.

---

## 6. Phase 1 Enhancements (existing data only)

Design principle: **parity tabs stay pixel-for-pixel comparable to Tableau** for UAT. New visuals are additive panels or a new tab, so they never block reconciliation sign-off.

### 6.1 Feature Summary

| # | Feature | User question answered | Location | Data used |
|---|---|---|---|---|
| E1 | **Maturity Ladder + Cumulative Outflow** | "When does cumulative maturing funding cross my buffer?" | Tab 1, below KPIs | `rem_days`, `ccy_code`, amount |
| E2 | **Maturity Calendar Heatmap** | "Are there single-day maturity cliffs?" | Tab 2, toggle "Table / Calendar" | `maturity_date`, `ccy_code` |
| E3 | **Country × Bucket Heatmap** | "Which entity has a crunch coming?" (Group TLM) | Tab 4 | `company_code`, bucket |
| E4 | **Counterparty Concentration (Pareto)** | "How dependent are we on the top names?" | Tab 4 | `counterparty`, `bis_entity_type`, `interco` |
| E5 | **Outstanding Trend** | "Is the FD book growing or shrinking, and in which segment?" | Tab 4 | all retained `report_date`s |
| E6 | **Intragroup toggle** | "What is third-party funding excluding DBS entities?" | Global filter bar | `interco` / `interco_type` |
| E7 | **Restricted-currency toggle** | "What is the gap in currencies that can't be freely moved?" | Global filter bar | `ccy_code` + config list |

### 6.2 Global Toggles

- **Intragroup (`intragroup`)**: `all` (default, parity) | `exclude` | `only`. The predicate uses the normalised `interco` flag. The **"DBS BANK LTD" counterparty is the #1 name**, so concentration (E4) defaults to `exclude`.
- **Restricted currencies (`restricted_ccy_only`)**: driven by config `FD_RESTRICTED_CCY`, initial value `CNY,INR,IDR,TWD,KRW,VND` (to be confirmed with TLM). When `true`, it limits the dataset to these currencies. All charts also show a small "R" badge on restricted currencies.

### 6.3 New Endpoints

#### E1. `POST /api/v1/fixed-deposit/maturity-ladder`

```json
{
  "bucket_columns": ["Matured / Today", "1 day", "2 days", "...", "> 1 year"],
  "series": [
    {"currency": "USD", "values": [0, -2993.04, -824.60, "..."]}
  ],
  "cumulative": [0, -4818.24, -5924.87, "..."]
}
```

- **Chart:** ECharts stacked vertical bars (top 6 currencies by volume + "Other") with a cumulative line on a secondary axis.
- **Optional:** a user-entered buffer threshold line. It is stored in local UI state only, and persistence is deferred to Phase 2.

#### E2. `POST /api/v1/fixed-deposit/maturity-calendar`

- **Params:** `horizon_days` (default 90, max 365)
- **Response:** `[{date, currency, amount_sgd_mn, deal_count}]` for `report_date < maturity_date ≤ report_date + horizon`.
- **Chart:** ECharts calendar heatmap (one row per week, colour = |amount|). Clicking a day drills into Tab 3 filtered on that maturity date.
- **Limitation:** no business-day calendar is available, so weekend/holiday maturities are shown on their contractual date (§10).

#### E3. `POST /api/v1/fixed-deposit/country-bucket-matrix`

- **Response:** `{rows: [{country, total, buckets: {...}}], bucket_columns}`
- **Chart:** ECharts heatmap with rows = `company_code` and columns = buckets up to `6-12 mths`. Colour is set by the cell's share of the country's total, so that small countries remain readable. The tooltip shows the absolute amount.

#### E4. `POST /api/v1/fixed-deposit/concentration`

- **Params:** `top_n` (default 20), `exclude_indiv` (default `true`; the masked `'Null'` INDV bucket would otherwise rank as one giant "counterparty")
- **Response:** `[{rank, counterparty, bis_entity_type, amount_sgd_mn, share_pct, cumulative_share_pct}]` plus `total_sgd_mn`, `top10_share_pct`, `top20_share_pct`.
- **Chart:** Pareto, with bars for amount and a line for cumulative %. KPI chips show the Top-10 and Top-20 shares.

#### E5. `POST /api/v1/fixed-deposit/trend`

- **Params:** `group_by`: `bis_entity_type` | `ccy_code` | `company_code`
- **Response:** `[{report_date, date_type: "daily"|"month_end", group, amount_sgd_mn}]` across all retained dates (≤14).
- **Chart:** stacked area. The x-axis is **categorical**, because dates are irregular (11 month-ends + 3 recent days). Month-end points use hollow markers and the three daily points use solid markers, with a visual break between them.
- **Note:** the trend depth is capped by the retention rule (§4.2). Extending history is a business decision with storage impact (~1k rows/date).

### 6.4 LCR Fields (candidate, not committed)

The table already holds `lcr_amount_outflow`, `is_operational`, `is_protected` and `run_off_sme_retail_pref`. A **behavioural (LCR-weighted) outflow ladder** could be built in Phase 1, but only once TLM confirms these values are populated and authoritative for FDs. Until then it remains a Phase 2 item, to avoid publishing an unvalidated regulatory-style figure.

---

## 7. Frontend Architecture & React Implementation (`ilms-rpt-web`)

### 7.1 Component Hierarchy

```
ilms-rpt-web/src/components/FixedDeposit/
├── FixedDepositPage.tsx              # Main container: 4 MUI Tabs, global Filter Bar, header (report date + last loaded)
├── components/
│   ├── FixedDepositFilterBar.tsx     # 6 multi-selects + 2 date-range pickers + Intragroup / Restricted toggles + Reset
│   ├── FixedDepositOverviewTab.tsx   # 3 KPI cards + 3 horizontal stacked bars + MaturityLadderChart
│   ├── FixedDepositMaturityTab.tsx   # Currency × Tenor pivot (AG Grid) ⇄ MaturityCalendarHeatmap
│   ├── FixedDepositDetailsTab.tsx    # Paginated AG Grid + Excel export
│   ├── FixedDepositTrendTab.tsx      # Trend, Concentration Pareto, Country × Bucket heatmap   (Phase 1)
│   ├── FixedDepositKpiCard.tsx       # Reusable styled KPI card
│   └── charts/
│       ├── StackedHoldingsBar.tsx
│       ├── MaturityLadderChart.tsx         (Phase 1)
│       ├── MaturityCalendarHeatmap.tsx     (Phase 1)
│       ├── CountryBucketHeatmap.tsx        (Phase 1)
│       ├── ConcentrationPareto.tsx         (Phase 1)
│       └── OutstandingTrendChart.tsx       (Phase 1)
├── hooks/
│   ├── useFixedDepositFilters.ts     # Filter state, URL sync (shareable links), reset
│   └── useFixedDepositData.ts        # React Query hooks per endpoint (keyed on filter state)
├── utils/
│   ├── formatters.ts                 # Accounting format ($X,XXX.XX), DD/MM/YYYY dates
│   └── bisEntityColors.ts            # Single source of truth for BIS entity colours
├── types/
│   └── fixedDeposit.types.ts         # TS interfaces for requests / responses
└── fixedDeposit.css                  # Component styles adhering to ILMA theme
```

### 7.2 Cross-cutting UI Rules

- **Filters:** Report Date is a single-select (default latest). Country, Book Code, Currency, BIS Entity Type and Counterparty are multi-selects. Value Date and Maturity Date are **date-range pickers**, replacing Tableau's long date dropdowns. Filter state is synced to the URL query string.
- **Number format:** `($44,058.58)` with the unit `mio`. Negative values are shown in brackets, never with a minus sign. Values are right-aligned with 2 decimals.
- **BIS entity colours:** fixed mapping in `bisEntityColors.ts`, used identically by every chart and legend.
- **Empty state:** "No fixed deposits match the current filters", with a Reset link.
- **Loading:** skeletons per panel. A slow panel must not block the others.

### 7.3 View Specifications

#### Tab 1: Overview
- **Header:** red banner styled identically to existing ILMA dashboards (`FD Outstanding Volume - Overview`).
- **KPI cards:** Outstanding Volume `($44,058.58)` mio · Weighted Remaining Tenor `190 days` · Weighted Original Tenor `241 days`.
- **BIS entity legend:** horizontal colour badges (`BANK`, `CORP`, `FINA`, `INDIV`, `PSE`, `SOV`).
- **3 horizontal stacked bar charts** (ECharts):
  1. Holdings: Country × BIS Entity Type
  2. Holdings: Currency × BIS Entity Type
  3. Holdings: Counterparty × BIS Entity Type (top 20, sorted descending by |amount|)
- **Phase 1:** Maturity Ladder + Cumulative line (E1) below the bar charts.

#### Tab 2: Maturity by Currency
- **Row dimension:** Currency (alphabetical).
- **Column dimensions:** Total, then buckets in `bucket_order` (`Matured / Today` and `Open / No Maturity` are shown only when non-zero).
- **Summary row/column:** Grand Total pinned to the top row and first column. Empty cells are rendered blank.
- **Phase 1:** "Table / Calendar" toggle switching to the Maturity Calendar Heatmap (E2).

#### Tab 3: Details
- AG Grid (server-side pagination) with a pinned Grand Total row `($44,058.58)`.
- **Columns:**
  1. Contract Reference (String, left-aligned)
  2. Country (String, centre)
  3. Book Code (String, centre)
  4. Currency (String, centre)
  5. BIS Entity Type (String, centre)
  6. Counterparty (String, left-aligned)
  7. Value Date (`DD/MM/YYYY`)
  8. Maturity Date (`DD/MM/YYYY`)
  9. Outstanding Amount (Mn) (number, right-aligned, `($XX.XX)`)
- **Actions:** Export to Excel (XLSX), column reorder, quick text search.

#### Tab 4: Trend & Concentration (Phase 1)
- Outstanding Trend (E5) with a group-by selector.
- Counterparty Concentration Pareto (E4) with Top-10 / Top-20 share chips.
- Country × Bucket Heatmap (E3).

---

## 8. Quality Assurance, Testing & Reconciliation

### 8.1 Tableau vs. React Reconciliation Checklist

Run automatically before UAT sign-off, for **at least 3 report dates including one month-end**:

| Check Item | Validation Rule | Acceptance Criteria |
|---|---|---|
| Grand Total Outstanding | `SUM(outstanding * exchange_rate / 1e6)` in React vs Tableau | Exact match (0.00% variance) |
| Weighted Remaining Tenor | Formula output vs Tableau calculated field, identical filters | Within ±0.5 day (rounding) |
| Weighted Original Tenor | As above | Within ±0.5 day |
| Maturity Bucket Cell Values | Export the Maturity by Currency matrix from both | Cell-by-cell delta ≤ $0.01M, **except** documented bucket-logic fixes (§5.3 D), which are listed in the UAT pack |
| Holdings Charts | Country / Currency / Counterparty × BIS totals | Exact match per bar segment |
| Record Count | Count of `CONTRACT_TYPE = 'L_FD'` rows | 100% match |
| Cross-tab Consistency | Overview total = Details grand total = Maturity grand total | Exact match (mirrors the legacy "red tick" checks) |
| Filter Reactivity | Country / Book / Currency filter permutations recompute all cards and charts | Zero frontend runtime errors; totals match Tableau for 10 sampled permutations |

### 8.2 Unit & Integration Testing

**ETL (`pytest`)**
- Excel parsing with mock files having varying timestamps (`091500`, `091502`, re-push at `103000` → latest wins).
- Report-date resolution: Monday run loads Friday's file; holiday gap is caught up within `LOOKBACK_DAYS`.
- Non-`L_FD` contracts are discarded.
- Missing required column → job fails, audit row `FAILED`, no rows deleted.
- Re-run with the same `source_file` → `SKIPPED`, row count unchanged (idempotency).
- Atomicity: an exception injected mid-insert leaves the previous data for that report date intact.
- Housekeeping retains exactly 3 latest dates + 11 previous month-ends, over a 15-month synthetic dataset.
- `exchange_rate` precision: an IDR row at `0.0000815432` round-trips without truncation after the DDL change.

**Backend (`pytest` + `httpx`)**
- Bucket edge cases: `rem_days` = −3, 0, 1, 3, 4, 6, 7, 8, 14, 15, 28, 29, 60, 61, 90, 91, 180, 181, 365, 366, and NULL.
- Weighted tenors with zero total (all filters exclude everything) → `null`, not a divide-by-zero error.
- Filter SQL is fully parameterised; a `sort_col` outside the whitelist → HTTP 422.
- `intragroup=exclude` removes all `interco`-flagged rows; `only` returns their complement.

**Frontend (Jest + React Testing Library; Playwright E2E)**
- Accounting formatter: `-44058.58 → ($44,058.58)`, `0 → $0.00`.
- Filter URL round-trip restores identical state.
- E2E: change Report Date → all four tabs refresh; Export produces an XLSX whose total equals the on-screen grand total.

### 8.3 Data Profiling Before Build (one-off)

```sql
-- Retained history depth
SELECT DATE(report_date) d, COUNT(*) n, SUM(outstanding*exchange_rate)/1e6 sgd_mn
FROM ILMA.tbl_txn_fixed_deposit GROUP BY d ORDER BY d DESC;

-- FX precision risk
SELECT ccy_code, MIN(exchange_rate), MAX(exchange_rate)
FROM ILMA.tbl_txn_fixed_deposit GROUP BY ccy_code ORDER BY MIN(exchange_rate);

-- Flag value domains used by Phase 1 toggles
SELECT interco, interco_type, COUNT(*) FROM ILMA.tbl_txn_fixed_deposit GROUP BY 1,2;
SELECT asset_liability, is_double_counted, COUNT(*) FROM ILMA.tbl_txn_fixed_deposit GROUP BY 1,2;

-- Bucket edge populations
SELECT SUM(maturity_date IS NULL) null_mat,
       SUM(DATEDIFF(maturity_date, report_date) <= 0) matured_or_today
FROM ILMA.tbl_txn_fixed_deposit;
```

---

## 9. Deployment & Operations

- **ETL schedule:** hourly from 09:00 to 18:00 SGT on business days. Each run is a no-op if nothing new has arrived.
- **Alerting:** `FAILED` audit row → alert. No `SUCCESS` for the expected report date by 11:00 SGT → "file not received" alert.
- **Config (env vars):** `ILMA_DB_URL`, `FD_DRIVEX_FOLDER`, `FD_LOOKBACK_DAYS`, `FD_TMP_DIR`, `FD_RESTRICTED_CCY`, Logstash host/port.
- **Secrets:** DB and DriveX credentials come from the platform secret store, never from the repository.
- **Access control:** endpoints are protected by the ILMS Java auth service on OpenShift. Country-level row restriction (if required by TLM) is applied server-side on `company_code`.
- **Rollback:** the legacy Tableau workbook stays available read-only until two consecutive month-ends reconcile cleanly in production.

---

## 10. Phase 2 Backlog (requires new data or decisions)

| Item | Blocker |
|---|---|
| Holiday-adjusted maturity calendar | Business-day calendar per country |
| Rollover / renewal rate | Link between matured and renewed contract references |
| FD pricing vs benchmark (SORA / SOFR / HIBOR) | Contract interest rate field (only `interest_paid` / `interest_received` cashflows today) |
| Behavioural (LCR-weighted) outflow ladder | TLM validation of `lcr_amount_outflow` and runoff flags for FDs |
| Daily movement waterfall (new / matured / FX revaluation) | Consecutive daily snapshots (currently only 3 recent days retained) |
| Group-level counterparty concentration | Group / parent counterparty ID |
| Persisted buffer thresholds and alerts | Threshold ownership and storage |
| Longer trend history | Retention policy change |

---

## 11. Open Items / Assumptions to Confirm

| # | Item | Owner |
|---|---|---|
| 1 | DriveX folder spelling `DBSSSGSG` vs `DBSSGSG` | DevOps |
| 2 | `exchange_rate` precision in the DL_Transaction source and the DDL change approval | Data Eng / DBA |
| 3 | Do all mapped columns (e.g. `INTERCO`, LCR fields) exist in the DL_Transaction file? | Data Eng |
| 4 | Does Tableau filter on `asset_liability` or `is_double_counted`? Must replicate exactly. | BA |
| 5 | Tableau bucket logic for Δ ≤ 0 / NULL maturity (accepted-difference list for UAT) | BA / TLM |
| 6 | `interco` / `interco_type` value domain and the intended "intragroup" definition | TLM |
| 7 | Restricted-currency list | TLM |
| 8 | Meaning of `attribute_2/3/7/9`, `customer_field10`, `*_date_status` | Data Eng |
| 9 | Country-level access restriction requirement | TLM / Security |
