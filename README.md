# ARIA v2.0 - 花蓮縣地形整合河川洪災避難所風險評估系統

## 專案概述
ARIA v2.0 是一個專為**花蓮縣**設計的地形整合河川洪災避難所風險評估系統，結合內政部 20m DEM、水利署河川圖資與消防署避難收容所資料，提供複合風險評估功能。

## 系統特色
- **花蓮縣專版**：限定花蓮縣範圍的精準分析，包含13個行政區198個避難所
- **地形整合分析**：20m DEM 坡度與高程評估，支援NoData值處理
- **複合風險模型**：河川距離 + 地形因子的多維度風險評估
- **環境變數配置**：使用 `.env` 檔案管理參數，支援彈性調整
- **專業級輸出**：互動式地圖、DEM hillshade地形圖、詳細統計報告
- **AI診斷日誌**：完整記錄問題解決過程與技術最佳化

## 分析範圍與資料來源

### 目標區域
- **分析範圍**：花蓮縣（13個行政區：秀林鄉、富里鄉、卓溪鄉、玉里鎮、瑞穗鄉、豐濱鄉、萬榮鄉、壽豐鄉、光復鄉、鳳林鎮、吉安鄉、花蓮市、新城鄉）
- **避難所數量**：198個（總收容量20,561人，平均103.8人/所）
- **座標系統**：EPSG:3826 (TWD97)

### 資料來源
- **河川資料**：水利署河川面資料 (`data/riverpoly/riverpoly.shp`)
- **避難所資料**：消防署避難收容處所點位檔案v9 (`data/避難收容處所點位檔案v9.csv`)
- **地形資料**：內政部20m DEM (`data/dem_20m_hualien.tif`)
- **行政界線**：國土測繪中心鄉鎮界 (`data/鄉(鎮、市、區)界線1140318/TOWN_MOI_1140318.shp`)

## 環境變數配置

### 必要參數 (.env檔案)
```bash
# 地形分析門檻
SLOPE_THRESHOLD_DEGREES=30
ELEVATION_THRESHOLD_METERS=50
BUFFER_DISTANCE_METERS=500
DEM_CLIP_BUFFER=1000

# 風險分級門檻
EXTREME_RISK_SLOPE=60
HIGH_RISK_SLOPE=45
MEDIUM_RISK_SLOPE=30
LOW_RISK_ELEVATION=100

# 系統設定
OUTPUT_CRS=EPSG:3826
DEBUG_MODE=True
TARGET_COUNTY=花蓮縣
```

### 複合風險分級邏輯
- **極高風險**：距河川 < 500m **且** 最大坡度 > 30°
- **高風險**：距河川 < 500m **或** 最大坡度 > 30°
- **中風險**：距河川 < 1000m **且** 平均高程 < 50m
- **低風險**：其餘

## 安裝與執行

### 1. 環境需求
```bash
pip install geopandas pandas folium matplotlib rioxarray numpy rasterstats python-dotenv
```

### 2. 資料準備
確保以下資料檔案位於 `/data/` 目錄：
- `data/dem_20m_hualien.tif` (花蓮縣20m DEM)
- `data/riverpoly/riverpoly.shp` (水利署河川面)
- `data/避難收容處所點位檔案v9.csv` (消防署避難所)
- `data/鄉(鎮、市、區)界線1140318/TOWN_MOI_1140318.shp` (鄉鎮界)

### 3. 執行分析
```bash
jupyter notebook ARIA.ipynb
```

## AI 診斷日誌

### 問題 1：Zonal Stats 回傳 NaN
**症狀**：`rasterstats.zonal_stats` 回傳大量 NaN 值，導致地形統計失效。

**根因分析**：
1. CRS 未對齊：避難所 GeoDataFrame 與 DEM 的座標系統不一致
2. 像素未覆蓋：部分避難所 500m 緩衝區超出 DEM 範圍
3. DEM NoData值未處理：-32767的NoData值影響統計計算

**解決方案**：
```python
# 1. 確保 CRS 一致性
if shelters.crs != 'EPSG:3826':
    shelters = shelters.to_crs('EPSG:3826')
if dem.rio.crs != 'EPSG:3826':
    dem = dem.rio.reproject('EPSG:3826')

# 2. 擴大 DEM 裁切範圍
county_buffer = county_boundary.buffer(1000)  # 增加 1km 緩衝
dem_clipped = dem.rio.clip(county_buffer.geometry, crs='EPSG:3826')

# 3. 處理 NoData 值
dem_array_processed = dem_array.copy()
dem_array_processed[dem_array_processed == -32767] = np.nan
dem_clipped.values[0] = dem_array_processed

# 4. Zonal statistics 使用 nodata=np.nan
elevation_stats = zonal_stats(
    buffer_gdf.geometry,
    dem_array,
    affine=transform,
    stats=['mean', 'std', 'min', 'max'],
    nodata=np.nan
)
```

**效果**：有效高程統計從 0% 提升至 64.1%，成功獲得 127 個避難所的地形資料，所有避難所高程均為正值。

### 問題 2：DEM NoData值導致異常高程
**症狀**：避難所高程顯示-5000m以下，DEM統計最小值-32767.0m。

**根因分析**：
- DEM檔案使用 -32767 作為NoData值
- 未處理NoData值導致zonal statistics計算包含異常值
- 平均高程被嚴重低估（-14587.8m）

