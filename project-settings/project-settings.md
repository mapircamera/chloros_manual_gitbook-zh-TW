# 專案設定

Chloros

中的「專案設定」<img src="../.gitbook/assets/icon_project-settings.JPG" alt="" data-size="line">

側邊欄，可讓您針對專案中的影像處理、校準目標偵測、多光譜指數計算以及匯出選項等各方面進行設定。這些設定會與您的專案一同儲存，並可儲存為範本，以便在多個專案中重複使用。

## 存取專案設定

要存取「專案設定」：

1. 在 Chloros

中開啟一個專案
2. 點擊左側邊欄中的 **專案設定** <img src="../.gitbook/assets/icon_project-settings.JPG" alt="" data-size="line">

標籤頁
3. 設定面板將依類別顯示所有可用的配置選項



<!-- SCREENSHOT-NEEDED: Full Project Settings sidebar of a LATTICE project, scrolled so the Processing category is visible showing the per-product export checkboxes (Export sensor response, Export vignette corrected, Export debayered, Export preview, Export radiance, Export reflectance) and the Debayer method row. -->

{% hint style="info" %}
**取決於其他設定的選項會顯示為灰色。** 當父級開關導致某項設定無法進行時（例如，取消勾選 *反射率校正 / 白平衡* 會使 *匯出反射率* 無法執行），依賴的控制項將被停用，其工具提示會標示必須變更的開關名稱。
{% endhint %}

***

## 顯示

### 圖片縮圖解析度

* **類型**：下拉式選單
* **選項**：`Default (512 px)`、`1024 px`、`2048 px`、`Full resolution`
* **預設值**：預設 (512 像素)
* **說明**：渲染影像網格縮圖時所採用的解析度（以最長邊的像素數為準）。數值越高，放大時畫面越清晰，但載入速度較慢且會佔用更多記憶體。全解析度則採用原始影像的大小。
* **註**： 僅供顯示 — 此設定絕不會影響處理或匯出檔案。***

## 目標偵測

這些設定控制 Chloros

如何在您的影像中偵測並處理校準目標。這兩項設定僅在啟用 **反射率校準 / 白平衡** 時才生效（否則會顯示為灰色，因為此時會完全跳過目標偵測）。

### 最小校準樣本面積 (px)

* **類型**：數字
* **範圍**：0 至 10,000 像素
* **預設值**：25 像素
* **說明**：設定被偵測區域需具備的最小面積（以像素為單位），方能被視為有效的校準標靶樣本。較小的數值可偵測較小的標靶，但可能會增加誤報。較大的數值則需要更大、更清晰的標靶區域才能進行偵測。
* **何時調整**：
  * 若因小型影像瑕疵導致誤檢，請增加此數值
  * 若校準目標在影像中顯得過小且無法被偵測到，請減少此數值

### 最小目標聚類 (0-100)

* **類型**：數字
* **範圍**：0 至 100
* **預設值**：60
* **說明**：控制在偵測校準目標時，將顏色相似區域分組的聚類閾值。較高的數值要求更多相似顏色才能被分組，導致目標偵測結果較為保守；較低的數值則允許目標群組內存在更多顏色變化。
* **何時調整**：
  * 若校準目標被分割成多個檢測結果，請增加此數值
  * 若顏色變化較大的校準目標未能被完整檢測，請減少此數值

***

## 處理

這些設定控制 Chloros

如何處理及校準您的影像。

### 暗角校正

* **類型**：核取方塊
* **預設值**：啟用（已勾選）
* **說明**：套用暗角校正以補償影像邊緣因鏡頭特性所導致的變暗現象。暗角是一種常見的光學現象，指因鏡頭特性使影像的角落和邊緣看起來比中心更暗。
* **副作用**：此開關同時決定執行程序將寫入哪種 *未校正的備用產品*（詳見下文）。

### 反射率校正／白平衡

* **類型**：核取方塊
* **預設**：啟用（已勾選）
* **說明**：啟用反射率校正 — 根據相機類型及可用資源，透過偵測到的畫面內校正目標和／或 DAQ 光感測器的下行光數據進行校正。此功能可將資料集中的反射率值進行標準化，並確保無論光線條件如何，測量結果皆保持一致。
* **當停用時**：將完全跳過標靶偵測，且**任何相機均無法產生反射率產出** —— 無論是 Survey3

