---
metaLinks:
  alternates:
    - https://app.gitbook.com/s/o044KN3Ws0uIDvOmSkcR/supported-cameras
---

# 支援的相機

Chloros 可在 **所有平台**（Windows、Linux amd64 及 Linux arm64/Jetson）上處理來自兩個 MAPIR 相機系列的影像：

* **Survey3** — Survey3W（廣角）與 Survey3N（窄角）相機。輸入格式：`RAW+JPG`。
* **LATTICE** — M3C 和 M3M 多光譜相機模組。輸入：`.tif`/`.tiff` 擷取的影像。 LATTICE 相機亦可透過 Chloros 進行 **即時控制** — 透過 GUI 的「相機」分頁（Windows）或 `chloros-cli lattice` / Python SDK（Windows 和 Linux）— 包括同步的多相機陣列。請參閱 [LATTICE 指南](lattice/)。

處理流程亦支援 `.dng` 輸入檔案。

## Survey3

<table data-header-hidden><thead><tr><th width="156">製造商</th><th width="250">相機型號</th><th width="138">濾光片型號</th><th width="187">影像類型</th></tr></thead><tbody><tr><td><strong>製造商</strong></td><td><strong>相機型號</strong></td><td><strong>濾鏡型號</strong></td><td><strong>影像類型</strong></td></tr><tr><td>MAPIR</td><td>Survey3W, Survey3N</td><td>RGB</td><td>RAW+JPG、JPG</td></tr><tr><td>MAPIR</td><td>Survey3W, Survey3N</td><td>RGN</td><td>RAW+JPG、JPG</td></tr><tr><td>MAPIR</td><td>Survey3W, Survey3N</td><td>OCN</td><td>RAW+JPG、JPG</td></tr><tr><td>MAPIR</td><td>Survey3W, Survey3N</td><td>NGB</td><td>RAW+JPG、JPG</td></tr><tr><td>MAPIR</td><td>Survey3W, Survey3N</td><td>RE</td><td>RAW+JPG、JPG</td></tr><tr><td>MAPIR</td><td>Survey3W, Survey3N</td><td>NIR</td><td>RAW+JPG、JPG</td></tr></tbody></table>## LATTICE

LATTICE 系列是一款基於 Sony IMX265 全域快門感光元件（3.1 MP，3.45 µm 像素）所建構的模組化多光譜相機系統。每台相機皆將其識別資訊儲存為模型字串：

```
<sensor>-<lens>-F<filter>        e.g.  M3C-L41-FRGN,  M3M-L87-F550
```

Chloros 會以 `LATT-` 作為前綴顯示（例如 `LATT-M3M-L41-F550`）， 而該型號字串將驅動後續所有流程——感測器配置檔、波段佈局及校準皆會自動解析；無需針對每台相機進行個別設定。 鏡頭編號代表 **水平視場角（單位：度）**：`L41` = 窄視場 41°，`L87` = 廣視場 87°。

共有兩種感測器配置：

| 配置 | 感測器      | 濾光片類型                           | 每台相機的波段數                                                        |
| ------------- | ----------- | ------------------------------------- | ----------------------------------------------------------------------- |
| **M3C**       | 拜耳彩色 | 三波段通帶                       | 單次曝光可獲 3 個光譜波段                                 |
| **M3M**       | 單色  | 單一窄帶干涉濾光片 | 1 個校準波段 — 結合多台 M3M 相機以計算植被指數 |

### M3C（拜耳）濾光片選項

| 濾光片 | 波段（名稱 @ 中心波長 nm / FWHM nm）       |
| ------ | ---------------------------------------- |
| `FRGB` | Blue 475/30 · Green 550/30 · Red 625/30  |
| `FRGN` | Red 660/21 · Green 550/30 · NIR 850/30   |
| `FOCN` | Orange 618.4/46 · Cyan 494.1/40 · NIR 821.6/54（實測值 — 詳見下文） |
| `FNGB` | Blue 475/30 · Green 550/30 · NIR 850/30  |

針對 `FOCN`，這些數值為濾光片**實際測量**的半最大波長（Chloros 1.3.0 及後續版本；較早版本採用規格表上的數值 615/21 · 490/38 · 808/14）。波段名稱仍維持為 `Orange_615`、`Cyan_490` 及 `NIR_808`，而**實際測量**的中心波長與寬度則是 Chloros 標記於每個經校準的匯出檔中的數值。

### M3M（單色）濾光片型錄 — 23 項 SKU

F 數值即為 SKU 標籤；經測量之波段（鐫刻於每片經校準的出貨產品上）則是每批濾光片的掃描數據：

| SKU    | 中心波長 (nm, 實測) | FWHM 邊寬 (nm) | 寬度 (nm) |
| ------ | --------------------- | --------------- | ---------- |
| F385   | 379.4                 | 367–392         | 25         |
| F405   | 403.9                 | 390–417         | 27         |
| F450   | 443.7                 | 430–458         | 28         |
| F485   | 489.7                 | 478–502         | 24         |
| F520   | 519.9                 | 504–536         | 32         |
| F550   | 548.4                 | 531–566         | 35         |
| F590   | 589.0                 | 570–608         | 38         |
| F615   | 623.8                 | 614–634         | 20         |
| F632   | 633.4                 | 616–651         | 35         |
| F650   | 651.1                 | 636–666         | 30         |
| F685   | 686.2                 | 675–698         | 23         |
| F715   | —（名義）           | 706–724         | 18         |
| F725   | 725.2                 | 712–738         | 26         |
| F750   | 746.0                 | 729–763         | 34         |
| F780   | 775.1                 | 754–796         | 42         |
| F808   | 810.3                 | 789–832         | 43         |
| F832   | 826.1                 | 810–843         | 33         |
| F850   | 846.5                 | 828–865         | 37         |
| F880   | —（名義）           | 867–893         | 26         |
| F905   | —（名義）           | 892–920         | 28         |
| F940   | 940.6                 | 923–958         | 35         |
| F950   | 945.1                 | 929–961         | 32         |
| F988 † | 985.3                 | 968–1003        | 35         |

_「頻帶邊緣是根據MAPIR 每批次濾光片掃描所得的半峰寬值測量而來——這與Chloros 在每份經校準的輸出資料中標記的數值相同。」_ "— (名義值)" = 尚未進行批次掃描；對於這些 SKU，所列的中心波長即為 SKU 編號，而寬度則採用製造商的數值。

† 「F988 反射率是使用場景內反射率面板進行校準的：該波段超出 DAQ 光感測器的校準範圍，因此 Chloros 會套用您最近的面板擷取值，並在兩次面板觀測之間維持該值。」 請參閱 [校準標靶](calibration-targets.md)。

關於即時相機控制、陣列、網路設定及輻測處理鏈，請參閱 [LATTICE 指南](lattice/)。