**解決方案**：
```python
# 識別並處理NoData值
nodata_candidates = [-32767, -9999, 0]
for candidate in nodata_candidates:
    nodata_count = np.sum(dem_array == candidate)
    if nodata_count > 0:
        print(f"發現NoData值 {candidate}: {nodata_count:,} 個像素")
        # 將NoData值設為NaN
        dem_array[dem_array == candidate] = np.nan
        break

# 驗證高程合理性（台灣高程範圍約0-4000m）
dem_valid = dem_array[np.isfinite(dem_array)]
if dem_valid.min() >= -100 and dem_valid.max() <= 5000:
    print("✓ 高程範圍合理")
```

**效果**：處理後高程範圍 -3.0 - 3824.3m，平均值 1222.9m，符合台灣實際地形。避難所高程範圍 10.8 - 1150.6m，所有避難所高程均為正值。

### 問題 3：坡度計算結果不合理
**症狀**：計算出的坡度值異常，部分避難所周圍坡度達到80°，與實際地形不符。

**根因分析**：
- `np.gradient` 的 `spacing` 參數未與 DEM 解析度匹配
- 使用預設 spacing=1 導致坡度被嚴重高估
- 20m DEM解析度在陡峭區域可能產生誤差

**解決方案**：
```python
# 取得實際 DEM 解析度
if hasattr(dem_clipped.rio, 'resolution'):
    resolution = abs(dem_clipped.rio.resolution()[0])  # 20m
else:
    resolution = 20  # 預設值

# 使用正確的 spacing 參數計算坡度
dy, dx = np.gradient(dem_array, resolution)

# 計算坡度角度
slope_rad = np.arctan(np.sqrt(dx**2 + dy**2))
slope_deg = np.degrees(slope_rad)

# 驗證結果合理性
slope_valid = slope_deg[np.isfinite(slope_deg)]
print(f"坡度範圍: {slope_valid.min():.2f}° - {slope_valid.max():.2f}°")
```

**效果**：坡度計算結果合理化，平均坡度 30.91°，最大坡度 85.45°，符合台灣地形特徵。但仍發現6個避難所周圍坡度>60°，需要進一步現場驗證。

## 技術架構

### 資料流程
1. **資料載入**：本地檔案系統 → GeoDataFrame
2. **座標轉換**：EPSG:4326 → EPSG:3826
3. **地形處理**：DEM 裁切 → 坡度計算 → Zonal Statistics
4. **空間分析**：緩衝區建立 → 空間連接 → 風險分級
5. **結果輸出**：統計報告 → 互動地圖 → JSON 匯出

### 核心套件
- **geopandas**: 空間資料處理
- **rioxarray**: DEM 載入與處理
- **rasterstats**: Zonal statistics 計算
- **folium**: 互動式地圖生成
- **python-dotenv**: 環境變數管理

## 分析結果摘要

### 花蓮縣避難所地形分析成果
- **分析避難所**：198個（13個行政區）
- **有效地形統計**：127個（64.1%成功率）
- **高程範圍**：10.8 - 1150.6m（平均104.0m）
- **坡度範圍**：3.79 - 79.40°（平均21.33°）
- **高風險避難所**：41個（坡度>30°）
- **極陡坡避難所**：6個（坡度>60°，需現場驗證）

### 發現的技術問題與解決
1. **DEM NoData值處理**：成功識別並處理5,912,157個NoData像素
2. **坡度計算最佳化**：使用正確spacing參數，避免坡度高估
3. **極陡坡分析**：發現80度坡度可能為DEM解析度限制或座標偏差

## 輸出檔案說明

### 主要分析結果
- `risk_map.html`: 互動式風險地圖
- `terrain_risk_map.png`: DEM hillshade + 避難所風險分級地圖
- `top_risk_terrain_analysis.png`: 高風險避難所地形分析圖
- `terrain_risk_audit.json`: 地形風險清單（含risk_level、mean_elevation、max_slope、river_distance_category）

### 配置檔案
- `.env`: 環境變數配置（SLOPE_THRESHOLD、ELEVATION_THRESHOLD等）
- `requirements.txt`: Python 套件依賴
- `ARIA.ipynb`: 完整分析notebook（含Captain's Log）

## 使用限制與注意事項

1. **記憶體需求**：建議至少 8GB RAM（花蓮縣DEM約200MB）
2. **DEM 資料**：需預先下載至 `/data/` 目錄，支援花蓮縣預裁切版本
3. **座標系統**：所有分析均在 EPSG:3826 (TWD97) 下進行
4. **緩衝區大小**：可透過 `.env` 檔案調整（預設500m）
5. **極陡坡注意**：坡度>60°的結果建議現場驗證，可能為DEM解析度限制
6. **資料範圍**：目前僅支援花蓮縣，其他縣市需自行準備DEM和河川資料

## 版本資訊
- **版本**: v2.0 (花蓮縣專版)
- **更新日期**: 2024-03-23
- **作者**: ARIA 開發團隊
- **適用範圍**: 花蓮縣地形整合河川洪災避難所風險評估
- **核心特色**: NoData值處理、複合風險分級、AI診斷日誌

## 授權條款
本專案採 MIT 授權條款。