標靶驅動模式或 LATTICE DAQ 驅動模式皆然。相關設定（*匯出反射率*、*最小重新校準間隔* 以及標靶偵測閾值）將顯示為灰色且無法編輯。

### 未校準的備用產出：匯出感測器響應 / 匯出暗角校正

* **類型**：兩個核取方塊
* **預設值**：兩者皆啟用（已勾選）
* **說明**：當幀影像無法進行反射率校準（未找到校準目標，或反射率校準已關閉）時，該幀將被寫入為 *未校準的備用產品*。**每台相機型號**，每組運行僅存在其中一種備用產品，由 *暗角校正* 開關決定：
  * 暗角校正 **開啟**→ `Vignette_Corrected_Images/`（受**匯出經暗角校正的檔案** 設定控制）
  * 暗角校正 **關閉**→ `Sensor_Response_Images/`（受**匯出感測器響應** 設定控制）
* 未啟用的備用選項會顯示為灰色。取消勾選當前啟用的選項，將完全停止寫入該檔案。

### LATTICE 匯出產品

對於包含 LATTICE 擷取影像的專案，每個匯入的 LATTICE 幀將在單次處理過程中，擴展為所有已啟用**且適用**的產品。 有四個核取方塊用來控制此扇出功能（預設皆為**開啟**）：

| 設定 | 輸出資料夾 | 輸出內容 |
| --- | --- | --- |
| **匯出去拜爾化** | `Debayered_Images/` | 線性去拜爾化影像： 已移除校準套件的暗場偏移，並進行邊緣感知去馬賽克處理，未進行線性校正、平場校正或色彩分離 — 此即擷取檔在去馬賽克層級所儲存的原始檔案。維持感測器的原始方向（記錄於 XMP 中）。適用於 RGB

及多光譜相機。 |
| **匯出預覽** | `Preview_Images/` | 螢幕預覽。RGB

= 白平衡（若有 DAQ 光源則採用該光源，否則採用灰階世界）+ 伽瑪校正；多光譜 = 假色拉伸。 |
| **匯出輻照度** | `Radiance_Images/` | 以 W/m²/sr/nm 為單位的 Float32 光譜輻照度。僅適用於多光譜（M3C/M3M）——不適用於 RGB

原始檔。 無論 *校準影像格式* 設定為何，皆以 32 位元 TIFF

格式寫入。 |
| **匯出反射率**| `Reflectance_Calibrated_Images/` | Uint16 反射率，經縮放使**32768 = 反射率 1.0** （標記為 XMP `Chloros:PixelScale`）。僅限多光譜影像，當有相符的 `.daq` 下行輻射記錄（或通過品質保證的幀內目標）覆蓋該幀時才會寫入。 |

* RGB

主相機輸出去拜爾化 + 預覽影像；因其不適用，故跳過輻照度／反射率資料。
* 去拜爾化／預覽影像的位元深度遵循 *校準影像格式* 設定；輻照度始終為 float32。
* Survey3

