# 擷取設定與模式

在「相機」分頁中，擷取操作由一個紅色的 **「擷取全部」**按鈕以及一個**「擷取設定」** 窗格驅動，該窗格決定該按鈕的輸出結果：哪些相機參與擷取、每台相機儲存的匯出類型，以及快門是單次觸發、連續觸發，還是按間隔觸發。 本頁面詳述整個流程——包含設定、實際擷取操作、檔案在磁碟上的儲存位置，以及日後如何將其重新處理成經校正的產出成果。相機與陣列的控制項本身位於 [相機設定](camera-settings.md)。

{% hint style="info" %}
**進行擷取前必須開啟專案。** 在開啟專案之前，「全數擷取」及「擷取設定」齒輪圖示將處於停用狀態（「建立或開啟專案以儲存擷取結果」）。每次擷取的檔案皆儲存於 `captures/` 中的專案資料夾內。
{% endhint %}

## 「擷取設定」窗格

可透過側邊欄相機清單中 **「全部擷取」旁的齒輪圖示**，或任何單一相機設定窗格底部的**「開啟擷取設定…」** 按鈕開啟此窗格。窗格標題為「擷取設定」，並附有 ← 返回按鈕。

<!-- SCREENSHOT-NEEDED: the full Capture Settings pane — Single/Continuous/Interval mode buttons at top, the bulk export-type toggle rows (All Raw … All Index), the orange Fastest Capture toggle, an array group card with the Aligned checkbox and Record buttons, and an expanded per-camera row showing per-type checkboxes. -->



您在此處的選項——包含的攝影機、各類型核取方塊以及擷取模式——會 **依專案** 儲存，並在您重新開啟專案時還原。

### 擷取模式

面板頂端有三個模式按鈕：

| 模式 | 功能說明 | 子設定（預設值） |
| --- | --- | --- |
| **單次** *(預設)* | 對所有選取的攝影機進行一次擷取。 | — |
| **連續**| 連續擷取直至滿足停止條件。 | 根據**擷取次數**（預設 1）*或* **擷取持續時間**（預設 10 秒；單位：秒／分鐘／小時／天）停止。 |
| **間隔**（縮時攝影） | 依計時器進行連拍。 |**每次間隔的擷取次數**(預設 1) ·**每**N 單位 (預設 5 秒) ·**持續** N 單位 (預設 1 分鐘)。 |

