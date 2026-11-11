# 單色相機與植被指數

## 一台相機 = 一個波段

**M3M**相機是拜耳**M3C** 的單色版本：其後方配備單一窄帶干涉濾光片，並搭載 IMX265 單色感光元件。 型號字串即代表波段名稱 — `M3M-<lens>-F<wavelength>`，例如 `M3M-L87-F685`（在 Chloros

中顯示為 `LATT-M3M-L87-F685`）。 該感光元件提供 **單一灰階波段**，且無拜耳馬賽克結構：無需進行去馬賽克處理、無需分離通道間的串音，也無需設定白平衡。

在規劃單色系統前，應了解以下要點：

* **各波段的輻照度與反射率均已完全定義。**這些是按波段劃分的輻照度圖，因此一台 M3M 相機所產出的校準後 float32 輻照度（W/m²/sr/nm）與 uint16 反射率（`32768` = ρ 1.0），與 M3C 波段的輸出完全相同。 單色影像攜帶**單位矩陣** 感測器響應矩陣——無需進行或套用 3×3 解混操作。
* **單一單色相機無法產生植被指數。** NDVI

、NDRE

及其相關指數至少需要兩個波段。若要透過單色硬體計算指數，需結合多台 M3M 相機——詳見下文。
* M3M 相機傳輸 **Mono12** （12 位元：傳輸時為 2 位元組/像素，若陣列採用 [壓縮 12 位元傳輸](arrays.md#packed-12-bit-transport)，則為 1.5 位元組/像素），這點對 [陣列頻寬規劃](arrays.md#bandwidth-how-many-cameras-fit) 至關重要。

## Chloros

在單色模式下會跳過哪些步驟——以及如何通知使用者

色彩處理管線的各階段對於單波段感測器而言根本不適用。 Chloros

會 **以一行訊息跳過這些階段** 而非拋出錯誤，且在同一執行階段中，對於任何 M3C（拜耳）相機仍會正常執行這些階段：

| 階段 | 單色 (M3M) 行為 | M3C 行為 |
| --- | --- | --- |
| 去馬賽克 / 拜耳去馬賽克 | 跳過 — `debayered` 輸出層級為 1 通道灰階影像。 | 3 通道去馬賽克。 |
| 白平衡 (`lattice white-balance`) | 跳過並顯示一行訊息。 | 正常執行。 |
| 色彩配置檔 (`lattice color-profile`) | 跳過並顯示一行訊息。 | 正常執行。 |
| 飽和度／對比度 (`lattice color`) | 跳過並顯示一行訊息。 | 正常執行。 |
| 光譜串擾分離 | 單位矩陣（無 3×3 矩陣）。 | 套用各相機專屬的 3×3 矩陣。 |
| 輻照度／反射率 | **執行中** — 按波段處理，完全校正。 | 按波段執行。 |

GUI 採用相同的篩選機制：對於單色相機，每台相機的設定面板會隱藏僅限 RGB

的列（白平衡、伽瑪、色彩配置檔、飽和度、對比度、通道分割），且即時直方圖會鎖定為單一 **MONO** 軌跡。 整個堆疊過程中的區分標誌是模型字串中的 `M3M` 標記，該標記在 GUI/SDK

中顯示為 `is_mono`。

## 索引需 ≥ 2 個波段：對齊 → 堆疊 → 索引

單色索引的工作流程始終包含以下三個步驟：

1. **對齊** — 將數台 M3M 相機對準不同波長（例如一台 F650「Red

」和一台 F850「NIR

」），將它們連接成 [多相機陣列](arrays.md)，並讓 Chloros

計算相機間的共註冊變形。
2. **堆疊** — 對齊後的幀將合併為一張多波段影像（每台相機貢獻一個命名波段）。
3. **索引** — 針對堆疊中的各波段評估索引公式，並可選擇透過 LUT 進行渲染。

在 GUI 中，這整條處理鏈即為 **組合相機**陣列顯示模式：即時合成影像已預先對齊，而陣列的「索引計算器」（如下所示）則定義其渲染所用的公式。 透過**對齊** 擷取選項，可將擷取的輸出檔案變形至相同的對齊狀態。

## 索引計算器

索引計算器負責編寫即時預覽及各相機索引輸出所使用的索引表達式。它是一個共享介面，可從「相機」分頁側邊欄的兩個位置開啟：

* **單一相機**— 即時預覽 →**索引** 齒輪圖示（僅限 RGN

／OCN

／NGB

拜耳相機；單一單色相機無索引控制，因單一波段無法構成索引）。
* **按陣列**— 陣列設定 → 即時預覽 →**索引**齒輪圖示。此為單色通道路徑：波段清單涵蓋**所有成員相機**，因此單色成對相機會在此貢獻其兩個波段。

<!-- SCREENSHOT-NEEDED: Index Calculator pane opened for a combined array of two mono cameras (e.g. F650 + F850): band chips row showing the two bands with wavelength labels, the operator buttons, the expression textarea containing "(NIR - Red) / (NIR + Red)", the green "Valid expression" banner, the LUT controls (Apply LUT checked, Level 7-stop, Min 0.2 / Max 1), and the live histogram with p2/p98 percentile lines. -->

其控制項，由上而下：

* **波段晶片**（「波段 — 點擊以新增至表達式」） — 每個可用波段對應一個按鈕，標示為顏色名稱 + 波長（單位為 nm）；若顏色名稱重複，則會以「顏色 850」等方式區分。點擊後，波段標記會插入至光標位置。 無法產生單波段輻度（RGB

/FRGB）的相機所屬波段將被過濾掉。
* **運算子與函數按鈕** — `+ - * / ( ) ^ ,` 以及 `abs() sqrt() log() log10() exp() min() max() pow()`。
* **表達式文字輸入區** — 自由輸入公式；佔位符顯示經典的 NDVI

格式 `(NIR - Red) / (NIR + Red)`。其上方的唯讀標記化預覽會將波段圖塊、數字及標誌渲染為未知標記。
* **有效性橫幅**— 灰色「空白 — 不會套用任何索引」；綠色「有效表達式」；紅色並顯示具體的解析錯誤（未知波段、因多台相機導致的波段歧義、缺失括號等）；或當表達式有效但包含**常數** 時顯示琥珀色 （例如 `X/X`，或將 NDVI

分母輸入為 `−` 而非 `+`）—— 常數會將整個畫面映射為單一顏色。
* 若所套用的表達式無誤，但**即時畫面呈單一色調**（平淡或飽和的場景），則會顯示獨立的琥珀色警告——系統會自動偵測直方圖塌陷現象。
* **套用 LUT**（預設開啟；關閉 = 灰階拉伸）、**級數**2/3/5/7 級（預設 7 級），以及位於漸變條兩側的**最小值 / 最大值**輸入欄位。 「最小值」預設為**

0.2**— 此設定會將色彩漸變縮放至與植被相關的範圍，而低於此數值的範圍將以灰階呈現；若將「最小值」設為 −1，則可涵蓋完整的指數範圍（**重置** 按鈕可恢復為 −1…+1）。「最大值」預設為 1。
* 指數分佈的 **即時直方圖** — 採用平方根縮放的柱狀圖、琥珀色的第 2/98 百分位數線、白色中位數線，以及超出範圍的尾部讀數（「◀ N% &lt; lo」/「hi &lt; N% ▶”），當數值超過 1% 時會轉為琥珀色，作為擴大「最小值／最大值」範圍的提示。
* **套用** 會將表達式套用至即時串流；LUT 調整無需按下「套用」即可即時生效。若開啟專案，表達式與 LUT 設定將隨專案儲存，並於重新開啟時還原；若未開啟專案，則僅在當次工作階段中有效。

<!-- SCREENSHOT-NEEDED: Combined-array live tile rendering NDVI from a mono pair through the default 7-stop LUT, with the array name pill and fps readout visible — the result of applying the expression from the previous screenshot. -->



## CLI

路徑

相同的對齊 → 堆疊 → 索引鏈，可透過腳本進行端到端控制：

```bash
chloros-cli lattice align-calibrate --serials SN_RED,SN_NIR --save align.json
chloros-cli lattice index --live --profile align.json \
  --preset NDVI --channel red=Red_660 --channel nir=NIR_850 \
  --save-multiband -o output/
```

`align-calibrate` 使用一個連接的陣列來存放其相機，或自行開啟相機以進行單次執行。 `index --live` 總是自行開啟攝影機（一個 [暫存陣列](../reference/cli-reference.md#multi-camera-transient-sessions)），且沒有 `--serials`：它會開啟網路上的每台攝影機。 因此，請在 Chloros

釋放其持有的所有攝影機之前——包括 GUI、`cam-connect`、`array-connect`（`chloros-cli lattice array-disconnect --all`）： 若再次開啟已被佔用的攝影機，可能會中斷其持有者的串流，或導致後端當機。若攝影機未傳送任何畫面，執行程序即告失敗，且不會寫入堆疊或索引。

`--channel` 將預設值的符號映射至堆疊的波段名稱。 以下兩項規則可避免執行失敗：

* **符號區分大小寫**，且必須與預設值的通道名稱完全吻合 — 預設值使用小寫（NDVI

的預設值為 `red`、`nir`； 請參閱 `--list-presets`）。`--channel red=Red_660` 可正常運作；`--channel RED=660` 則會因 `channel_map missing entries` 錯誤而失敗。
* 波段側必須指定對齊圖疊中的某個波段（`lattice align-info --profile align.json` 列出了這些波段）。離線模式亦接受以 0 為起點的波段索引，例如 `--channel red=0 --channel nir=1`。

`lattice index` 亦可離線執行，並以已儲存的對齊多頻段 TIFF

檔案為目標。 所有形式皆在後端執行，因此後端必須正在運行，並已登入付費的 Chloros

+ 層級；相對路徑的 `--input` 或 `--output` 會根據您執行命令的資料夾進行解析：

```bash
chloros-cli lattice index --input aligned.tif --preset NDVI \
  --output ndvi.tif --colorize --gradient RdYlGn
```

### 索引預設值

`lattice index --preset`（以及「影像」分頁中的 [索引/LUT 沙盒](../image-viewer-gui/index-lut-sandbox.md)，其採用相同的引擎）內建以下 **22 組預設值**：

`NDVI, GNDVI, BNDVI, NDRE, ENDVI, SAVI, OSAVI, MSAVI, EVI, EVI2, CVI, MSR, TDVI, LAI, GLI, NGRDI, VARI, TGI, EXG, CIRE, CIGREEN, NDWI`

執行 `chloros-cli lattice index --list-presets` 可查看各預設值的公式與通道符號，執行 `--list-gradients` 則可查看可用的色彩漸變。自訂公式請使用 `--formula EXPR`，其語法與「索引計算器」相同。 請注意，此預設值清單專屬於 LATTICE 指數引擎 — 匯入影像時的「專案設定」中「處理」下拉選單所顯示的清單則不同（請參閱 [多光譜指數公式](../project-settings/multispectral-index-formulas.md)）。

完整的標誌集（`--output-format`、`--vmin/--vmax/--percentile`、 `--bg-mode`、`--live` 的對齊變形旋鈕等）詳見 [CLI

參考手冊 § 指數／植被數學](../reference/cli-reference.md#index--vegetation-maths)； SDK

對應項目請參閱 [SDK

參考手冊](../reference/sdk-reference.md)。

## 從單色陣列擷取索引產品

當陣列已連接且已套用索引表達式後，`array-capture`（或 GUI 中的 **擷取全部**) 會儲存每台相機的輸出層級 *以及* 索引渲染結果 — `--index`/`--no-index` 可在 CLI

上切換此功能，且預設擷取會包含所有適用的層級。 單色攝影機對每個擷取群組的貢獻，包含其單一波段在原始/去拜耳化（灰階）/輻射度/反射率層級的資料，以及當陣列以組合模式運行時所產生的共用組合索引合成影像。 請參閱 [多相機陣列 § 擷取](arrays.md#capturing-monitoring-vs-analysis)。
