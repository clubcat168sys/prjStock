# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Taiwan stock market quantitative analysis engine. Downloads and processes Taiwan stock data (corporate info, pricing, broker activities), performs technical analysis, and maintains a PostgreSQL database for filtering and scoring stocks.

## Running Jobs

```bash
# Download raw stock info files
python src/twstock/jobs/download_stock_info.py

# ETL: parse CSV and upsert to DB (--data-date required)
python -m twstock.jobs.etl_stock_info --data-date 20251225

# ETL with a specific file
python -m twstock.jobs.etl_stock_info --data-date 20251225 --file /path/to/file.csv --reason "manual"

# Via Docker
docker exec -it twstock_python bash -lc "cd /app/src && python -m twstock.jobs.etl_stock_info --data-date 20251225"
```

## Environment

Copy `.env.example` to `.env` and set `PG_DSN` (PostgreSQL connection string). The DB schema defaults to `twstock`.

## Architecture

### Job Framework (`src/twstock/jobs/runner.py`)

All jobs follow the same pattern:
- `run_job()` — acquires a PostgreSQL advisory lock (keyed by job name + run date), generates a UUID `run_id`, logs RUN-level events to `job_logs`
- `job_step()` — context manager for step-level timing, row counts, and error tracking
- Advisory locks prevent duplicate daily runs; jobs can block (wait) or fail-fast (try mode)

New jobs should use `template/py/job_template.py` as a starting point.

### Database Layer (`src/twstock/db/`)

- **`pool.py`** — psycopg3 `ConnectionPool` with health checks; sets `search_path` per connection; uses dict row factory. Call `get_conn()` to acquire.
- **`lock.py`** — Session-level and transaction-level PostgreSQL advisory locks; uses stable hash of string keys.
- **`repo/lookup_code.py`** — DB-backed key-value config store (category + code_key → code_value). URLs and file paths come from here, not hardcoded.
- **`repo/job_logs.py`** — Writes structured RUN/STEP/ERR events to `job_logs` table.

### Configuration (`src/twstock/settings.py`)

Pydantic v2 settings loaded from `.env`. Singleton `settings` is imported by other modules. DB connection pool parameters (min/max size, timeout) are here.

### Data Flow

1. **Download** (`download_stock_info.py`) — Reads URLs from `lookup_code(STOCK_INFO)`, downloads raw bytes via httpx, writes to `DL_PATH` (no parsing).
2. **ETL** (`etl_stock_info.py`) — Reads CSVs from `DL_PATH`, decodes (utf-8-sig → utf-8 → cp950), parses stock fields, upserts to `stock_info`. DB trigger auto-inserts to `stock_info_chg` on relevant field changes.
3. **Calc layer** (`src/twstock/calc/`) — Not yet implemented. Planned: indicators (TA-Lib), filters (3-layer funnel), scoring.

### Planned Filter Logic (from `knowledge/filter/filter_concept.txt`)

Three-layer funnel:
1. **State filter** — Liquidity (dollar volume), volatility (BBW/ATR), close quality (CLV/upper wick)
2. **Trend filter** — VWMA position/slope, HH/HL structure, two-day confirmation
3. **Quality/veto filter** — Day trade detection, inventory trends, geographic proximity

Outputs: `state_score`, `trend_score`, `quality_score`, `quality_flag` (ok/veto).

## Key Database Tables

| Table | Purpose |
|---|---|
| `job_logs` | Append-only event log (RUN/STEP/ERR) for all jobs |
| `lookup_code` | DB-driven config KV store (URLs, paths, parameters) |
| `stock_info` | Stock master (ID, name, industry, shares, address) |
| `stock_info_chg` | Change log populated by DB trigger on `stock_info` updates |
| `stock_price` | Daily OHLCV data |
| `broker_inventory_daily` | Daily broker inventory by stock |
| `trading_date` | Trading calendar |

DDL is in `db/sql/tables/`. Initial seed data in `db/sql/seeds/seed_lookup_code.sql`.

## Documentation

Design specs live in `knowledge/design/` (job specs, module layout). Database schema docs in `db/doc/`. New table DDL should follow `template/sql/table_template.sql`.