在「連續」或「間隔」模式下，執行期間「全數擷取」按鈕會變為 **「停止 (N)」** 按鈕，並會隨擷取完成而累計擷取次數。 當所有影像皆已擷取完畢時，執行程序將停止（參見 [「擷取全部」流程](#the-capture-all-flow)）。**陣列中的「連續」模式在後端執行（1.3.0 版新增功能）。** 當清單中的每台攝影機均屬於陣列時（獨立攝影機被隱藏或排除），連續擷取模式將不再針對每次擷取發送單一請求：Chloros

後端會自行執行擷取迴圈，在選取下一組資料的同時寫入一組同步資料，而按鈕則會輪詢其計數。若清單中包含獨立攝影機，Chloros

將如以往般以逐幀方式進行擷取。

{% hint style="warning" %}
**Windows

上的快速擷取：請將 Chloros

後端排除在 Microsoft Defender 掃描範圍外。**根據 MAPIR

的測試，對高幀率擷取所寫入的每個檔案進行即時掃描，會消耗約 30% 的擷取速率 （在小範圍 ROI 下每秒數百個檔案）。Windows

安裝程式提供 **「將 Chloros

後端排除於 Microsoft Defender 掃描範圍外（可大幅提升攝影機擷取速度；建議）」** 選項，預設為勾選狀態，並會在解除安裝時移除此排除設定。 若您曾取消勾選此選項，請重新執行安裝程式並勾選該選項，或為 `chloros-backend.exe`（位於 Chloros

安裝資料夾內的 `backend` 資料夾中）新增 Defender 程序排除項目。 當 Defender 在陣列的「連續擷取」過程中佔用一個或多個 CPU 核心時，Chloros

後端會在其日誌中記錄一次，並提供上述相同的兩種解決方案。
{% endhint %}

<!-- SCREENSHOT-NEEDED: the capture-mode area of Capture Settings with Interval selected — showing the "Captures / interval", "Every N (unit)" and "For N (unit)" rows with their defaults (1, 5 s, 1 m). -->




### 選擇攝影機與匯出類型

此窗格的說明文字已概括說明：選擇「全數擷取」功能將產出的攝影機與匯出類型 — 預設為全部啟用，且選項會與此專案一同儲存。

* **「全選」／「全不選」** 按鈕可一次切換所有攝影機的納入選項核取方塊。
* **批次匯出類型切換按鈕**（兩行按鈕）：**全部原始檔 / 全部去拜耳化 / 全部預覽 / 全部輻射度 / 全部反射率 / 全部色澤指數**。每項皆採用三態色彩標示：綠色 ✓ = 所有支援該類型的相機皆啟用，琥珀色 – = 部分相機啟用，灰色 = 無相機支援。 若無任何已連接的相機支援該類型，該切換按鈕將被停用。當「最快擷取」功能開啟時，所有切換按鈕皆會變為灰色。
* **各相機專屬列**：包含一個「包含」核取方塊，以及該相機適用之匯出類型的可展開（▸/▾）清單，清單中各項目皆附有獨立核取方塊。 該行會顯示啟用數量，例如「4/6」。

### 匯出類型及其適用相機

共有六種匯出類型：**Raw、去拜耳化、輻射度、反射率、預覽、索引**。僅適用於該相機的類型才會顯示在對應的行中：

| 匯出類型 | 內容 | RGB

（FRGB） | 拜耳多光譜（FRGN/FOCN/FNGB） | 單色（M3M） |
| --- | --- | --- | --- | --- |
| **原始檔** | 直接來自感光元件的拜耳馬賽克（單色：單一波段） | ✓ | ✓ | ✓ |
| **去拜耳化** | 線性去馬賽克：從原始檔中移除校準套件的暗偏移量（DSNU），接著進行邊緣感知去馬賽克 — 不包含線性校正、平場校正或解混。儲存為 3 通道 RGB

TIFF

檔（單色：1 通道灰階） | ✓ | ✓ | ✓ |
| **預覽** | 完整顯示鏈（依據相機設定檔進行白平衡與伽瑪校正，並加上RGB

相機的「高光」設定；多光譜：假色拉伸） | ✓ | ✓ | ✓ |
| **輻照度** | 透過完整輻度測量鏈計算的 float32 W/m²/sr/nm | —（未提供）| ✓ | ✓ |
| **反射率** | uint16 ρ（32768 = 1.0）| —（未提供） | ✓ — 僅在相機具備 DAQ 光感測器時顯示（自身具備或繼承自其陣列） | 與多光譜相同 |
| **指數** | 植被指數 (LUT) 渲染 | — | ✓ — 需在相機上啟用且非空的指數表達式，且不提供給組合陣列成員（陣列擁有一個共用指數） | —（指數需 ≥2 波段；參見 [單色相機與植被指數](mono-indices.md)) |

RGB

相機絕不提供輻照度與反射率選項 — 對於寬頻光度感測器而言，每像素拜耳陣列的輻照度並無實質意義。

### 最快擷取

**⚡ 最快擷取 — 僅限原始檔**切換鈕（開啟時為橘色）會覆寫所有匯出選項，使其僅輸出**原始檔** — 並為陣列提供免費的組合指數合成檔 — 因此畫面能以最快速度完成擷取：在擷取時會完全跳過輻照度／反射率／顯示的運算步驟。

