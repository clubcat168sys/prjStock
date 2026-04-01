## 下載 / ETL 流程與除錯

**現有 Job**

| Job | 模組 | 功能 |
|-----|------|------|
| download_stock_info | `jobs/download_stock_info.py` | 從 STOCK_INFO URL 下載原始 CSV |
| etl_stock_info | `jobs/etl_stock_info.py` | 解析 CSV → upsert `stock_info` |

`stock_info` 有 DB trigger：相關欄位異動時自動 insert `stock_info_chg`，ETL upsert 後不需手動處理 change log。

**除錯步驟**
1. 確認 `lookup_code('STOCK_INFO', 'LIST_URL')` 可取到正確 URL
2. 確認 `DL_PATH` 目錄存在且有寫入權限
3. 查 `job_logs` 確認 RUN/STEP/ERR 事件
4. 若 advisory lock 卡住：確認無其他程序持有同一 lock key

**常見問題**

| 問題 | 原因 | 解法 |
|------|------|------|
| 解碼失敗 | 新來源使用其他編碼 | 更新解碼順序或加新編碼 |
| upsert 無效 | ON CONFLICT key 不對 | 確認 stock_id 為 conflict key |
| advisory lock timeout | 前次 Job 未正常結束 | 確認 DB session，或手動釋放 lock |
