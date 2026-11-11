---
metaLinks: {}
---

# 開始使用

<div data-full-width="false"><figure><img src=".gitbook/assets/chloros_logo_transparent.png" alt=""><figcaption></figcaption></figure></div>

Chloros
是由 [MAPIR
](https://www.mapir.camera) 開發的一款軟體應用程式，用於處理多光譜影像、即時控制 MAPIR
硬體，以及記錄感測器資料。Chloros
1.3.0 支援完整的 MAPIR
產品系列：

* **Survey3
相機** — 將 RAW+JPG 影像處理成經校正的反射率與植被指數圖。請參閱 [支援的相機](supported-cameras.md)。
* **LATTICE 相機** — 即時連接 GigE 多光譜相機模組，可單獨使用或作為同步的多相機陣列：預覽、擷取並處理成經校正的輻照度與反射率產品。請參閱 [LATTICE 專區](lattice/README.md)。
* **DAQ 光感測器** — DAQ-U（USB）、DAQ-M（藍牙）以及 DAQ-E 和 DAQ-E-S（乙太網路）光譜感測器：即時校正光譜、 `.daq` 記錄，以及用於反射率處理的向下輻射照度。請參閱 [DAQ 章節](daq/README.md)。

{% hint style="success" %}
**Chloros
1.3.0 新增功能**：AI 助理可透過 `chloros-mcp` 驅動 Chloros
，而 CLI
、SDK
及 `chloros-mcp` 亦提供相同功能。 LATTICE 索引產品已獲得改進，完整感測器陣列運作速度更快且適用於任何網路，後端會在指令之間持續運行。請參閱下方的「新功能」清單，並[下載](download.md)安裝程式。
{% endhint %}

{% hint style="info" %}
**正在使用 AI 助理操作 Chloros
嗎？** 本手冊正是為此而設計。請讓您的助理查詢：

* `https://mapir.gitbook.io/chloros/llms.txt` — 所有頁面的機器可讀索引。
* 任何頁面的原始 Markdown 格式 — 在其 URL
後方附加 `.md`（例如：`https://mapir.gitbook.io/chloros/reference/cli-reference.md`）。
* [CLI
參考手冊](reference/cli-reference.md) 與 [SDK
參考手冊](reference/sdk-reference.md) — 專為大型語言模型 (LLM) 設計的完整、精確值參考頁面。

提示語範例：*「閱讀 https://mapir.gitbook.io/chloros/reference/cli-reference.md,，然後撰寫一個腳本，用以登入並將資料夾 ~/flights/flight_001 處理成反射率 + NDVI
GeoTIFF 檔案。」*

或者讓它直接執行 Chloros
：`chloros-mcp` 是一款隨 SDK
安裝的 MCP 伺服器，可與 Claude Desktop、Claude Code、Cursor 及其他 MCP 客戶端配合使用。

完整指南：[如何透過 AI 助理使用 Chloros
](ai-assistants.md)。
{% endhint %}

***

## Chloros
1.3.0 的新功能

* **AI 助理可驅動 Chloros
。** `chloros-mcp` 是一台已安裝 Python
SDK
的 MCP 伺服器，可讓 Claude Desktop、Claude Code、Cursor 及其他 MCP 客戶端處理影像，並運作 LATTICE 相機、陣列及 DAQ 光感測器。請參閱 [如何搭配 AI 助理使用 Chloros
](ai-assistants.md)。
* **CLI
、SDK
和 `chloros-mcp` 提供相同的功能。** 基於安全性考量，僅有登入、DAQ-E 配置以及直接 DAQ 伺服器功能仍保留在 CLI
中。 部分 1.2.0 版本的腳本需要進行修改：請參閱 [CLI
參考手冊](reference/cli-reference.md) 及 [SDK
參考手冊](reference/sdk-reference.md) 頂端的 **

1.3.0 版本變更事項**。
* **改進 LATTICE 指數產品。** 無論在何種處理模式下，從 LATTICE 擷取資料中匯出的 NDVI
及其他指數產品均已獲得改進。
* **Windows
安裝程式選項。**在**系統設定**下，安裝程式提供**啟用 Windows
長路徑**及**將 Chloros
後端排除於 Microsoft Defender 掃描範圍外** 兩項選項，兩者預設皆為勾選狀態。 前者允許專案資料夾路徑長度超過 260 個字元；後者則可防止即時掃描導致攝影機擷取速度變慢（在高擷取率下，掃描會使擷取率降低約 30%）。此外，安裝程式現僅會將CLI
資料夾新增至 `PATH` 一次，並移除先前安裝程式留下的重複項目。請參閱 [下載](download.md)。
* **過深的專案資料夾將被直接拒絕。** 當 Windows
長路徑功能關閉時（此為 Windows
預設設定），若項目資料夾深度超過 260 字元的路徑限制，系統會在執行任何操作前直接拒絕該資料夾，並顯示符合限制的最長項目資料夾名稱，以及如何啟用長路徑功能。先前，此類執行情況下將不會產生任何輸出。 啟用長路徑後，位於極深資料夾中的產出檔現在能保留其元資料（GPS 和 XMP 標籤）。
* **遮罩限制。** 當 LATTICE 輻度或反射率產品的波段分離產生負數波段值時，該值將被截斷為零，並在產品資料夾旁的 `clamp_masks/` 資料夾中，以每像素遮罩記錄受影響的像素與波段。 產品會在 `Chloros:UnmixClampMask` 標籤中為其遮罩命名。即時預覽現在會與匯出結果完全一致地進行裁剪。請參閱 [輸出影像格式](output-image-formats.md)。
* **產品會自行描述其特性。** 索引產品會攜帶自身的處理標籤（`Chloros:ProcessingLevel=index`、索引表達式，以及是否由輻照度或反射率計算而來），而非來源原始檔的標籤。Survey3
產品在每種處理模式下都會攜帶 Chloros
處理標籤。 LATTICE 的原始檔和去拜耳化檔案會保留感測器的原始方向，並將其方向記錄在 XMP 中；影像檢視器僅在顯示時才會旋轉這些檔案。
* **Linux
套件以未解壓縮狀態發行。** 後端與 CLI
會安裝在 `/usr/lib/chloros/` 下的資料夾中，因此不再於每次啟動時自動解壓縮。 在 x86_64 平台上，後端啟動時間不再耗時約 100 秒，且 `.deb` 的檔案大小約為原先的一半。 隨附的 ExifTool 現已可在乾淨安裝的 Ubuntu 系統上運行。`.deb` 現亦會為系統安裝 Python
SDK
（透過 `python3` 安裝）， （請透過 `sudo apt install ./chloros_1.3.0_<arch>.deb` 進行安裝），並會增加相機串流所需的內核套接字接收緩衝區，同時新增一個以您的使用者身分執行的後端服務。 Jetson 套件 `chloros_1.3.0_arm64_jp6.deb` 是針對 JetPack 6 編譯的，並使用 JetPack 的 CUDA 版本 PyTorch 在 Jetson 的 GPU 上進行處理。 在 Linux
系統上，每個匯出的影像現在僅需一次寫入磁碟（先在記憶體中寫入並標記，再複製至目標位置）：在 Jetson 平台上（其運算速度取決於磁碟），執行 288 張影像的 LATTICE 任務僅需 14 分鐘，而非原先的 24 分鐘。請參閱 [Linux
安裝指南](linux/linux-installation.md)。
* **LATTICE OCN
相機在陽光下的讀取精度更高。** 經過波段解混後，輻照度、反射率及光學指數會進行快速校正，因此灰階目標在陽光下測得的 NDVI
值接近中性。RGN
相機則維持不變。
* **應用程式內錯誤回報。** 「除錯日誌」分頁頂端的錯誤圖示可將錯誤描述、最多 10 張影像、日誌及系統詳細資訊傳送至 MAPIR
。請參閱 [GUI：導覽](navigation.md)。
* **LATTICE 陣列。** 陣列預設會保留完整的感測器資料，而採用 12 位元壓縮傳輸（`--packed`，預設為 `auto`）可將完整感測器的幀率提升約三分之一； 陣列可在任何網路環境下，以該網路允許的幀率運作（低速連線會降低幀率，而非拒絕陣列或縮減其幀大小）；若陣列中的攝影機發生電源重啟，系統會自動恢復運作並重新啟用 PTP；在光線昏暗時，自動曝光不再降低該幀率（系統會先填滿週期，再提高增益）； 連續擷取功能可跟上高速陣列的節奏；RGB
攝影機新增 **高光** 控制功能；以及 SDK
的 `ArraySession` 新增了 `grab()`、 `frame_stream()` 及 `continuous()`。請參閱 [LATTICE 章節](lattice/README.md)。
* **更穩定的自動曝光。** 在光線恆定的情況下，曝光值會保持不變；光線變化後，曝光值會在數秒內穩定下來；微小的亮部不再使畫面變暗；手動曝光時增益仍維持在「自動」模式；而在強光下停止回應的相機，會根據光線感測器讀取的數值（若未連接光線感測器，則根據相機自身的影像幀）來維持其曝光水準。 請參閱 [相機設定](lattice/camera-settings.md)。
* **光感測器。** **DAQ-E-S 支援：** 其固定式漫射器無需蓋帽校正，且每次讀數皆包含裝置的傾斜、滾動與俯仰角度。 錄製檔會標示其蓋板是經選擇或預設的，且處理過程會針對預設蓋板發出警告：DAQ-U、DAQ-M 和 DAQ-E 出廠時未附蓋板，因此請為已安裝的蓋板命名。 `.daq` 的即時記錄可在斷電後存續（請保留 `.daq-journal` 檔案及其對應的 `.daq`）， 若 DAQ-E 停止傳輸，重新連接時其記錄中會標記中斷點。 若專案僅包含光感測器記錄，系統會透過應用程式和CLI
列出並處理這些記錄；使用 `chloros_scripts` 所做的記錄，會以該 DAQ 為來源列出，並附有可自訂的標籤。每次擷取都會儲存所使用的 DAQ 讀數。請參閱 [DAQ 章節](daq/README.md)。
* **處理。**LATTICE 幀會在目標偵測期間開始處理。目標與 DAQ 參考值會依頻段分別決定。移動專案時，系統會重新連結其檔案。8 位元擷取的輸出為 8 位元，採用新的**TIFF
（8 位元）** 格式。 未使用校準目標的 Survey3
執行結果將以備用產品（`Vignette_Corrected_Images` 或 `Sensor_Response_Images`）形式儲存，而非反射率影像。反射率影像的各波段 DLS 標籤始終屬於該幀影像本身。 LATTICE 擷取不支援「Texture Aware」功能，此類擷取始終採用標準去拜爾演算法。
* **CLI
與 SDK
。** 相對輸出路徑現會指向您執行命令或腳本的資料夾（過去已安裝的後端會嘗試在自身安裝資料夾中解析路徑，導致失敗）。 後端會在命令與腳本之間持續運行，並在閒置 30 分鐘後自動停止；此功能由 `chloros-cli backend status` 和 `backend stop` 負責管理。 擷取指令會列出所有無法寫入的檔案名稱，並以代碼 1 退出；若磁碟空間已滿，重複執行的擷取作業將停止。 `process` 執行成功時，會回報 `Image products written: N (plus M clamp masks)`。在 Windows
系統上，SDK
的 `ChlorosLocal` / `process_folder` 會尋找已安裝的後端（並使用已正在運行的後端），而 `read_image_audit_tags` 則可直接從標準的 `pip install` 執行。

***

## Chloros
1.2.0 的新功能

* **即時攝影機控制 — 新增「攝影機」分頁。** 可單一連接 LATTICE 攝影機，或作為同步的多攝影機陣列（PTP 時間同步、硬體觸發擷取），並具備即時預覽疊加、各波段直方圖、智慧自動曝光、即時索引計算器，以及應用程式內攝影機韌體更新功能。
* **光感測器 — 新增「光感測器」分頁。** 可連接 DAQ-U（USB）、DAQ-M（藍牙）及 DAQ-E（乙太網路）感測器； 檢視即時校準光譜（W/m²/nm）、將 `.daq` 檔案記錄至您的專案中、選擇蓋帽校正配置檔，並透過網路更新 DAQ-E 韌體。
* **擷取模式與記錄器。** 單次／連續／間隔擷取，以及僅限原始資料的「最快擷取」模式；可針對每個專案選擇「擷取全部」功能所產出的相機型號與匯出類型；陣列記錄器可提供監控級索引影片，以及具備離線影片建構功能的分析級原始快拍。
* **LATTICE 處理流程。** 匯入 LATTICE 擷取資料夾，並將每個原始幀分別處理為去拜耳化、預覽、float32 輻照度 (W/m²/sr/nm) 及反射率產出，並可針對各產出類型進行切換。 反射率可來自畫面內的校準標靶或數據擷取（DAQ）的下行輻射；匯出時會套用陣列對齊；若缺少工廠校準資料，系統會根據相機序列號自動下載。
* **專案會記住硬體設定。** 已連接的相機和光感測器會與專案一同儲存（`cameras.json` / `sensors.json`），當您重新開啟專案時，這些裝置會根據儲存的設定重新連線。 請參閱 [GUI：專案](projects.md)。
* **影像檢視器升級。** 提供游標像素/索引讀取功能，並具備正確的每檔反射率縮放、圖層直方圖、GSD 分箱滑桿、每觸發/每相機網格模式、LATTICE 產品檢視，以及將索引/LUT 沙盒資料匯出至磁碟的功能。
* **CLI
與 SDK
大幅擴充。** 新增 `lattice`、`daq pool-*`、`project` 及 `time-sync` 命令系列； 新增 `process` 選項（`--input-level`、單一產品切換開關、`--reflectance-source`、陣列對齊標誌）； SDK
智慧連接句柄（`connect_camera` / `connect_array` / `connect_daq_sensor`），可自動啟動後端； `open_project()` 自動化；SDK
套件已與安裝程式捆綁，並以 `chloros-sdk` 之名發佈至 PyPI。
* **誠實的失敗語義。** 若 `chloros-cli process` 執行時請求了產品但未寫入任何產品，現在會明確報錯並以非零狀態碼退出；成功執行的任務則會回報已寫入多少影像產品。
* **新的輸出佈局。** 產品會存放於 `<project>/<camera>/<format>/<Product>_Images/` 資料夾中，並保留原始檔案名稱 — 由資料夾名稱（而非檔案名稱後綴）來識別產品。請參閱 [輸出影像格式](output-image-formats.md)。
* **更多輸入選項、方案與語言。** 新增對 `.dng` 輸入格式的支援；38 種介面語言皆已完整上線；各方案設有硬體限制，免費（無需登入）版本最多可使用 4 台攝影機與 2 個光感測器。
* **可靠性。**「停止處理」功能會乾淨俐落地結束，並提供真實的執行摘要；多攝影機專案會匯出每台攝影機的資料；安裝程式升級時不再會將您登出。***

Chloros
提供 3 種應用程式介面：

## Chloros
：桌面圖形使用者介面 (GUI) 應用程式

獨立視窗，具備所有功能，包括「即時攝影機」與「光感測器」分頁。_僅限 Windows 系統。_

## [Chloros
CLI
：命令列介面](CLI.md)

