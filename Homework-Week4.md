# Week 4 Assignment: ARIA v2.0 (Integrated Impact Auditor)

# 第 4 週作業：全自動區域受災衝擊評估系統（地形整合版）

**繳交期限：下次上課前**

---

## 1. 任務情境 (Scenario)

單純看距離河川的遠近是不夠的。你需要升級你的 **ARIA** 系統，整合 **內政部地政司 20m DEM**，評估各設施在極端降雨下的「地形風險」。

**跟課堂 Lab 的差異：**
- Lab 1 用鄉鎮界做 dissolve + groupby 練習 → **作業要結合 Week 3 河川分析，加入地形因素**
- Lab 2 用預裁切的小範圍 DEM → **作業用完整的目標縣市 DEM，自行裁切**
- Lab 只算單一統計 → **作業要計算複合風險指標（距離 + 高程 + 坡度）**

---

## 2. 資料來源 (Data Sources)

### A. 向量資料 — 延續 Week 3

沿用第三週的資料：
- 水利署河川面 Shapefile（`gpd.read_file(WRA_URL)`）
- 消防署避難收容所 CSV（data.gov.tw/dataset/73242）
- 國土測繪中心鄉鎮市區界

### B. 網格資料 — 內政部地政司 20m DEM

- **來源**：[內政部 20m DEM](https://data.gov.tw/dataset/176927)
- **格式**：GeoTIFF（`.tif`），解析度 20m × 20m
- **CRS**：EPSG:3826（TWD97 / TM2）
- **檔案大小**：全台 DEM 非常大（> 500MB），**務必先裁切再分析**

```python
import rioxarray

# 在 Colab 中從 Google Drive 讀取
dem = rioxarray.open_rasterio('/content/drive/MyDrive/GIS_data/dem_20m.tif')
```

> **注意**：請勿將 .tif 檔 push 到 GitHub！大檔案放在 Google Drive 或 .gitignore。

---

## 3. 核心要求 (Requirements)

必須以 **`.ipynb` (Jupyter Notebook)** 格式繳交，展示步進式分析過程。

### A. 資料介接 (Data Ingestion)

1. **向量資料**：延續 Week 3 的避難所 + 河川面資料（已清理、已轉 EPSG:3826）
2. **網格資料**：讀取內政部 20m DEM
3. **裁切**：用目標縣市的鄉鎮界（dissolved + buffer(1000)）裁切 DEM → 減少記憶體用量，同時確保邊緣避難所的 500m 緩衝區不會超出 DEM 範圍
4. **CRS 對齊**：確認向量和網格都在 EPSG:3826

> **⚠️ 防呆檢查（Sanity Check）**：載入 Week 3 河川資料後，請先確認河川有涵蓋你的目標縣市：
> ```python
> rivers_in_county = gpd.sjoin(rivers, county_boundary, predicate='intersects')
> print(f"河川面與目標縣市交集：{len(rivers_in_county)} 筆")
> assert len(rivers_in_county) > 0, "⚠️ 河川資料未涵蓋目標縣市！請重新下載完整河川資料，不要篩選前 N 條"
> ```
> 如果結果為 0，代表你的 Week 3 河川資料可能有篩選問題，請重新下載完整的水利署河川面資料。

### B. 地形分析 (Terrain Analysis)

1. **坡度計算**：從 DEM 計算 slope（度）
   ```python
   import numpy as np
   dy, dx = np.gradient(dem.values[0], 20)  # 20m resolution
   slope = np.degrees(np.arctan(np.sqrt(dx**2 + dy**2)))
   ```
2. **Zonal Statistics**：為每個避難所的 500m 緩衝區計算：
   - **平均高程** (mean elevation)
   - **最大坡度** (max slope)
   - **高程標準差** (std elevation) — 地形起伏度

### C. 複合風險判定 (Composite Risk)

1. **從 `.env` 讀取門檻值**：
   ```
   SLOPE_THRESHOLD=30
   ELEVATION_LOW=50
   BUFFER_HIGH=500
   TARGET_COUNTY=花蓮縣
   ```
   > **建議使用花蓮縣**：Pre-lab 已提供預裁切的花蓮縣 DEM（`dem_20m_hualien.tif`）。若你想挑戰其他縣市，需自行從 [data.gov.tw](https://data.gov.tw/dataset/176927) 下載全台 DEM（> 500MB）再裁切，並確保河川資料也涵蓋該縣市。

2. **風險分級邏輯**：
   - **極高風險**：距河川 < 500m **且** 最大坡度 > SLOPE_THRESHOLD
   - **高風險**：距河川 < 500m **或** 最大坡度 > SLOPE_THRESHOLD
   - **中風險**：距河川 < 1000m **且** 平均高程 < ELEVATION_LOW
   - **低風險**：其餘

3. **產出**：每個避難所附帶 `risk_level`、`mean_elevation`、`max_slope`、`river_distance_category`

### D. 視覺化 (Visualization)

1. **DEM + 避難所地圖**：用 matplotlib 疊合 DEM hillshade + 避難所點位（依風險等級著色）
2. **統計圖**：Top 10 高風險避難所的坡度 vs. 高程散佈圖
3. **儲存**：`terrain_risk_map.png`

### E. 專業規範 (Infrastructure First)

1. **Colab 整合**：DEM 放 Google Drive，在 Colab 上運算；程式碼回存 GitHub
2. **環境變數**：門檻值放 `.env`，用 `python-dotenv` 讀取
3. **Markdown Cells**：每個分析步驟前寫 Captain's Log 說明
4. **AI 診斷日誌**：在 README 中描述你如何解決以下問題（至少一個）：
   - 「Zonal Stats 回傳 NaN」（CRS 未對齊或像素未覆蓋）
   - 「DEM 太大導致 Colab 記憶體不足」（需先裁切）
   - 「坡度計算結果不合理」（gradient 的 spacing 參數需與解析度匹配）

---

## 4. 推薦的 Vibe Coding Prompt

> "I need to upgrade my Week 3 ARIA system with terrain intelligence. I have:
> 1. Week 3 shelter GeoDataFrame (EPSG:3826) with river buffer risk levels
> 2. MOI 20m DEM (.tif) on Google Drive
>
> Help me in separate Jupyter cells:
> 1. Load the DEM with rioxarray and print its shape, CRS, and transform
> 2. Clip the DEM to my target county boundary
> 3. Compute slope from the clipped DEM using np.gradient
> 4. Create 500m buffers around shelters (EPSG:3826)
> 5. Use rasterstats.zonal_stats to get mean elevation and max slope per buffer
> 6. Merge terrain stats back to the shelter GeoDataFrame
> 7. Apply composite risk logic (river distance + slope + elevation)
> 8. Create a DEM hillshade map with shelter risk overlay"

---

## 5. 繳交清單 (Deliverables)

1. **GitHub Repo URL**
2. **`ARIA_v2.ipynb`** — 完整分析 Notebook（含 Markdown 說明）
3. **`terrain_risk_audit.json`** — 避難所地形風險清單（含 shelter_id、name、risk_level、mean_elevation、max_slope）
4. **`terrain_risk_map.png`** — DEM + 避難所風險地圖
5. **`README.md`** — 包含 AI 診斷日誌

---

## 6. 評量標準

| 項目 | 比重 |
|------|------|
| DEM 載入 + 裁切 + CRS 對齊 | 20% |
| 坡度計算 + Zonal Statistics | 25% |
| 複合風險邏輯（河川距離 + 地形） | 20% |
| 視覺化品質（DEM 地圖 + 統計圖） | 15% |
| Colab + .env + Markdown + AI 診斷日誌 | 20% |

---

## 7. 提示與注意事項

- **安裝套件**：`pip install rioxarray rasterstats geopandas python-dotenv matplotlib`
- **DEM 檔案**：從 data.gov.tw 下載後放在 Google Drive，不要 push 到 GitHub
- **記憶體管理**：全台 DEM 太大，一定要先用 `rio.clip()` 裁切到目標區域
- **CRS 對齊**：DEM 是 EPSG:3826，向量資料也要轉成 3826 才能做 zonal stats
- **坡度計算**：`np.gradient(dem, 20)` 的第二個參數是像素間距（20m），不要忘了
- **Zonal Stats NaN**：通常是因為 buffer 範圍超出 DEM 邊界，或 CRS 不一致
- **Week 3 連結**：ARIA v2.0 是 v1.0 的升級版，保留河川風險 + 新增地形風險

---

*"The professional disaster engineer doesn't just look at location — they measure environmental intensity. This week, we evolve from 'seeing maps' to 'computing risk.'"*
