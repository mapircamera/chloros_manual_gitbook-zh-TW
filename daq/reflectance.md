# 反射率工作流程

DAQ 光感測器會將輻量影像轉換為反射率。共有兩種截然不同的工作流程：

1. **單感測器** — 一台 DAQ 測量下行輻照度，同時相機進行拍攝，而 Chloros 會將相機測得的輻度除以該參考值。
2. **雙感測器** — 兩組 DAQ 感測器，一組監測天空，另一組監測物體，無需相機即可產生即時光譜反射率曲線。

## 單一感測器 + 相機（下行輻照度參考）

DAQ 充當下行輻照度感測器 (DLS)： 相機測量上行輻照度 **L**（W/m²/sr/nm），DAQ 測量下行輻照度**E**（W/m²/nm），Chloros 則依下列公式計算各波段的反射率：

> ρ = π · L / E

DAQ 的讀數始終與 **曝光時間戳相匹配** —— 這就是為什麼 DAQ 和相機共享一個受 PTP 規範的時鐘（參見 [DAQ-E 網路與時間同步](ethernet-ptp.md)）。 進行戶外作業時，請安裝 Sunshine 餘弦遮光罩並正確宣告其參數；遮光罩的宣告會直接影響 E 的量值（參見 [遮光罩輪廓與校準範圍](caps-and-range.md)）。 進行定量分析時，請謹記儀器特性：定量輻照度須基於至少 15 秒的讀數平均值。

### 即時擷取

在「相機」分頁中將 DAQ 與相機綁定：每台相機的設定面板皆設有 **光感測器** 下拉選單，其中列出「光感測器」分頁中所有已連接的 DAQ（DAQ-U/M/E）；若為同步陣列，全陣列範圍的光感測器選項將套用至每個成員 （個別相機仍可覆寫設定）。綁定後，感測器的光譜會傳入相機的 DLS 插槽，而反射率輸出值則會除以對應的讀數。 在陣列中，擁有獨立感測器的成員會使用自身的讀數——包括即時合成影像中的波段、匯出資料以及自動曝光設定——而其他成員則使用陣列的設定。

<!-- SCREENSHOT-NEEDED: Cameras tab per-camera settings panel showing the "Light Sensor" dropdown open, with a connected DAQ sensor listed and selected. -->


有兩點行為值得注意：

* **未綁定 DAQ → 反射率將被拒絕，而非偽造。** Chloros 會拒絕該反射率結果並記錄跳過原因，而非默默回傳較差的結果。
* **所使用的讀數將被保留。** 對於每個反射率幀，實際套用的 DAQ 讀數會以 `.daq` 檔案的形式寫入擷取檔的 `daq/` 資料夾中，以便日後重新處理該擷取檔（[錄製與 .daq 格式](recording.md))。其旁的 `.daq.cameras.json` 檔案會標示其所描述的相機名稱，而重新處理時僅會將該檔案與這些相機的影像進行配對。

### 處理記錄的影像

進行飛行後處理時，請在作業期間錄製一個 `.daq` 檔案，並將其與影像一同保存——處理流程會自動解析時間戳記匹配的下行影像，並在首次使用時從 MAPIR 的雲端取得任何缺失的工廠校準資料。GUI 錄製檔案在停止時會自動新增至開啟中的專案中。

反射率參考值可在處理時選擇 — 例如在 `chloros-cli process` 上使用 `--reflectance-source`，或透過 GUI 的「專案設定」中的「反射率來源」設定：

| 值 | 行為 |
| --- | --- |
| `auto`（預設） | 通過品質保證（QA）的幀內校準目標為絕對參考；若未通過，則以 DAQ 下行波（ρ = π·L/E）作為備用參考。 若校準目標雖通過品質保證（QA）但帶有標記（例如面板脫落或波段異常），系統不會自動優先選用該目標：系統會依其在保留面板上測得的評分順序，依次嘗試「目標」、「目標+DAQ」及「僅DAQ」這三種鏈路。 |
| `daq` | DAQ 具有最終決定權 |
| `target` | 嚴格的幀內標靶；不允許 DAQ 替代 |

請參閱 [校準標靶](../calibration-targets.md) 以了解標靶工作流程，以及 [LATTICE 章節](../lattice/README.md) 以及 [CLI 參考手冊](../reference/cli-reference.md) 以了解完整的處理流程。讀取匯出的反射率像素時，請使用標記的尺度 （LATTICE：32768 = ρ 1.0，XMP `Chloros:PixelScale`；Survey3：65535）——請參閱 [輸出影像格式](../output-image-formats.md)。

