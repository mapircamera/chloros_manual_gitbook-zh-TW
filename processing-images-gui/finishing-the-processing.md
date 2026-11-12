# 完成處理

當 Chloros 完成處理後，便是檢視結果、驗證輸出品質，並將處理後的影像準備好以供工作流程使用的時刻。本頁面將引導您完成最後的步驟及後續行動。

## 處理完成指示

當處理成功完成時，您將看到以下幾項指示：

* ✅ **進度條**：達到 100% 完成
* ✅ **除錯日誌**：顯示最後幾行 `[RUN-SUMMARY]` 內容，並附有計數（影像、相機群組、目標、校準影像、寫入的檔案）
* ✅ **開始按鈕**：重新啟用（準備進行下一次處理）
* ✅ **輸出檔案**：所有已處理的影像皆已儲存至專案的輸出目錄中（如下所示）

{% hint style="warning" %}
**若執行過程未寫入任何影像，即視為失敗。** 若您請求了影像產出，但執行過程未寫入任何影像，Chloros 會將其報告為失敗 — 日誌名稱中的 `[RUN-SUMMARY]` 提示了可能的原因（未匯入任何資料、未偵測到目標，或所有請求的產出皆因不適用而被跳過）。對應的 CLI 指令會以非零狀態退出。 若刻意執行僅產生元資料的執行程序（所有匯出產品皆關閉，且無索引），仍會被視為成功。請參閱 [CLI 參考手冊](../reference/cli-reference.md#a-run-that-writes-no-images-fails)。
{% endhint %}

***

## 尋找已處理的影像

### 開啟輸出資料夾

1. 點擊 **主選單** <img src="../.gitbook/assets/image (1) (1) (1) (1).png" alt="" data-size="line">
圖示（左上角）
2. 選擇 **「開啟專案資料夾」**

3. 檔案總管將開啟並跳轉至專案目錄
4. 根據名稱定位您的專案

### 輸出目錄結構

處理後的檔案會儲存於 **專案資料夾之下，並依相機型號分組，再依檔案格式分類**：

```
<project>/
└── LATT-M3M-L41-F550/                  # one folder per camera
    ├── tiff16/
    │   ├── Reflectance_Calibrated_Images/
    │   ├── Debayered_Images/
    │   ├── Preview_Images/
    │   └── NDVI_Index_Images/           # one folder per selected index
    └── tiff32/
        └── Radiance_Images/             # float32 radiance always lands here
```

* **相機資料夾**：LATTICE 為 `LATT-<sensor>-<lens>-F<filter>` （與擷取影像的 EXIF 資料 `Model` 相符），Survey3 則為 `<model>_<filter>`（例如 `Survey3N_RGN`）。 兩台共用相同感光元件與濾鏡、但鏡頭不同的相機，會分別建立獨立的資料夾結構——因其暗角、視野及畸變效果各異。
* **格式資料夾**：遵循您的匯出格式設定 — `tiff16`、`tiff8`、`png8`、`jpg8`， 或 `tiff32`（適用於 TIFF 格式，32 位元，百分比）。 輻照度始終為 float32 格式，並始終存放於 `tiff32` 目錄下。
* **產出資料夾**：
  * `Reflectance_Calibrated_Images/` — 經校正的反射率
  * `Debayered_Images/` — 線性去拜耳化 (LATTICE)：已移除暗偏移並進行去馬賽克處理，未進行平場校正或解混，且維持感測器原生方向
  * `Preview_Images/` — 顯示預覽 (LATTICE)
  * `Radiance_Images/` — float32 光譜輻度，W/m²/sr/nm (LATTICE 多光譜)
  * `Vignette_Corrected_Images/` **或** `Sensor_Response_Images/` — 針對無反射率參考的幀所產生的未校正備用檔；每次運行僅存在其中一個，由「暗角校正」設定決定
  * `<INDEX>_Index_Images/` — 每個選定的索引對應一個資料夾（例如 `NDVI_Index_Images`）
* **`clamp_masks/`**（1.3.0 版新增，僅適用於 LATTICE 多光譜 M3C 相機）：當波段在輻照度或反射率產品中將剪裁的負值歸零時，此資料夾會出現在格式資料夾中，位於產品資料夾旁邊。 該資料夾針對每個受影響的產品，各包含一個像素級遮罩，例如 `tiff16/clamp_masks/Reflectance_Calibrated_Images/<name>_clampmask.tif`。遮罩絕不會存放在產品資料夾內，因此產品資料夾可直接傳遞給攝影測量軟體使用。 請參閱 [截斷遮罩](../output-image-formats.md#clamp-masks)。

{% hint style="info" %}
**每個匯出的產品均保留原始檔案的名稱。**`capture_..._raw.tif` 的輻照度匯出檔仍稱為 `capture_..._raw.tif` —— 它只是存放在 `tiff32/Radiance_Images/` 目錄中。**資料夾用來識別產出結果，而非檔案名稱**，因此搜尋 `*radiance*.tif` 將一無所獲；請改以資料夾為搜尋依據。
{% endhint %}

<!-- SCREENSHOT-NEEDED: Windows Explorer open on a processed project folder showing the tree: a LATT-M3C multispectral camera folder expanded with tiff16 (Reflectance_Calibrated_Images, Debayered_Images, Preview_Images, NDVI_Index_Images, clamp_masks) and tiff32 (Radiance_Images, clamp_masks) subfolders visible -->


### 應有多少個檔案？

請勿依公式計算 — 輸出檔案的數量取決於啟用了哪些產品，以及哪些產品適用於各台相機（例如：RGB 相機不會產生輻照度／反射率資料）。 權威的計數資料在日誌中：最後一行 `[RUN-SUMMARY]` 會精確報告寫入的檔案數量，而提示行則會說明任何被跳過的項目。

***

## 檢視已處理的影像

### 在檔案總管中快速預覽

** Windows 內建預覽功能：**

1. 導航至產品資料夾（例如 `tiff16/Reflectance_Calibrated_Images/`）
2. 選取一個影像檔案
3. 預覽畫面將顯示於Windows 檔案總管預覽窗格中
4. 使用方向鍵瀏覽影像

### 在外部影像檢視器中預覽

**推薦的檢視器：**

* **QGIS** - 免費地理資訊系統軟體（最適合進行具地理參考的多光譜分析）
* **IrfanView** - 快速、輕量級的影像檢視器（支援 TIFF）
* **Adobe Photoshop** - 專業編輯軟體（支援 TIFF）
* **GIMP** - Photoshop 的免費替代方案
* **Windows 照片** - 基本檢視功能（可能不支援 16 位元 TIFF）

### 在 Chloros 影像檢視器中預覽

使用 Chloros 內建的影像檢視器進行進階視覺化：

1. 在檔案瀏覽器中點擊影像縮圖
2. 影像將在主預覽區域中開啟
3. 點擊左側邊欄中的 **影像檢視器** <img src="../.gitbook/assets/icon_image-viewer.JPG" alt="" data-size="line">
標籤頁
4. 使用 [索引/LUT 沙盒](../image-viewer-gui/index-lut-sandbox.md) 進行互動式分析

詳細說明請參閱 [影像檢視器](../image-viewer-gui/opening-an-image-full-screen.md)。

***

## 讀取反射率像素值 (GIS / Pix4D / 腳本)

反射率以整數 DN 形式儲存，而 **表示 ρ = 1.0 的 DN 值取決於來源相機**：

| 來源            | ρ = 1.0 對應值 | 如何判別                                        |
| --------------- | ---------- | -------------------------------------------------- |
| LATTICE (M3C/M3M) | **32768**（容許範圍最高至 ρ 2.0） | 檔案中標有 XMP 標籤 `Chloros:PixelScale=32768` |
| Survey3          | **65535**（在 ρ 1.0 處被截斷）     | 沒有 `Chloros:PixelScale` 標籤 —— 該標籤的缺失即是訊號（Survey3 產品確實包含其他 `Chloros:*` 處理標籤） |

**應讀取 `Chloros:PixelScale` 標籤並以其為除數**，而非一概假設為 65535 —— 若將 LATTICE 反射率除以 65535，會默默地將每個數值減半。 1.3.0 版本之前以 8 位元格式寫入的反射率檔案是經過裁剪（clipped）而非重新縮放（rescaled），且沒有比例標籤：請以 16 位元或 32 位元重新匯出，而非進行除法運算。 詳情請參閱 [輸出影像格式](../output-image-formats.md)。***

## 匯出時保留的元資料

每個產品都會保留原始擷取檔的 **GPS 區塊**及其**EXIF 子 IFD**，因此
匯出檔會包含 `FocalLength`、`FNumber`、`ExposureTime`、 `ISO`、`DateTimeOriginal` 以及
`CameraSerialNumber`，同時也包含地理參考資訊。

{% hint style="warning" %}
**若正射鑲嵌圖的縮放比例異常，請先檢查 `FocalLength`。**
Pix4D 會根據焦距加上高度來計算地面樣本距離。若缺少此標籤，
系統將回退至極其錯誤的比例尺——在一趟測量為 49 張影像的飛行任務中，一處 411 公尺 × 160 公尺
的橘園被重建為 47.8 公里 × 13 公里，產生了一張主要由
空白區域組成的 4.55 億像素正射影像。拼接速度緩慢以及檔案大小出乎意料地龐大，正是此問題的徵兆，而非獨立的
問題。

```bash
exiftool -FocalLength -GPSLatitude "YourProject/.../some_export.tif"
```
{% endhint %}

並非 *所有* 標籤都會被複製。IFD0 的結構標籤是刻意未被複製的（複製
這些標籤會損壞 LATTICE 的輸出結果），而 `ExifImageWidth` / `ExifImageHeight` 則被排除
，因為它們描述的是原始擷取內容——若將已調整尺寸的匯出檔包含在內，
其聲稱的尺寸將與其自身的光柵資料相矛盾。

Chloros 還會寫入其專屬的 `Chloros:*` XMP 標籤，用以描述對每個產品所進行的處理
（例如 `Chloros:ProcessingLevel`、`Chloros:CalibrationApplied`、
`Chloros:VignetteCorrected`）。自 1.3.0 版本起，Survey3 產品在每種處理
模式中皆包含這些標籤；索引產品則會描述索引本身（`Chloros:ProcessingLevel=index`、
公式，以及該索引是根據反射率還是輻照度計算而來），而非重複
原始資料的標籤；此外，存放於路徑長度超過 260 個字元的資料夾中的產品，其元資料
也會被保留。請參閱 [輸出影像格式](../output-image-formats.md#index-products)。

***

## 檢視除錯日誌

### 檢查警告或錯誤

1. 開啟 **「除錯日誌」** <img src="../.gitbook/assets/icon_log.JPG" alt="" data-size="line">
標籤頁
2. 瀏覽訊息
3. 尋找黃色警告或紅色錯誤
4. 閱讀 `[RUN-SUMMARY]` 相關行及任何提示
5. 聯絡 MAPIR 技術支援以獲取協助

### 儲存日誌

若要保留處理紀錄或傳送給 MAPIR 技術支援：

1. 點擊 **「複製」**或**「下載」** 按鈕
2. 將其儲存為文字檔於專案資料夾中
3. 隨專案文件一併提供
4. 若遇到問題，請寄送給 MAPIR 技術支援

若要直接向 MAPIR 回報問題，請使用「除錯日誌」索引標籤頂端的錯誤圖示（**回報錯誤**）——請參閱 [從日誌索引標籤回報錯誤](monitoring-the-processing.md#reporting-a-bug-from-the-log-tab)。***

## 常見輸出問題與解決方案

### 問題：輸出檔案遺失

**可能原因：**

* 該產品不適用於該相機（例如：RGB 相機的輻照度／反射率 — 日誌中會明確標示）
* 缺少必要的參考資料（例如：反射率分析未指定目標且無 `.daq` 下行輻射）
* 在「專案設定」中，該產品的「匯出」核取方塊已被停用
* 匯出過程中磁碟空間不足

**解決方案：**

1. 檢查除錯日誌中的 `[RUN-SUMMARY]` 提示與 `[EXPORT-CHECK]` 記錄 — 這些內容說明了各相機的跳過情況
2. 請在 [專案設定](adjusting-project-settings.md) 中確認產品匯出核取方塊的狀態
3. 請確認磁碟空間是否充足
4. 排除原因後重新處理

### 問題：邊緣過暗或過亮（暗角仍可見）

**可能原因：**

* 暗角校正已停用
* 相機／鏡頭未收錄於 Chloros 配置檔資料庫中
* 暗角過於嚴重，超出校正能力範圍

**解決方案：**

1. 確認已在「專案設定」中啟用暗角校正
2. 檢查是否正確偵測到相機型號
3. 若暗角問題仍持續存在，請聯絡 MAPIR 技術支援

### 問題：顏色或數值不正確

**可能原因：**

* 未偵測到校準標靶
* 選取了錯誤的校準標靶模型
* 反射率校準已停用
* 標靶影像品質不佳

**解決方案：**

1. 確認已啟用反射率校準
2. 檢查除錯日誌中的「已找到標靶」訊息
3. 檢視標靶影像品質
4. 標記正確的標靶後重新處理

### 問題：NDVI 數值似乎不正確

**預期NDVI 範圍：**

* **水體、岩石、土壤**：-0.1 至 0.2
* **稀疏／不健康的植被**：0.2 至 0.4
* **中等密度植被**：0.4 至 0.6
* **健康、茂密的植被**：0.6 至 0.9**若數值超出上述範圍：**

1. 確認已套用反射率校正
2. 確認已包含光感測器日誌
3. 檢查是否偵測到校正目標
4. 確保已正確偵測到相機型號
5. 檢視標靶影像的拍攝時間與條件
6. 若您自行從反射率檔案計算指標，請確認已將數值除以檔案中的 `Chloros:PixelScale`（參見上文）

***

## 使用已處理的影像

### 用於攝影測量／正射鑲嵌圖製作

**建議工作流程：**

1.**將已校正的反射率影像**匯入攝影測量軟體：
   * Pix4Dmapper
   * Agisoft Metashape
   * DroneDeploy
   * WebODM
2. **保留 EXIF 元資料**：確保 GPS 資料得以保留，以便進行地理標記
3. **校準工作流程**：使用反射率影像以確保科學精度 — LATTICE 反射率影像包含 Pix4D 可讀取的 XMP 校準標籤
4. **處理指數拼接圖**：從個別指數影像建立 NDVI 正射拼接圖
5. **匯出經地理參照的 GeoTIFF**：供 GIS 應用程式使用

### 用於 GIS 分析

**建議工作流程：**

1.**載入至 QGIS、ArcGIS 或類似軟體**

2.**使用 16 位元 TIFF** 反射率影像進行多波段分析（需除以檔案中的 `Chloros:PixelScale`）
3. **使用指數影像**（NDVI、NDRE）作為可直接使用的植被圖層
4. **柵格計算器**：合併波段以進行自訂分析
5. **匯出**：建立分類圖、變化檢測圖及植被健康圖

### 直接分析／報告用途

**建議工作流程：**

1.**使用帶有 LUT 色彩的指數影像** 製作視覺化報告
2. **擷取統計數據**：各田塊／樣區的平均 NDVI 值
3. **時間序列**：比較多個觀測時段的指數值
4. **生成報告**：包含地圖、統計數據及視覺化圖表***

## 歸檔與備份

### 建議的備份策略

**應保存的內容：**

* ✅ **原始 RAW/JPG 影像或 LATTICE 原始擷取檔** — 存檔於獨立硬碟／雲端；原始檔是處理流程的來源，其餘所有內容均可從中重新生成
* ✅ **`.daq` / `.csv` 光感測器檔案** — 日後重新推導反射率時所需
* ✅ **處理後輸出** — 保留經校正的影像與指標
* ✅ **專案資料夾**（`project.json` 及相關檔案）—— 內含所有設定，供日後重新處理時使用
* ✅ **除錯日誌**—— 記錄處理細節
* ✅ **校正目標影像**—— 用於驗證與重新處理

**儲存建議：**

* **即時備份**：外接硬碟
* **長期歸檔**：雲端儲存（Google Drive、Dropbox 等）
* **關鍵資料**：在不同位置保留 2-3 份副本***

## 後續處理流程

### 重複使用專案設定

若未來需處理類似資料集：

1. **儲存專案範本**（若尚未執行）
2. 使用已儲存的範本 **建立新專案**

3.**匯入新影像**

4. 為確保一致性，使用相同的設定進行**處理**

### 批次處理多個會話

針對多個會話／資料集：

**選項 1：GUI - 多個專案**

* 為每個會話建立獨立專案
* 使用一致的範本設定
* 逐一進行處理

**選項 2：Chloros CLI（僅限 Chloros+）**

* 自動化批次處理
* 透過腳本處理多個資料夾
* 請參閱 [CLI 文件](../CLI.md) 及 [CLI 參考手冊](../reference/cli-reference.md)

**選項 3：Python SDK（僅限 Chloros+）**

* 程式化控制
* 與分析流程整合
* 請參閱 [API 文件](../api-python-sdk.md) 及 [SDK 參考手冊](../reference/sdk-reference.md)

***

## 後處理疑難排解

### 採用不同設定重新處理

若結果不盡理想：

1. 保留原始影像（切勿刪除）
2. 在 Chloros 中開啟同一專案3. 於「專案設定」面板中調整設定
4. 重新處理 — 輸出檔案將存放在相同的产品資料夾中，因此會覆寫前次執行時同名的檔案

### 處理影像子集

若要僅重新處理特定影像：

1. 建立新專案
2. 僅匯入需要重新處理的影像
3. 使用相同的設定範本
4. 處理較小的資料集

### 尋求協助

若遇到問題：

* 📧 **電子郵件**：info@mapir.camera（請附上除錯日誌）
* 🌐 **技術支援**：[https://www.mapir.camera/community/contact](https://www.mapir.camera/community/contact)
* 📚 **常見問題**：[常見問題](../faq.md)
* 📖 **文件**：[Chloros 手冊](../)

***

## 摘要：完整工作流程

您現已完成完整的 Chloros 處理工作流程：

1. ✅ **建立專案** - 請參閱 [專案](../projects.md)
2. ✅ **新增檔案** - 請參閱 [新增檔案](adding-files-to-a-project.md)
3. ✅ **調整設定** — 請參閱 [調整專案設定](adjusting-project-settings.md)
4. ✅ **標記目標** — 請參閱 [選擇目標影像](choosing-target-images.md)
5. ✅ **開始處理** - 請參閱 [開始處理](starting-the-processing.md)
6. ✅ **監控進度** - 請參閱 [監控處理進度](monitoring-the-processing.md)
7. ✅ **檢視結果** - 本頁面**您已校正並經過反射率校正的多光譜影像，現已準備就緒，可進行分析！**

***

## 其他資源

### 進階功能

* [**影像檢視器**](../image-viewer-gui/opening-an-image-full-screen.md) - 互動式視覺化與分析
* [**指數/LUT 沙盒**](../image-viewer-gui/index-lut-sandbox.md) - 自訂指數測試
* [**多光譜指數公式**](../project-settings/multispectral-index-formulas.md) - 完整的指數參考資料

### 自動化與整合

* [**命令列介面 (CLI) 文件**](../CLI.md) - 命令列批次處理
* [**Python SDK**](../api-python-sdk.md) - 程式化自動化
* [**Chloros+ 功能**](../#chloros) - 進階處理功能

### 支援與學習

* [**常見問題 (FAQ)**](../faq.md) - 常見問題解答
* [**校準標靶**](../calibration-targets.md) - 了解反射率校準
* [**支援的相機**](../supported-cameras.md) - 相容硬體