處理不受這四項開關設定影響。
* 多光譜 M3C 相機的輻照度與反射率產出，亦可將每像素的 **裁剪遮罩** 至產品資料夾旁的 `clamp_masks/` 資料夾中，記錄波段解混時將負值裁剪為零的位置。詳見 [輸出影像格式](../output-image-formats.md#clamp-masks)。

這四個開關在無介面模式下亦以 `chloros-cli process --debayered / --preview / --radiance / --reflectance` 形式存在。它們取代了舊版的 CLI

標誌 `--radiometric-output`，該標誌現已不存在。

{% hint style="warning" %}
**若關閉所有適用的產出項目，將導致處理任務失敗。** 自 1.2.0 版本起，若處理任務被要求產生產出項目但未寫入任何影像產出，系統將回報失敗，且 CLI

會以非零狀態退出，而非回報無聲成功。日誌中會列出無法寫入的產出項目及其原因。刻意僅產生元資料的處理任務 （未請求任何產品）仍會被視為成功。
{% endhint %}

### 反射率來源（專案設定，透過 CLI

設定）

該專案亦會儲存 LATTICE 反射率產品所使用的 **反射率參考值**。 設定面板中沒有專用的控制項；該值以 `Processing → "Target reflectance source"` 形式儲存於專案配置中，並透過 `chloros-cli process --reflectance-source {auto,target,daq}` 進行設定 （SDK

中的 `configure()` 並無對應的關鍵字）：

* **`auto`**（預設）：通過品質保證（QA）的幀內校準目標將成為絕對參考；若無目標或 QA 失敗，則回退至 DAQ 下行分界值（ρ = πL/E）。 若標靶在品質保證（QA）中雖通過但帶有標記（例如面板脫落或波段異常），則不會自動被優先選用：系統會依其在保留面板上測得的評分順序，依序嘗試「標靶」、「標靶+DAQ」及「僅DAQ」這三種鏈路。
* **`target`**：嚴格的目標驅動反射率 — 不進行 DAQ 替代。
* **`daq`**：以 DAQ 為權威的反射率；幀內目標不會用作參考。

儲存的值在比對時不區分大小寫，且允許若干拼寫作為別名：`target`、`target_image`、 `empirical` 及 `empirical_line` 均代表 **目標**； `daq`、`dls`、`light_sensor` 以及 `sensor` 均代表**daq**。其他任何情況——包括未存在的鍵——均解析為**auto**。

按單位進行的 **測量** 目標掃描，會根據目標單位的序列號/QR（例如 `<serial>.csv`）在三個位置進行查詢： 由 `--target-reflectance-dir` 指定的目錄（儲存為 `Processing → "Target reflectance dir"`）、專案本身的 `target_reflectance/` 資料夾， 以及 `CHLOROS_TARGET_REFLECTANCE_DIR` 環境變數所指定的路徑。若該單元不存在實測掃描資料，則改用目標機型的公佈名義曲線。

### 去拜爾演算法

* **類型**：下拉式選單
* **選項**：
  * 標準（快速，中等品質）
  * 紋理感知（緩慢，最高品質）\[Chloros

+\]
* **預設值**：標準（快速，中等品質）
* **說明**：選擇用於將原始拜耳陣列感測器資料轉換為全彩影像的去馬賽克演算法。「標準（快速、中等品質）」方法在處理速度與影像品質之間提供了最佳平衡。「紋理感知（緩慢、最高品質）」 \[Chloros

+] 採用高品質的邊緣感知去拜耳化演算法，並結合 AI/ML 降噪模型，可消除幾乎所有的去拜耳化噪點。紋理感知模型需要 GPU 記憶體（VRAM）才能運行。建議在可用 VRAM 超過 4GB 時使用此模式，以獲得更快的處理速度。
* **當該行為下拉式選單時**：僅當**兩者**皆為真時才會出現——即您已使用符合資格的 Chloros

+ 訂閱登入，**且**該專案中不含任何 LATTICE 擷取畫面。否則，該行將以純文字形式顯示為 `Standard (Fast, Medium Quality)`，且無任何選項可供選擇。
* **LATTICE 注意事項**： 目前尚無經 LATTICE 訓練的「紋理感知」模型，且無論儲存的值為何，處理流程都會強制對 LATTICE 幀套用標準去馬賽克演算法。若您將 LATTICE 資料夾新增至已選取「紋理感知」的專案中，Chloros

會將設定重寫回「標準」，而非在 `project.json` 中保留過時的設定值。

### 最小重新校準間隔

* **類型**：數字
* **範圍**：0 至 3,600 秒
* **預設值**：0 秒
* **說明**：設定使用校準標靶之間的最小時間間隔（以秒為單位）。當設定為 0 時，Chloros

將使用每個偵測到的校準標靶。當設定為較高數值時，Chloros

僅會使用間隔至少為此數秒的校準標靶，從而減少在頻繁擷取校準標靶的資料集中所需的處理時間。
* **何時調整**：
  * 當光線條件變化時，設定為 0 可獲得最高的校準精度
  * 當光線條件穩定且您擁有大量校準目標影像時，可增加此值（例如設定為 60 至 300 秒）以加快處理速度

### 光感測器時區偏移

* **類型**：數字
* **範圍**：-12 至 +12 小時
* **預設值**：0 小時
* **說明**：指定光感測器資料時間戳記的時區偏移量（以小時為單位，相對於 UTC），用於將光感測器日誌與影像擷取時間進行比對。較新的 `.daq` 錄製檔會攜帶自身的時間區來源資訊，因此此設定主要適用於以當地時間記錄的舊版日誌。

### 套用 PPK 校正

* **類型**：核取方塊
* **預設值**：停用 （未勾選）
* **說明**：啟用來自內建 GPS（GNSS）的 MAPIR

DAQ 記錄器的後處理運動學（PPK）校正。啟用後，Chloros

將使用專案目錄中任何包含曝光引腳資料的 .daq 日誌檔案，並對您的影像套用精確的地理位置校正。
* **需求**：專案目錄中必須存在包含曝光針記錄的 .daq 日誌檔案
* **何時啟用**：若您的 .daq 日誌檔案中含有曝光回饋記錄，建議始終啟用 PPK 校正。

### 曝光引腳 1

* **類型**：下拉式選單
* **可見性**：僅在以下情況下可見 「套用 PPK 校正」已啟用，且第 1 針腳有可用曝光資料時才可見
* **選項**：
  * 專案中偵測到的相機型號名稱
  * 「勿使用」－ 忽略此曝光針腳
* **預設值**：根據專案設定自動選取
* **說明**：將特定相機指派至曝光引腳 1，以進行 PPK 時間同步。曝光引腳會記錄相機快門觸發的確切時間點，這對於精確的 PPK 地理定位至關重要。
* **自動選取行為**：
  * 單一相機 + 單一引腳：自動選取相機
  * 單一相機 + 兩個引腳：引腳 1 會自動指派給該相機
  * 多個相機：需手動選取

### 曝光引腳 2

* **類型**：下拉式選單
* **可見性**：僅在「套用 PPK 校正」已啟用且引腳 2 有可用曝光資料時才會顯示
* **選項**：
  * 專案中偵測到的相機型號名稱
  * 「勿使用」－ 忽略此曝光接腳
* **預設值**：根據專案設定自動選取
* **說明**：在使用雙相機設定時，將特定相機指派給曝光接腳 2，以進行 PPK 時間同步。
* **自動選取行為**：
  * 單一相機 + 單一引腳：引腳 2 會自動設定為「勿使用」
  * 單一相機 + 兩個引腳：引腳 2 會自動設定為「勿使用」
  * 多個相機：需手動選取
* **注意**：同一台相機無法同時指派給第 1 針腳和第 2 針腳。***

## DAQ 光感測器

此區段出現在「專案設定」中，並列出專案中的所有 DAQ 下行記錄檔 — `.daq` 錄製檔以及 DAQ-M `.csv` 下行輻射日誌。在「光感測器」分頁中建立的記錄會自動新增至開啟中的專案。

<!-- SCREENSHOT-NEEDED: Project Settings "DAQ Light Sensor" section of a project containing at least one .daq file, showing the "Cap override (all files)" dropdown and a per-file row with its resolved cap. -->



每行顯示該檔案、感測器型號，以及該檔案實際生效的漫射蓋校正設定。行上方有一個適用於整個專案的控制項：

### 漫射蓋覆寫（所有檔案）

* **類型**：下拉式選單
* **選項**：`Auto` 加上專案中現有感測器類型所適用的散射蓋校正曲線
* **預設值**：自動
* **儲存為**：`Processing → "DAQ cap id"` （預設為 `auto`）
* **說明**：`Auto` 會使用各檔案所記錄的頂蓋參數（若未記錄任何資料，則預設採用 Sunshine 頂蓋；但 DAQ-U、DAQ-M 和 DAQ-E 出廠時未配備頂蓋，且 Chloros

無法辨識實際安裝的頂蓋， 因此處理過程會針對假設的蓋子發出警告：請選擇實際安裝的蓋子）。選擇特定蓋子將覆寫專案中**所有**下行輻射檔案，唯 DAQ-E-S 除外，因其擴散器是校準的一部分；僅包含 DAQ-E-S 的專案不會顯示下拉選單：原始記錄將以此進行校正， 而已帶有校正值的記錄則會重新參照（即撤銷原先的校正並套用所選的校正值）。
* **重要**：所選的校正值必須與記錄過程中實際安裝的校正值相符。無論是感測器還是軟體都無法偵測實體校正值——校正值 ID 不匹配會導致光譜校正錯誤。

此處刻意採用 **單一** 適用於整個專案的控制項，而非逐檔下拉選單：此設定將套用至專案中的每個下行光源。***

## 陣列對齊

本節 **僅** 在專案中至少有一張影像攜帶 LATTICE 陣列於擷取時標記的模組間對齊變換（XMP `Chloros:Alignment*` 標籤）時才會顯示。此處會顯示帶有對齊標籤的影像數量、哪台相機作為參考（`REF` 標記），以及各相機的影像數量表格。

<!-- SCREENSHOT-NEEDED: Project Settings "Array Alignment" section for an imported LATTICE array capture set, showing the tagged-image count, the per-camera rows with the REF badge, and the three controls (Apply array alignment, Crop to common overlap, Resampling). -->

### 套用陣列對齊

* **類型**：核取方塊
* **預設值**：啟用（已勾選）
* **儲存格式**：`Processing → "Array alignment"`
* **說明**：使用拍攝時標記的變換，將每個已處理的產品（去拜耳化／預覽／輻射度／反射率／索引）變形至陣列的共用參考幾何結構中。關閉時，將以原生各感測器幾何結構進行匯出。

### 裁切至共同重疊區域

* **類型**：核取方塊（僅在 *套用陣列對齊* 啟用時有效）
* **預設值**：啟用（已勾選）
* **儲存為**：`Processing → "Array alignment crop"`
* **說明**：將對齊後的匯出檔案裁切至所有相機模組共有的區域，使每個波段具有相同的覆蓋範圍。關閉時則保留完整的感測器畫布（來源區域外以黑色填充）。

### 重新取樣

* **類型**：下拉式選單（僅在 *套用陣列對齊* 啟用時有效）
* **選項**：`Bilinear (smooth, default)`、`Nearest (preserve exact values)`、`Cubic (sharpest)`
* **預設**：雙線性
* **儲存為**：`Processing → "Array alignment interpolation"`
* **說明**：對齊變形所使用的插值方式。「最近鄰」可保留原始影像的精確數值（無像素間混合），適用於嚴格的輻射測量分析；「雙線性」則最適合地圖繪製與視覺呈現。

這三個選項在無頭模式下亦以 `chloros-cli process --array-alignment`、`--array-alignment-crop` 及 `--array-alignment-interp {bilinear,nearest,cubic}` 形式存在。

***

## 指標

這些設定可讓您配置用於分析與視覺化的多光譜指標。

### 新增指標

* **類型**：特殊指數設定面板
* **說明**：開啟一個互動式面板，您可在其中選取並設定多光譜植被指數（NDVI

、NDRE

、EVI

等），以便在影像處理過程中進行計算。您可以新增多個指數，每個指數皆可擁有各自的視覺化設定。
* **可用指標**：GUI 下拉選單包含**27** 個預定義的多光譜指標公式（完整清單請參閱 [多光譜指標公式](multispectral-index-formulas.md)，其中包含 CLI

/SDK

`--indices` 選項所支援的名稱）。
* **功能**：
  * 從預定義的指數公式中進行選擇
  * 將相機的濾波器通道拖曳至公式的波段插槽
  * 設定視覺化色彩漸層（LUT - 查找表）
  * 設定閾值與剪裁模式
  * 建立自訂指數公式
* **注意事項**：針對單波段的 LATTICE M3M 單色相機，系統不會計算指數 — 單一波段下，多波段指數的定義將失效。Survey3

與 LATTICE M3C 則不受此影響。



<!-- SCREENSHOT-NEEDED: Project Settings > Index section with one index added and expanded: the filter dropdown, the formula dropdown open showing preset names, the coloured channel circles above the rendered formula, and the "+ Add LUT" button below it. -->

您新增的每個指數都會以數學公式形式呈現，每個波段插槽對應一個彩色圓圈：紅色 = 紅光（Red

）、綠色 = 綠光（Green

）、藍色 = 藍光（Blue

）、橘色 = 橘光（Orange

）、青色 = 青光（Cyan

）、 紫色 = NIR

，洋紅色 = RE。將公式上方行中的圓圈拖曳至插槽即可綁定；雙擊已綁定的插槽即可清除。僅當公式所用的每個插槽皆有對應通道時，該指數才會進行計算。

### 自訂公式（Chloros

+ 功能）

* **類型**：自訂公式定義陣列
* **可用性**：需使用符合資格的 Chloros

+ 訂閱帳戶登入。
* **說明**：允許您使用波段運算來建立並儲存自訂多光譜指數公式。自訂公式會與您的專案設定一同儲存，並可如同內建指數般使用。
* **建立方式**：
  1. 在「指數設定」面板中，開啟自訂公式計算器
  2. 撰寫公式時請使用 **波段槽位符號**，而非波段名稱
  3. 以具描述性的名稱儲存公式 — 該公式隨後會出現在公式下拉選單的底部，您可將相機的通道圓圈拖曳至其插槽中，操作方式與內建預設值完全相同
* **公式語法**：
  * 波段插槽： `x`、`y`、`z`、`a`、`b`、 `c` — 六個位置，可透過拖曳映射至實際通道
  * 運算子：`+`、`-`、`*`、`/`、 `^` 以及 `()` 用於分組
  * 函數：`sqrt()`、`log()`、`ln()`、 `abs()`、`sign()`、`log1p()`、`log2()`
* **為何使用符號而非樂團名稱**：以 `(y-x)/(y+x)` 形式書寫的公式適用於任何相機，因為拖放映射會決定 `y` 是 RGN

濾鏡的 850 nm NIR

，還是 OCN

濾鏡的 808 nm NIR

。 內建預設值也是以相同方式儲存的——請參閱 [多光譜指數公式](multispectral-index-formulas.md)，了解全部 27 種公式的確切符號形式。
* **適用範圍**：自訂公式會與專案設定一同儲存，並可在 [索引/LUT 沙盒](../image-viewer-gui/index-lut-sandbox.md) 以及影像處理過程中使用。 CLI

／SDK

的 `--indices` 名稱清單**不**支援這些公式，該清單僅擴充 22 個內建預設名稱。***

## 匯出

這些設定用於控制匯出後經處理影像的格式與品質。

### 校準影像格式

* **類型**：下拉式選單
* **選項**：
  * **TIFF

（16 位元）** - 未壓縮的 16 位元 TIFF

格式
  * **TIFF

(8 位元)** - 未壓縮的 8 位元 TIFF

格式
  * **TIFF

(32 位元，百分比)** - 32 位元浮點 TIFF

，反射率值以百分比表示
  * **PNG

(8 位元)** - 壓縮的 8 位元 PNG

格式
  * **JPG (8 位元)** - 壓縮的 8 位元 JPEG

格式
* **預設**：TIFF

(16 位元)；若專案全為 8 位元，則顯示 TIFF

(8 位元)，且僅提供 8 位元格式
* **說明**：選擇用於儲存經處理及校正後影像的檔案格式。匯出的檔案將存放於各相機資料夾內對應該格式的子資料夾中 (`tiff16`、`tiff32`、`png8`、`jpg8`)，每個產品對應一個 `<Product>_Images/` 資料夾。匯出檔案將保留原始檔案名稱——由資料夾名稱（而非檔案名稱後綴）來識別產品。
* **格式建議**：
  * **TIFF

（16 位元）**：建議用於科學分析與專業工作流程。能最大程度保留資料品質，且無壓縮失真。最適合多光譜分析及在 GIS 軟體中的後續處理。
  * **TIFF

（32 位元，百分比）**：最適合需要以百分比（0-100%）表示反射率值的工作流程的工作流程。可為輻射測量提供最高精度。
  * **PNG

（8 位元）**：適用於網頁瀏覽與一般視覺化。採用無損壓縮，檔案大小較小，但動態範圍較窄。
  * **JPG (8 位元)**：檔案大小最小，僅適用於預覽及網頁顯示。採用有損壓縮，不適合科學分析。
* **注意**：無論此設定為何，LATTICE 輻照度皆會以 32 位元浮點數 TIFF

格式匯出。

***

## 儲存專案範本

此功能可讓您將當前專案設定儲存為可重複使用的範本。

* **類型**：文字輸入欄位 + 儲存按鈕
* **說明**：為您的設定範本輸入具描述性的名稱，並點擊儲存圖示。該範本將儲存您當前專案的所有設定（目標偵測、處理選項、指標、 及匯出格式），以便日後在專案中輕鬆重複使用。範本儲存於專案儲存資料夾內的 `Project Templates/` 資料夾中，亦可從主選單中選取或匯出（*選取範本* / *儲存範本* / *匯出範本*）。
* **應用情境**：
  * 為不同相機系統建立範本（RGB

、多光譜、NIR

）
  * 儲存特定作物類型或分析工作流程的標準設定
  * 在團隊內共享一致的設定
* **使用方法**：
  1. 設定所有所需的專案設定
  2. 輸入範本名稱（例如：「RedEdge

Survey3

NDVI

標準」）
  3. 點擊儲存圖示
  4. 建立新專案時即可載入此範本

***

## 專案儲存資料夾

此設定指定新專案的預設儲存位置。

* **類型**：目錄路徑顯示 + 編輯按鈕
* **預設 (Windows

)**：位於「文件」資料夾中的 `Documents\MAPIR\Chloros Projects` — 通常為 `C:\Users\[Username]\Documents\MAPIR\Chloros Projects`，或 OneDrive 及漫遊設定檔將「文件」資料夾移動至的任何位置
* **預設 (Linux

)**：`~/Documents/MAPIR/Chloros Projects`
* **說明**：顯示目前用於建立新 Chloros

專案的預設目錄。點擊編輯圖示可選擇其他目錄。此覆寫設定將以單行文字形式儲存於 `~/.chloros/working_directory.txt` 檔案中 — 在 Windows

系統上則為 `C:\Users\<Username>\.chloros\working_directory.txt`。 若該檔案遺失，或其指定的路徑已不存在，Chloros

將回退至上述預設值。CLI

與 GUI 皆讀寫同一個檔案，因此 `chloros-cli` 與 GUI 對專案存放位置的設定始終一致。
* **專案範本** 存放於此目錄下的 `Project Templates/` 子資料夾中。
* **何時應變更**：
  * 為團隊協作而設定為網路磁碟機
  * 針對大型資料集，變更至儲存空間較大的磁碟機
  * 依年份、客戶或專案類型將專案整理至不同資料夾
* **注意**：變更此設定僅會影響新建立的專案。現有專案仍保留在原始位置。
* **Windows

路徑長度**：除非已啟用 Windows

長路徑（安裝程式的 **啟用 Windows

長路徑** 選項），否則路徑長度超過約 130 個字元的專案資料夾將在處理前被拒絕，因為其產出檔案會超過 Windows

的 260 個字元路徑限制。 請參閱 [啟動處理](../processing-images-gui/starting-the-processing.md#the-project-folder-is-too-deep-for-this-computer)。

***

## 設定的持久性

Chloros

專案即為一個 **資料夾**。所有專案設定皆儲存於其中的 `project.json` 檔案中； 已連接的硬體資訊則與之一同儲存於 `cameras.json` 和 `sensors.json` 中，因此重新開啟專案時，其攝影機和光感測器也會自動重新連線。當您重新開啟專案時，所有設定都會完全恢復至您上次離開時的狀態。 儲存的專案亦可透過 `chloros-cli project` 或 SDK

的 `open_project` 進行無介面操作。

### 設定層級

設定的套用順序如下：

1. **系統預設值** — 由 Chloros

定義的內建預設值2. **範本設定** — 若您在建立專案時載入範本
3. **已儲存的專案設定** — 與專案檔案一同儲存的設定
4. **手動調整** — 您在當前工作階段中所做的任何變更

### 設定與影像處理

處理設定會在處理執行開始時讀取。變更設定並不會追溯性地修改已儲存於磁碟上的產出結果 — 請重新執行處理以套用新設定。部分設定完全不會影響處理：

* 影像縮圖解析度（僅供顯示）
* 儲存專案範本
* 儲存專案資料夾

***

## 配置金鑰參考

針對自動化操作（CLI

`--config`、SDK

`configure` 或直接讀取 `project.json`），以下是 `Project Settings` 下的確切設定鍵：

| 金鑰路徑 | 類型 | 預設值 |
| --- | --- | --- |
| `Display → Image Thumbnail Resolution` | `"512" \| "1024" \| "2048" \| "full"` | `"512"` |
| `Target Detection → Minimum calibration sample area (px)` | 數值 0-10000 | `25` |
| `Target Detection → Minimum Target Clustering (0-100)` | 0 至 100 之間的數值 | `60` |
| `Processing → Vignette correction` | 布林值 | `true` |
| `Processing → Reflectance calibration / white balance` | 布林值 | `true` |
| `Processing → Export sensor response` | 布林值 | `true` |
| `Processing → Export vignette corrected` | 布林值 | `true` |
| `Processing → Export debayered` | 布林值 | `true` |
| `Processing → Export preview` | bool | `true` |
| `Processing → Export radiance` | bool | `true` |
| `Processing → Export reflectance` | 布林值 | `true` |
| `Processing → Array alignment` | 布林值 | `true` |
| `Processing → Array alignment crop` | 布林值 | `true` |
| `Processing → Array alignment interpolation` | `"Bilinear" \| "Nearest" \| "Cubic"` | `"Bilinear"` |
| `Processing → Debayer method` | `"Standard (Fast, Medium Quality)" \| "Texture Aware (Slow, Highest Quality)"` | 標準 |
| `Processing → Minimum recalibration interval` | 數值 0-3600 | `0` |
| `Processing → Light sensor timezone offset` | 數值 -12..12 | `0` |
| `Processing → Apply PPK corrections` | 布林值 | `false` |
| `Processing → DAQ cap id` | 電容特徵曲線識別碼 或 `"auto"` | `"auto"` |
| `Processing → Target reflectance source` | `"auto" \| "target" \| "daq"` | `"auto"` |
| `Index → Add index` | 索引配置清單 | `[]` |
| `Export → Calibrated image format` | `"TIFF (16-bit)" \| "TIFF (8-bit)" \| "TIFF (32-bit, Percent)" \| "PNG (8-bit)" \| "JPG (8-bit)"` | `"TIFF (16-bit)"` |

當「陣列對齊」區段首次渲染，或透過自動化呼叫設定時，會寫入 `Array alignment` 鍵值。若這些鍵值不存在，處理流程將使用上文所示的相同數值 （`true`、`true`、雙線性），因此未包含這些參數的專案.json，其運作行為與包含這些參數的專案完全相同。

### 儲存於 `project.json` 中且設定面板中無控制項的鍵值

這些鍵位於相同的 `Project Settings` 樹下，並會在處理過程中被讀取，但您在側邊欄中找不到對應的控制項：

| 鍵路徑 | 類型 | 預設值 | 設定來源 |
| --- | --- | --- | --- |
| `Processing → LATTICE input level` | `"auto" \| "raw" \| "debayered" \| "processed"` | `"auto"` | `chloros-cli process --input-level`, SDK

`input_level=`。覆寫 LATTICE 輸入 TIFF

檔案的解析方式； `auto` 會根據每個檔案的 `Chloros:ProcessingLevel` XMP 標籤及通道數進行推斷。對於 Survey3

`.raw` 擷取資料，此設定將被忽略。 刻意不設為 GUI 設定 — 在所有正常情況下，「自動」皆為正確設定。 |
| `Processing → Target reflectance dir` | 路徑字串 | `""` | `chloros-cli process --target-reflectance-dir`，或專案目標 API

|
| `Processing → Target reflectance config` | 以相機序號為鍵的字典 | `{}` | 註冊幀內目標 （模式 `fixed_block` / `fixed_strip` / `aruco`） |
| `Processing → DAQ-U log path` | 路徑字串 | `""` | SDK

`process_folder(daq_log_path=…)`。指向一個 `.daq` 錄影檔或其資料夾 |
| `Target Detection → Minimum calibration target squares` | 數字 | `4` | 舊版預設值；無控制項且無CLI

參數 |
| `UI → Grid thumbnail size` | 數字 | `160` | 影像網格專屬的縮圖縮放滑桿 |

有兩項檢視器偏好設定儲存於 **`project.json` 的頂層**，完全位於 `Project Settings` 之外，因為它們屬於顯示狀態而非處理設定：

| 鍵值路徑 | 類型 | 預設值 | 設定來源 |
| --- | --- | --- | --- |
| `viewer_display → gsd_bin` | 整數 1–256 | `1` | 影像分頁的 GSD (px) 控制項 — 請參閱 [以全螢幕開啟影像](../image-viewer-gui/opening-an-image-full-screen.md) |

***

## 最佳實務

1. **從預設值開始**：預設設定對於大多數 MAPIR

相機系統和典型工作流程而言效果良好。
2. **建立範本**：一旦針對特定工作流程或相機優化了設定，請將其儲存為範本，以確保各專案間的一致性。
3. **全面處理前先進行測試**：嘗試新設定時，請先在少量影像上進行測試，再處理整個資料集。
4. **記錄您的設定**：使用能明確標示相機系統、處理類型及預期用途的描述性範本名稱（例如：「Survey3

\_RGB\_NDVI\_Agriculture」）。
5. **匯出格式選擇**：根據最終用途選擇匯出格式：
   * 科學分析 → TIFF（16 位元或 32 位元）
   * GIS 處理 → TIFF

（16 位元）
   * 快速視覺化 → PNG

（8 位元）
   * 網路分享 → JPG（8 位元）

***

有關 Chloros

中多光譜指標的更多資訊，請參閱 [多光譜指標公式](multispectral-index-formulas.md) 頁面。
