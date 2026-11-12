# Chloros 中的「DAQ」分頁

「DAQ」分頁——在 Chloros 側邊欄中標示為 **光感測器**——是 [DAQ-U、DAQ-M 及 DAQ-E 光感測器](README.md) 的即時控制介面： 可透過任何傳輸協定連接感測器、即時檢視校準光譜、從感測器配對中計算即時反射率，並將 `.daq` 檔案直接記錄至您的專案中。

當 Chloros 後端完成啟動後，此分頁即會啟用。分頁中的圖表資料由 Chloros 的 DAQ 服務透過即時連線提供；若連線中斷，系統會自動重新連線（2–10 秒重試間隔）；當服務無法連線時，感測器的「狀態」列會顯示 **無伺服器**。

佈局包含一個 **感測器側邊欄**（每個已連線的感測器佔一行）以及一個**圖表區域**（每個感測器或群組對應一個圖表方塊）。

<!-- SCREENSHOT-NEEDED: full DAQ (Light Sensors) tab in list view with one DAQ-E connected — sensor sidebar on the left (Connect Sensor + Record All buttons, one sensor row), spectrum chart with rainbow fill in the main area, live data table below the chart -->

***

## 連接感測器

點擊側邊欄頂端的 **連接感測器**。連接對話方塊將在主區域開啟（若正在新增其他感測器，則以覆蓋視窗形式顯示——此時會出現「取消」按鈕）。

| 控制項 | 行為 |
| --- | --- |
| **裝置類型** | `DAQ-U (USB)`（預設）、`DAQ-M (Bluetooth)` 或 `DAQ-E (Ethernet)`。切換後將重新掃描新選定的傳輸協定。 |
| **埠 / BLE 裝置 / 主機名稱 / IP** | 將偵測到的裝置列為 `device - description`；系統會自動選取第一個被識別為感測器的項目。 掃描期間會顯示 `Scanning...`（USB）、帶有 8 秒倒數計時的 `Scanning (N)...`（BLE），或帶有 5 秒倒數計時的 `Discovering ethernet sensors (N)...` （乙太網路）。若無結果，則顯示 `No ports` / `No BLE devices` / `No ethernet sensors found`。 |
| **↻ 重新整理** | 立即重新掃描所選傳輸協定（在 BLE／乙太網路掃描進行中時此功能將停用）。 |
| **連線感測器** | 選取裝置後即啟用；連線建立期間標籤將變更為 `Connecting...`。 |

探索功能僅在 **「連線」對話方塊顯示於螢幕上時** 執行，且僅針對選定的傳輸協定每 15 秒重複一次 — 僅開啟分頁並不會觸發掃描。若連線失敗，對話方塊將顯示：*「連線失敗。請嘗試拔除並重新插入感測器，然後再次點擊『連線』。」*

當第一個感測器連線成功時，側邊欄會自動彈出。

{% hint style="info" %}
**DAQ-E 未顯示？**DAQ-E 沒有狀態指示燈 — 請檢查其連接的交換機或注入器埠上的 PoE/鏈路指示燈，並在通電後等待數秒讓其完成開機。Chloros 機器必須位於同一廣播域中（mDNS 無法穿越路由器）。 安裝程式中的**Chloros後端（UDP-In）** 防火牆規則會允許 mDNS（UDP 5353）、DAQ-E 資料（UDP 5002）及 PTP（UDP 319/320）通過； 若使用第三方防火牆，則需開放相同埠號。同一局域網（LAN）上的兩台 DAQ-E 裝置將被分別偵測到，每台皆擁有各自的 `daq-e-<id>.local` 主機名稱。
{% endhint %}

<figure><img src="../.gitbook/assets/v120-daq-device-type.png" alt=""><figcaption>「裝置類型」提供 DAQ-U（USB）、DAQ-M（藍牙）及 DAQ-E（乙太網路）</figcaption></figure>

***

## 感測器側邊欄

每個已連線的感測器都會佔有一行（此外，每個「環境+物件」群組還會佔用一行）。行可透過拖曳重新排序，其順序也會重新排列圖表區塊。點擊一行可將該感測器／群組設為清單檢視中的活躍圖表。

