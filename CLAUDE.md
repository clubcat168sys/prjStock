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
# Download raw stock info files (TWSE/MOPS CSV)
python src/twstock/jobs/download_stock_info.py

# ETL: parse CSV and upsert to DB (--data-date required)
python -m twstock.jobs.etl_stock_info --data-date 20251225

# ETL with a specific file
python -m twstock.jobs.etl_stock_info --data-date 20251225 --file /path/to/file.csv --reason "manual"

# ETL FinMind daily datasets (stock_price / inst_inv / trading_daily_rpt / margin)
# 需先在 DB 設定 lookup_code(FINMIND, TOKEN)
python -m twstock.jobs.etl_finmind_daily --data-date 20251225

# Via Docker
docker exec -it twstock_python bash -lc "cd /app/src && python -m twstock.jobs.etl_stock_info --data-date 20251225"
docker exec -it twstock_python bash -lc "cd /app/src && python -m twstock.jobs.etl_finmind_daily --data-date 20251225"
```

## FinMind API Setup

1. 在 [FinMind](https://finmindtrade.com/) 取得 token
2. 更新 DB：`UPDATE twstock.lookup_code SET code_value = '<your_token>' WHERE category = 'FINMIND' AND code_key = 'TOKEN';`
3. `REQ_DELAY`（預設 1.0 秒）可依 plan 調整

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
| `stock_info` | Stock master (ID, name, industry, shares, is_warning) |
| `stock_info_chg` | Change log via DB trigger on `stock_info` |
| `stock_price` | Daily OHLCV data（FinMind: TaiwanStockPrice） |
| `inst_inv_buy_sell` | 三大法人每日買賣超（FinMind: TaiwanStockInstitutionalInvestorsBuySell） |
| `trading_daily_rpt` | 分點進出明細（FinMind: TaiwanStockTradingDailyReport），月分區 |
| `margin_pur_short_sale` | 融資融券餘額（FinMind: TaiwanStockMarginPurchaseShortSale） |
| `broker_inventory_daily` | 分點庫存日序列（衍生表），月分區 |
| `trading_date` | Trading calendar |

## Key SQL Functions

DDL in `db/sql/funcs/`.

| Function | Purpose |
|---|---|
| `fn_calc_inventory_slope(date, window_days)` | 計算分點庫存線性回歸斜率 |
| `fn_detect_accumulation(date, ...)` | 吸籌偵測主函式（5層濾網） |