支援命令列批次處理，並可執行即時 `lattice`、`daq pool-*`、`project` 及 `time-sync` 指令。 非常適合自動化、腳本編寫及無頭操作。支援 **Windows
、Linux
amd64 及 Linux
arm64 (NVIDIA Jetson)**。_使用 CLI
需訂閱付費的 Chloros
+ 方案。_

## [Chloros
API
：Python
SDK
](api-python-sdk.md)

用於自動化與自訂工作流程的 Python
程式介面：完整管線處理、即時攝影機／陣列工作階段、DAQ 感測器工作階段，以及儲存專案的自動化功能。 隨桌面版／CLI 套件一併安裝，亦以 `pip install chloros-sdk` 為名發布。_使用 API 需訂閱付費的 Chloros
+ 方案。_

***

## 支援的平台

| 平台 | GUI | CLI
| Python
SDK
|
| --- | --- | --- | --- |
| **Windows
10/11 (x64)** | 是 | 是 | 是 |
| **Linux
amd64 (x86_64)** | 否 | 是 | 是 |
| **Linux
arm64 (NVIDIA Jetson)** | 否 | 是 | 是 |

有關 Linux
的安裝說明，請參閱 [Linux
與邊緣運算](linux/linux-overview.md) 章節。

***

## 三個步驟開始使用

