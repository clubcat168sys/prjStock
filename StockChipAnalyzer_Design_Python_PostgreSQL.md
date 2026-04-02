# 台股吸籌偵測系統

## 系統設計架構文件

- Tech Stack: Python 3.12 + PostgreSQL 16
- Data Source: FinMind API
- 週期: 日K

---

## 1. 專案概述

本系統旨在透過台股籌碼面資料，偵測個股是否正被特定法人或主力吸籌。系統採用多層濾網策略，結合價量特徵、法人動向、券商分點進出、庫存斜率趨勢及融資融券等維度，篩選出高機率吸籌標的。

### 1.1 核心邏輯

吸籌 = 量縮價穩 + 法人持續買超 + 分點庫存上升 + 籌碼集中 + 融資不增

---

## 2. 系統架構

### 2.1 整體 Data Flow

```text
FinMind API (HTTP GET / JSON)
    |
    v
Python ETL Service (httpx + pydantic)
    |
    v
Validation & Transform (去重 / 型別檢查 / 空值處理)
    |
    v
PostgreSQL COPY / INSERT ... ON CONFLICT
    |
    v
PostgreSQL Tables (6 tables)
    |
    v
SQL Function / View: detect_accumulation
    |
    v
吸籌訊號結果 (排序輸出)
```

### 2.2 Tech Stack

- Backend: Python 3.12
- Database: PostgreSQL 16
- Data Source: FinMind API (`api.finmindtrade.com/api/v4/data`)
- 週期: 日K（每日收盤後執行）

### 2.3 為何建議 Python 3.12 + PostgreSQL 16

- Python 適合 ETL、資料處理、回測與策略迭代
- Python 3.12 已相對成熟，常用套件支援度佳
- PostgreSQL 適合分析查詢、Window Function、分區、後續擴充 materialized view
- 對未來策略研究、排程、自動化部署較彈性

---

## 3. Database Design

### 3.1 Table 總覽（共 6 張）

| # | Table | 用途 | 資料頻率 |
|---|---|---|---|
| 1 | stock_info | 股票基本資料（代號/名稱/產業/警示） | 異動時更新 |
| 2 | daily_price | 日K線 OHLCV + 成交金額 | 每日 |
| 3 | institutional_flow | 三大法人每日買賣超 | 每日 |
| 4 | broker_branch_daily | 券商分點進出明細 | 每日 |
| 5 | margin_transaction | 融資融券餘額 | 每日 |
| 6 | etl_log | ETL 執行紀錄與錯誤追蹤 | 每次執行 |

### 3.2 Table Schema

#### Table 1: `stock_info`

```sql
CREATE TABLE stock_info (
    stock_id        varchar(10) PRIMARY KEY,
    stock_name      varchar(50) NOT NULL,
    industry        varchar(50),
    is_warning      boolean NOT NULL DEFAULT false,
    updated_at      timestamp NOT NULL DEFAULT now()
);
```

#### Table 2: `daily_price`

```sql
CREATE TABLE daily_price (
    stock_id        varchar(10) NOT NULL,
    trade_date      date NOT NULL,
    open_price      numeric(10,2),
    high_price      numeric(10,2),
    low_price       numeric(10,2),
    close_price     numeric(10,2),
    volume          bigint,
    turnover        bigint,
    PRIMARY KEY (stock_id, trade_date),
    CONSTRAINT fk_daily_price_stock
        FOREIGN KEY (stock_id) REFERENCES stock_info(stock_id)
);
```

#### Table 3: `institutional_flow`

```sql
CREATE TABLE institutional_flow (
    stock_id        varchar(10) NOT NULL,
    trade_date      date NOT NULL,
    foreign_net     bigint,
    site_net        bigint,
    dealer_net      bigint,
    PRIMARY KEY (stock_id, trade_date),
    CONSTRAINT fk_institutional_flow_stock
        FOREIGN KEY (stock_id) REFERENCES stock_info(stock_id)
);
```

#### Table 4: `broker_branch_daily`

```sql
CREATE TABLE broker_branch_daily (
    stock_id        varchar(10) NOT NULL,
    trade_date      date NOT NULL,
    broker_id       varchar(10) NOT NULL,
    branch_id       varchar(10) NOT NULL,
    buy_volume      bigint NOT NULL DEFAULT 0,
    sell_volume     bigint NOT NULL DEFAULT 0,
    net_volume      bigint GENERATED ALWAYS AS (buy_volume - sell_volume) STORED,
    PRIMARY KEY (stock_id, trade_date, broker_id, branch_id),
    CONSTRAINT fk_broker_branch_daily_stock
        FOREIGN KEY (stock_id) REFERENCES stock_info(stock_id)
);
```