### 超出 DAQ 校準範圍的波段

DAQ 的輻測校準範圍約為 374–974 nm。若任何相機波段在該範圍內的譜權重低於一半，Chloros 將拒絕基於 DAQ 的反射率數據，並回報跳過原因 `dls-uncalibrated-band-<nm>`。 在現有產品型號中，此情況僅影響 F988：F988 的反射率是透過場景內反射率面板進行校準的；由於該波段超出 DAQ 光感測器的校準範圍，因此 Chloros 會採用您最近一次的面板擷取數據，並在兩次面板觀測之間維持該值。 若 F988 相機僅以 DAQ 模式運行，Chloros 將拒絕該波段基於 DAQ 的反射率，並回報跳過原因 `dls-uncalibrated-band-988` —— 此時應採用反射板工作流程。

## 雙感測器（環境光 + 物體）

兩組 DAQ 感測器（任意組合，跨任何傳輸介質）即可在無需相機的情況下提供即時反射率光譜：其中一組感測器朝向天空（**環境光源**），另一組朝向被測物（**物體掃描器**），Chloros 會根據波長進行計算：

> R(λ) = 物體(λ) / 環境(λ)

（當環境光 ≤ 0 時，此值為零）。

### 在 GUI 中

在「光感測器」分頁中連接兩組感測器後，開啟新增感測器覆蓋層（網格檢視中圖表磁磚上的「+」按鈕），並選擇 **結合環境光 + 物體**。 在「環境光源」和「物體掃描器」下拉選單中選擇兩個感測器，然後點擊「建立」。該群組將以獨立圖表的形式顯示，並在側邊欄中以帶有綠色**REF** 標籤的行呈現。

<!-- SCREENSHOT-NEEDED: The add-sensor overlay's "Combine Ambient + Object" panel with two connected DAQ sensors selected in the Ambient Light Source and Object Scanner dropdowns, Create button enabled. -->

<!-- SCREENSHOT-NEEDED: A live Apparent Reflectance chart from an Ambient+Object DAQ pair in list view, with the vegetation-index table visible below the chart (NDVI etc. showing live values). -->


在反射率圖表下方（清單檢視），一個即時 **植被指數表** 會根據藍光 450 / 綠光 550 / 紅光 670 / NIR 800 nm 的波段中心，從曲線中計算出各項指數。基於比率的指數（NDVI、GNDVI、ENDVI、WDRVI、GRVI、 CVI、GCI、MSR）的比值型指數會始終顯示；而需要絕對反射率的指數（EVI、SAVI、OSAVI、GSAVI、GOSAVI、MSAVI2、RDVI、TDVI、LAI、NLI、MNLI、FCI、GEMI）則僅在兩個感測器皆為經功率校正的型號時才會顯示。

### 表觀與相對 — 標籤規則

Chloros 根據感測器對實際能達成的標準來標示雙感測器輸出：

| 感測器對 | 標籤 |
| --- | --- |
| 兩感測器均經校正 — 已載入工廠校正套件 | **表觀反射率** |
| 任一感測器未經校準 | **相對反射率** |

這三種模型皆為輻測型：一旦載入感測器的出廠校準套件，其光譜即為絕對值 W/m²/nm，因此一對經校準的感測器所產生的比值將對應於絕對的表觀反射率——此值並非由傳輸函數決定。 若其中一個感測器仍在傳輸原始計數（無法取得校準套件），則結果將降級為相對曲線（光譜形狀仍有效）。兩個感測器均應具備正確宣告的上限值（[上限值配置檔與校準範圍](caps-and-range.md)）。

### 透過 Python


在整合式 SDK 介面中並無專用的雙感測器呼叫函式：請使用 `chloros_sdk.connect_daq_sensor()` 開啟兩個工作階段，並自行將其光譜與 `latest()` 進行比對，同時採用相同的標記規範。 （MAPIR 內部直接硬體介面中亦存在雙感測器記錄工具，為求完整性，已列於 [CLI 參考手冊](../reference/cli-reference.md) 中——但該工具並非隨附 CLI 的一部分；上述 GUI 工作流程才是受支援的正式操作路徑。）
