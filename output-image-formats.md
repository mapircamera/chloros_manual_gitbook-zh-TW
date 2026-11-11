---
metaLinks:
  alternates:
    - https://app.gitbook.com/s/o044KN3Ws0uIDvOmSkcR/output-image-formats
---

# 輸出影像格式

Chloros 可將處理後的產品以四種檔案格式匯出。請於「專案設定」（GUI）中選取格式，或透過 `--format`（CLI）或 `export_format`（SDK）進行設定。CLI 與 SDK 僅接受以下精確的字串。

| 格式字串 | 副檔名 | 像素類型 | 像素範圍 | 備註 |
| --- | --- | --- | --- | --- |
| `TIFF (16-bit)` *(預設)* | `.tif` | uint16 數位數值 | 0 – 65535 | 建議用於攝影測量／GIS。 |
| `TIFF (8-bit)` | `.tif` | uint8 數位數值 | 0 – 255 | 適用於全 8 位元擷取，此類擷取僅提供 8 位元格式：Chloros 不會提升位元深度。 |
| `TIFF (32-bit, Percent)` | `.tif` | float32 | 0.0 – 1.0 | 1.0 = 100% 反射率。部分應用程式無法讀取浮點數 TIFF 檔案；檔案大小較大。 |
| `PNG (8-bit)` | `.png` | uint8 數位數值 | 0 – 255 | 無損壓縮，適合網頁瀏覽與視覺化。 |
| `JPG (8-bit)` | `.jpg` | uint8 數位數值 | 0 – 255 | 有損壓縮，檔案大小最小。 |

## 輸出檔案的儲存位置

產出檔案會寫入專案資料夾下，依相機分組，再依檔案格式分類：

```
<project>/
└── LATT-M3M-L41-F550/                  # one folder per camera (model+lens+filter)
    ├── tiff16/                          # follows --format: tiff16, tiff8, png8, jpg8, or tiff32
    │   ├── Reflectance_Calibrated_Images/
    │   ├── Debayered_Images/
    │   ├── Preview_Images/
    │   └── NDVI_Index_Images/           # one <INDEX>_Index_Images/ folder per requested index
    └── tiff32/
        └── Radiance_Images/             # float32 radiance always lands here
```

LATTICE 的相機資料夾為 `LATT-<sensor>-<lens>-F<filter>`，Survey3 的相機資料夾則為 `<model>_<filter>`（例如 `Survey3N_RGN`）。 **每個匯出的產品均保留原始檔案的名稱 — 產品由資料夾識別，而非檔案名稱後綴。** 完整規則請參閱 CLI 參考手冊中的 [輸出檔案存放位置](reference/cli-reference.md)。

## LATTICE 產出（擷取與匯出層級）

單次處理中，一個 LATTICE 原始幀會拆分為所有請求的產出。每種產出類型皆有其專屬的切換選項（GUI 核取方塊，或 CLI 指令 `--debayered` / `--preview` / `--radiance` / `--reflectance`，預設皆為開啟)：

| 層級 | 內容 | 資料類型 |
| --- | --- | --- |
| `raw` | 直接來自感測器的拜耳資料（單色相機：單一波段）。處理始終從原始資料開始。 | 原始擷取 |
| `debayered` | 線性去馬賽克 — M3C 為 3 通道，M3M 為 1 通道灰階：先移除校準包中的暗電偏移，接著進行邊緣感知去馬賽克，不包含線性校正、平場校正或解混。 在任何處理模式下，`process` 匯出格式所產生的檔案，皆與此層級下擷取功能所儲存的檔案相同。 | 線性 DN |
| `radiance` | 來自完整輻測鏈的絕對光譜輻度，單位為 **W/m²/sr/nm**。 無論選取何種匯出格式，皆以 32 位元 TIFF 格式寫入（`tiff32/Radiance_Images/`）。 | float32 |
| `reflectance` | 反射率 ρ，其中 **DN 32768 = ρ 1.0 (100%)**，並保留至 ρ 2.0 的動態範圍。相容於 Pix4D。 | uint16 |
| `preview` | 顯示就緒渲染：RGB = 白平衡 + γ 值；多光譜 = 假色拉伸。 | 8 位元顯示 |

### 方向

`raw` 和 `debayered` 檔案始終保持感測器的 **原生** 方向；相機的翻轉／旋轉設定僅套用至其他產品（預覽、輻照度、反射率、指數）。 陣列擷取會保留每個成員檔案的原生方向，除非您主動指定方向（CLI 中為 `--orient`，SDK 中為 `orient=True`）。 每個檔案都會在 XMP 中記錄其像素所屬的方位角 — `ChlorosRotation`、`ChlorosFlipHorizontal`、 `ChlorosFlipVertical` — 而 `ChlorosOrientationApplied` 則標示其像素是否攜帶此資訊（原始檔與去拜耳化檔中為 `false`）。 Chloros 影像檢視器僅根據該記錄對 RAW 及去拜耳化檔案進行顯示調整；檔案本身絕不會被覆寫。若檔案沒有該記錄（即在 1.3.0 版本之前所寫入，或由其他工具處理），則會以原始儲存狀態顯示。

## 裁剪遮罩

多光譜 M3C 相機的輻照度與反射率產出會進行 3×3 波段解混。若解混過程在某波段產生負值，該值將被裁剪為零（即時預覽的裁剪方式完全相同）。 1.3.0 版本新增功能：若某產出中至少有一個像素被截斷，該產出將附帶一個 **截斷遮罩**，明確標示出哪些像素和波段受到影響：