#### Table 5: `margin_transaction`

```sql
CREATE TABLE margin_transaction (
    stock_id         varchar(10) NOT NULL,
    trade_date       date NOT NULL,
    margin_buy       bigint,
    margin_sell      bigint,
    margin_balance   bigint,
    short_buy        bigint,
    short_sell       bigint,
    short_balance    bigint,
    PRIMARY KEY (stock_id, trade_date),
    CONSTRAINT fk_margin_transaction_stock
        FOREIGN KEY (stock_id) REFERENCES stock_info(stock_id)
);
```

#### Table 6: `etl_log`

```sql
CREATE TABLE etl_log (
    log_id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    dataset_name     varchar(50) NOT NULL,
    stock_id         varchar(10),
    start_date       date,
    end_date         date,
    rows_fetched     integer,
    rows_inserted    integer,
    status           varchar(10),
    error_message    text,
    executed_at      timestamp NOT NULL DEFAULT now(),
    duration_sec     integer
);
```

---

## 4. ETL Layer

### 4.1 FinMind Dataset 對應

| FinMind Dataset | Target Table | 備註 |
|---|---|---|
| TaiwanStockInfo | stock_info | 含產業分類 |
| TaiwanStockPrice | daily_price | OHLCV |
| TaiwanStockInstitutionalInvestors | institutional_flow | 外資/投信/自營 |
| TaiwanStockTradingDailyReport | broker_branch_daily | 分點進出 |
| TaiwanStockMarginPurchaseShortSale | margin_transaction | 融資融券 |

### 4.2 ETL Flow

```python
for dataset in datasets:
    # 1. HTTP GET -> api.finmindtrade.com/api/v4/data
    #    ?dataset={dataset}&start_date={date}&token={token}
    # 2. Deserialize JSON -> list[Model]
    # 3. Validate (去重, 型別, 空值)
    # 4. 寫入 staging 或記憶體批次資料
    # 5. COPY / executemany -> PostgreSQL
    # 6. INSERT ... ON CONFLICT DO UPDATE
    # 7. INSERT etl_log
```

### 4.3 Python ETL Service 骨架

```python
from __future__ import annotations

import asyncio
import time
from datetime import date

import httpx
import psycopg


class FinMindEtlService:
    def __init__(self, token: str, dsn: str) -> None:
        self._token = token
        self._dsn = dsn
        self._base_url = "https://api.finmindtrade.com/api/v4/data"

    async def run_daily(self, eval_date: date) -> None:
        datasets = [
            ("TaiwanStockPrice", "daily_price"),
            ("TaiwanStockInstitutionalInvestors", "institutional_flow"),
            ("TaiwanStockTradingDailyReport", "broker_branch_daily"),
            ("TaiwanStockMarginPurchaseShortSale", "margin_transaction"),
        ]

        async with httpx.AsyncClient(timeout=30.0) as client:
            for dataset, target_table in datasets:
                start_ts = time.time()
                log = {
                    "dataset_name": dataset,
                    "status": "Success",
                    "error_message": None,
                    "rows_fetched": 0,
                    "rows_inserted": 0,
                }

                try:
                    payload = await self.fetch_from_finmind(client, dataset, eval_date)
                    rows = self.deserialize(dataset, payload)
                    log["rows_fetched"] = len(rows)
                    inserted = self.bulk_upsert(target_table, rows)
                    log["rows_inserted"] = inserted
                except Exception as ex:
                    log["status"] = "Failed"
                    log["error_message"] = str(ex)
                finally:
                    log["duration_sec"] = int(time.time() - start_ts)
                    self.save_log(log)
                    await asyncio.sleep(1.0)

    async def fetch_from_finmind(self, client: httpx.AsyncClient, dataset: str, eval_date: date) -> dict:
        response = await client.get(
            self._base_url,
            params={
                "dataset": dataset,
                "start_date": eval_date.isoformat(),
                "token": self._token,
            },
        )
        response.raise_for_status()
        return response.json()

    def deserialize(self, dataset: str, payload: dict) -> list[dict]:
        return payload.get("data", [])

    def bulk_upsert(self, target_table: str, rows: list[dict]) -> int:
        if not rows:
            return 0
        with psycopg.connect(self._dsn) as conn:
            with conn.cursor() as cur:
                # 實務可改用 COPY 到 staging table，再 INSERT ... ON CONFLICT
                pass
        return len(rows)

    def save_log(self, log: dict) -> None:
        with psycopg.connect(self._dsn) as conn:
            with conn.cursor() as cur:
                cur.execute(
                    """
                    INSERT INTO etl_log (
                        dataset_name, rows_fetched, rows_inserted,
                        status, error_message, duration_sec
                    )
                    VALUES (%s, %s, %s, %s, %s, %s)
                    """,
                    (
                        log["dataset_name"],
                        log["rows_fetched"],
                        log["rows_inserted"],
                        log["status"],
                        log["error_message"],
                        log["duration_sec"],
                    ),
                )
                conn.commit()
```

