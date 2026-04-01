## 當前專案狀態

> 狀態改變時更新此區段；每節不超過 3 條。_最後更新：2026-04-01_

**當前焦點**
- 基礎 ETL（download_stock_info + etl_stock_info）已完成並穩定
- 下一里程碑：實作 `src/twstock/calc/` calc 層（技術指標 + 三層濾網）
- DB schema 與 job framework 已定型，新 job 直接套用 template

**進行中工作**
- Calc 層尚未實作：indicators（TA-Lib）、filters（state/trend/quality）、scoring 均待開發
- `knowledge/filter/filter_concept.txt` 已有設計概念，需轉為程式碼

**已知風險**
- Calc 層若直接查 `stock_price` 大量歷史資料，需注意記憶體與效能
- TA-Lib 安裝在 Windows 環境需特殊處理（建議走 Docker）
- `lookup_code` 種子資料（`seed_lookup_code.sql`）需與實際 `.env` DL_PATH 保持一致
