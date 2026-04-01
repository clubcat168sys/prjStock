# CLAUDE.md — Taiwan stock quantitative analysis engine

@claude/project_state.md

@claude/decisions.md

@claude/job-pattern.md

@claude/sop-new-job.md

@claude/sop-download-etl.md

@claude/calc-design.md

---

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

## Database Layer (`src/twstock/db/`)

- **`pool.py`** — psycopg3 `ConnectionPool`; sets `search_path` per connection; dict row factory. Call `get_conn()` to acquire.
- **`lock.py`** — Session-level and transaction-level PostgreSQL advisory locks; stable hash of string keys.
- **`repo/lookup_code.py`** — DB-backed KV config store (category + code_key → code_value).
- **`repo/job_logs.py`** — Writes RUN/STEP/ERR events to `job_logs`.
- **`settings.py`** — Pydantic v2 settings from `.env`. Singleton `settings` imported by all modules.

## Key Database Tables

DDL in `db/sql/tables/`. New tables follow `template/sql/table_template.sql`. Seed data in `db/sql/seeds/seed_lookup_code.sql`.

| Table | Purpose |
|---|---|
| `job_logs` | Append-only event log (RUN/STEP/ERR) |
| `lookup_code` | Config KV store (URLs, paths, parameters) |
| `stock_info` | Stock master (ID, name, industry, shares, address) |
| `stock_info_chg` | Change log via DB trigger on `stock_info` |
| `stock_price` | Daily OHLCV data |
| `broker_inventory_daily` | Daily broker inventory by stock |
| `trading_date` | Trading calendar |