### 4.4 ETL 注意事項

- Rate Limit: FinMind 有 API 頻率限制，每次 request 間加 delay
- Upsert: 建議使用 `INSERT ... ON CONFLICT DO UPDATE`，避免重複寫入
- 歷史回補: 支援日期區間批次抓取，不只單日
- Error Handling: 單一 dataset 失敗不影響其他 dataset 執行
- 大批量匯入: 建議先進 staging table，再 merge 進正式表
- Token 管理: 建議放 `.env` 或系統環境變數，不可 hard code

---

## 5. 吸籌偵測策略

### 5.1 多層濾網架構

| 濾網層 | 名稱 | 條件 |
|---|---|---|
| Layer 1 | 量縮價穩 | MA20 Vol < MA60 Vol, 20日振幅 < 10%, 股價近 MA60 |
| Layer 2 | 法人連續買超 | 外資或投信近10日中 >= 7 日淨買超，累積量 > 日均量 x 2 |
| Layer 3 | 分點庫存上升 | 分點庫存 `slope_5d > 0`, `slope_20d > 0`, 且短斜率 >= 長斜率 |
| Layer 4 | 籌碼集中 | 多分點同步連續買超 >= 7 日，融資不增 |
| Layer 5 | 排除雜訊 | 日均成交額 > 500萬，非處置股，近5日無單日暴量 |

### 5.2 分點庫存斜率

分點庫存 = 該分點對某檔股票的累積淨買賣張數。透過線性回歸計算 5 日及 20 日斜率，判斷吸籌趨勢。

斜率公式：

```text
Slope = (N * SumXY - SumX * SumY) / (N * SumX2 - SumX * SumX)
```

| 庫存斜率 | 搭配量能 | 解讀 |
|---|---|---|
| 穩定正斜率 | 量縮 | 典型吸籌，最有價值的訊號 |
| 加速正斜率 | 量增 | 可能進入拉抬階段 |
| 斜率轉平/反轉 | 量增 | 出貨訊號，需警戒 |
| 負斜率 | 任何 | 倒貨中，應排除 |

### 5.3 最強訊號組合

庫存正斜率加速（`slope_5d >= slope_20d > 0`）+ 股價量縮盤整 + 多分點同步連續買超 + 融資不增

---

## 6. PostgreSQL Function / Query 設計

### 6.1 `detect_accumulation`

```sql
CREATE OR REPLACE FUNCTION detect_accumulation(
    p_eval_date date,
    p_min_avg_turnover numeric DEFAULT 5000000
)
RETURNS TABLE (
    stock_id varchar,
    stock_name varchar,
    avg_vol_ratio numeric,
    consecutive_buy_days integer,
    accum_branch_count integer,
    slope_5d numeric,
    slope_20d numeric,
    margin_trend varchar
)
LANGUAGE sql
AS $$
    WITH price_base AS (
        -- MA, 振幅, 量能萎縮比
        SELECT 1
    ),
    institution_buy AS (
        -- 法人連續買超天數 & 累積量
        SELECT 1
    ),
    branch_accum AS (
        -- 分點連續買超偵測
        SELECT 1
    ),
    inventory_slope AS (
        -- 庫存斜率計算
        SELECT 1
    ),
    margin_filter AS (
        -- 融資不增 + 融券觀察
        SELECT 1
    ),
    noise_filter AS (
        -- 成交額 / 處置股 / 暴量排除
        SELECT 1
    )
    SELECT
        s.stock_id,
        s.stock_name,
        0::numeric AS avg_vol_ratio,
        0::integer AS consecutive_buy_days,
        0::integer AS accum_branch_count,
        0::numeric AS slope_5d,
        0::numeric AS slope_20d,
        'flat'::varchar AS margin_trend
    FROM stock_info s
    WHERE false;
$$;
```

### 6.2 `calc_inventory_slope`

