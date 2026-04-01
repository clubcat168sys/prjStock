## Job 開發規範

**標準結構**

```python
# 從 template/py/job_template.py 開始
from twstock.jobs.runner import run_job, job_step

def main():
    with run_job("job_name", run_date) as ctx:
        with job_step(ctx, "step_name") as step:
            # 業務邏輯
            step.row_count = n
```

**lookup_code 存取**

```python
from twstock.db.repo.lookup_code import lookup_code
url = lookup_code("STOCK_INFO", "LIST_URL")
```

**job_logs 事件類型**

| type | 觸發時機 |
|------|----------|
| RUN | `run_job()` 開始/結束 |
| STEP | `job_step()` 開始/結束，含 row_count、elapsed |
| ERR | 任何 exception |

**新 Job 檢查清單**
- [ ] 繼承 `job_template.py` 結構
- [ ] `run_job()` 帶正確 job name（snake_case）
- [ ] 每個邏輯步驟包在 `job_step()` 內
- [ ] 設定值從 `lookup_code` 取得，不寫死
- [ ] 更新 CLAUDE.md Running Jobs 區段的執行範例
