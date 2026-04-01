## Calc 層設計

> 參考：`knowledge/filter/filter_concept.txt`（尚未實作）

**三層濾網架構**

```
stock_price / broker_inventory_daily
         ↓
  [Layer 1] State Filter   → state_score
         ↓
  [Layer 2] Trend Filter   → trend_score
         ↓
  [Layer 3] Quality Filter → quality_score, quality_flag (ok/veto)
```

| Layer | 指標 |
|-------|------|
| State（流動性+波動度） | 成交金額、BBW/ATR、CLV/上影線比例 |
| Trend（趨勢） | VWMA 位置/斜率、HH/HL 結構、兩日確認 |
| Quality/Veto（品質） | 當沖偵測、券商庫存趨勢、地緣接近度 |

**輸出欄位**：`state_score`、`trend_score`、`quality_score`（NUMERIC 0~1）、`quality_flag`（'ok'/'veto'）

**實作位置**
- `src/twstock/calc/indicators/` — 技術指標（TA-Lib）
- `src/twstock/calc/filters/` — 三層濾網邏輯
- `src/twstock/calc/scoring.py` — 分數彙整

**注意**：TA-Lib 在 Windows 建議透過 Docker；大量歷史查詢需加日期範圍限制避免 OOM。
