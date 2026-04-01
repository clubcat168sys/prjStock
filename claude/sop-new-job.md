## SOP：新增 Job

1. **複製 template**
   ```bash
   cp template/py/job_template.py src/twstock/jobs/{job_name}.py
   ```

2. **實作業務邏輯** — `run_job()` 內，每段邏輯用 `job_step()` 包裹，設定值從 `lookup_code()` 取得

3. **新增 lookup_code 種子資料**（若需要新 key）
   ```sql
   INSERT INTO twstock.lookup_code (category, code_key, code_value, description)
   VALUES ('CATEGORY', 'KEY_NAME', 'value', '說明')
   ON CONFLICT (category, code_key) DO UPDATE SET code_value = EXCLUDED.code_value;
   ```

4. **更新 CLAUDE.md** — 在 Running Jobs 區段補充執行範例

5. **驗證**
   ```bash
   python -m twstock.jobs.{job_name} --data-date 20251225
   docker exec -it twstock_python bash -lc "cd /app/src && python -m twstock.jobs.{job_name} --data-date 20251225"
   ```

6. **確認 job_logs**
   ```sql
   SELECT * FROM twstock.job_logs
   WHERE job_name = '{job_name}'
   ORDER BY created_at DESC LIMIT 20;
   ```
