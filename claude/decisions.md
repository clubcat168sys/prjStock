## 已核准架構規則

> 確立新規則時新增至此；廢棄規則直接刪除。

**D-001: 所有設定透過 lookup_code 管理**
URL、下載路徑、參數等設定一律存於 `lookup_code` 表，不在程式碼中寫死。環境切換只改 DB，不動程式碼。新增任何需設定化的值前，先確認 `lookup_code` 是否已有對應 key。

**D-002: 所有 Job 使用統一 framework**
新 Job 必須使用 `run_job()` + `job_step()`，並從 `template/py/job_template.py` 開始。保證 advisory lock、UUID run_id、job_logs 結構化記錄一致性。

**D-003: Advisory Lock key = job name + run date**
Lock key = `{job_name}_{run_date}`，防止同一天重複執行同一 Job。避免並行下載/ETL 造成資料重複或衝突。

**D-004: DB 查詢結果使用 dict row factory**
`pool.py` 統一設定 dict row factory，所有查詢回傳 `dict`，以欄位名存取。避免 index 存取錯誤，可讀性高。

**D-005: CSV 解碼順序 utf-8-sig → utf-8 → cp950**
ETL 讀取台股 CSV 時依序嘗試此三種編碼，失敗才往下試。台灣證交所檔案編碼不一致，此順序覆蓋已知所有來源。