* **位置：**位於 `clamp_masks/` 資料夾中，**緊鄰** 產品資料夾，絕不會置於產品資料夾內部： `tiff16/Reflectance_Calibrated_Images/x.tif` → `tiff16/clamp_masks/Reflectance_Calibrated_Images/x_clampmask.tif`，以及 `tiff32/Radiance_Images/x.tif` → `tiff32/clamp_masks/Radiance_Images/x_clampmask.tif`。
* **內容：** 一個單波段的 `uint8` 格式TIFF檔案，基於產品本身的像素網格（包含方向、ROI、像素合併及陣列對齊）。 當產品 R 通道中的波段發生裁剪時，位元 0 (`mask & 1`) 會被設定；位元 1 （`& 2`）表示 G 通道，位元 2（`& 4`）表示 B 通道。 各波段被裁剪像素的比例亦以 `Chloros:BandRUnmixClampFraction`、`Chloros:BandGUnmixClampFraction` 及 `Chloros:BandBUnmixClampFraction` 的形式標記於產品中。
* **產品如何標示：** XMP 標籤 `Chloros:UnmixClampMask` 包含相對於產品資料夾的遮罩路徑（`../clamp_masks/<product folder>/<name>_clampmask.tif`， 始終以 `/` 分隔），若無像素被裁切（則未寫入遮罩），代碼為 `none`；若無法寫入遮罩，則代碼為 `unavailable`。 未帶有標籤的產出檔表示從未執行過色彩分離（RGB 和單色相機、原始檔及去拜耳化層級）。

由於沒有任何產品資料夾包含遮罩，您可以將產品資料夾（例如 `Reflectance_Calibrated_Images/`）連同其子資料夾一併直接傳遞給 Pix4D、Metashape 或其他攝影測量軟體——請傳遞產品資料夾，而非其上層的格式資料夾。 Chloros 自身的匯入功能會跳過鉗位遮罩。CLI 會將它們分別計數：`Image products written: N (plus M clamp masks)`。

## 索引產品

每個索引都會存放在各自的 `<INDEX>_Index_Images/` 資料夾中（例如 `NDVI_Index_Images/`）。

* **整數格式**（`TIFF (16-bit)`、`TIFF (8-bit)`、`PNG (8-bit)`、 `JPG (8-bit)`）儲存的是**每張影像的線性拉伸**，而非 −1…1 的映射：DN 0 代表該幀的最低索引值，全尺度則代表最高值。 該檔案包含 `Chloros:IndexScaling=linear_minmax`、`Chloros:IndexMin` 及 `Chloros:IndexMax`，因此也包含 `value = IndexMin + DN / full_scale × (IndexMax − IndexMin)`。 由於每個幀都有其自身的端點，因此應以數值而非 DN 來比較各幀。
* **`TIFF (32-bit, Percent)`** 將索引值本身以 float32 格式儲存，且未經重新縮放——這是進行整趟飛行定量分析的最佳選擇。

1.3.0 版本新增功能：索引產品會描述 **自身**，而非重複原始資料的標籤。其 GPS 和相機標籤會從計算來源的產品（若有反射率產品則取自該產品）複製而來，接著：

| 標籤 | 值 |
| --- | --- |
| `Chloros:ProcessingLevel` | `index`（因此該檔案絕不會被重新匯入為擷取檔） |
| `Chloros:IndexApplied` / `Chloros:IndexExpression` | `True` / 所使用的公式 |
| `Chloros:IndexLevel` | `reflectance` 或 `radiance` —— 該指數的計算依據（若兩者皆無則不顯示） |
| `Chloros:IlluminationInvariant` | 若指數是根據反射率計算，則為 `True`；若根據輻度或感測器數值計算，則為 `False` |
| `Chloros:ChlorosPipelineVersion` | 生成該指數的 Chloros 版本 |

`Chloros:PixelScale` 已從索引產品中移除 — 請使用上方的 `IndexMin` / `IndexMax`。 完整的讀回配方請參閱 [CLI 參考手冊](reference/cli-reference.md) 中「整數索引柵格是每張影像的對比度拉伸」一節。

## 讀取反射率像素值

反射率以整數數位數值 (DN) 儲存，而 **表示 ρ = 1.0（100% 反射率）的 DN 值取決於來源相機**：

| 來源相機 | ρ = 1.0 對應的 DN | 如何判別 |
| --- | --- | --- |
| LATTICE (M3C / M3M) | `32768`（動態範圍上限為 ρ 2.0） | 檔案中標有 XMP 標籤 `Chloros:PixelScale=32768`。 |
| Survey3 | `65535`（在 ρ 1.0 處截斷） | 沒有 `Chloros:PixelScale` 標籤——此缺失即為識別標誌。 （自 1.3.0 版本起，Survey3 產品在所有處理模式下均會包含其他 `Chloros:*` 處理標籤，例如 `Chloros:CalibrationApplied`。） |

**應讀取 `Chloros:PixelScale` XMP 標籤並以其為基準進行除法運算**，而非假設其為常數。 該標籤定義於 uint16 域中，因此即使在會重新縮放的輸出格式下，其值仍維持為 `32768` —— 請先將儲存的資料類型歸一化回 uint16（8 位元乘以 257，float32 乘以 65535）。

{% hint style="warning" %}
**8 位元擷取資料。** 8 位元擷取資料的輸出為 8 位元。其反射率會經過重新縮放並標記為 `Chloros:PixelScale=32768`，因此比例關係得以維持； 8 位元 DN 產品，以及 1.3.0 版本之前寫入的反射率檔案，均會被裁剪至 0–255 範圍，且不帶有比例標籤。若 LATTICE 反射率檔案中缺少此標籤，請勿假設其具有比例；應改以 16 位元或 32 位元重新匯出。
{% endhint %}

完整的規則（包括與 MicaSense 相容的標籤），請參閱 [CLI 參考手冊](reference/cli-reference.md) 中的 **「讀取反射率像素」**。