| 元素 | 含義 |
| --- | --- |
| 彩色左邊框 | 該感測器的圖表顏色。 |
| 傳輸標誌 | `DAQ-U` / `DAQ-M` / `DAQ-E`，若為 Ambient+Object 反射率群組，則顯示綠色 `REF` 標章。 |
| 裝置名稱 | 預設為感測器的序號（用於校準、`.daq` 檔案名稱及匯入比對的穩定識別碼）；自訂名稱會依專案保留。 |
| **已校準** 標籤（綠色） | 當感測器的原廠校準套件已載入時顯示，即光譜值為真實的 W/m²/nm。 |
| **有更新可用** 標籤（琥珀色，僅限 DAQ-E） | 當前執行的韌體版本早於此 Chloros 版本所附帶的映像檔。更新期間會顯示即時進度（`Flashing… N%`、 `Restarting sensor…`，接著是 `Updated X → Y` 或 `Failed`）。 |
| 眼睛圖示 | 切換此感測器在圖表中的顯示狀態。 |
| 齒輪 | 開啟各感測器的設定模態視窗（如下）。 |
| ✕（紅色） | 斷開感測器連接，或移除「環境+物件」群組。 |

各行上方有兩個按鈕：

* **連接感測器** — 開啟連接對話方塊 （處理中時會重新標示為 `Connecting...`）。
* **全部錄製 / 全部停止**— 啟動或停止**所有**已連接感測器的 `.daq` 錄製。 需至少一個感測器**以及一個開啟的專案**（工具提示：「開啟專案以進行記錄」）；當任何記錄正在執行時，此按鈕會轉為紅色。

空狀態下顯示「無感測器連接」。

<!-- SCREENSHOT-NEEDED: sensor sidebar with three rows — a DAQ-E showing both the green Calibrated pill and the amber Update Available pill, a DAQ-U row, and a green REF group row — plus the Connect Sensor and Record All buttons -->

***

## 單一感測器設定（齒輪模態視窗）

點擊感測器行上的齒輪圖示即可開啟。內容依序為：

