## Calc 層設計

> 詳細邏輯：`knowledge/filter/filter_concept.txt`
> SQL 函式：`db/sql/funcs/fn_calc_inventory_slope.sql`、`db/sql/funcs/fn_detect_accumulation.sql`

---

### 吸籌核心公式

```
吸籌 = 量縮價穩 + 法人持續買超 + 分點庫存上升 + 籌碼集中 + 融資不增
```

---

### 吸籌偵測：5 層濾網（`fn_detect_accumulation`）

| Layer | 名稱 | 條件 |
|-------|------|------|
| 1 | 量縮價穩 | 日均成交額 > 500萬、近5日無暴量（< 日均量 × 3） |
| 2 | 法人連續買超 | 外資或投信近10日中 ≥ 7 日買超 |
| 3 | 分點庫存上升 | `slope_5d > 0`, `slope_20d > 0`, `slope_5d >= slope_20d` |
| 4 | 籌碼集中 | 多分點連續買超 ≥ 7 日 + 融資不增（非 up） |
| 5 | 排除雜訊 | `is_warning = false`（非處置/注意股） |

**最強訊號組合**：slope_5d 加速（≥ slope_20d > 0）+ 量縮盤整 + 多分點同步 + 融資不增

---

### 庫存斜率（`fn_calc_inventory_slope`）

斜率公式（線性回歸）：
```
slope = (N×ΣXY − ΣX×ΣY) / (N×ΣX² − (ΣX)²)
```

| 庫存斜率 | 搭配量能 | 解讀 |
|----------|----------|------|
| 穩定正斜率 | 量縮 | 典型吸籌，最有價值的訊號 |
| 加速正斜率（5d ≥ 20d） | 量增 | 可能進入拉抬階段 |
| 斜率轉平/反轉 | 量增 | 出貨訊號，需警戒 |
| 負斜率 | 任何 | 倒貨中，應排除 |

---

### 技術指標層（Python calc）：三層濾網

```
stock_price / broker_inventory_daily
         ↓
  [Layer 1] State Filter   → state_score  (流動性+波動度)
         ↓
  [Layer 2] Trend Filter   → trend_score  (趨勢結構)
         ↓
  [Layer 3] Quality Filter → quality_score, quality_flag (ok/veto)
```

| Layer | 指標 |
|-------|------|
| State（流動性+波動度） | 成交金額、BBW/ATR 相對化、CLV/上影線比例 |
| Trend（趨勢） | VWMA 位置/斜率、HH/HL 結構、兩日確認 |
| Quality/Veto（品質） | 當沖偵測、券商庫存趨勢、地緣接近度 |

**輸出欄位**：`state_score`、`trend_score`、`quality_score`（NUMERIC 0~1）、`quality_flag`（'ok'/'veto'）

**最終排序**：`where quality_flag = 'ok' order by state_score×a + trend_score×b + quality_score×c desc`

---

### 實作位置

- `db/sql/funcs/fn_calc_inventory_slope.sql` — 庫存斜率 SQL 函式 ✅
- `db/sql/funcs/fn_detect_accumulation.sql` — 吸籌偵測主函式 ✅
- `src/twstock/calc/indicators/` — 技術指標（TA-Lib）
- `src/twstock/calc/filters/` — 三層濾網邏輯
- `src/twstock/calc/scoring/` — 分數彙整

**注意**：TA-Lib 在 Windows 建議透過 Docker；大量歷史查詢需加日期範圍限制避免 OOM。