1. **安裝** — 下載並執行適用於您平台的安裝程式。請參閱 [下載](download.md)。
2. **登入（GUI 版本為選填）** — GUI 版本無需帳戶即可免費處理影像。透過 [Chloros
+ 登入](chloros+-login.md)，可解鎖平行處理、GPU 加速、更高的裝置限制，以及 CLI
／SDK
存取權限。
3. **建立您的第一個專案** — 開啟 Chloros
，建立 [新專案](projects.md)，[新增您的影像](processing-images-gui/adding-files-to-a-project.md)，並 [開始處理](processing-images-gui/starting-the-processing.md)。若要直接驅動實體硬體，請開啟「攝影機」或「光感測器」分頁 — 請參閱 [GUI：導覽](navigation.md)。

***

## Chloros
+

雖然 Chloros
對於大多數任務皆可免費使用，但您可能會發現自己需要更多功能。此時，購買 Chloros
+ 的付費授權將能為您帶來實質效益。透過 Chloros
+ 授權，您可以解鎖以下新功能：

* **多執行緒處理**：透過在處理管道中同時處理影像，大幅加快大型專案的影像處理速度。
* **GPU (CUDA) 加速**：善用當今更高規格的 GPU 記憶體選項，進一步加速影像處理管道。建議使用 4GB 或以上的 VRAM 以獲得最佳效果。
* **Chloros
+**[**CLI**](CLI.md) **存取權限**：透過命令列執行 Chloros
+，以實現自動化並整合至您自己的軟體中。適用於任何付費方案；於伺服器端強制執行。
* **Chloros
+**[**API**](api-python-sdk.md) **存取方式：** 透過 Python
執行 Chloros
+ 以進行程式化控制，實現與您的研究流程、資料分析工作流程及自訂應用程式的無縫整合。適用於任何付費方案；於伺服器端強制執行。
* **更高的硬體限制**：可同時連接更多攝影機與光感測器。若未登入，GUI 最多可連接 4 台攝影機和 2 個 DAQ 光感測器；付費方案將提高這兩項上限：