* **資訊行** — 裝置類型 (DAQ-U/M/E)、連線方式 (`Serial (USB)` / `Bluetooth` / `Ethernet`）、連接埠（COM 埠、BLE 位址或主機）以及序列號。
* **校準報告：下載** — 擷取該裝置的 NIST 可追溯校準證書（PDF），並在您的 PDF 檢視器中開啟。 需先知曉序列號方可使用；首次連線時會將證書快取。
* **裝置名稱** — 點擊鉛筆圖示可重新命名；設定將依專案保留。
* **圖表線條顏色** — 顏色色塊；設定將依專案保留。
* **積分時間 (ms)**— 滑桿 + 數值，**1–500 ms**，預設**32 ms**。當 AE 開啟時此選項將被停用。
* **幀平均值**— 滑桿 + 數值，**1–50 幀**，預設**20**。
* **AE：開啟／關閉**— 自動曝光切換開關；連線時**預設為開啟**。若要手動設定積分時間，請將其關閉。
* **停止串流／開始串流** — 暫停或恢復即時串流。
* **錄製／停止錄製** — 針對每個感測器的 `.daq` 錄製（需開啟專案）。
* **Cap** — 曝光上限校正設定檔（詳見下一節）。
* **即時資訊列** — 積分時間（毫秒）、FPS、樣本數、錄製狀態（紅色 `REC` 或 `Off`），以及狀態（`Streaming` / `Paused` / `SATURATED` / `No Server`）。

### 僅限 DAQ-E：網路、韌體及 PTP 資料列

* **主機名稱 / IP** — 裝置的當前位址。
* **韌體** — 即時韌體版本，以及一個動作儲存格： 當此 Chloros 版本包含較新的 DAQ-E 韌體映像檔時，會出現 **更新至 \<version\>** 按鈕。更新過程將透過網路在約 30 秒內完成；感測器會自動重新啟動並重新連線，若傳輸過程中斷，當前韌體將保持不變。 進度值會即時更新（`Flashing… N%` → `Restarting sensor…` → `Updated X → Y`），當值為最新時，該欄位將顯示 `Up to date`。
* **PTP 同步** — 即時 PTP 狀態（回退至 `unknown`）。DAQ-E 韌體 v1.2.0 及以上版本以「僅從屬時鐘」身分參與 IEEE 1588 PTPv2； Chloros 主機的後端為PTP總主時鐘，局域網（LAN）上的每台DAQ-E及LATTICE攝影機均在第0域中作為其從屬裝置，將時間戳記誤差控制在約1毫秒以內。
* **網路**— 裝置的運作位址及其取得方式（DHCP、鏈路本地或靜態），以及重啟後仍會套用的儲存設定。**變更…**會開啟一個表單：**自動 (DHCP)**或**靜態 IP**，其中包含 IP 位址、子網遮罩以及可選的閘道。**套用並重新啟動** 會儲存設定、重新啟動感測器並重新連線。 若設定的靜態 IP 位址位於本電腦網路範圍之外，系統將拒絕該設定，因為 Chloros 在重新啟動後將無法連線至感測器；此時會出現一個確認方塊，供需重新分配位址或將感測器路由至該網路的電腦選擇「仍允許」此設定。**開機即串流**設定是否讓感測器在通電後立即開始串流，且在任何軟體連線之前即開始運作。**重新啟動感測器** 會重新啟動感測器，並以其當前設定重新連線。當感測器正在記錄時，系統將拒絕重新啟動。此功能為 1.3.0 版本新增。

對於「環境+物件」群組，齒輪模態視窗僅顯示該群組的來源感測器、裝置名稱及圖表線條顏色。

<!-- SCREENSHOT-NEEDED: per-sensor settings modal for a DAQ-E — info rows, Calibration Report Download, Hostname/IP + Firmware row with an "Update to <ver>" button, PTP Sync row, Integration Time / Frame Average sliders, AE ON toggle, and the Cap dropdown all visible (scrolled composite acceptable) -->



### 蓋帽選擇

**蓋帽** 下拉選單用於告知 Chloros 感測器擴散片上安裝的是哪種實體蓋帽，並將該蓋帽的出廠測量校正曲線套用至每個光譜。選項取決於型號：

| 型號 | 蓋帽選項 |
| --- | --- |
| DAQ-U | 無（裸感測器）、視場角 15°、視場角 30°、視場角 45°、視場角 60°、視場角 90°、Sunshine（餘弦校正器） |
| DAQ-M | 無（裸感測器）、Sunshine（餘弦校正器） |
| DAQ-E | 無（裸感測器）、15° 視場角、45° 視場角、90° 視場角、Sunshine（餘弦校正器） |

**所有型號的預設設定皆為「陽光模式」（餘弦校正器）** — MAPIR 出貨的每台 DAQ 均已安裝「陽光」蓋，此為標準戶外配置： 180° 半球視野，在 60° 以內餘弦誤差 ≤ ±4%，在 70° 以內 ≤ ±4.5%（不建議在太陽高度角低於 ~15° 時使用），設計上具有衰減效果 (~12×)。您的選項將保留於專案中。

{% hint style="warning" %}
**蓋板選項必須與實際安裝的蓋板相符。**無論是感測器還是軟體，均無法偵測已安裝的蓋板類型。此選項將同時影響即時校正及寫入每個 `.daq` 檔案的標記——由於「Sunshine」蓋板具有約 12 倍的衰減係數， 若未申報更換蓋子，光譜校正值將因此產生約同等倍數的誤差。（拆下並重新安裝同一顆蓋子，誤差約為 1.5%。）僅當蓋子已實際移除時，才應選擇**無（裸感測器）**； 在 DAQ-E 上，「無」仍會套用其內嵌式玻璃擴散片出廠幾何輪廓 —— 這並非無操作 —— 而裸露的 DAQ-E 屬於實驗台配置，並非受支援的現場配置。
{% endhint %}

{% hint style="info" %}
從早期手冊升級說明：1.1.0 版本中瀏覽器端的「陽光擴散片已安裝」切換開關已移除。蓋子處理現改為此種依感測器設定的蓋子配置檔，並於伺服器端套用。
{% endhint %}

***

## 圖表區域

頂部固定欄包含 **清單 ⇄ 網格檢視切換鈕**以及**圖表縮放** 滑桿（磁磚大小 200–2000 像素）。當存在多個圖表群組時，檢視會自動切換為網格模式；若僅有一個或更少圖表群組，則會切回清單模式。 檢視模式與圖表大小會依專案保留設定。

每個感測器的 **光譜圖表** 顯示：

* **X 軸** — 波長 (nm)。 感測器網格範圍為 340–1010 nm，間隔 5 nm（共 135 個點），顯示時會插值至 1 nm。
* **Y 軸**— 針對已校正的感測器（綠色**已校正**標籤），**功率 (W/m²/nm)**— 基於所有三種傳輸方式的輻量校正光譜輻照度 — 並根據峰值自動選取 SI 前綴 (m/µ/n)。若未載入校準套件，感測器將以**強度 (計數)** 形式繪製原始讀數，且未經縮放。 當圖表同時疊加這兩種數據時，功率顯示於左軸，計數則顯示於右軸。
* 單一曲線下方會顯示彩虹色譜填充；同一張圖表上疊加的多個感測器則以彩色線條顯示，填充色會變暗。
* **懸停**— 垂直游標會顯示波長及各感測器的數值（已校準感測器為 W/m²/nm，否則為總計數）；**拖曳**可進行縮放（縮放期間會出現縮放按鈕）。
* **+** 按鈕（僅限網格檢視），用於將感測器新增至此圖表或建立群組（詳見下文）。
* 裝置名稱居中顯示於頂部，並在收到第一幀資料前顯示旋轉圖示。

**飽和**狀態不會直接標示在圖表上：飽和的感測器會在即時資料表中顯示紅色 `SATURATED` 狀態文字，並呈現紅色 `Saturated: Yes` 行。請降低積分時間或重新啟用自動曝光 (AE) 以解除此狀態。

<!-- SCREENSHOT-NEEDED: grid view with at least two chart tiles visible, the Chart Zoom slider and list/grid toggle in the top bar, and the "+" add-sensor button visible on one tile -->



***

## 即時資料表（清單檢視）

位於圖表下方的清單檢視，每 500 毫秒刷新一次：

* **所有型號**：光色色塊（sRGB 源自 CIE XYZ）、飽和狀態（是/否）、CIE 1931 X/Y/Z、色度 x/y、CIE u′/v′、CCT（K）、 顯色指數 (Ra)、主波長 (nm)、峰值波長 (nm)、激發純度、Duv、CIE L\*/a\*/b\* 以及孟塞爾 H/V/C。
* **僅限經校準的感測器**（DAQ-U / DAQ-M / DAQ-E 中的任一款，只要已載入其出廠校準套件——感測器行上的綠色**已校準** 標章即為識別標記）：總功率（W/m²）、明視勒克斯（lx）、暗視勒克斯（lx）、 S/P 比、PPFD 以及 PPFD Red /Green /Blue（µmol/m²/s），以及各類視錐細胞輻照度 — S-錐、黑色素視錐、視紫質視錐、M-錐、L-錐（單位皆為 W/m²）。

<!-- SCREENSHOT-NEEDED: list view live data table for a DAQ-E showing both the colorimetric rows and the power-calibrated rows (Total Power, Photopic/Scotopic Lux, PPFD, opic irradiances) -->

***

## 反射率群組 （環境光 + 物體）

兩個相連的感測器可組合成即時反射率顯示——無需使用相機：

1. 在網格視圖中，點擊圖表方塊上的 **+**，並選擇**組合環境光 + 物體**。
2. 選擇一個 **環境光源**感測器與一個**物體掃描器**感測器（兩者必須不同），然後點擊**建立**。

Chloros 會根據這兩組即時數據流，針對每個波長計算 R(λ) = 物體(λ) / 環境(λ)（當環境值 ≤ 0 時，結果為 0）。 該群組的標籤取決於感測器的校準類別：

* 兩個感測器均已校準（已載入校準包）→ **「表觀反射率」**。
* 任一感測器未校準 → **「相對反射率」**。

該群組將以綠色 `REF` 行顯示於側邊欄，並擁有獨立的圖表（彩虹色填充，懸停時顯示 4 位小數，可拖曳縮放）。

**+**選單亦提供**新增感測器** 功能，並有三種放置方式：*合併新感測器*（加入此圖表）、*將現有感測器移至此處*，或 *檢視新感測器*（獨立圖表）。

<!-- SCREENSHOT-NEEDED: the "+" add-sensor overlay open on a chart tile showing the menu (Add New Sensor / Combine Ambient + Object / Cancel), and the Ambient + Object sub-dialog with its two sensor selects -->



### 植被指數表

在清單檢視中，植被指數表位於反射率群組圖表下方，其數值是根據波段中心處的即時反射率計算得出：**藍色 450 / 綠色 550 / 紅色 670 / NIR 800 nm** 的即時反射率計算而成（數值精確至小數點後 4 位，若無法計算則顯示為 `---`；將游標懸停於指數名稱上可查看其全名）：

* **始終顯示**（尺度不變，適用於任何感測器組合）：NDVI、GNDVI、ENDVI、WDRVI、GRVI、 CVI、GCI、MSR。
* **僅當兩個感測器均已進行功率校正時**（兩個感測器套件均已載入）：EVI、SAVI、OSAVI、GSAVI、GOSAVI、MSAVI2、RDVI、TDVI、LAI、NLI、MNLI、FCI、GEMI。

<!-- SCREENSHOT-NEEDED: an Ambient+Object reflectance group in list view — reflectance chart labeled "Apparent Reflectance" with the vegetation index table below it showing live NDVI etc. -->

***

## 錄製 `.daq` 檔案

* 錄製需 **開啟專案** — 否則「全數錄製」（側邊欄）及各感測器的「錄製」按鈕均會停用。
* 檔案將寫入 **`<project folder>/light_sensor/`**；檔案名稱包含感測器 ID 和時間戳記，且裝置名稱會與錄製資料一併儲存。
* 當記錄停止時（按下「停止」、「全部停止」，或記錄中途斷線），完成的 `.daq` 檔案會 **自動新增至開啟中的專案** — 它會直接出現在專案的檔案清單中，無需手動新增，並可立即作為 [反射率處理](README.md) 的下行輻射數據使用。
* 錄製期間，設定模態視窗的即時資料列中會顯示一個紅色的 `REC` 指示符。

關於定量輻照度數值，請至少取 15 秒的資料進行平均計算 — 這是儀器的特性，並非缺陷。

<!-- SCREENSHOT-NEEDED: recording in progress — sidebar Stop All button in its red state and the settings modal live rows showing Recording: REC -->



***

## 多感測器佈局與專案持久性

* 將多個感測器合併至同一張圖表（共用軸線）、維持獨立圖表（自動網格佈局）、在圖表間移動感測器、拖曳重新排列行／磁磚，並透過眼睛圖示切換按鈕隱藏個別感測器。
* 針對每個專案，Chloros 會保留：裝置名稱、圖表顏色、圖表大小、檢視模式，以及每個感測器的設定（積分時間、幀平均、AE 狀態、封套選擇）。
* **重新開啟專案時，系統會根據位址自動重新連線其感測器** — DAQ-U 為 COM 埠、DAQ-M 為 BLE 裝置、 DAQ-E 的 mDNS 主機名稱（即使裝置的 IP 地址變更仍可解析）——並重新套用每個感測器已儲存的 cap 設定檔、幀平均、AE 狀態及手動積分時間。***

## 相機配對 (DLS)

無需進行配對。與無人機 DLS 工作流程（需預先將光感測器綁定至相機）不同，Chloros 是在後端將 DAQ 資料與影像進行配對：在匯入／處理時，`.daq` 的讀數會被插值至每個擷取畫面的曝光時間戳記。 使用任何已連接的感測器進行錄製（`.daq` 會自動加入專案中），反射率處理會根據時間找出正確的讀數——請參閱 [DAQ 光感測器](README.md) 以了解下行數據的運用方式。 對於 **即時**的 LATTICE 擷取，請改在「相機」分頁的**光感測器** 下拉選單中選擇感測器； 此時每次擷取都會儲存其使用的讀數，且僅與該次擷取所使用的相機配對（請參閱 [反射率工作流程](reflectance.md)）。