```sql
CREATE OR REPLACE FUNCTION calc_inventory_slope(
    p_eval_date date,
    p_window_days integer DEFAULT 5
)
RETURNS TABLE (
    stock_id varchar,
    branch_id varchar,
    slope numeric
)
LANGUAGE sql
AS $$
    WITH numbered AS (
        SELECT
            stock_id,
            branch_id,
            trade_date,
            SUM(net_volume) OVER (
                PARTITION BY stock_id, branch_id
                ORDER BY trade_date
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
            ) AS inventory,
            ROW_NUMBER() OVER (
                PARTITION BY stock_id, branch_id
                ORDER BY trade_date
            ) AS rn
        FROM broker_branch_daily
        WHERE trade_date <= p_eval_date
    ),
    windowed AS (
        SELECT *
        FROM numbered
    )
    SELECT
        stock_id,
        branch_id,
        NULL::numeric AS slope
    FROM windowed
    WHERE false;
$$;
```

### 6.3 PostgreSQL 實作建議

- 純查詢邏輯可先用 View / CTE 驗證，再封裝成 Function
- 若分點資料量很大，可考慮：
  - 依 `trade_date` 做 range partition
  - 預先計算日累積庫存表
  - 使用 materialized view 存放中繼結果
- 回測與正式偵測可拆成兩套查詢，避免互相影響效能

---

## 7. Python 開發指引

### 7.1 建議開發順序

- Phase 1 - DB：建立 6 張 tables + indexes
- Phase 2 - ETL：實作 Python FinMindEtlService，逐一完成 dataset 抓取與 upsert
- Phase 3 - Query：先完成 `calc_inventory_slope`，再完成 `detect_accumulation`
- Phase 4 - 驗證：回測訊號出現後 20 / 40 / 60 日漲幅，確認策略有效性
- Phase 5 - 優化：分數化（每層濾網給權重算總分），排序優先度通常比二元篩選更實用

### 7.2 專案結構建議

```text
stock_chip_analyzer/
|-- src/
|   |-- app/
|   |   |-- services/
|   |   |   |-- finmind_etl_service.py
|   |   |-- models/
|   |   |   |-- finmind_response.py
|   |   |   |-- etl_log.py
|   |   |-- db/
|   |   |   |-- connection.py
|   |   |   |-- repositories/
|   |   |-- main.py
|-- sql/
|   |-- 01_create_tables.sql
|   |-- 02_create_indexes.sql
|   |-- 03_calc_inventory_slope.sql
|   |-- 04_detect_accumulation.sql
|-- tests/
|-- pyproject.toml
|-- .env.example
|-- README.md
```

### 7.3 建議套件

- `httpx`: 非同步 HTTP Client
- `pydantic`: API 回傳資料驗證
- `psycopg[binary]`: PostgreSQL 存取
- `python-dotenv`: 載入環境變數
- `pandas`: 回測/分析用（非必要，但常用）
- `tenacity`: 重試機制
- `pytest`: 測試

### 7.4 關鍵 Index 建議

```sql
CREATE INDEX ix_bbd_stock_date
    ON broker_branch_daily(stock_id, trade_date);

CREATE INDEX ix_bbd_branch_stock
    ON broker_branch_daily(branch_id, stock_id, trade_date);

CREATE INDEX ix_dp_stock_date
    ON daily_price(stock_id, trade_date);

CREATE INDEX ix_if_stock_date
    ON institutional_flow(stock_id, trade_date);
```

### 7.5 注意事項

- `broker_branch_daily` 資料量最大，預估每日數十萬筆，需注意 partition 或定期歸檔
- 庫存屬於衍生計算，初期可用查詢即時計算；若效能不足，再建中繼表或 materialized view
- PostgreSQL 沒有 SQL Server `MERGE` 的必要依賴，建議以 `INSERT ... ON CONFLICT DO UPDATE` 為主
- 回測模組建議獨立，避免與每日 ETL 執行流程耦合過深
- 部署可採 Linux + systemd timer / cron，或容器化執行

---

## 8. 建議下一步

1. 先完成 PostgreSQL DDL 與 Index
2. 完成 FinMind ETL MVP
3. 補 60~120 天歷史資料做驗證
4. 先做單一偵測查詢版本，不急著過早封裝
5. 確認策略有效後，再做分數化、通知、回測報表

---

## 9. 補充建議

若此專案後續要擴大，建議再加上：

- 排程：APScheduler / cron
- 通知：LINE Notify 替代方案、Telegram Bot、Email
- 回測表：signal_result、signal_backtest
- 前端介面：Streamlit 或簡單 FastAPI + Vue
- 觀察名單機制：watchlist + 每日差異報告