{% hint style="info" %}
**仍會儲存一個 `.daq` 檔案。** 當指派了光感測器時，「最快擷取」仍會將 DAQ 的下行輻射讀數寫入（至擷取檔的 `daq/` 資料夾中）—— 因此，輻照度、反射率及折射率產出皆可於日後透過重新處理來建立（參見 [將擷取資料重新處理為經校正的產出](#re-processing-captures-into-calibrated-products))。此外，「最快擷取」功能不會影響您的核取方塊選項：關閉此功能後，選項將恢復原狀。
{% endhint %}

### 陣列層級控制項

每個已連接的陣列在面板中皆擁有專屬的群組卡片：

* **「包含」核取方塊**（各成員狀態三態）以及陣列名稱與其顯示模式：「(合併 | 獨立)」。
* **對齊**核取方塊（預設為**開啟**）：將成員檔案變形為陣列的對齊設定檔，使各相機間的匯出檔案能實現像素級對齊。 原始檔保持未變形狀態，但其元資料中會包含變換資訊。（該配置檔本身是在 [陣列設定窗格](camera-settings.md#alignment-co-registration) 中計算得出的。）陣列成員檔案會以各台相機的原生方向儲存，任何「鏡像/旋轉」設定僅記錄於檔案中，而非實際套用。
* 成員相機的行資料會嵌套在卡片內。

陣列卡片還包含兩個錄影器。可將其視為 **監控與分析** 的區別：

| 錄影器 | 等級 | 錄製內容 |
| --- | --- | --- |
| **● 錄製索引影片 / ■ 停止錄製** *(僅限組合陣列)* | **監控** | 將即時組合索引合成影像以 10 fps 錄製為影片 — 8 位元、預覽解析度、內嵌 LUT。 需開啟專案並啟用串流即時預覽。錄製時會顯示幀數與已用時間。 |
| **⦿ 錄製原始連拍 / ■ 停止原始連拍** *（任何陣列）* | **分析**| 原始拜耳幀以即時擷取速率輸出（未經處理），並附帶每幀清單及 `.daq` 讀數，儲存至 `captures/bursts/`。 連拍結束後，會出現**建立影片** 按鈕：它會離線重新處理連拍資料，生成經校準的影片——包含合併折射率及／或各相機的輻照度／反射率／折射率——並可選配 TIFF 檔案。當您停止連拍時，合併折射率建檔會自動啟動。 |

<!-- SCREENSHOT-NEEDED: an array group card in Capture Settings while a raw burst is recording — the ⦿/■ burst button in its recording state with frame count, and (in a second capture) the Build video button that appears after stopping. -->

## 「全數擷取」工作流程

<!-- SCREENSHOT-NEEDED: the sidebar during a capture — Capture All showing live "Capturing… 3/6" progress text, and (second capture) the result flash "Saved N files". -->



在側邊欄的相機清單中按下 **全數擷取**：

1. 所有包含在內、可見且未暫停的攝影機，將依照其選定的匯出類型進行擷取。**陣列會以單一同步觸發方式運作**（所有成員皆屬於單一同步群組 — 參見 [多攝影機陣列](arrays.md)）；獨立攝影機則分別進行擷取。
2. 隱藏（眼睛圖示）或暫停的攝影機將被跳過。陣列僅在*所有*成員皆為隱藏或暫停狀態時，才會被完全封鎖。
3. 每當指派光感測器時，對應的 DAQ 下行輻射讀數都會以 `.daq` 檔案格式儲存於擷取檔案的 `daq/` 資料夾中——即使是僅擷取原始資料的情況亦然——以便日後隨時能衍生出輻測產品。
4. 按鈕會顯示即時進度 —「正在擷取… 完成/總計」— 而在「連續/間隔」模式下會變為 **停止 (N)**。每個擷取項目設有 300 秒的超時限制。
5. 當掃描完成時，結果提示會顯示 **「已儲存 N 個檔案」**或**「已儲存 N 個，F 個失敗」**，若曾跳過相機，則會額外顯示「(S 隱藏/暫停/跳過)」 若儲存裝置空間已滿，執行程序將停止並顯示**「已停止：儲存裝置已滿。已儲存 N 個，F 個失敗」** —— 每個無法寫入的檔案均計為失敗。

## 擷取檔案的儲存位置

擷取檔案會儲存於開啟的專案下，路徑為 `<project>/captures/`。每種匯出類型都會存放於其**專屬子資料夾**中，因此多層級擷取絕不會混淆不同類型：

```
<project>/captures/
├── raw/           capture_<ts>_SN<serial>_raw.tif
├── debayered/     capture_<ts>_SN<serial>_debayered.tif
├── radiance/      capture_<ts>_SN<serial>_radiance.tif
├── reflectance/   capture_<ts>_SN<serial>_reflectance.tif
├── preview/       capture_<ts>_SN<serial>_display.tif
├── index/         per-camera vegetation-index (LUT) render, when Index is selected
├── composite/     array foreground/background live-view composite, when produced
├── clamp_masks/   radiance/…_clampmask.tif, reflectance/…, index/… — unmix clamp masks,
│                  one subfolder per product folder, only when a band was clamped
├── bursts/        raw-burst recordings (frames + manifest + .daq per burst)
└── daq/           the downwelling reading(s) matched to each capture (.daq), each with a
                   .daq.cameras.json naming the cameras it describes
```

* `<ts>` 為擷取時間戳記，`<serial>` 為相機序號。 獨立擷取檔的名稱為 `capture_<ts>_SN<serial>_<level>`；來自單一同步觸發器的陣列擷取檔名稱為 `sync_<ts>_SN<serial>_<level>`，且 **該群組內所有相機共用同一個時間戳記** （當攝影機僅儲存單一層級時，層級後綴將被省略）。
* **需注意的一項不對稱現象：** 顯示層級儲存於名為 `preview/` 的資料夾中，而檔案名稱則保留 `_display` —— 僅此層級的資料夾名稱與後綴會有所不同。
* 未知層級會回退至以其自身名稱命名的資料夾；若無法建立子資料夾，檔案將寫入擷取根目錄，而非遺失。
* 擷取的 TIFF 檔案預設採用無損壓縮（DEFLATE），並將完整的校準與處理元資料 **封裝於檔案的 XMP 中** — 擷取檔具有自描述性，除了 `.daq` 讀取檔（及其對應的 `.daq.cameras.json`）外，沒有其他輔助檔案。
* **鉗位遮罩（1.3.0 版新增功能）。**當 3×3 光譜解混必須將某波段在至少一個像素上鉗位為 0 時，輻照度、反射率及折射率產出將獲得一個逐像素遮罩， `clamp_masks/<product folder>/<product name>_clampmask.tif`（位元 0 = 產品 R 通道中的波段，位元 1 = G，位元 2 = B）；產品的 `Chloros:UnmixClampMask` XMP 標籤會指向該遮罩。 這些遮罩位於**產品資料夾旁**，絕不會置於資料夾內，因此像 `reflectance/` 這樣的產品資料夾可直接傳遞給攝影測量軟體。Chloros

自身的匯入功能會跳過這些遮罩。
* **方向（1.3.0）。** 獨立相機的「預覽」、「輻射度」、「反射率」及「索引」產品會根據其「鏡像/旋轉」設定進行旋轉；而「原始檔」和「去拜耳化」則維持感測器讀取時的原始狀態。陣列成員檔案均維持原始格式。 每個檔案都會在其 XMP 中記錄該設定，因此重新處理時會以相同方式套用該設定。
* **`.daq` 屬於哪台相機 (1.3.0)。** 在處理時，擷取的 `.daq` 僅會與其 `.daq.cameras.json` 所指定的相機所拍攝的影像進行配對。 因此，未配備自有光感測器的獨立攝影機將不再借用其他攝影機的讀數——請將感測器綁定至該攝影機，或在「光感測器」分頁中記錄一個感測器。 光感測器記錄檔（以及任何未包含該檔案的 `.daq`，包括較舊的擷取檔）仍可服務所有攝影機。

這與 `chloros-cli lattice capture` / `array-capture` 寫入其 `-o` 目錄的佈局相同——詳見 [CLI

參考指南 § 擷取資料夾的結構](../reference/cli-reference.md#what-a-captures-folder-looks-like)中已有記載。

<!-- SCREENSHOT-NEEDED: OS file explorer showing a real <project>/captures/ folder after a multi-level array capture — the raw/debayered/radiance/reflectance/preview subfolders, the daq/ and clamp_masks/ folders, and sync_<ts>_SN<serial>_<level>.tif filenames visible inside one subfolder. -->

## 將擷取檔重新處理為校正後產品

已擷取的原始影像幀加上儲存的 `.daq`，便是處理流程所需的一切——這正是為何「最快擷取」模式適用於實際工作。

* **GUI**：將擷取資料夾新增至專案（[將檔案新增至專案](../processing-images-gui/adding-files-to-a-project.md)），並依常規進行處理。
* **CLI**：將 `process` 指向 **擷取資料夾根目錄**：

```bash
chloros-cli process "C:/ChlorosProjects/MyField/captures"
```

`process` 通常僅會匯入您指定的資料夾，但當該資料夾內無影像且含有子資料夾時，系統會自動向下遍歷 — 因此層級子資料夾與 `daq/` 資料夾將一次全部被擷取。 每次擷取都會以 **單一影像** 的形式匯入，並將其他層級附加為可檢視的模式，而非每層級各為一張影像。

直接為層級子資料夾命名（例如 `…/captures/raw/`）同樣有效，但會遺漏 `daq/` 資料夾——請在重新匯入時將其一併複製過來-從 `raw/` 衍生輻測產品時，請一併複製該資料夾，否則時間戳記的比對將無對應對象可解析。

{% hint style="warning" %}
**處理流程總是從 `raw` 開始。** 在每次擷取中，原始幀是處理管線的來源；`debayered`、`radiance`、`reflectance`、 以及 `preview` 雖作為可視化模式存在，但絕不會被送回處理管道——重新處理衍生產品將導致已烘焙至其像素中的暗角、色彩及輻射度運算再次套用，因此 Chloros

會選擇拒絕處理，而非進行重複處理。 `index/` 和 `composite/` 的渲染結果根本不會經過任何處理（它們是輸出檔，而非擷取檔）。 一個**未**包含原始檔匯入的擷取資料夾仍能正常顯示，但 `process` 會跳過該資料夾並明確標示；`--input-level {raw,debayered,processed}` 則是刻意設計的「逃生通道」，用以強制建立處理起點。 請參閱 [CLI

參考手冊](../reference/cli-reference.md#what-a-captures-folder-looks-like) 以了解確切的跳過訊息。
{% endhint %}

在編寫重新處理腳本時，還有兩項值得了解的行為：

* 若執行 `chloros-cli process` 時請求產出，但**未寫入任何影像產出**，則會明確報錯並以非零狀態退出**——您絕不會遇到靜默的空運行。成功執行會回報其產出數量。（刻意僅執行元資料的運行仍被視為成功。）
* 重新匯入的已處理輸出絕不會佔用擷取的原始資料槽 — 原始資料始終是管線的來源。

## 命令列介面（CLI

）對應指令

本頁所述的所有操作皆可透過無視窗模式執行。GUI 擷取模式可直接對應至 `chloros-cli lattice array-capture`：

| GUI | CLI

|
| --- | --- |
| 單次 | `chloros-cli lattice array-capture` |
| 連續 | `array-capture --continuous [--count N] [--duration S]` |
| 間隔 | `array-capture --interval S [--duration S]` |
| 最快擷取 | `array-capture --fastest` |
| 對齊核取方塊 | `--aligned / --no-aligned` |
| 匯出類型核取方塊 | `--processing LEVEL` 或 `--levels L1,L2,…`（預設為 `all`） |
| 錄製索引影片 | `chloros-cli lattice array-record` |
| 錄製原始連拍 / 建立影片 | `chloros-cli lattice array-burst` / `array-build-video` |

完整的標誌表、智慧自動曝光（smart-AE）穩態擷取選項（`--smart`），以及恆定速率模式，皆詳載於 [CLI

參考指南 § 擷取模式、錄影機與離線重新處理](../reference/cli-reference.md#capture-modes-recorders--offline-reprocess)。