| 方案 | 攝影機 | DAQ 光感測器 |
| --- | --- | --- |
| 鐵級 (免費，無需登入) | 4 | 2 |
| 銅級 / 青銅級 | 6 | 3 |
| 銀級 | 10 | 6 |
| 金級 | 20 | 12 |

* **多裝置使用**：每份 Chloros
+ 授權可註冊 2 台以上的裝置。請使用您的 MAPIR
Cloud 帳戶來管理已註冊的裝置。透過升級您的 Chloros
+ 授權，即可增加對更多裝置的支援。
* **進階紋理感知去拜爾演算法：** 結合高品質邊緣感知去拜爾演算法與 AI/ML 降噪模型，可消除幾乎所有去拜爾過程產生的噪點。
* **自訂多光譜指數公式：** 在 Chloros
柵格計算器中輸入自訂多光譜指數，適用於影像處理及影像檢視沙盒。
* **Linux
與邊緣運算：** 可在 Linux
x86_64 和 ARM64 平台（包括 NVIDIA Jetson）上執行 Chloros
，以進行現場與邊緣處理。請參閱 [Linux
概覽](linux/linux-overview.md)。

<p align="center"><a href="https://cloud.mapir.camera/pricing" class="button primary" data-icon="envira">Chloros+ 定價與註冊</a></p>

<figure><img src=".gitbook/assets/plus_prog.JPG" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/chloros_grid_zoom.gif" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/chloros_grid_mode.gif" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/chloros_grid_meta.gif" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/chloros_map_markers.gif" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/cli.JPG" alt=""><figcaption></figcaption></figure>
<!-- SCREENSHOT-UPDATE: cli.JPG shows the 1.1.0 CLI banner. Re-shoot a terminal running `chloros-cli --version` + `chloros-cli status` on the 1.3.0 build so the banner prints "Chloros CLI 1.3.0". -->
