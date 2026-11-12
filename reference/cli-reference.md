# Chloros CLI 參考手冊

**版本：**

1.3.0**生成日期：**2026-07-29 19:19 ·**修訂日期：** 2026-11-06**目標讀者：** 針對大型語言模型（LLM）使用進行優化；同時兼具人類可讀性。**範圍：** `chloros-cli` 所有面向使用者的子指令，包含選項及可直接複製貼上的範例。

本文件是隨 MAPIR Chloros 附帶的 `chloros-cli` 命令列工具之完整參考手冊。本文件刻意編寫得極為詳盡，以便大型語言模型（或人類）無需檢視原始碼，即可根據以下清單組建任何受支援的工作流程。

若您僅需重點內容，請跳至：
- [五分鐘快速入門](#five-minute-quickstart)
- [LATTICE 相機首次連線工作流程](#lattice-camera-first-connect-workflow)
- [DAQ 感測器首次連線工作流程](#daq-sensor-first-connect-workflow)
- [Smart-AE / Smart-Capture](#smart-ae--smart-capture)
- [擷取模式、記錄器與離線重新處理](#capture-modes-recorders--offline-reprocess)

> **透過 AI 助理驅動 Chloros？** Claude Desktop、Claude Code、Cursor 及其他 MCP 客戶端可透過 `chloros-mcp` 執行處理、操控 LATTICE 相機與陣列，以及 DAQ 光感測器， 該 MCP 伺服器會隨 Python SDK 一併安裝（亦包含於 Linux 的 .deb 套件中，詳見 `/usr/bin`）：`claude mcp add chloros -- chloros-mcp`。相關設定、選項及工具請參閱 SDK 參考手冊中的 [MCP 伺服器（AI 助理）](sdk-reference.md#mcp-server-ai-assistants)中。具備 shell 功能的助手亦可直接執行本頁上的所有指令。

> **

1.3.0 版本的變更**（現有 1.2.0 腳本或工作流程會察覺到的變更）。
>
> **

1.2.0 腳本可能需要處理的變更：**
>
> - **`--preset` 適用。** `lattice capture`、`continuous`、`calibrate`、`multi-capture`、`multi-stream`、 `align-calibrate`、`align-apply` 以及 `align-stream` 已在 1.2 版中移除。0，並改為採用相機本身的設定。現在它會設定預設值的格式、曝光、增益和幀率：`--preset high_speed` 以 5 毫秒、6 dB、 60 fps。`lattice cam-connect --preset` 同樣保留預設值的格式（1.2.0 版本在套用預設值後會將相機設定為 12 位元）。請參閱 [常用選項](#common-options-shared-by-most-subcommands)。
> - **在 M3M（單色）相機上，拜耳像素格式即代表其位元深度。** 預設檔名稱中提及的拜耳格式，是單色感光元件無法拍攝的：在 1.2.0 版本中，相機拒絕了該格式並保留了原有的格式， 因此 `--preset high_speed` 在該情況下並非 8 位元。現在它已改為相同深度的單色格式：`high_speed` 和 `--pixel-format BayerRG8` 會產生 `Mono8`；`high_quality`、 `high_quality`、`triggered` 以及 `BayerRG12` 則會產生 `Mono12`。在混合的 M3C + M3M 陣列中，每台相機皆採用其自身系列中的共用格式，而 `network-analysis` / `analyze-array --pixel-format` 會將各台相機投影至該深度。反之亦然：若 M3C 要求 Mono 格式，則會採用該深度的拜耳格式，因此當陣列的主控為 M3M 時，其 M3C 相機將不再以 Mono 模式重新開啟。請參閱 [單色 (M3M) 單波段相機](#mono-m3m-single-band-cameras)。
> - **`lattice array-capture` 的匯出檔案預設會進行對齊，與應用程式中的設定相同。** 每個成員的 `debayered`、 `display`、`radiance`、`reflectance` 以及索引檔案，都會被變形至主攝影機的畫面範圍內，並裁切至成員間的重疊區域 （因相機偏移量而比感光元件小，`XMP-mapir:AlignmentJSON` `"applied": true`），因此 CLI 的擷取檔案與 Chloros 應用程式對同一場景的擷取檔案相同；`raw` 則維持原始格式。先前，若未使用 `--aligned` 進行擷取，所有檔案皆會以原始尺寸儲存且未經變形處理。`--no-aligned` 保留此設定。相同預設值亦適用於 `--smart`、 `--continuous` 及 `--interval`，以及 SDK 和 `chloros-mcp`。請參閱 [`array-capture` 模式]（#array-capture-modes）。
> - **`lattice array-capture --compression` 預設為 `none`。** 檔案以未壓縮格式寫入（每幀全解析度約 6.3 MB），以實現最快的持續傳輸速率；1.2.0 版本採用 `deflate` 格式（約 4.1 MB）。 若需較小的檔案，請傳入 `--compression deflate`。兩者皆為無損格式。
> - **PNG 和 JPEG 僅支援 `display` / `preview` 以及索引產品。** 若擷取的 `--format png` 或 `jpg` 檔案包含 8 位元檔案無法容納的亮度級別，系統將在開始擷取前拒絕該請求（HTTP 400，退出 `1`）， 同樣地，預設的 `--processing all` 搭配 `--format png`/`jpg` 也會被拒絕。請參閱 [擷取匯出層級](#capture-export-levels-the-all-default)。
> - **`lattice dls` 需要 `--mac`。**已不再有預設的 MAC 位址（原先為某個單元的位址）；`daq pool-discover --only ble` 會尋找感測器的 MAC 位址。**`lattice dls --camera SERIAL`** 為新增項目。
> - **更多錯誤會導致 `1` 退出。**當執行程序的輸出無法驗證，或其遺失了映像檔時，`process` 會觸發此錯誤，儘管其他產品已寫入；若執行擷取指令時，有任何檔案無法寫入 （其餘檔案仍會儲存）； 當磁碟空間已滿時會觸發 `lattice continuous` 和 `multi-stream`；當錄製器回報錯誤時會觸發 `array-burst` / `array-record`； 當請求的產品被跳過或未按要求建置時，會出現 `array-build-video`；當擷取過程中途失敗時，會出現 `project burst` / `stream`； 當錄製內容中沒有讀取資料時，會出現 `project sensor log`；當值為攝影機無法接受時，會出現 `lattice exposure --set`、`gain --set` 及 `configure --set`。 `lattice network-analysis --json` 會隨著摘要而退出 （`1` 對應於 `needs_force_slip`）。反之：當感測器已不在池中時，會觸發 `daq pool-disconnect`，並退出 `0`X。請參閱 [退出代碼](#exit-codes)。
> - **當陣列已連接時，運作緩慢的 `lattice array-connect` 絕不會失敗。** 它會等待最多 600 秒（1.2.0 版本為 180 秒；`project connect` 為 90 秒），並在過程中輸出後端的執行步驟；若屆時連線仍未完成，後端會將 — 每台攝影機都會恢復至被偵測時的原始狀態 — 因此出現 `Error:`（退出 `1`）即表示未連接任何陣列。 在 1.2.0 版本中，若在相機通電後立即進行連接，可能會持續時間超過等待時限，並在指令失敗後仍將陣列連接成功。現在按下 Ctrl+C 可取消該連接。請參閱 [持久連接](#persistent-connections-smart-prep-gui-equivalent-flow)。
> - **`lattice array-connect` 會保留完整的感測器。** 若未設定 `--width` / `--height` / `--binning`，每台攝影機皆會設定為全感測器模式，幀率將降至該電腦與網路所能提供的水準；連線輸出清單中會列出更快的選項 （像素合併或較小的 ROI）及其預估幀率。1.2.0 版本會縮小或合併影像幀以維持幀率。請參閱 [`array-connect` 選項](#array-connect-options)。
> - **網路速度緩慢時，陣列不再被拒絕。** 每台攝影機的傳輸速度會配合其網路頻寬分配，幀率亦隨之降低。若攝影機無法以如此緩慢的速度傳輸，陣列將以串列發射模式運行 (`slip-emit-and-capture`) 模式：每個幀都會送達，但相機不會同時曝光。`--force-tier sim-capture-sim-emit` 則堅持同時曝光。請參閱 [陣列 fps 與連拍模型](#array-fps--burst-model)。
> - **`lattice network-analysis` / `analyze-array` 說明「connect」的適用範圍**： 若未指定 `--pixel-format`，則由 connect 設定的格式（Mono12 / BayerRG12，原先為 BayerRG8）以及 [壓縮 12 位元](#packed-12-bit-transport---packed) 的設定將生效。 因此，相同的指令可能會顯示與 1.2.0 版本不同的幀率。
> - **相對路徑以執行指令的位置為基準。** `-o` 以及後端開啟的其他路徑 （`--input`、`--burst-dir`、`--profile`、`--save`、`--mask`），配方步驟的 `output`、 以及 `lattice index --live` 的預設 `output/`，皆會根據您執行 `chloros-cli` 的資料夾進行解析。 在 1.2.0 版本中，後端會根據其自身的安裝資料夾來解析這些路徑，但在已安裝的建置環境中會因「存取被拒絕」而失敗。請參閱 [規範](#conventions)。
> - **`lattice index` 無論以何種形式皆在後端執行**，如同其他 `lattice` 指令，因此請先啟動後端。離線索引 (`--input`) 由後端計算產生，後端會讀取 TIFF 檔案，並將產出結果寫入該檔案旁邊或存至 `-o`， 而 `--list-presets` / `--list-gradients` 則會輸出後端的表格。在 Windows 上，這些功能現已正常運作： 先前，Windows CLI 會將所有形式的 `lattice index`（包含 `--live` 在內）皆以 `Error: lattice_sdk not available` 作為回應。在 Linux 上， 離線索引過去是在CLI中無後端且無登入狀態下執行；現在則需同時具備這兩項條件，如同其他所有CLI指令。請參閱 [索引 / 植被數學](#index--vegetation-maths)。
> - **產品旁新增的檔案。** 凡是因波段解混而對像素進行夾緊處理的輻照度、反射率及指數產品，其產品資料夾旁都會有一個 `clamp_masks/` 資料夾，其中包含一個遮罩檔案，並在輸出結果中印出一行 `clamp mask:`。 請參閱 [分帶混合鉗位遮罩](#unmix-clamp-mask-which-pixels-a-band-was-clamped-on)。
> - **相機方位。** 單鏡頭擷取會將 `raw` 和 `debayered` 儲存為感測器讀取時的原始狀態，並將相機的翻轉／旋轉資訊記錄在 XMP 中；`process` 會將此資訊套用至所產生的產品中。 陣列成員檔案將保持原生格式，除非您傳入新的 `--orient` (`array-capture`、`array-burst`、`project capture` / `burst` / `stream`）。請參閱 [相機方向](#what-a-captures-folder-looks-like)。
> - **若匯出層級名稱不為「一」，則會被拒絕。** `lattice capture --processing` 以及所有 `--levels` 清單皆接受 `raw`、`debayered`、`radiance`、`reflectance` 或 `display`（或其別名，例如 `preview`；`--processing` 以及 `all`）。在觸碰相機之前，任何其他名稱均會被拒絕（HTTP 400 `UNKNOWN_LEVEL`，退出 `1`）； 1.2.0 為其儲存了 `raw` 並退出 `0`。
> - **`daq pool-set-cap` 需要 `--cap-id`。** 若缺少此參數，指令將退出 `2`；1.2.0 版本移除了遮光罩校正，僅保留裸感測器。 `--cap-id none` 要求使用未經校正的感測器。
> - **`lattice resolution --binning-mode` 已移除。** 像素合併模式始終為「平均」；該標誌仍會被接受，但會顯示警告訊息且不執行任何操作。 請參閱 [感測器調校](#sensor-tuning)。
> - **對於 LATTICE 擷取資料，`process --debayer texture-aware` 將被拒絕。** 對於存放 LATTICE 擷取檔的資料夾，`process` 會在匯入後退出 `1`，並列出檔案數量，之後才進行任何處理。 LATTICE 擷取檔始終會進行標準去馬賽克處理，因此 1.2.0 版本雖接受此選項但會忽略它；Chloros 應用程式對於這類檔案亦不提供此選項。請參閱 [`chloros-cli process`](#chloros-cli-process)。
> - **相機將恢復至被發現時的狀態 — 除非使用 `--keep-settings`。** `lattice array-disconnect`，每個暫時性指令（未指定陣列的 `multi-capture`、`multi-stream`、 `multi-test`、`align-calibrate`、`align-apply`、 `align-stream`、`index --live`）以及 `lattice continuous` 會將每台相機恢復至原始狀態： 其快門觸發與同步線接線、PTP 設定，以及本次拍攝過程中變更的所有相機設定（曝光與增益、自動曝光目標、限制與阻尼、白平衡、黑電平、像素格式、像素合併與感測範圍、翻轉、幀率）。而 `--keep-settings` 則會將每台相機的的設定儲存至相機自身的記憶體中，使其開機時即採用這些設定；`--clear-settings` 則會將該記憶體重置為出廠設定。無論採用何種方式，相機皆會保持自由運行狀態。專案在連接相機時，會從自身儲存的設定中為相機設定參數。 請參閱 [相機記憶體](#camera-memory----keep-settings----clear-settings)。
> - **所有模式皆以相同方式儲存專案設定。** 對專案下所連接的相機、陣列或光感測器所做的任何變更——無論是透過應用程式、CLI、SDK 或 MCP 伺服器進行——都會儲存至該專案的 `cameras.json` / `sensors.json` 目錄中，且在任何模式下開啟該專案都會根據其中設定來配置裝置。先前僅有應用程式會儲存專案，且其下次儲存時會覆寫其他所有內容。請參閱 [單一專案記錄，適用所有模式](#one-project-record-every-mode)。
> - **由多台相機產生的檔案會列出每台相機的名稱。** 多波段堆疊（`align-apply`、`align-stream`、`index --live --save-multiband`）、由多台相機波段組成的即時索引，以及陣列擷取的 `composite/` 影像，均以清單形式載有各來源相機的序列號、型號及校準包（`XMP-mapir:SourceCameraSerials`、 `SourceCameraModels`、`SourceCalBundleSha256s`、……)，而 EXIF `SerialNumber` 則以逗號分隔列出這些序號。 過去這些檔案要麼完全不包含任何相機資訊，要麼僅包含第一台相機的序號。請參閱 [由多台相機產生的檔案](#files-made-from-several-cameras)。
> - **僅有 `raw` 檔案會包含相機的校準標籤。** 相容於 MicaSense 的資料集（`BlackLevel`、`RadiometricCalibration`、各通道的 `Band{R,G,B}RadiometricCalibration`、 `MicaSenseVignetting*` 套組）以及 `CalibrationJSON` 有效載荷描述感測器的原始數值，因此它們僅載於 `raw` 擷取檔案中。 其餘所有檔案——擷取檔的 `debayered`、`display`、`radiance`、 `reflectance` 及索引檔案、`lattice continuous` 與 `multi-stream` 幀， 以及每個 `process` 產出檔案——皆包含相機的識別碼（`XMP-mapir:CameraSerial`、`CameraModel`、`FilterType`、 `LensModel`、`CalBundleSha256`、……），並在擷取時包含該幀的專屬記錄（`XMP-mapir:FrameJSON`：曝光、增益、溫度、像素格式、擷取時間）。 在 1.2.0 版本中，這些檔案也包含原始資料集，當工具將其 （如 Pix4D、Metashape）會將其套用至非原始值的像素。`process` 會根據其序列號及 `CalBundleSha256` 來查找此類檔案的校準資料。
>
> **其他變更：**
>
> - **陣列的自動幀率將根據陣列的實際輸出進行調整。** 若幀率損失超過 1% 的觸發次數，系統會降低幀率（此判斷基於多次長時間觀察而非單次），且會不定期重新嘗試先前降低過的幀率。您手動設定的幀率則維持不變。
> - **自動曝光更為穩定。** 系統會在光線恆定的情況下維持曝光值，光線變化後數秒內穩定下來，忽略微小的亮部細節，並能恢復在強陽光下停止回應的相機。 請參閱 [自動曝光的運作方式](#how-auto-exposure-behaves)。
> - **`process` 僅統計您的影像與錄影檔案。** 若在包含專案 `camera_cal/` 的資料夾上重新執行，將不再計入這些校準檔案（例如在 90 張影像的集合中顯示「找到 96 張影像」），而已將其納入的專案則會刪除這些檔案。
> - **`lattice multi-test` 會根據相機的開機狀態進行處理。** 它會在執行時開啟相機的 PTP，並在首次觸發前等待相機時鐘鎖定並穩定下來——每台相機需從開機後等待 120 秒（約 2 分鐘） （由 `--ptp-settle SECONDS` 設定），因為剛鎖定的時鐘在約 2 分鐘內誤差可能高達 ~200 微秒——並將每台相機的 PTP 恢復至原始狀態。若某台相機仍維持先前測試時的鎖定狀態，則不再等待其重新鎖定。 過去在單一交換機網路中，若無 DAQ-E 回應，該程式會保持 PTP 關閉狀態，並將每台攝影機的自由運行時鐘視為無法鎖定的 PTP 時鐘（`PTP is not locked`，每次執行）。請參閱 [多相機（暫態會話）](#multi-camera-transient-sessions)。
> - **`.daq` 記錄會標示其電容是經選擇還是預設的。** DAQ-U、DAQ-M 和 DAQ-E 出廠時未附帶蓋帽，且 Chloros 無法偵測是否已安裝蓋帽，因此若感測器連接時未指定蓋帽名稱，預設會被標記為 `sunshine_cosine`，這是一種 *假設*。現在每個 Chloros 記錄（架構 1.30）都會在 `cap_id` 旁邊儲存 `cap_id_source`： 若您為保護蓋命名（包含 `none`），則會儲存 `operator`；若採用預設值，則儲存 `auto_default`。處理一則假設已安裝保護蓋的記錄時，若專案的 DAQ 保護蓋設定維持為「自動」，會輸出一個 `[daq-dls] WARNING` 並標示檔案名稱：在陽光校正下，未套用電容的 DAQ-U 讀數約高出 30 倍。連接時 **請為您所套用的電容命名**，或設定專案的 DAQ 電容，警告訊息便會消失。
> - **Windows 系統上不再出現 pip 控制台視窗。** 在已安裝 Python 的電腦上，後端及其每個影像處理程序在啟動時會各執行一次 `pip`， 且每個都會開啟一個控制台視窗。現在已不再如此。
> - **`.daq` 的即時錄製資料能在斷電後存活。**錄製資料會在每次提交時，透過日誌和完整的磁碟同步寫入，這是在感測器後端進行的， 因此即使磁碟速度較慢，也只會延遲檔案寫入，絕不會影響讀數。若錄製在提交過程中中斷（斷電、強制終止），可能會在原始檔案旁留下 `<name>.daq-journal`：**請將這兩者合併。**Chloros 在首次讀取檔案時會回滾日誌，因此記錄檔案開啟時會包含停止前已提交的所有資料（約最後 3 秒的資料會遺失）；在 1.2.0 版本中，若在錯誤的時機發生斷電，可能會導致整個檔案無法讀取（「資料庫磁碟映像格式錯誤」）。
> - **在繁忙的磁碟上進行記錄，不再會導致感測器停滯。** `daq pool-record` 會將讀取資料排入佇列，並透過獨立的執行緒進行寫入；無法寫入的讀取資料會被計入 （`daq pool-list` 顯示 `N write error(s)` 的最後一個錯誤），而非導致感測器凍結（在四台相機同時儲存 TIFF 檔的情況下，單次寫入耗時 8.4 秒）。
> - **`lattice multi-capture` 若未使用陣列，當觸發器超時時，已儲存的幀將不再遺失。** 若攝影機的觸發器超時（`SC_ERR_TIMEOUT -1011`，此情況發生於單一 1 GbE 交換機上連接四台攝影機時），系統將在該幀群組處終止運行：已儲存的群組會以 `Saved:` 列出，接著顯示錯誤訊息， 退出 `1`。在 1.2.0 版本中，此情況會產生 HTTP 500 錯誤且無清單顯示，儘管檔案實際已儲存於磁碟上。
> - **軟體觸發的攝影機不再遺漏每兩個觸發訊號中的其中一個。** 處於軟體觸發模式的攝影機現在會將讀出期間收到的觸發訊號排入佇列（或由 Chloros 等待其準備就緒）， 這與線纜同步相機的運作方式相同。
> - **`daq pool-discover` 命名了 DAQ-E-S `DAQ-E-S`** （而 `--json` 的 `model` 欄位亦同）。先前針對每個乙太網路感測器皆顯示為 `DAQ-E`； 即使有感測器在約 2 秒內未能通過身分驗證，仍會以 `DAQ-E` 的名稱列出。
> - **改進的 LATTICE 指數產品**（NDVI 及其他指數）自 `process` 起，適用於所有處理模式。
> - **OCN 相機：在陽光下改進輻照度、反射率及指數產品。** 頻段解混後會套用機隊校正；RGN 相機維持不變，而 OCN 的數值與 1.2.0 版本不同。此校正亦適用於使用較早版本拍攝的 OCN 影像，只要在儲存該影像拍攝時所用校正資料的電腦上進行處理 （Chloros 會在相機連接後保留其校正資料）。
> - **`lattice continuous` 會記錄原始影像幀。** 執行 `tiff` 時，會寫入經去馬賽克處理的 3平面影像，而 `process` 無法以此為基礎進行處理。其影像現已轉為 `raw` 檔案——這正是陣列擷取所產生的檔案：包含一個以感測器原始數值呈現的馬賽克平面，未經壓縮，並附有校準封包。 一個從這些檔案讀取三個平面資料的腳本，實際上只讀取其中一個。
>
> - **每次擷取都會儲存其使用的 DAQ 讀數，這點與 GUI 的運作方式相同。** `lattice capture`、 `lattice continuous`（每幀一個 `.daq`）、`array-capture` 以及 `multi-capture`，當連接陣列（及 SDK）時，會寫入 `.daq` 及其對應的 `.match.json`；此行為包含僅原始資料的擷取，且在 1.2 版本中，0 版本中，儲存原始或去拜耳化擷取時，僅會透過 `--force-daq` 儲存其中一個。`--no-daq` 會關閉此功能，`--daq-tolerance-ms MS` 則設定該擷取的匹配容差，而新的 `daq pool-bind` 可透過 CLI 將相機或陣列綁定至 DAQ。`.daq` 所儲存的擷取，在 `process` 中僅會與其指定的相機進行配對。 現在，擷取會等待 DAQ 的下一次讀取，以便將曝光前後兩側的讀數進行融合。 請參閱 [擷取所儲存的 DAQ 讀數](#the-daq-reading-a-capture-saves) 以及 [遠離曝光時間的 DAQ 讀數](#a-daq-reading-far-from-the-exposure)。
> - **`process --indices NAME`** 會為每台相機的波段在專案中新增索引，並包含一個新專案。在 1.2.0 版本中，若專案尚未列出某個名稱，則會透過 `Unknown index name(s) ignored` 忽略該名稱。
> - **`process` 位於即時陣列旁：**它不再重新啟動包含相機或 DAQ 的後端，並會等待最多 15 秒，直到運行即將結束。**Ctrl+C** 會停止後端的運行，而不僅是CLI。 請參閱 [`process` 說明](#notes)。
> - **若專案資料夾層級過深，超出 Windows 的 260 字元路徑限制**，系統會直接拒絕該路徑，並採用最長且符合限制的資料夾名稱，而非執行一個未寫入任何資料的執行程序。 Windows 安裝程式現已提供「啟用 Windows 長路徑」選項，並為後端設定 Microsoft Defender 排除項目，兩者預設皆為勾選狀態。請參閱 [Windows (.exe)](#windows-exe)。
> - **`lattice array-capture --continuous`** 會執行後端自身的連續擷取功能：每 2 秒顯示一條進度條，以摘要取代逐檔清單，且若遇任何未寫入的檔案或磁碟空間已滿，則會以非零狀態結束程式。 請參閱 [`array-capture` 模式](#array-capture-modes)。
> - **在 `array-connect` 之後的第一個 `array-capture`** 會等待最多 20 秒以取得陣列的第一組幀，而非因 HTTP 503「陣列仍在預熱中」而失敗。
> - **陣列中 ROI 區域的曝光：** 狹小的 ROI 區域不再將陣列成員的曝光時間限制在感光元件的自由運行週期內（768 × 304 解析度下為 4.9 毫秒）。請參閱 [陣列成員的曝光](#exposure-on-an-array-member)。
> - **自動曝光不再降低陣列的自動幀率。** 透過纜線同步的陣列會以網路和相機讀出所允許的速率運行；自動曝光會先填滿該速率所對應的週期，然後再提高增益。 若手動曝光時間超過該週期（`lattice exposure --set`、`configure --set`），系統會放慢觸發速度以配合曝光時間，而非拒絕該操作；當元件恢復為自動模式時，幀率將恢復正常。請參閱 [陣列成員的曝光](#exposure-on-an-array-member)。
> - **陣列產生的不完整幀組較少**，且較少出現 503 錯誤（「無新的同步幀組」）：每個從屬相機的觸發輸入現已經過去抖處理。
> - **壓縮 12 位元傳輸**（`--packed`，預設為 `auto`）可將全感測器陣列的幀率提升約三分之一。 現已套用 `array-connect --packet-size`（1.2.0 版本雖已接受該設定，但實際上予以忽略）。 請參閱 [封裝 12 位元傳輸](#packed-12-bit-transport---packed)。
> - **陣列中的攝影機在電源重啟後可自動恢復運作。** 交換機電源重啟後，`array-connect` 的陣列會-當攝影機回應時重新開啟，並在 PTP 重新啟用後恢復串流；在 1.2.0 版本中，該功能會永久停止運作，因此需要新的 `array-connect`。`array-status` 會回報 `ptp_enabled: False`（以及 `ptp_off:`）的狀態，而當攝影機讀取 PTP 時，系統會報告陣列原先連接的設定。詳見 [持久連線](#persistent-connections-smart-prep-gui-equivalent-flow)。
> - **`lattice array-record` 在未開啟 GUI 的情況下進行錄製。** 當 GUI 的合併檢視關閉時，後端會自行渲染檢視畫面：陣列的的NDVI，或是您透過新的 `--expr` 傳入的索引。若錄製過程未寫入任何幀，現在會導致失敗（`no-frames: …`，退出 `1`）。 先前，它僅在 GUI 的「合併檢視」開啟時才會進行錄製， 而即使錄製內容為空，系統仍會退出 `0`。請參閱 [`array-record`](#array-record--combined-index-videogif-monitoring-grade)。
> - **Chloros 已持有的攝影機不會被再次開啟。** 若嘗試透過指令自行開啟該攝影機，系統將回傳 HTTP 409 錯誤（`CAMERA_HELD`），並標明持有該攝影機的實體以及釋放該攝影機的指令。 這包括目前正被其他指令使用的攝影機（例如：`multi-test` 或 `multi-capture` 正在開啟其攝影機，或 `lattice capture`）：第二個指令將被拒絕，直到第一個指令完成為止。 再次開啟該裝置可能會中斷持有者的串流，或導致後端當機。請參閱 [持久連線](#persistent-connections-smart-prep-gui-equivalent-flow)。
> - **暫存型多攝影機指令會傳送所有攝影機的影像，否則即失敗。** `lattice multi-capture` 未連接陣列時，`multi-stream`、`multi-test`、`align-calibrate`、 `align-apply`、`align-stream` 及 `index --live` 會退出並將 `1` 命名為未傳送任何幀的攝影機。 串流會跳過缺少某台攝影機的幀組，並在連續三次後停止；`multi-test` 會針對每個觸發事件輸出一行，`(no frame)` 則標示哪台攝影機錯過了觸發；`align-stream` 也會因工作錯誤而退出 `1` 因工作錯誤而退出。隨後每次執行都會將每台攝影機恢復至原始狀態：其觸發器與同步線接線、其 PTP (`PtpEnable`) 以及運行過程中變更的各項相機設定——包括曝光與增益、自動曝光目標、限制與阻尼、白平衡、黑電平、像素格式、像素合併與 ROI、翻轉、幀率——除非 `--keep-settings` / `--clear-settings`（參見 [相機記憶體](#camera-memory----keep-settings----clear-settings)）。 先前，這些指令會保留所有傳入的幀，並退出 `0`（`align-stream` 會輸出工作錯誤訊息，如 `Note:`）， 並將所有攝影機保留在 `TriggerMode` `On` 中。此外，在保留幀之前，系統會根據前一場景設定各攝影機的曝光值（最多數秒），並將其設定為連接陣列中各攝影機的起始參數：40 多光譜、 60 RGB（由 `--brightness N` 設定），並在整個運行過程中維持此設定； Chloros 未持有的單一 `lattice capture` 亦同。先前，每台相機的第一幀皆以該相機自身的曝光極限（約 100 毫秒）拍攝，無論光線條件如何，且經常發生剪裁現象。 參見 [多攝影機（暫存會話）](#multi-camera-transient-sessions)。
> - **暫存多攝影機指令會像 `array-connect` 那樣共享網路，且僅開啟 `--serials`。** 每台攝影機的傳輸速率會根據其在主機鏈路中的分配比例進行調控，因此四台攝影機可在 1 Gbps 鏈路上運行；此前，這四台攝影機總共要求 125 MB/s 的傳輸速率，佔滿了 1 GbE 鏈路，導致每幀畫面都會丟失封包。 若陣列超出網路承載能力，系統會在任何攝影機開始串流前即予以拒絕。現在由 `--serials` 選擇攝影機，並由第一個觸發器啟動；先前則無論 `--serials` 如何指示，每個指令都會開啟網路上的所有攝影機。 若發生 GVSP 封包遺失的故障，系統現在會明確標示為此問題，而非歸咎於 M8 同步線纜。請參閱 [多攝影機（暫態連線）](#multi-camera-transient-sessions)。
> - **`lattice array-disconnect` 會讓攝影機維持自由運行狀態，如同偵測到的情況。** `TriggerMode` `Off`，以及連接時偵測到的所有其他設定——觸發器和同步線設定（`TriggerSource`、 `TriggerActivation`、`TriggerOverlap`、`TriggerDelay`、`LineMode` / `LineSource`）、PTP，以及該會話所變更的攝影機設定——因此，對前任成員發出單一針對前任成員的單一「-camera」指令即可一次取得所有畫面，且主控端不再停留在 `TriggerSource` `Software` 狀態。`--keep-settings` 會將該會話的設定儲存至每台攝影機的記憶體中，而 `--clear-settings` 則會重置該記憶體（參見 [攝影機記憶體](#camera-memory----keep-settings----clear-settings)）。 `lattice continuous` 還會將其偵測到處於「已啟用」狀態的攝影機設定為該運行期間的「自由運行」模式（並在運行結束後恢復原狀），而沒有畫面輸出的運行會退出 `1`。
> - **`lattice configure --get` / `lattice exposure`** 會輸出一個 `note:`，其中包含攝影機實際曝光值或增益值輸出 `note:`。若 `lattice exposure --set` / `configure --set` 被拒絕，則會顯示相機可接受的範圍，並將自動模式恢復原狀，而非關閉自動曝光。請參閱 [感測器調校](#sensor-tuning)。
> - **新增相機控制指令：**`lattice highlights`（在RGB顯示渲染中降低明亮白色表面的亮度）及 `lattice exposure --brightness-auto`。**`lattice multi-detect`** 可在單次探測循環中偵測到有線相機，並預設將主相機設為 RGB 相機。
> - **新增 DAQ 指令：** `daq pool-bind`； `pool-enable-ptp`；以及用於 DAQ-E 自身設定的 `pool-info`、`pool-set-network`、`pool-set-auto-stream`、 `pool-reboot` 以及 `pool-update-firmware`。請參閱 [DAQ-E 裝置設定](#daq-e-device-settings)。
> - **當 DAQ-E 停止傳輸時，後端會重新連線。** 當彙總的 DAQ-E 串流陷入靜默狀態（例如供電交換機重新啟動， 或相機連線中斷），後端會根據其原有設定重新連線，並在連線中斷期間以逐漸延長的間隔進行重試。該裝置上的錄製會繼續在同一個 `.daq` 中進行，檔案中會標記中斷間隔， 而 `pool-record --stop` 則會列出每個中斷區段。在 1.2.0 版本中，感測器會持續靜默，直到 `pool-disconnect` + `pool-connect` 為止，此時錄製也會隨之結束。 請參閱 [停止傳輸的 DAQ-E](#a-daq-e-that-stops-delivering)。
> - **`time-sync status`** 會回報 PTP 主時鐘的封包是否傳送到網路（`health`），若未傳送成功，則會在標準錯誤輸出（stderr）中顯示 `WARNING:`。 請參閱 [`time-sync`](#chloros-cli-time-sync)。
> - **Linux .deb：** 此套件會增加相機串流所需的內核套接字接收緩衝區（若採用內核預設值，將導致畫面遺失及幀率下降），並新增一個以您的使用者身分執行的後端服務，同時由 `lattice network` 負責檢查緩衝區。 CLI 絕不會停止由服務執行的後端，且升級後會重新啟動已啟用的後端服務。請參閱 [將後端作為服務執行](#running-the-backend-as-a-service-headless) 以及 [Linux 主機 — 套接字接收緩衝區](#linux-hosts--socket-receive-buffers)。
> - **`lattice array-capture` 會於無法對齊時顯示：**每次擷取資料因未對齊而儲存時（無對齊設定檔、成員解析度不同、陣列未與纜線同步， 或幀無法映射至配置檔）都會以 `Alignment:` 行顯示。**`chloros-cli daq help`** 即為 `daq --help`（其已退出至 `2`），而在 Linux 系統上，說明文件會列出該建置版本所包含的直接硬體 `daq` 指令 。
> - **Jetson：** 套件為 `chloros_1.3.0_arm64_jp6.deb`，適用於 JetPack 6，其 GPU 處理在搭載 JetPack CUDA PyTorch 的 Orin 上運行。請參閱 [Linux arm64 — Jetson](#linux-arm64--jetson-jetpack-6)。

---

## 命名規範

- 所有指令均以 `chloros-cli` 為前綴。在 Windows 系統上，二進位檔名為 `chloros-cli.exe`；在 Linux/Jetson 系統上則為 `chloros-cli`。
- 可選參數以 `--flag` 形式呈現。必填的位置參數則不帶方括號。
- 若已指定預設值，省略該參數時將採用該值。
- CLI 是一個基於 Chloros 後端的輕量級 HTTP 客戶端（該伺服器位於 `127.0.0.1:5000`，桌面應用程式亦使用此伺服器）。**哪些指令可啟動它：**
  - 核心指令 — `process`、`login`、`logout`、`status`、 `export-status`、`time-sync` 以及 `selftest` — 會使用已運行的後端，若無後端運行則會啟動一個，並在執行完畢後停止該後端。
  - **`lattice`、`project` 和 `daq` 指令絕不會啟動後端。** 它們需要一個已經正在運行的後端：Windows 桌面應用程式的後端、Linux [後端服務](#running-the-backend-as-a-service-headless)，或是您自行啟動的後端（請參閱 [啟動後端](#start-the-backend))。若無後端服務，這些指令會輸出 `Error: Chloros backend not reachable at http://127.0.0.1:5000`（並附上啟動該後端所需的指令），然後以 `1` 狀態退出。
- `CHLOROS_BACKEND_URL=<url>` 會將 **`lattice`**、**`project`**及**`daq pool-*`** 指令群指向另一個後端（`lattice network-analysis` 則指向 `--backend-url`）。核心指令會刻意鎖定 `http://127.0.0.1:<port>` 並忽略它 （IPv4 字面值可避免 Windows 的 `localhost`→`::1` 每請求約 2 秒的效能開銷）。請參閱 [環境變數](#environment-variables)。
- **相對路徑會以您執行 `chloros-cli` 的資料夾為基準進行解析。** 後端是開啟輸出與輸入資料夾的程序，因此在傳送路徑之前，CLI 會將其轉為絕對路徑： `-o` / `--output`（包括其預設值 `output`）、`--input`、 `--burst-dir`、`--profile`、`--save` 以及 `--mask`，一個 [配方](#recipe-dsl) 步驟的 `output`（預設為 `output`），以及 `lattice index --live` 的預設 `output` 資料夾。因此，`-o captures/` 會寫入 `<current folder>/captures/`，且列印出的路徑為絕對路徑。 `~` 會被展開。針對遠端後端（`CHLOROS_BACKEND_URL` 指定另一台機器），路徑會以輸入的原始形式傳送，並在該機器上進行解析。
- 所有 SDK/CLI 呼叫均需使用 Chloros+ 帳戶登入（每台機器執行一次 `chloros-cli login`；結果會快取於 `~/.chloros/`）。
- 範例使用 Linux 路徑；在 Windows 上，請將 `/home/user/...` 替換為 `C:/Users/.../...`。DAQ-U'的序列埠在 Windows 上採用 `COM3` 格式，而在 Linux 上則採用 `/dev/ttyUSB0` / `/dev/ttyACM0` 格式。
- 功能摘要將互斥旗標寫為 `{--a | --b}` 格式寫入，而可選標誌則以 `[...]` 格式寫入。

---

## 頂層概述

```
chloros-cli [global options] COMMAND [command options]
```

### 全局選項

| 標誌 | 說明 |
| --- | --- |
| `--backend-exe PATH` | 覆寫自動偵測到的後端可執行檔。 |
| `--port N` | 後端 HTTP 埠 (預設： `5000`）。僅供核心指令使用（`process`、`login`、`logout`、`status`、`export-status`、`time-sync`、 `selftest`）；`lattice`、`project` 及 `daq` 則使用 `CHLOROS_BACKEND_URL`（預設的 `http://127.0.0.1:5000`）。|
| `-v, --verbose` | 啟用詳盡輸出。 |
| `--restart` | 強制重新啟動後端（終止任何正在執行的 `backend_server.py`）。由 systemd 服務 （Linux .deb）所執行的後端絕不會被停止：該指令會改為輸出 `sudo systemctl restart <unit>` 並退出 `1`。 |
| `--version` | 顯示版本資訊 (`Chloros CLI 1.3.0`)。 |
| `--help` | 顯示頂層說明。 |

### 指令索引

| 指令 | 用途 |
| --- | --- |
| [`process`](#chloros-cli-process) | 端到端處理一個包含 Survey3 或 LATTICE 擷取資料的資料夾。 |
| [`login`](#chloros-cli-login) | 使用 Chloros+ 帳戶對本機進行驗證。 |
| [`logout`](#chloros-cli-logout) | 清除快取的憑證。 |
| [`status`](#chloros-cli-status) | 顯示當前授權／驗證狀態。 |
| [`export-status`](#chloros-cli-export-status) | 在執行 `process` 期間，即時顯示 Thread-4 匯出進度。 |
| [`language`](#chloros-cli-language) | 設定或列出 CLI 顯示語言（支援 38 種）。 |
| [`set-project-folder`](#project-folder-commands) / [`get-project-folder`](#project-folder-commands) / [`reset-project-folder`](#project-folder-commands) | 預設專案資料夾（與 GUI 共用）。 |
| [`update`](#chloros-cli-update) | 檢查並安裝 Chloros 更新（Linux/Jetson）。 |
| [`selftest`](#chloros-cli-selftest) | 系統診斷 + 基本功能測試。 |
| [`time-sync`](#chloros-cli-time-sync) | PTP 主時鐘狀態／控制。 |
| [`lattice`](#chloros-cli-lattice) | LATTICE 攝影機控制與擷取（45 多個子指令）。 |
| [`daq`](#chloros-cli-daq) | DAQ 光譜感測器控制（DAQ-U / DAQ-M / DAQ-E）。 |
| [`project`](#chloros-cli-project) | 開啟並執行已儲存的 Chloros 專案（相機 + DAQ）。 |

---

## 安裝

`chloros-cli` 已內建於所有受支援平台的 Chloros 桌面安裝程式中 ——無需另行下載 CLI 版本。安裝平台套件後，`chloros-cli` 將與桌面應用程式及其驅動的後端二進位檔一同安裝至您的 `PATH` 目錄中。

最新下載： [`https://mapir.gitbook.io/chloros/download`](https://mapir.gitbook.io/chloros/download)

> 安裝程式同時附帶了便利的啟動腳本（`Chloros_CLI.bat` / `Chloros_CLI.ps1`、`Launch_CLI.*`、 `chloros-cli.sh`），這些腳本可開啟一個可立即使用的 CLI 命令列介面。

### Windows (.exe)

1. 從下載頁面下載 Windows 安裝程式。
2. 執行 `Chloros Installer 1.3.0.exe` 並依照精靈的指示操作。 預設安裝路徑為 `C:\Program Files\Chloros\`（CLI 位於 `C:\Program Files\Chloros\cli\`，安裝程式會將此路徑加入 PATH 變數）。
3. 開啟一個新的終端機（`cmd.exe`、PowerShell 或 Windows Terminal），以便載入已更新的 `PATH`。

```powershell
chloros-cli --version
```

安裝程式會自動將 `chloros-cli.exe` 加入您的系統 `PATH`，並捆綁 LATTICE 攝影機驅動程式。後端為 `C:\Program Files\Chloros\backend\chloros-backend.exe`（安裝資料夾中的 `backend` 資料夾），該元件並未包含在 `PATH` 中。

**系統設定。** 安裝程式的 *選擇其他任務* 頁面中，**系統設定**下方有兩個選項，兩者皆**預設為勾選狀態**：

| 選項（安裝程式中的文字說明） | 功能說明 | 取消靜默安裝 |
| --- | --- | --- |
| **啟用 Windows 長路徑**（允許專案資料夾位於超過 260 個字元的深度；建議啟用） | 將 Windows 的系統級設定 `LongPathsEnabled` 設為 1。Windows 預設為關閉此設定， 此時任何路徑長度均不得超過 260 個字元；Chloros 產品會位於專案資料夾下方約 130 個字元處，因此若專案資料夾路徑過深將被拒絕（參見 [疑難排解](#疑難排解提示))。此設定僅對安裝後啟動的程式生效。卸載時仍會保留此設定，因為其他程式可能依賴此設定。 | `/MERGETASKS="!longpaths"` |
| **將 Chloros 後端排除於 Microsoft Defender 掃描範圍外**（大幅加快攝影機擷取速度；建議啟用） | 在安裝資料夾中為 `chloros-backend.exe` 新增 Defender**程序** 排除項目，因此即時防護不會掃描後端寫入的每個檔案，無論其寫入位置為何。 不會排除其他程式；卸載時會移除此排除設定。若未安裝 Defender 或其遭政策鎖定，此步驟將靜默失敗，安裝程序仍會繼續進行。 | `/MERGETASKS="!defenderexclusion"` |

**為何 Defender 選項至關重要：**在高擷取率下，對擷取檔案進行即時掃描會消耗約 30% 的擷取率（以 9 台攝影機連續擷取為例： 未受 Defender 影響時為 77.7 幀/秒，受影響時為 52.7 幀/秒）。當 `lattice array-capture --continuous` 偵測到 Defender 在擷取過程中佔用一個或多個 CPU 核心時，會顯示提示訊息；請勾選此選項後重新執行安裝程式。

若採用靜默安裝（`/SILENT` 或 `/VERYSILENT`），除非選擇停用（例如 `"Chloros Installer 1.3.0.exe" /VERYSILENT /MERGETASKS="!defenderexclusion"`），否則這兩項設定均會生效。

**Windows 防火牆。** 安裝程式會於每個網路設定檔中，為 `chloros-backend.exe` 新增兩條傳入規則，而解除安裝程式則會移除這些規則：`Chloros Backend (UDP-In)`（適用於攝影機串流、PTP、 mDNS 以及 DAQ-E 資料串流；另一則為 `Chloros Backend (TCP-In)`，僅限於本機子網，用於 DAQ-E 韌體更新，此時感測器會連回這台電腦以擷取影像（參見 [DAQ-E 裝置設定](#daq-e-device-settings))。後端服務的 HTTP API 仍維持與 `127.0.0.1` 綁定。

### Linux amd64 (.deb)

適用於 Ubuntu 22.04 LTS 或更新版本／基於 Debian 的 x86_64 工作站。

> **Ubuntu 20.04 不受支援。** 此套件的依賴清單源自
> 後端實際連結的物件，其中包含 `libc6 (>= 2.35)`；
> Focal 預設搭載 glibc 2.31。 `apt` 會直接拒絕安裝，而非在
> 執行時才導致安裝失敗。 該清單亦列出 `libgl1`，此為內建的 OpenCV 所載入，而精簡版
> 伺服器映像檔可能缺乏；`apt install` 則會一併提供。

```bash
sudo apt install ./chloros_1.3.0_amd64.deb
chloros-cli --version
```

此 .deb 檔案會安裝：
- `chloros-cli` 至 `/usr/bin/chloros-cli`，以及 `chloros-backend`（將後端）至 `/usr/bin/chloros-backend`
- 已編譯的後端與 CLI，解壓縮後分別位於 `/usr/lib/chloros/backend/` 和 `/usr/lib/chloros/cli/`
- LATTICE 相機驅動程式
- 降噪模型、校準套件及更新通道設定
- 系統專用的 Python SDK：`python3`（`import chloros_sdk`）。`apt install` 亦包含其所需套件，即 `python3-numpy` 與 `python3-requests`（建議但非必要：上述程式皆為獨立運作； `dpkg -i` 會跳過這些套件，且安裝輸出會列出缺失的項目）。在 virtualenv 環境中：`pip install /usr/lib/chloros/sdk/chloros_sdk-*.whl`
- `/etc/sysctl.d/60-chloros-gige.conf`，此套件會將相機的 串流的套接字接收緩衝區提升至 32 MB，並立即套用設定（若該值已設定得更高則維持原狀）。移除套件將刪除該檔案。請參閱 [Linux 主機 — 套接字接收緩衝區](#linux-hosts--socket-receive-buffers)。
- 後端有兩個可選的 systemd 服務，目前均未啟用（詳見下一節）

#### 將後端作為服務執行（無圖形介面）

若未透過服務執行，後端（以及隨之運作的 PTP 主時鐘）僅會在有人手動啟動時運行： 在終端機中執行 ``chloros-backend``、SDK 腳本，或核心 ``chloros-cli`` 指令，其運作時間即為該程序的執行時長。若要在無頭機器上建立始終處於運作狀態的後端，請以 **您的使用者身分** 啟用該服務：

```bash
sudo systemctl enable --now chloros-backend@$USER.service    # $USER expands to your user name
systemctl status chloros-backend@$USER.service
```

`chloros-backend@<user>.service` 會以該使用者身分執行後端。其寫入的擷取資料屬於該使用者，詳細日誌記錄於該使用者的 `~/.cache/chloros/logs` 中， 並使用該用戶的 `chloros-cli login`（`~/.chloros`），因此請先以該用戶身分登入一次。PTP 無需 root 權限。如同所有已安裝的後端，它僅監聽 `127.0.0.1:5000` 埠。

`chloros-backend.service` 會以 **root** 身分執行後端。 此後端仍會安裝，以確保啟用它的機器能持續運作，但 root 後端所寫入的一切皆歸 root 所有，因此使用者無法刪除自己的擷取資料。若要切換至您的使用者帳戶：

```bash
sudo systemctl disable --now chloros-backend.service
sudo systemctl enable --now chloros-backend@$USER.service
```

root 服務的詳細日誌位於 `/var/log/chloros`，`adm` 群組的成員可讀取該日誌（該服務會設定 `CHLOROS_LOG_DIR`； 請參閱 [環境變數](#environment-variables))。安裝套件時會顯示已填入您使用者名稱的 `enable` 指令，並在 root 服務啟用時一併顯示切換指令。

**啟用一個後端單元。** 每次僅能運行一個後端。若啟用第二個單元（例如除 `chloros-backend@<user>.service` 之外的 `chloros-backend.service`，或是兩位使用者的 `@` 單元） 將無法啟動，並維持為 `failed` 狀態且退出狀態為 98；systemd 不會重新啟動它。 其 `systemctl status` 說明了原因：`Another instance is already running. Exiting.`，接著是佔用後端資源的項目（5000 號埠，或鎖定檔案及其所属使用者）， 接著是 `systemd will not restart <unit> (exit status 98)`。重新開機後，這兩組單元中任一組都可能被選中運行。若要切換，請依照上述步驟停用您不使用的組。**CLI 絕不會停止服務的後端。** `chloros-cli` 會直接使用現有後端，這與 GUI 和 SDK 的做法相同：`process` 在其上運行，並載入自己的專案來取代上一個專案。 在原本CLI會重新啟動後端的情況下（例如 `--restart`，或是進口操作未找到任何項目的 `process`）， 它會指定單元名稱、輸出 `sudo systemctl restart <unit>` 並退出 `1`， 因為以這種方式停止的後端將保持停用狀態。`GET /api/status` 將單元命名為 `service`（若後端未執行任何服務，則命名為 `null`）。**升級與移除。** 套件升級會先停止所有後端服務，然後依序重新啟動每個已啟用的服務，優先處理 `@<user>` 單元（若某個單元曾手動啟動但從未啟用，則會保持停止狀態，如同重新開機後的情況）。 若啟用多個後端服務，僅會執行其中一個，其餘則如上所述維持失敗狀態；升級過程會輸出 `Chloros: N backend units are enabled: …` 和 `Keep one enabled: sudo systemctl disable --now <unit>`。移除套件會停止並停用這些服務。 若執行「清除」操作，亦會刪除 `/var/log/chloros`。

### Linux arm64 — Jetson (JetPack 6)

```bash
sudo apt install ./chloros_1.3.0_arm64_jp6.deb
chloros-cli --version
```

與 amd64 .deb 檔案的佈局相同，包含接收緩衝區及 [後端服務](#running-the-backend-as-a-service-headless)，適用於 **JetPack 6**（L4T R36）上的 Jetson Orin / Orin NX / Orin Nano。 其 PyTorch 採用 JetPack 6 針對 Orin 所建置的 CUDA 版本，因此 GPU 處理（包含支援紋理的去拜爾演算法）將在 Jetson 的 GPU 上執行。 此套件本身並不包含 CUDA：它使用 JetPack 6 所安裝的 CUDA 和 cuDNN 函式庫（若映像檔中缺少這些函式庫，則使用 `sudo apt install nvidia-jetpack`）。 若無法使用 CUDA GPU，處理將在 CPU 上執行，且「紋理感知」功能將使用標準去拜爾演算法（參見 [`process`](#chloros-cli-process)）。

### 每台機器僅需驗證一次

每個平台在執行 SDK 或 CLI 呼叫之前，都需要進行一次性的 Chloros+ 登入：

```bash
chloros-cli login user@example.com 'YourPassword'
```

憑證會快取於 `~/.chloros/user_session.json` 中。

### 啟動後端

`lattice`、`project` 和 `daq` 指令會與已運行的後端進行通訊；它們絕不會啟動後端（請參閱 [規範](#conventions)）。使用這些指令時，請確保後端持續運行：

| 平台 | 操作方式 |
| --- | --- |
| Windows | 開啟 Chloros 桌面應用程式（其後端同時為 CLI 提供服務），或在獨立終端機中執行 `"C:\Program Files\Chloros\backend\chloros-backend.exe"`（位於安裝資料夾內的 `backend` 資料夾中），並保持其開啟狀態。 |
| Linux / Jetson | 在獨立的終端機中執行 `chloros-backend`，或透過 [後端服務](#running-the-backend-as-a-service-headless) 讓其持續運行。 |

每次僅能運行一個後端；所有客戶端（圖形介面、CLI、SDK 腳本）皆共用該後端。 核心指令（`process`、`login`、`status`、…）也會使用它，並在沒有後端執行時啟動一個。

**後端會在指令之間持續運行**，因此下一個指令無需再次支付啟動費用。 當您登出時（`chloros-cli logout`，除非 Chloros 應用程式仍開啟，或正在執行運行或離線建置），或執行 `chloros-cli backend stop` 時，或當 30 分鐘內無任何元件使用它時，它會自動停止： 無任何命令、腳本或應用程式，無任何執行任務或離線建置，亦無連接的攝影機或光線感測器。當執行任務或離線建置正在進行時，`backend stop` 會拒絕停止並標示該任務名稱；`--force` 會立即停止它，而該執行任務或建置將遺失 （會保留一個快照，並由 `array-build-video` 重新建置）。 請在後端啟動前設定 `CHLOROS_BACKEND_IDLE_MIN` 以變更 30 分鐘的時限（`0`：永不）；[後端服務](#running-the-backend-as-a-service-headless) 絕不會自行停止。`chloros-cli backend status` 會顯示是否有後端正在運行、哪些程式正在使用它，以及是否有任何離線建置正在執行。`process` 會在正在運行的後端中開啟其專案，除非 Chloros 應用程式已開啟並載有該專案 開啟的專案：此時會拒絕操作，您需先從應用程式處理或關閉該專案。先前由 `process` 或 SDK 腳本留下的開啟專案，將直接被取代。

### 驗證安裝

```bash
chloros-cli --version           # prints "Chloros CLI 1.3.0"
chloros-cli selftest            # 7-step diagnostic (backend, GPU, models, CUDA)
chloros-cli status              # shows license tier + logged-in user
```

`selftest` 僅在通過所有七項檢查後才會退出 `0`。最後一項檢查需要 CUDA GPU（NVIDIA GPU、 或 Jetson Orin 的 GPU）以及去噪模型，因此若電腦未配備此類 GPU，系統會顯示 `Failed: CUDA + Denoiser` 並退出 `1`；此時除僅限 GPU 運算的「支援紋理的去拜爾演算法」外，其餘功能仍可正常運作。

> **需訂閱 Chloros+。**CLI 需具備有效的 Chloros+ 方案。**Copper**是 Chloros+ 的入門層級 — 所有付費的 Chloros+ 層級皆可存取 CLI/SDK；僅免費的**Iron** 層級除外。（方案 ID 對應表：`0`=Iron/免費， `1`=Copper、`2`=Bronze、`3`=Silver、`4`=Gold。） 請於 [`https://cloud.mapir.camera/pricing`](https://cloud.mapir.camera/pricing) 進行升級。
>
> 此下限由後端強制執行，而不僅限於 CLI：若 SDK 或 CLI 標記的請求未附帶付費方案，無論該請求來自 `chloros-cli`、Python SDK 還是手動編寫的 HTTP 客戶端，皆會因 `403 PLAN_UPGRADE_REQUIRED` 而被拒絕。 已登出狀態的呼叫者則會收到 `401 AUTH_REQUIRED` 錯誤。在方案的寬限期內（月費方案為 30 天，年費方案則至到期日）可離線存取 ，過期後即停止；`chloros-cli status` 則會持續運作，以便顯示錯誤原因（這是唯一一個不受層級限制的 SDK/CLI 路徑 — `GET /api/license-status`）。

---

## 五分鐘快速入門

```bash
# 1. Sign in once on this machine
chloros-cli login user@example.com 'YourPassword'

# 2. Survey3 / LATTICE folder → finished radiance + NDVI in one call
chloros-cli process "/home/user/captures/flight_001" \
  --vignette --reflectance --indices NDVI GNDVI OSAVI

# 3. The lattice / daq commands need a running backend: on Windows open the
#    Chloros desktop app; on Linux start one (or see "Start the Backend")
chloros-backend &

# 4. Take a single LATTICE photo with the first camera found
chloros-cli lattice capture -o output/

# 5. Connect a 4-cam LATTICE array with the GUI's smart-prep flow
chloros-cli lattice array-connect \
  --serials 213800234,214000533,214701288,214701292

# 6. Read a spectrum from a connected DAQ-U (Windows: --port COM3)
chloros-cli daq pool-connect --port /dev/ttyUSB0
chloros-cli daq pool-latest --sensor-id CB-7C-A8-2E-5F   # id from 'daq pool-list'
```

`process`（步驟 2）會在沒有後端執行時為自身執行啟動一個後端，並在執行完成時停止該後端，因此請從步驟 4 開始啟動一個後端。 請參閱 [啟動後端](#start-the-backend)。

---

## `chloros-cli process`

將一個圖像資料夾透過完整的 Chloros 處理流程進行處理（目標偵測 → 校正 → 暗角處理 → 反射率 → 參數匯出）。

### 概要

```
chloros-cli process INPUT [OPTIONS]
```

### 位置參數

| 參數 | 說明 |
| --- | --- |
| `INPUT` | 包含 `.raw + .jpg`（Survey3）、`.tif/.tiff`（LATTICE）或 `.dng` 檔案的輸入資料夾路徑。 |

### 常用選項

| 標誌 | 預設值 | 說明 |
| --- | --- | --- |
| `-o, --output PATH` | 在預設專案資料夾中建立一個名為 `-n` 的新資料夾 | 專案資料夾。 專案名稱將以此為依據，若某資料夾中已存在 `project.json` 檔案，則會建立 `_1` / `_2` 作為並列資料夾，而非被覆寫。預設專案資料夾為 `set-project-folder`（或 GUI）所選取的資料夾，否則即為 `Documents/MAPIR/Chloros Projects`。 |
| `-n, --project-name NAME` | 自動 (時間戳記) | 專案名稱，僅在 **不包含** `-o` 時使用（若包含 `-o`，則專案名稱將採用資料夾名稱）. |
| `--debayer {standard,texture-aware}` | `standard` | `texture-aware` 針對 Survey3 影像採用 Chloros+ 神經網路去馬賽克演算法；速度較慢但品質更高。此功能僅適用於 Survey3： LATTICE 擷取影像始終使用標準去馬賽克演算法，因此對於存放 LATTICE 擷取影像的資料夾，`process` 會在匯入後拒絕處理 `texture-aware`，並顯示錯誤訊息指出影像數量，隨後退出 `1` 並未進行任何處理便會退出（Chloros 應用程式對此類影像亦不提供「Texture Aware」功能）。此功能需配備 CUDA GPU（Windows / Linux x64 系統上的 NVIDIA GPU，或 Jetson 上的 Orin GPU）： 若未配備，執行時將改用 `standard`，且後端日誌會記錄一行 `[DEBAYER] … needs a CUDA GPU …` 說明原因（如 `--verbose` 所示）。 |
| `--vignette / --no-vignette` | `--vignette` | 暗角校正。 |
| `--reflectance / --no-reflectance` | `--reflectance` | 反射率校正（若找到面板標靶則使用該標靶；LATTICE 則使用 NIST 依序列校正）。對於 LATTICE 多光譜資料，此步驟同時也作為反射率 **乘積** 切換 — 請參閱 [按產品導出切換選項](#per-product-export-toggles-lattice-multispectral)。 |
| `--ppk` | 關閉 | 套用 sidecar 檔案中的 PPK GNSS 校正。 |
| `--exposure-pin-1 CAMERA_MODEL` | 無 | 連接至曝光引腳 1 的相機型號，用於 PPK 時間同步。 |
| `--exposure-pin-2 CAMERA_MODEL` | 無 | 連接至曝光引腳 2 的相機型號，用於 PPK 時間同步。 |
| `--recal-interval SECONDS` | `0` | 最小重新校準間隔，單位為秒（專案的 *最小重新校準間隔*）。 |
| `--timezone-offset HOURS` | `0` | 光感測器相對於 UTC 的時區偏移，單位為小時（專案的 *光感測器時區偏移*）。 |
| `--format FORMAT` | `TIFF (16-bit)` | 下列其中一項：`TIFF (16-bit)`、 `TIFF (32-bit, Percent)`、`TIFF (8-bit)`、`PNG (8-bit)`、`JPG (8-bit)` 中的其中一個。Chloros 絕不會提高影像的位元深度： 對於所有 8 位元影像，較深的格式會以 `TIFF (8-bit)` 寫入，且執行程序中會明確說明一次。 在混合使用 8 位元與 12/16 位元擷取影像的專案中，每張擷取影像皆以其原始位元深度或更低位元深度進行匯出。Radiance 始終為 32 位元浮點數。 |
| `--indices NAME [NAME ...]` | 無 | 依預設名稱分類的植被指數 (`NDVI`、`GNDVI`、`OSAVI`、`SAVI`、 `EVI`、`GCI`、……)，不區分大小寫。每個預設值會根據其相機的波段新增至專案中，並包含一個新專案。若名稱並非預設值，或該相機的波段無法生成 （例如 `NDRE` 需要紅光（RE）和近紅外線（NIR），而沒有任何單一相機能同時提供這兩者），該名稱會被標記為警告，且不會產生索引影像。 |
| `--input-level {auto,raw,debayered,processed}` | `auto` | 強制將 LATTICE TIFF 設為處理流程的起點（Survey3.raw 不受影響）。 此外，這也是讓 **無原始檔** 的擷取資料仍能進行處理的「後門」——請參閱 [擷取資料資料夾的樣貌](#what-a-captures-folder-looks-like)。 |
| `--debayered / --no-debayered` | 啟用 | 輸出線性去拜耳化產出（`Debayered_Images`）。請參閱 [各產出項目的匯出切換選項](#per-product-export-toggles-lattice-multispectral)。 |
| `--preview / --no-preview` | 啟用 | 輸出顯示預覽 (`Preview_Images`)：RGB = 白平衡（若有可用，則採用 DAQ 光源；否則採用灰度世界）+ gamma；多光譜 = 假色拉伸。 |
| `--radiance / --no-radiance` | 啟用 | 輸出 float32 輻照度 (`Radiance_Images`, W/m²/sr/nm)。 |
| `--reflectance-source {daq,target,auto}` | `auto` | LATTICE 反射率產出的參考基準： `auto` = 通過品質保證（QA）的幀內目標值為絕對參考值（帶有 QA 標記的目標值會與 DAQ 針對測量分數進行競爭），若 DAQ 下行（ρ = π·L/E）則採用備用方案； `target` = 嚴格模式（不允許 DAQ 替代）；`daq` = DAQ 權威模式。請參閱 [各產品匯出切換選項](#per-product-export-toggles-lattice-multispectral)。 |
| `--target-reflectance-dir DIR` | 無 | 各單元 **測量** 目標反射率掃描的目錄（`<serial>.csv`）； 若未找到則回退至標稱 T3/T4P 光譜。|
| `--array-alignment / --no-array-alignment` | 啟用 | LATTICE 陣列： 將每個擷取檔的 `Chloros:Alignment*` XMP 中標記的模組間對齊資訊，套用至所有處理後的產品（去拜耳化／預覽／輻照度／反射率／索引）。若影像未含該標籤，則不執行此操作。 |
| `--array-alignment-crop / --no-array-alignment-crop` | 裁切 | 將對齊後的輸出裁切至陣列的共同重疊區域，使所有模組共用同一影像範圍；`--no-…` 則保留完整的感測器畫布（光源外側以黑色填充）。 |
| `--array-alignment-interp {bilinear,nearest,cubic}` | `bilinear` | 針對對齊變形進行重採樣。`nearest` 保留原始資料的精確動態範圍 (DN) 值（不進行輻射值之像素間混合）。 |

### 目標偵測選項

| 標誌 | 說明 |
| --- | --- |
| `--min-target-size PIXELS` | 偵測器所需的最小校準目標尺寸（以像素為單位）。 |
| `--target-clustering 0-100` | 目標聚類百分比（預設值為 `60`）。 |
| `--target / --targets` | 僅在輸入檔案的 `target` 或 `targets` 子資料夾中搜尋校準目標（速度較快）。 |

### 範例

```bash
# Simplest: defaults are good for most surveys
chloros-cli process "/home/user/images/survey_001"

# Multi-index with explicit format
chloros-cli process "/home/user/images/survey_001" \
  --vignette \
  --reflectance \
  --format "TIFF (32-bit, Percent)" \
  --indices NDVI GNDVI OSAVI

# Texture-aware debayer for highest quality (Chloros+ only)
chloros-cli process "/home/user/images/survey_001" \
  --debayer texture-aware \
  --indices NDVI

# Process LATTICE captures explicitly (auto-detects from EXIF normally)
chloros-cli process "/home/user/captures/lattice_flight" \
  --input-level processed

# LATTICE multispectral → float32 radiance only (no DAQ downwelling needed)
chloros-cli process "/home/user/captures/lattice_flight" \
  --no-debayered --no-preview --no-reflectance

# LATTICE reflectance anchored to an in-frame target (strict, no DAQ fallback),
# with per-unit measured target scans looked up by serial
chloros-cli process "/home/user/captures/lattice_flight" \
  --reflectance-source target --target-reflectance-dir "/home/user/target_scans"

# LATTICE array capture: keep native geometry (ignore stamped alignment)
chloros-cli process "/home/user/captures/array_flight" \
  --no-array-alignment

# Aligned, uncropped, value-preserving resampling
chloros-cli process "/home/user/captures/array_flight" \
  --no-array-alignment-crop --array-alignment-interp nearest

# Save to a project folder of your choice (the project is named after it)
chloros-cli process "C:/input" -o "C:/projects/Field_A_2026-05-26"

# Name the project and let it go to the default project folder
chloros-cli process "C:/input" -n "Field_A_2026-05-26"
```

### 按產品設定的匯出開關 (LATTICE 多光譜)

LATTICE 處理會在 **單次處理中**擴展至**所有適用的產品**。這四項按類型設定的開關 — `--debayered`、 `--preview`、`--radiance`、`--reflectance` — 均**預設為開啟**；請使用 `--no-<type>` 格式來停用其中一項。 RGB 主攝影機僅會輸出去拜耳化後的預覽影像（不含各波段的輻照度／反射率）， 因此 `--radiance`/`--reflectance` 對其而言無效。對於 Survey3 的 `.raw`（其遵循標準反射率／目標路徑），這些切換選項將被忽略。*（舊版 `--radiometric-output {reflectance,radiance,sensor-response}` 標誌已**移除**並由這些切換選項取代；現已不再有 `sensor-response` 層級。）*

| 產品 | 輸出 | 是否需要 DAQ 下行資料？ |
| --- | --- | --- |
| `--debayered` | 線性去馬賽克 (`Debayered_Images`)：與擷取時在 `debayered` 層級所儲存的檔案相同 — 先從原始資料中扣除校準套件的暗偏移量 (DSNU)，再進行 OpenCV 的邊緣感知去馬賽克處理；不包含線性化 LUT、平場校正或解混色。 | 否 |
| `--preview` | 顯示預覽 (`Preview_Images`)：RGB = 白平衡 + γ 校正；多光譜 = 假色拉伸。 | 否。 |
| `--radiance` | 來自完整輻測鏈 (`Radiance_Images`) 的 float32 單位 W/m²/sr/nm 值。 | 編號 |
| `--reflectance` | uint16 反射率 ρ（`32768` = 1.0）， Pix4D 相容。 | **是**，除非有通過品質保證（QA）的幀內目標用作錨點（參見下文）。 |

`--reflectance-source` 選取反射率參考點：**`auto`**（預設）將通過品質保證（QA）的畫面內標靶設為**絕對參考點**— 以標靶錨定的經驗性線鏈會針對保留的面板進行交叉評分，並按波段逐一套用測得的最優結果：若某波段無法透過 DAQ 進行校準，或其 DAQ 斜率擬合失敗，則採用該標靶自身的線 （由**`ReflectanceElBandMethods`**指定）——若目標雖通過 QA 但帶有標記（例如遺漏面板、波段順序錯誤），其測量分數亦會與 DAQ 分割值進行競爭；若該幀在競爭中落敗，則會被標記為**`ReflectanceChain=daq`**— 當無目標存在或品質保證失敗時，將回退至 DAQ 下行閾值（ρ = π·L/E）；**`target`**採用嚴格模式（不進行 DAQ 替換）；**`daq`**選擇採用 DAQ 權威行為。目標幾何（ArUco / 固定 ROI / 條帶）來自專案目標配置；`--target-reflectance-dir DIR` 根據目標單元的序列號/QR 查閱每個單元的**測量** 掃描（`<serial>.csv`），並以標稱 T3/T4P 光譜作為備用方案。

DAQ 反射率路徑會自動從記錄的 **`.daq`**（DAQ-U/M/E）**或與影像一同存取的 DAQ-M 原生 `.csv`**中，自動解析出**時間戳記匹配的下行光譜**。若未在本地端快取每台相機或 DAQ 的校正套件， 處理流程將在首次使用時**自動從 AWS 擷取**（需連網一次；儲存於 `~/.chloros/` 中）。**這是哪台相機的讀數？**擷取檔所儲存的 `.daq` 值，即為其所描述之相機（或多台相機）的讀數——該讀數名稱載於其旁附帶的 `<file>.daq.cameras.json` 輔助檔中 （`lattice capture`、`multi-capture`、 `array-capture` 及 GUI 皆然；原始連拍檔則在其清單中列出各成員的讀數）——而 `process` 僅將其**與來自該等相機的影像**。因此，沒有自身讀數的獨立攝影機將不再借用其他攝影機的 DAQ；請將 DAQ 綁定至該攝影機，或為其錄製一個。DAQ 自身的錄製 （`daq pool-record`，或在 GUI 中啟動的錄製），或是任何未附帶該 sidecar 的 `.daq`（包含較舊的擷取檔），仍可服務所有相機。

#### 讀取反射率像素（Pix4D / Metashape / 您自己的腳本）

反射率以整數 DN 儲存，且 **表示 ρ = 1.0 的 DN 取值取決於來源相機**：

| 來源 | ρ = 1.0 對應值 | 如何判別 |
| --- | --- | --- |
| LATTICE (M3C / M3M) | `32768`（具有 ρ 2.0 的餘裕） | 檔案中標有 XMP `Chloros:PixelScale=32768` 標記。 |
| Survey3 | `65535`（在 ρ 1.0 處被截斷） | 沒有 `Chloros:*` XMP 標籤 —— 這種缺失 *本身* 就是訊號。 |

**應讀取 `Chloros:PixelScale` 並以此為基準進行除法**，而非假設其為常數。 該標籤定義於 uint16 域中，因此即使在進行重新縮放的輸出格式中，其值仍維持不變 — `32768`、`TIFF (16-bit)`、`TIFF (8-bit)`、 `PNG (8-bit)`、`JPG (8-bit)` 以及 `TIFF (32-bit, Percent)` 皆為自描述型 （請先將儲存的資料類型歸一化回 uint16：8 位元資料乘以 257，浮點數乘以 65535）。

> **來自較舊版本的檔案可能不包含比例因子。** 一個 8 位元反射率 TIFF 檔案會被重新縮放，並像其他檔案一樣標記為 `Chloros:PixelScale`。1.3.0 之前的版本則會將其 *裁剪* 至 0..255 範圍，因此所有大於 ρ≈0.008 的值都會被壓縮為 255， 且同時省略了 `Chloros:PixelScale` 與 `MicaSense:RadiometricCalibration` 這個元組。**若 LATTICE 反射率檔案中缺少此標籤，請勿假設其具有比例尺——應重新處理該檔案**，而非對原本無法被整除的像素進行除法運算。

#### 讀取指數像素（NDVI 及其他植被指數）

**整數指數柵格是針對每張影像進行的對比度拉伸，而非 −1…1 的映射。** `TIFF (16-bit)`、`TIFF (8-bit)`、 `PNG (8-bit)` 以及 `JPG (8-bit)` 等指數產品——包括 8 位元擷取影像所寫入的 8 位元 TIFF 檔案——儲存

```

DN = (value − IndexMin) / (IndexMax − IndexMin) × full_scale
```

其中 `IndexMin` / `IndexMax` 分別是 **該幀本身的最低與最高有限索引值**。 DN 0 並非 −1，且滿量程並非 +1。

Chloros 會在兩個端點上加蓋標記，以確保映射保持可逆：

| 標籤 | 含義 |
| --- | --- |
| `Chloros:IndexScaling` | `linear_minmax` — 如下所示的慣例 |
| `Chloros:IndexMin` | 儲存為 DN 0 的索引值 |
| `Chloros:IndexMax` | 儲存為滿量程的索引值 |

讀取這些資料並進行反轉，首先根據儲存資料型的最大值，將儲存的 DN 歸一化為 0…1，以便單一表達式能涵蓋所有整數格式：

```python
import json, subprocess
import numpy as np, tifffile

tags = json.loads(subprocess.run(
    ["exiftool", "-j", "-XMP:all", path],
    capture_output=True, text=True).stdout)[0]

# float(): exiftool hands back a negative real as a quoted string, so
# IndexMin arrives as str on most real NDVI data while IndexMax is a number.
lo, hi = float(tags["IndexMin"]), float(tags["IndexMax"])

raw = tifffile.imread(path)                    # uint16 for TIFF (16-bit)
ndvi = lo + raw.astype(np.float64) / np.iinfo(raw.dtype).max * (hi - lo)
```

讀取時無需 `-config pix4d.config` — exiftool 可自行讀取這些 XMP 屬性。此設定僅在 *寫入* 時才需要。

> **`-config pix4d.config`** 表示該幀的每個像素都具有相同的索引值。柵格以中點刻度填充，且無需除以任何數值——直接取端點值即可。

> **若無 `Chloros:Index*` 標籤，即表示未指定映射關係。** LUT 匯出是一種上色的渲染結果，其像素為根據您的 LUT 閾值視窗設定的漸層顏色；而一幀畫面若無有限索引值的像素，則無需指定範圍。 無論何種情況，請勿假設存在映射關係——此規則與缺少 `PixelScale` 的情況相同。**由於拉伸是針對每張影像進行的，因此同一航次的兩幀影像在數位數值（DN）上無法相互比較**—— 每幀都攜帶其自身的端點，因此相同的NDVI值在不同影像中會落在不同的DN上。若要進行跨航次的定量分析（如分區統計、固定閾值、時間序列），請匯出**`TIFF (32-bit, Percent)`**：索引柵格將以 float32 格式寫入其中，內容為**索引值本身**，無需重新縮放亦無需標籤。儘管格式名稱如此，索引產品並非 ×100 —— 該縮放僅適用於整數資料型的反射率。

#### 解混剪裁遮罩（某波段在哪些像素上被剪裁）

3×3 光譜解混可能會導致某波段呈現負值——例如當光線幾乎不包含該波段，或場景超出相機解混矩陣所描述的範圍時。 這並非真實數值，因此匯出時會將其固定為 0 —— 即時預覽中同樣顯示 0。某波段可能僅在少數像素上被限制，也可能在幾乎所有像素上被限制；限制遮罩會記錄這兩種情況。 `Chloros:Band{R,G,B}UnmixClampFraction` 表示此現象發生於幀中的比例；而夾斷遮罩則指出 **哪些像素**：

- **`clamp_masks/<product folder>/<product name>_clampmask.tif`**— 位於產品資料夾**旁邊**的 `clamp_masks/` 資料夾中，絕不會位於其內部（`radiance/x_radiance.tif` → `clamp_masks/radiance/x_radiance_clampmask.tif`； `process` 匯出檔中的 `tiff16/Reflectance_Calibrated_Images/x.tif` → `tiff16/clamp_masks/Reflectance_Calibrated_Images/x_clampmask.tif`）——針對每個輻照度、 反射率及折射率產品中，只要至少有一個像素發生裁剪：無論是擷取檔或 `process` 匯出檔皆然（以及 `array-build-video --save-tiffs` 的每幀 TIFF 檔）。單波段 `uint8`， 基於該產品自身的像素網格（包含方向、ROI、像素合併及陣列對齊）。每個遮罩會標示其所屬產品的相機（`Make` / `Model` / 序列號）、擷取時間及鏡頭標籤，並列出其位元所代表的波段 （`XMP-Camera:BandName`，以位元 0 為首）；其 `ImageDescription` 和 `Software` 標記顯示其為鉗位遮罩，且未包含任何校正資料。
- **位元**：位元 0 = 產品 R 通道中的波段 (`Chloros:BandRName`)，位元 1 = G (`BandGName`)，位元 2 = B （`BandBName`）。若位元被設定，表示該頻段中該像素的值已被限制為 0 — 請將其（以及由此建立的任何索引）視為可信度較低。
- **`Chloros:UnmixClampMask`** 會標記在每個執行過解混色的產品上：遮罩的路徑以產品自身資料夾為基準（`../clamp_masks/<product folder>/<product name>_clampmask.tif`， 始終以 `/` 分隔 — 將其與影像資料夾名稱結合即可開啟遮罩），若無任何像素被限制，則標記為 `none`。 若無標籤，表示該產品從未執行過解混程（RGB 或單色相機，raw/去拜耳化級別）。
- `lattice capture` / `array-capture` 會將遮罩路徑印在儲存檔案下方 （`clamp mask: …`），且擷取結果清單會將其列為該幀的 `clamp_mask`。
- `lattice index --live --save-multiband` 會針對多波段堆疊的每個來源攝影機寫入一個遮罩 — 請參閱 [`lattice index` — 多波段堆疊遮罩](#lattice-index--multiband-stack-masks)。影片則無遮罩：`array-build-video` 會改為回報每幀的夾緊比例（參見 [`array-build-video`](#array-build-video--offline-reprocess-a-saved-burst)）。

由於任何層級的產品資料夾中均未包含遮罩，因此可將產品資料夾（`radiance/`、`reflectance/`、 `process` 匯出檔中的 `*_Images/` 資料夾）皆可直接傳遞給 Pix4D、Metashape 或其他攝影測量軟體，因為它 ——包含子資料夾在內——且該軟體僅接受影像檔案。請將產品資料夾直接傳入，而非其上層資料夾（該資料夾還包含 `clamp_masks/` 以及其他所有匯出類型）。Chloros 自身的匯入功能會跳過遮罩。讀取遮罩： `(mask & 1) != 0` 表示「此處的 R 通道波段已遭限制」，`& 2` 表示 G 通道，`& 4` 表示 B 通道。

#### 傳遞至匯出的 EXIF 資料

`process` 會將原始擷取檔的 **GPS 區塊及其 ExifIFD** 至每個產出檔案，因此
匯出時會包含 `FocalLength`、`FNumber`、`ExposureTime`、`ISO`、 `DateTimeOriginal` 以及
`CameraSerialNumber` 將與地理參考資料一同輸出。

**`FocalLength` 對於攝影測量而言並非可選項目。** Pix4D 會根據
焦距加上高度來計算地面樣本距離；若缺少此標籤，系統將回退至極其錯誤的比例尺。在一次
拍攝 49 張影像的橘園飛行任務中，由於缺少此標籤，導致原本 411 公尺 × 160 公尺的場地，在重建後
47.8 公里 × 13 公里的區域——這張 455 MP 的正射影像中大部分區域顯示為「無資料」，在
任何人檢查 GSD 之前，這被誤判為拼接問題和
BigTIFF 問題。若您的正射影像輸出比例尺顯得
不合理， 請先對匯出的產品執行 `exiftool -FocalLength` 指令。

此副本刻意**未**使用 `-all:all`：IFD0'的結構標籤在
複製時會破壞 LATTICE 的輸出，而 `ExifImageWidth` / `ExifImageHeight` 則被排除，因為它們描述的是
*原始* 擷取檔——若曾經調整尺寸的匯出檔，其尺寸
與其自身光柵相矛盾。XMP 會直接寫入而非複製，因為當 XMP 區塊被複製時，ExifTool
會捨棄同一呼叫中的 XMP 標籤（這將導致 MAPIR 校準標籤遺失）。

### 輸出檔案的存放位置

產出檔案將寫入 **專案資料夾下，依相機分組，再依檔案格式分類**：

```
<project>/
└── LATT-M3M-L41-F550/                  # one folder per camera model+lens+filter
    ├── tiff16/
    │   ├── Reflectance_Calibrated_Images/
    │   ├── Debayered_Images/
    │   ├── Preview_Images/
    │   └── <INDEX>_Index_Images/        # e.g. NDVI_Index_Images
    └── tiff32/
        └── Radiance_Images/             # float32 radiance always lands here
```

LATTICE 的相機資料夾為 `LATT-<sensor>-<lens>-F<filter>`（與擷取檔的 EXIF
`Model`），Survey3 則為 `<model>_<filter>` —— 兩台相機雖共用同一個感光元件和濾鏡，但因
鏡頭不同而產生暗角、視野及畸變差異，因此會分別建立獨立的目錄結構。格式
資料夾結構依循 `--format`：`tiff16`、`tiff8`、`png8`、`jpg8`，或 `tiff32` 對應於
`TIFF (32-bit, Percent)`。

> **每個匯出的產品皆保留原始檔案的名稱。** 將
> `capture_…_raw.tif` 匯出為輻照度檔案時，檔案名稱仍稱為 `capture_…_raw.tif` — 只是存放於
> `tiff32/Radiance_Images/` 目錄中。**目錄名稱用以識別產品，而非檔案名稱**， 因此，若使用通配符
> 搜尋 `*radiance*.tif` 將一無所獲；請改以資料夾為依據進行比對。

### 光感測器錄製檔 — 已校準的 `.daq` + `.csv`

`process` 亦會處理您輸入資料夾中的 `.daq` 記錄檔，且 **無需**
需要任何影像資料：單獨飛行的 DAQ-U / DAQ-M / DAQ-E 即構成完整的
擷取，而僅包含 `.daq` 檔案的資料夾亦為有效的輸入。

DAQ 即使**未經**校準也能進行錄製——這正是公開版
[`chloros_scripts`](https://github.com/mapircamera/chloros_scripts) 記錄器
(`record_daq.py`) 預設的運作方式：它們寫入原始感測器計數值並在檔案上標記時間戳記，以便
Chloros **透過序列埠**（優先從本地快取，
其次是 MAPIR 雲端）並套用該校準。`process` 會將結果寫回：

```
<project>/
└── Light Sensor/
    ├── <name>_calibrated.daq        # reprocessable archive, declares its bundle
    └── <name>_calibrated.csv        # W/m^2/nm per reading + photometric columns
```

`.csv` 每筆讀數包含一行資料：UTC 時間戳記、積分時間、總功率、
明視/暗視勒克斯、PPFD（及其藍光/綠光/紅光分量）、峰值波長，接著是
基於感測器自身波長網格的全光譜。`.csv` 檔案在未經
二次校準的情況下重新匯入。

`<name>` 即為該記錄本身的檔案名稱。 位於不同資料夾的兩份記錄可以共用
一組 — 由兩台相機在
同一毫秒內儲存的「每次擷取讀數」（`<capture>/daq/daq_data_<sensor id>_<ts>.daq`） — 且每份記錄仍會獲得各自的產出：第一份保留
`<name>`，下一則則在檔名**之後**加上區分其來源的資料夾名稱
(`<name>__<capture>_daq_calibrated.daq`)，因此錄製檔的日期與時間仍居首位。

執行成功時，執行報告會顯示 `Light-sensor products written: N (calibrated .daq + .csv)`。
括號內的內容描述了實際寫入的內容，因此讀作
若為無資料包的感測器，則顯示為
`(RAW COUNTS — this sensor has no calibration bundle)`；若為同時包含兩者的資料夾，則顯示為
`(N calibrated, M raw counts)`。 後端自身的
`[DAQ-EXPORT]` 與 `[RUN-SUMMARY]` 標題亦以相同方式衍生其措辭——這
三者皆不能將原始匯出資料稱為「已校準」。

若 DAQ-U / DAQ-M / DAQ-E 的記錄無法取得其校準套件——例如您
處於離線狀態，或該感測器檔案中無校準資料——則該記錄將 **會連同原因標示**一併跳過**，並顯示於
`[DAQ-EXPORT]` 行中，絕不會被寫入為包含原始計數的「已校準」檔案。
請連線至網際網路並重新執行。原因即為讀取器實際
針對該檔案所判定之原因（無法讀取的架構、無校準套件、寫入錯誤），而執行
摘要會列出**不同的**原因——二十個因同一原因被跳過的檔案，系統會顯示為單一
原因，而非重複列出二十次該原因。 CLI會在運行結束時列出每個被跳過的記錄，
並附上其原因：

```
⚠ Light-sensor recording not exported: <file>.daq: no calibration bundle available for …
```

若某個錄製檔案資料夾中**沒有**任何檔案被匯出，則會失敗：請參閱
[未寫入任何影像的執行會失敗](#a-run-that-writes-no-images-fails)。

名稱以 `_calibrated` 或 `_raw`（`field_raw.daq`）結尾的錄製檔 **並非****：這些是 `process` 賦予其自身產出檔案的名稱，若將產出檔案
當作錄製檔案讀回，將會被重複計數。 CLI會明確指出這一點，並標示該檔案名稱——
`Not imported as a light-sensor recording: field_raw.daq … Rename the file to import it.`
——而非回報為空資料夾。請在處理此類記錄前先重新命名。

#### DAQ-A 記錄會以原始計數形式匯出

**DAQ-A** 系列產品早於「按序列號捆綁」系統推出，且無需擷取校準包
—— 而是透過現場對照反射率標靶進行校準，這正是
它從未需要校準包的原因。若拒絕這些錄製檔，將導致其
數據完全無法輸出，因此它們會以 **不同名稱** 匯出：

```
<project>/
└── Light Sensor/
    ├── <name>_raw.daq        # NOT _calibrated
    └── <name>_raw.csv        # raw spectral sensor counts, NOT irradiance
```

採用不同的檔案名稱而非檔案內的標記，是因為該標示必須能在
僅以裸名透過電子郵件傳送時仍能保留其意義。 `.csv` 檔案標頭顯示
`raw spectral sensor counts (NOT irradiance)`，並警告這些數值僅在
**檔案內部**具有可比性——這正是基於標靶的校準所使用的目的——
而非跨感測器比較。 與功率相關的光度學欄位（總功率、明視與
暗視勒克斯、PPFD）會寫入 **NULL** 而非從計數值積分得出，且運行
摘要顯示為 `RAW COUNTS`，因此日誌中「匯出」的字樣，因此無法將其解讀為輻照度。

舊版 **v1.01 / v1.02** 記錄（由 DAQ-A-SD 寫入）不包含每筆讀數的時戊，
僅有檔案的寫入時間。 影像↔下行輻射配對器仍會拒絕這些記錄——將
幀與寫入時間進行配對會導致隱性錯誤——但匯出工具能讀取這些記錄，且
CSV 檔案會輸出 `clock=daq_created_on`，因此該產品會標示其所屬的時鐘。

### 注意事項

- `process` 會自動偵測您的資料夾屬於 Survey3、LATTICE 還是混合類型。
- 進度透過 Server-Sent Events 串流傳輸；CLI 會顯示各執行緒的即時進度（偵測中、分析中、處理中、匯出中）。
- 針對 Linux/Jetson，CLI 會檢查交換空間，並可能在處理大型資料夾前發出警告。支援紋理的去拜爾演算法也會在低功耗 Jetson 裝置（Nano、Orin Nano）上自動套用 GPU 頻率上限。
- 執行成功時，系統會回報已寫入的影像產出數量（`Image products written: N`；若解混過程對任何像素進行了裁剪，則顯示為 `(plus M clamp masks)`）。
- **`process` 可安全地共用正在執行的後端。** 它僅在後端處於閒置狀態（即已載入前一個專案，且未持有任何相機、陣列或 DAQ）時，才會自動重新啟動該後端。 若後端持有上述任何項目（例如 GUI 正在串流陣列、SDK 腳本或 DAQ 錄製），則會維持現狀使用，絕不重新啟動，因此 `process` 可與正在運作的陣列並行執行。若後端回報有執行中任務，`process` 會等待最多 15 秒直至其完成（剛結束的執行約需一秒鐘才能清除），隨後會以 `Backend is mid-run (processing). Wait for it to finish, or pass --restart to force a fresh backend (this kills the in-flight run).` 錯誤碼拒絕執行。全域標誌 `--restart` 仍會強制啟動新的後端，並終止其上的任何攝影機、陣列或 DAQ 工作階段。由 Linux systemd 服務執行的後端絕不會被重新啟動： `process` 會使用該後端，而 `--restart` 則會輸出 `sudo systemctl restart <unit>`（參見 [將後端作為服務執行](#running-the-backend-as-a-service-headless))。Chloros 應用程式自身的後端亦然：當應用程式開啟時，`process` 會使用該後端，並載入其專案以取代應用程式的專案（若在此處啟動新的後端，將導致已開啟的應用程式無法存取該後端）。
- **正在執行的處理任務會保留其專案。** 當處理執行程序正在進行中時——無論是由應用程式、`process`、SDK腳本或`chloros-mcp`所啟動——後端將拒絕建立、開啟、匯入、新增或移除檔案，亦拒絕移動、 刪除或變更專案的處理設定，並會回傳 HTTP 409 錯誤代碼 `PROCESSING_IN_PROGRESS` 和 `A processing run is in progress on project '<name>', so ... now is refused`；若收到 `process` 並成功處理，則會退出 `1` 後便會退出。顯示設定（縮圖大小）仍會儲存。處理模式與專案的目標反射率設定亦會以相同方式遭拒絕（執行程序將維持其啟動時的模式）。 在執行過程中儲存至專案的擷取資料（無論來自應用程式、CLI、SDK 或 `chloros-mcp`），皆會如常儲存，但僅在執行結束時才會加入專案的檔案清單； 其回覆訊息亦明確說明此點（`project_add`），而中途停止並匯入專案的光感測器記錄亦同。先前，第二個客戶端可在執行過程中切換已開啟的專案，且正在執行的任務會匯出至另一個專案中。
- **Ctrl+C 會停止後端中的執行，而不僅是CLI。** CLI 會要求後端停止處理，輸出 `Waiting for the backend to stop (Ctrl+C again to stop waiting)...`，並最多等待 120 秒以接收 `Processing stopped.`，因此 `chloros-cli process … && next` 不會在後端仍忙於執行該運行時啟動 `next`。再次按下 Ctrl+C 將終止此等待。隨後 `process` 會退出 `130`，如同任何被中斷的指令一樣。

#### 未寫入任何影像的執行會失敗

若您要求輸出產物，而執行過程未寫入**任何**產物——僅有 `project.json` 和
`calibration_data.json` —— `process` 會將此視為失敗：它會輸出
`Processing finished but wrote no image products.` 並以 **非零狀態退出**，因此腳本可
偵測到此狀況。 該訊息會列出專案資料夾名稱及常見原因：

- 輸入資料夾未被識別為擷取資料（請檢查佈局及 `--input-level`），或
- 所有請求的產品均因不適用於該批相機而被跳過（例如：要求
  僅支援 RGB 的相機卻要求輻照度／反射率）。

請搭配 `--verbose` 重新執行，並檢查後端日誌中是否出現 `[LATTICE-EXPORT]` / `[EXPORT-CHECK]` 相關記錄，
這些記錄會說明各相機被跳過的具體原因，而這些資訊通常不會出現在CLI的輸出中。

若刻意執行僅處理元資料的運行——將所有產品選項關閉且不使用 `--indices` ——仍屬
**成功**，因為在此情況下輸出空圖像即為正確結果。

#### 若輸出檔案無法驗證，則運行失敗

執行完成後，`process` 會使用 ExifTool 檢查匯出檔案的標籤。若 ExifTool 無法執行，則
會輸出 `匯出驗證失敗：ExifTool 無法執行，因此 M 個匯出檔案中的 N 個
未經檢查（請參閱後端日誌中的 [EXIFTOOL] 行）` and `處理結果已寫入並保留
於原位置；請重新執行以驗證這些結果。`, does not print `處理完成！`，並 **以非零狀態退出**。該
處理結果將予以保留。

#### 哪個擷取會被納入執行結果

在執行 `process` 之前完成的擷取會被納入。執行期間儲存的擷取則不會；`process`
會警告有多少擷取被遺漏，再次執行即可將其納入。

#### 遺失影像的執行會失敗

若執行過程中某張影像失敗，`process` 會輸出 `N image(s) failed and were not exported` 並標示
最常見的失敗原因，不會輸出 `Processing complete!`，並 **以非零狀態結束**。 其餘產出檔會
被寫入並保留原位，後端日誌中則會記錄每項失敗。

**僅光感測器的執行**亦同： 根據定義，包含 `.daq` 記錄檔的資料夾沒有可匯出的影像，
因此該執行會根據其寫入的已校準 `.daq` / `.csv` 來進行評估。
若未寫入**任何**記錄——即所有記錄皆被跳過——則會輸出
`Processing finished but wrote no light-sensor products.`，並以
跳過原因作為各記錄的名稱，且**以非零狀態退出**。

---

## `chloros-cli login`

使用 Chloros+ 雲端帳戶對這台機器進行驗證。憑證會安全地快取於 `~/.chloros/user_session.json` 中。

```
chloros-cli login EMAIL PASSWORD
```

### 範例

```bash
chloros-cli login user@example.com 'YourPassword'

# Passwords containing $ should use SINGLE quotes
chloros-cli login user@example.com 'my$ecret$pass'
```

> **PowerShell `$$` mangling is auto-corrected.** In double quotes PowerShell expands `$$`（刪除密碼中的部分內容，或複製密碼的某些片段）。若發生 401 錯誤，CLI 會自動使用 `$$` 重新，接著使用去重複處理後的密碼一半；若重試成功，系統將讓您登入並顯示下次應使用的正確單引號語法。

> **無圖形介面／腳本使用：無快取會話即表示會出現登入提示。** 若在無緩存會話的情況下執行核心指令（`process`、`status`、 `export-status`、`time-sync`、…）若在無快取會話的情況下執行，會在繼續處理前透過標準輸入（stdin）要求輸入 `Email:` / `Password:`。 若標準輸入已關閉（例如 cron 或 systemd 單元），程式會立即因 `login cancelled` 而失敗；若標準輸入為開啟的終端機或管線，則會等待輸入。在排程無頭作業前，請先在每台機器上執行一次 `chloros-cli login EMAIL PASSWORD`。

---

## `chloros-cli logout`

清除快取的會話，並在下次呼叫時強制重新登入。

在處理執行中登出並不會中斷該執行：該執行將以開始時的處理模式完成，而下一次執行則會使用標準處理模式。`logout` 已說明此點（`The processing run in progress finishes in <mode> mode; the next run uses standard`），當您登出時，應用程式也會顯示此訊息 。

```bash
chloros-cli logout
```

---

## `chloros-cli status`

顯示當前授權等級（鐵級／銅級／銅級／銀級／金級）、已驗證使用者，以及裝置綁定次數。

```bash
chloros-cli status
```

---

## `chloros-cli export-status`

查詢 Thread-4 即時匯出進度。可在另一個 shell 執行 `process` 期間安全地呼叫此指令。

```bash
chloros-cli export-status
```

---

## `chloros-cli language`

設定CLI的顯示語言（支援38種語言，包含中日韓、從右至左（RTL）及印度語系）。若遇到無法渲染該腳本的舊版主控台，將平穩回退至英文。

```
chloros-cli language [LANG_CODE] [--list]
```

### 範例

```bash
# List all available languages
chloros-cli language --list

# Switch to Spanish
chloros-cli language es

# Show the currently-active language
chloros-cli language
```

---

## 專案資料夾指令

這些指令用於管理預設專案資料夾位置（與 GUI 共用）。

```bash
chloros-cli set-project-folder "/home/user/Chloros Projects"
chloros-cli get-project-folder
chloros-cli reset-project-folder
```

`get-project-folder` 會輸出新專案儲存的資料夾路徑。若已設定的資料夾不再存在（已被刪除、重新命名，或位於未連接的磁碟機上）， CLI 將與 GUI 相同，回退至預設資料夾：預設資料夾路徑會輸出至標準輸出（stdout），包含缺失資料夾名稱的警告訊息會輸出至標準錯誤（stderr），且結束代碼為 `0`。

---

## `chloros-cli update`

僅限 Linux / Jetson。檢查 `version_url` 與 `/etc/chloros/update.conf`，若存在較新版本， 便會下載對應的 `.deb` 並立即透過 `sudo dpkg -i` 進行安裝（不會出現確認提示；`sudo` 可能會要求您輸入密碼）。 在 Jetson 裝置上，僅 JetPack 6 系統會收到此更新。在 Windows 系統上，此指令不會執行任何操作，並會回報「`up to date`：請使用新安裝程式進行更新」。

```bash
chloros-cli update            # check + install
chloros-cli update --check    # check only
```

在 Linux/Jetson 環境中，CLI 也會在每次啟動時執行 **自動更新檢查** （非阻塞式，絕不會延遲指令執行）：它會讀取 `/etc/chloros/update.conf`，將結果在 `~/.chloros/update_cache.json` 中快取 1 小時，並在存在更新版本時輸出 `Update available: vX.Y.Z / Run: chloros-cli update`。 若發生任何錯誤或在 Windows 環境下，此操作將靜默跳過；設定 `CHLOROS_DISABLE_UPDATE_CHECK=1` 可關閉此功能。由於有 1 小時的快取機制，若新版本在 1 小時內發布，系統仍可能讀取為最新版本。

---

## `chloros-cli selftest`

執行 7 步驟的煙霧測試：版本、埠可用性、後端啟動、`/api/test`、`/api/system-info`（GPU/CUDA/PyTorch）、去噪器模型存在性、 CUDA+去噪器就緒狀態。此程序會使用正在運行的後端，或為測試啟動一個後端並在測試後停止它。它會輸出 `N/7 checks passed` 以及失敗的檢查項目，並僅在所有七項檢查均通過時才退出 `0`； 若電腦未配備 CUDA GPU，則 CUDA+去噪器檢查會失敗（請參閱 [驗證安裝](#verify-the-install)）。

```bash
chloros-cli selftest
```

---

## `chloros-cli time-sync`

PTP 主時鐘狀態與控制。Chloros 主機運行 PTP 主時鐘；LATTICE 攝影機與 DAQ-E 裝置作為其從屬裝置，以實現跨裝置時間戳記 （跨裝置約 1 毫秒）。PTP 負責對齊 **時間戳記**：陣列中的相機能同步曝光，是因為 M8 硬體觸發器觸發它們（模組間間隔小於 100 微秒），而非仰賴 PTP。 未帶子指令的 `chloros-cli time-sync` 會執行 `status`。

| 子指令 | 說明 |
| --- | --- |
| `status` | 顯示主時鐘狀態、BMCA 優先級、時鐘識別碼，以及其封包是否已傳送至網路：當主時鐘未運行時，執行 `health` 即為 `OK` （當主節點未運行時為 `STOPPED`），或 `SENDS_FAILING` / `DEGRADED` / `NO_SOCKETS` / `PEERS_SILENT` 並伴隨 `health_detail`， 接著會將一行 `WARNING:` 輸出至 stderr（stdout 仍維持 JSON 格式）。`sync_sent` / `announce_sent` 僅統計已發送的訊息；`*_send_errors` 與 `last_send_error` 則統計未發送的訊息。 當相機連線中斷後恢復時，總節點會自行重建其套接字（`rebinds`、`last_rebind`）。 |
| `peers` | 列出透過 Delay_Req 偵測到的從機（相機 + DAQ-E 感測器）。 |
| `cameras` | 每台攝影機的 PTP 狀態（`PtpStatus`、`PtpOffsetFromMaster`、`PtpMeanPathDelay`）。 |
| `restart` | 重新啟動主控程序。 |
| `set-priority --priority1 N --priority2 N` | 覆寫 BMCA 優先級。 |

主控程序會透過攝影機的網路介面卡進行廣播 （任何包含 `169.254.x.x` 位址的網卡）。若無網卡處於運作狀態，它僅會在預設路由的網卡為有線連接時才回退至該網卡，絕不會使用 Wi-Fi 或虛擬網卡（Tailscale、WSL），因為沒有任何攝影機或 DAQ-E 位於其後。若無內容可公告，則以無套接字模式運行（`health` `NO_SOCKETS`），而下一台連線的攝影機或 DAQ 會於當時已處於運作狀態的網路介面卡上重建該功能；`time-sync restart` 會立即執行此操作。

主控程式在後端之外，以獨立的進程運行（`own_process` / `process_pid` 在 `time-sync status` 中），在 Windows 系統上具有高優先級（`process_priority`），因此即使後端處於繁忙狀態（例如正在執行處理任務、匯出作業， 同時執行多個指令）便無法延遲其時間戳記；先前，一次這樣的延遲便可能導致所有攝影機的時鐘同時偏移。該程序在後端結束；若無法啟動，主時鐘將如以往般在後端內部運行（`own_process` 為 false，且後端日誌會如此記錄）。

### 範例

```bash
chloros-cli time-sync status
chloros-cli time-sync peers
chloros-cli time-sync cameras
chloros-cli time-sync restart
chloros-cli time-sync set-priority --priority1 1 --priority2 1
```

---

## `chloros-cli lattice`

LATTICE 攝影機控制。每個子指令皆經由 Chloros 後端路由；後端擁有攝影機池，因此後續的 CLI 呼叫會重複使用同一個已開啟的處理程序。

### 常用選項（多數子指令共用）

| 標誌 | 說明 |
| --- | --- |
| `-d, --device N` | 攝影機索引 (預設：0)。 |
| `-s, --serial SN` | 特定序列號；會覆寫 `--device`。 |
| `--serials SN1,SN2,…` | 用於多攝影機操作的以逗號分隔的序列號。[暫時性多攝影機指令](#multi-camera-transient-sessions) 僅開啟這些攝影機，其中第一個攝影機將觸發快門。 |
| `--all` | 對所有已偵測到的攝影機進行操作。 |
| `--exposure US` | 曝光時間（單位：微秒）。 |
| `--gain DB` | 增益（單位：分貝）。 |
| `--pixel-format FMT` | 例如：`BayerRG8`、`BayerRG12`；`Mono8`、 `Mono12`。此名稱會設定該參數在相機自身系列中的位元深度： 在 M3M 上，拜耳（Bayer）名稱會設定其單色（Mono）格式的深度（`BayerRG8` 設定 `Mono8`）；在 M3C 上，單色（Mono）名稱則設定拜耳（Bayer）格式的深度 （`Mono12` 設定為 `BayerRG12`）。 |
| `--width N` / `--height N` | 影像尺寸。 |
| `--preset {default,high_quality,high_speed,triggered}` | 套用設定預設值：其像素格式、曝光、增益及幀率（`high_speed`：`BayerRG8`， 5 毫秒、6 dB、60 fps；`high_quality`：`BayerRG12`，20 毫秒、0 dB；`lattice presets` 列出這些設定）。 在 M3M 上，格式為相同深度的單聲道格式：`high_speed` 擷取 `Mono8`、 `high_quality` 以及 `triggered` `Mono12`。隨之提供的標誌 （`--exposure`、`--gain`、`--pixel-format`）將優先於預設值，而預設值未列出的設定則維持相機本身的設定。所有自由運作，唯獨 `triggered` 例外，該設定會使攝影機對第 2 號線的硬體邊緣變化保持觸發狀態——若該線路無訊號驅動，攝影機將無限期等待而非進行擷取。 |
| `-o, --output DIR` | 輸出目錄 （預設：`output`）。 |
| `--packet-size {auto,jumbo,standard,N}` | GVSP 封包大小。`auto` 執行 ICMP+GVSP 探測；`jumbo` = 9000； `standard` = 1500。 |

### LATTICE 攝影機首次連線工作流程

```bash
# 0. A backend must be running: the Chloros desktop app (Windows),
#    `chloros-backend`, or the Linux service (see "Start the Backend")

# 1. Discover cameras on the network
chloros-cli lattice info

# 2. Single-cam smoke test: capture one frame.
#    By default this saves EVERY export type applicable to the cam
#    (raw, debayered, radiance, reflectance, preview). Pass e.g.
#    `--processing debayered` to save just one.
chloros-cli lattice capture -o output/

# 3. Connect a synchronized array (RECOMMENDED ENTRY POINT for arrays).
#    This is the same "smart-prep" flow the Chloros GUI uses:
#      - Network capability probe (ICMP DF ping + GVSP probe)
#      - Tier auto-pick (sim-emit / ftd-stagger / slip)
#      - Full sensor (no binning, no ROI); the frame rate falls to
#        what the host and wire deliver, and the faster options are
#        printed
#      - PTP enabled by default
#      - Per-cam pixel format auto-pick
#      - AE seeding from the cam's saved state
#      - GPIO trigger config on Line2
chloros-cli lattice array-connect \
  --serials 213800234,214000533,214701288,214701292

# 4. Capture one synced frame group from the live array.
#    Defaults to --processing all (one file per export type per cam);
#    pass a single level to narrow it, e.g. --processing reflectance.
chloros-cli lattice array-capture --processing reflectance -o output/

# 5. Live-preview one cam in your browser
chloros-cli lattice viewer --serial 213800234

# 6. Tear down when done
chloros-cli lattice array-disconnect
```

### 子指令參考

#### 偵測與資訊

| 子指令 | 用途 |
| --- | --- |
| `lattice info` | 列出已連線的攝影機（廠商、型號、序號、IP、MAC）。 |
| `lattice info` | 分析主機系統以取得最佳攝影機配置。`--no-discover` 會跳過攝影機偵測（速度較快，僅分析網路介面卡）。 |
| `lattice network [--fix] [--estimate] [--cameras N] [--pixel-format FMT]` | 檢查／修正網路介面卡設定（在 Linux 上，亦包含核心的套接字接收緩衝區 — 參見 [Linux 主機](#linux-hosts--socket-receive-buffers))；估算頻寬／每秒幀數。 |
| `lattice network-analysis --master SN [--slaves SN1,SN2,…] [--width N] [--height N] [--pixel-format FMT] [--binning {1,2,4}] [--force-tier TIER] [--packed {auto,on,off}] [--backend-url URL] [--json]` | 穩定架構後端網路-能力 + 陣列建議（返回 `status` ∈ `ok` / `auto_shrunk` / `auto_capped_fps` / `needs_force_slip` / `error`)。`--width` / `--height` 預設為完整感測器， `2048` × `1536`。它會為 `ok`、`auto_capped_fps` 以及 `auto_shrunk`，並將 `1` 切換至 `needs_force_slip` 和 `error`， 無論是否包含 `--json`。它會傳遞至後端時設定為 `--backend-url`（預設為 `http://127.0.0.1:5000`），而非 `CHLOROS_BACKEND_URL`。 `auto_capped_fps` 會保留請求的解析度，但會限制目標每秒幀數（fps）——讀取 `recommended.recommended_target_fps` 並將其作為連線目標傳遞；將其視為成功， 而非錯誤。`auto_shrunk` 僅在無法以任何幀率傳送所請求的幀時才會出現；connect 絕不會自行縮減幀。 |
| `lattice analyze-array [--models M1,M2,…] [--binning {1,2,4}] [--n-active N] [--width N] [--height N] [--pixel-format FMT] [--force-tier TIER] [--packed {auto,on,off}] [--json]` | 無需開啟攝影機的假設分析（`--width` / `--height` 預設為 `2048` × `1536`）。此設定與 `network-analysis` 皆會預測 connect 將串流的內容：若未設定 `--pixel-format`， connect 所套用的每台相機格式（Mono12 / BayerRG12）；而當使用 `--packed auto`（預設）時，則會顯示 [12 位元壓縮](#packed-12-bit-transport---packed）——因此所列出的 fps 即為該陣列的運行速率。兩者皆會輸出封裝設定。**`--n-active` 代表線路上所有攝影機的總數，而不僅限於此陣列**— 當獨立攝影機同時串流時，或線路預算的計算所依據的需求低估了攝影機數量時，請提高此數值（預設值：`len(--models)`）。 始終會輸出彙總的 `Wire budget:`（需求 MB/s 與防衝突上限）和 `Max cameras:` 行，並在陣列對線路超額訂閱時標記 `** OVER-SUBSCRIBED**` — 請參閱 [陣列 fps 與突發傳輸模型](#array-fps--burst-model)。 |
| `lattice gpu` | 顯示 GPU 狀態。 |
| `lattice firmware [--update] [--force] [-y\|--yes]` | 檢查或更新攝影機韌體。 本地 `.fwa` 選項已鎖定：當 `firmware/<MODEL_PREFIX>/` 中存在與該建置的 `MIN_FIRMWARE_VERSION` 相符的檔案時，該檔案會被燒錄 （僅將最高版本作為備用方案），因此儲存於磁碟上的較新供應商映像檔在該固定選項被解除前將保持閒置狀態——較新的版本會透過簽名的 AWS 清單傳送至裝置，若版本較新，建議優先採用此方式。 |
| `lattice presets [--apply NAME]` | 列出或套用相機預設值。`--apply` 設定預設值的名稱（其像素格式、曝光、增益、幀率、觸發條件），並在後續讀取時將這些設定以相機讀取的格式輸出： 在 M3M 上，預設值的深度採用 Mono 格式；其運作的幀率，可能低於預設值所要求的數值（`high_speed` 要求 60 fps；但透過 1 Gbps 鏈路連接的全畫幅相機實際運行幀率約為 35）。 其「open」指令寫入的其他所有設定，均會按原始狀態還原。`--apply default` 會將相機重置為出廠預設值：即 Line2 上的自由運行自動曝光、關閉鏈路吞吐量限制與缺陷校正，並保留 RGB 相機的校準白平衡（其他類型則設為統一值）。 任何設定均不會儲存至相機記憶體中，因此下次開機時，相機將如先前般從其自身記憶體中載入設定。作為擷取或連線指令的 `--preset`，`default` 並非預設值。 |
| `lattice status` | 即時攝影機狀態。 |

#### 擷取

| 子指令 | 用途 |
| --- | --- |
| `lattice capture [--format tiff\|tif\|png\|jpg\|jpeg] [--jpeg-quality N] [--processing LEVEL] [--levels L1,L2,…] [--no-daq] [--daq-tolerance-ms MS]` | 單幀。 **預設會儲存所有匯出類型**(`--processing all`)；請參閱 [擷取匯出層級](#capture-export-levels-the-all-default)。 `--processing` 可搭配 `all`、`raw`、`debayered`、 `radiance`、`reflectance` 或 `display` 作為參數（`preview` 僅在 `display`) 亦被接受；在觸碰相機之前，任何其他名稱均會被拒絕（退出 `1`）。`--levels` 會儲存一個明確的子集（覆寫 `--processing`）。 若為擷取而開啟一台Chloros未持有的攝影機，其曝光設定將首先依據場景設定（最多數秒），如同 [暫時性指令](#multi-camera-transient-sessions) 所為； 無論擷取如何結束，該攝影機都會被恢復至初始狀態（格式、曝光、色彩處理流程，以及擷取過程中變更的所有設定——詳見 [多攝影機（暫存會話）](#multi-camera-transient-sessions) 下的清單）。 用於讀取該擷取的 DAQ 讀數將儲存為 `.daq` 附檔，其中包含僅限原始資料的擷取檔；可透過 `--no-daq` / `--daq-tolerance-ms` 進行調整 （[DAQ 讀取擷取時會儲存](#the-daq-reading-a-capture-saves)）。`--force-daq` 仍被接受。`--jpeg-quality` = JPEG 品質 1–100 （預設 95）。**儲存至開啟中專案的檔案將加入該專案。** 如同應用程式中的運作方式，若擷取任務的檔案儲存於 Chloros 中開啟的專案資料夾內，系統會將其加入該專案的檔案清單；儲存於其他位置的檔案則不予處理。該指令會輸出一行訊息說明此情況 (`Project: ...`)，或者，若處理程序正在執行中，則會顯示這些檔案已被暫緩處理，並將於處理結束後加入。`multi-capture`、`array-capture` (`--smart` 亦然) 以及 `project capture` 皆屬此類。 |
| `lattice continuous [--format tiff\|tif\|png\|jpg\|jpeg] [--jpeg-quality N] [--queue-depth N] [--no-daq] [--daq-tolerance-ms MS]` | 持續將資料串流至磁碟，直至按下 Ctrl+C 為止。一次 `tiff` 執行的幀即為 `raw` 檔案，這是該攝影機陣列擷取的寫入內容：感測器的馬賽克圖，每個平面皆以自身數值呈現，未經壓縮，並附帶校準封包 （無論場景為何，一個 2048 x 1536 的 12 位元幀皆為 6.0 MiB 加上該封包），因此 `process` 即以此為起點。 `png` 和 `jpg` 各佔 8 位元，因此這些資料流是經去馬賽克處理、可供檢視的影像幀，而 `process` 並非以此為起點。若採用 DAQ 綁定 （`daq pool-bind`）或採用幀池化時，每個幀都會在該運行序列的 `daq/` 資料夾中擁有專屬的 `.daq` 和 `.match.json`， 因為 `lattice capture` 會儲存一個；`--no-daq` 會關閉此功能，而停止時會輸出總幀數。每個幀都會寫入一次，並包含完整的元資料； 按下 Ctrl+C 時，會在釋放相機之前先寫入已擷取的幀。當儲存裝置空間已滿時，系統會自動停止運作，並輸出 `Failed:` / `Disk full:` 並退出至 `1`；若任何幀未能寫入，亦會退出至 `1`。若相機處於硬體觸發待命狀態 （`TriggerMode` `On`）的相機，在本次運行期間將設定為自由運行模式。當運行結束時，相機將恢復至初始狀態——其觸發設定以及運行期間變更的各項設定 （曝光、增益、格式等）—— 除非發生 `--keep-settings`（將運行設定儲存至相機自身記憶體中，即自由運行模式），或 `--clear-settings`（將該記憶體重置為出廠設定）。 若運行期間未錄製任何幀，即視為失敗：若連續 10 秒無幀，系統將自動停止，輸出包含原因的 `No frames:` 訊息，並以 `1` 狀態退出。 寫入器無法跟上的幀將被計為丟失，停止時會輸出 `Dropped:` 並說明原因（包括相機的幀率、寫入器的幀率以及佇列幀率）；若採用較低的幀率或 8 位元像素格式，則可完整保留所有幀。 此指令每 2 秒向後端進行一次狀態回報；若在未停止錄製的情況下終止運作（`kill -9`，記憶體不足），後端將在最後一次狀態回報後 30 秒停止錄製，並將攝影機恢復至 （`stop_reason` `client_gone`，並在後端日誌中記錄一行）。`multi-stream` 和 `align-stream` 亦執行相同操作。 若啟動時攝影機的開啟時間超過指令的等待時間（60 秒；`multi-stream` / `align-stream` 則為 180 秒）——即攝影機仍在開機中——後端會撤銷該操作： 攝影機將被還原並關閉，且不會留下任何正在運行的錄影（HTTP 504 `CLIENT_DEADLINE_PASSED`）。|
| `lattice viewer [--brightness N] [--ae-damping F] [--frame-rate FPS]` | 瀏覽器端 MJPEG 即時預覽：連線至攝影機，輸出串流URL並在您的瀏覽器中開啟。`--ae-damping` 設定自動曝光衰減（0.4–100）。 |

##### 即時串流（MJPEG；不支援 RTSP）

Chloros 透過 **HTTP 傳輸 MJPEG**（`multipart/x-mixed-replace`，邊界條件 `frame`）傳輸即時影片。**不支援 RTSP。** 在瀏覽器中開啟該URL，或使用任何 MJPEG 客戶端（OpenCV 的 `cv2.VideoCapture(url)`、ffmpeg、VLC）讀取該串流。攝影機必須先在後端連接好（`lattice cam-connect`、`lattice array-connect`、`lattice viewer` 或 GUI）。

| URL（`http://127.0.0.1:5000` + …） | 串流 |
| --- | --- |
| `/api/camera/stream/<serial>` | 預覽畫面，最高約 30 fps。若為陣列成員， `?max_h=`（預設值為 `720`）與 `?jpeg_q=`（預設值為 `80`）分別設定高度與 JPEG 畫質。 這是 `lattice viewer` 所開啟的 URL。 |
| `/api/camera/<serial>/stream-annotated` | 相同預覽畫面，但繪製了 GUI 的疊加層： `zebra=1`、`crosshair=1`、`grid=3x3`（或 `9x9`）， `histogram=1`、`peaking=1`、`spot=1`；若未進行查詢，則採用該相機在專案中儲存的疊加層設定。 |
| `/api/camera/<serial>/stream-reflectance` | 每波段 8 位元反射率：ρ = 像素值 ÷ 該部分`X-Chloros-Pixel-Scale` 標頭（除非觸發 `refl_max`，否則為 255）。`fps`（預設值為 `10`）， `jpeg_q`（預設為 `95`）、`max_h`（預設為 `0` = 原生）， `band=<name>`、`align=1`。JPEG 格式，因此為有損壓縮。當相機無反射率時（HTTP HTTP 409 狀態），將被拒絕（RGB 相機、無校準包、無 DAQ 讀數）。 |

- **跨相機幀對齊。**每個**陣列成員** 在所有三個資料流中傳送的每個部分，皆攜帶 `X-Chloros-Group-Id`（該陣列的同步群組計數器）、`X-Chloros-Frame-Id` 及 `X-Chloros-Timestamp-Ns` 標頭。來自不同相機但具有相同 `X-Chloros-Group-Id` 的部分，源自同一觸發事件。獨立運作的相機沒有群組，也不會傳送群組 ID。 瀏覽器會忽略各部分的標頭；由客戶端自行解析多部分串流時，則會讀取這些標頭。
- **當陣列檢視開啟時暫停。** 當陣列的合併即時串流開啟時（例如 GUI 的陣列檢視），陣列成員的 `/api/camera/stream/<serial>` 會傳送靜態的「暫停」幀，並在關閉時恢復運作。

#### 感測器調校

| 子指令 | 用途 |
| --- | --- |
| `lattice configure [{--get N1 N2… \| --set N=V N=V… \| --dump}] [--json]` | 讀取／寫入任何 GenICam 節點。 `--set ExposureTime=…`（`Gain=…`）若未先透過其 `ExposureAuto`（`GainAuto`）開關將自動模式關閉， 如同 `lattice exposure --set` 所做的那樣；若相機拒絕該值，則會在標準錯誤輸出 (stderr) 上印出 `Failed to set <node>: …` 並退出 `1`。`--get` 以及 預設分組檢視會在自動模式運行期間，於 `ExposureTime` / `Gain` 下輸出 `note:` — 請參閱 [在自動模式下讀取曝光與增益](#reading-exposure-and-gain-under-auto-mode)。 |
| `lattice exposure [{--auto \| --auto-once \| --off \| --set US}] [--brightness 0-255] [--brightness-auto] [--damping 0.4-100] [--upper-limit US]` | 曝光與自動曝光。系統會將 `--set` 曝光值或 `--brightness` 目標值視為您的設定 （Chloros 的自動曝光控制不會干預）；增益維持原值，因此在自動模式下，系統會持續朝目標值調整，直到您設定 `lattice gain` 為止。 `--auto` 會將曝光控制權交還，`--brightness-auto` 則將目標交還給 Chloros 的自動目標設定。若未設定任何標記，系統會讀取曝光節點， 其行為與 `note:` 相同。若相機拒絕某個 `--set` 值，則會在標準錯誤輸出 (stderr) 上印出 `Failed to set ExposureTime: ExposureTime=<value> refused (<camera error>); the camera accepts <min>..<max> now (ExposureAuto put back to <mode>)` — 當寫入操作關閉自動模式時，系統會恢復自動模式而非維持關閉狀態 — 不會輸出 `Exposure: … us (auto OFF)` 行，並退出 `1`。 |
| `lattice gain [{--auto \| --off \| --set DB}]` | 增益與自動增益。 若 `--set` 值超出相機允許範圍，則會在標準錯誤輸出 (stderr) 上輸出 `Failed to set Gain: …`，不輸出 `Gain: … dB (auto OFF)` 行，並退出 `1`。 |
| `lattice resolution [--set WxH] [--offset X,Y] [--binning N]` | 感測器 ROI 與像素合併。像素合併方式始終為 **平均** — 詳見下文。 |
| `lattice format [--set FMT] [--list]` | 像素格式。 |
| `lattice trigger [--mode On\|Off] [--source SRC] [--delay-us US] [--activation EDGE] [--list-sources] [--software]` | 硬體／軟體觸發。 |
| `lattice white-balance [--auto] [--off] [--red R] [--blue B]`（無標誌 = 單次白平衡） | 白平衡操作。 僅適用於 RGB／拜耳感光元件相機；在單色 M3M 上為無操作（跳過）。單次運算會保留其計算出的白平衡（`BalanceWhiteAuto` 及增益值）；其餘所有設定的開放寫入值將維持原狀，若單次運算無法執行，則所有設定均維持原狀。 |
| `lattice color-profile [--set raw\|linear\|natural\|enhanced\|custom_temp] [--cct 2000-10000] [--get]` | RGB顯示色彩處理管道。`natural`（預設）為簡易即時處理模式； `enhanced` 則會加入去色邊、鮮豔度及 CLAHE 局部對比度，以呈現完整的 Hub-Parity 效果，每幀處理成本約為 2 倍，因此會導致較低的 **即時** 幀率——無論如何，儲存的擷取畫面始終會獲得完整的處理效果。僅限 RGB／Bayer 感光元件相機； 在單色 M3M 上會被跳過。 |
| `lattice color [--saturation 0-200] [--contrast 0-200] [--reset] [--get]` | 即時預覽飽和度／對比度（RGB濾鏡相機）；`100` 為中性設定，`--reset` 將兩者皆設為 100。 已儲存的擷取畫面不受影響。在單色 M3M 上會被跳過。 |
| `lattice highlights [{--set -100..0 \| --reset \| --get}]`（無標誌 = 讀取） | 顯示 **高光**（RGB濾鏡相機）：將最亮、最白的像素調暗 — 例如，原本明亮的白色表面會呈現為平淡的白色塊狀。`0` = 關閉， `-100` = 最強。即時預覽與顯示級別的擷取畫面會遵循此設定；輻照度／反射率則永遠不會，且無法恢復已剪裁的感測器資料（請降低該情況下的曝光值）。在單色 M3M 上跳過此設定。 |
| `lattice filter [--list]` | 列出 Chloros 所知的濾鏡型號（`RGN-IMX265`、`OCN`、`NGB`、…）。相機的濾鏡資訊是從其型號字串中讀取的，無法手動設定：`--set NAME` 僅會查詢一個型號，隨後會以 `1` 退出並傳回 `setting the filter model is not supported`。 |
| `lattice power [--sleep]` | 偵測電源／熱節點；切換低功耗閒置模式。 `--sleep` 將相機置於睡眠狀態（其鏈路吞吐量限制已啟用（`DeviceLinkThroughputLimitMode` `On`）並設定為最低值，若相機具備節能模式則啟用該模式） 其餘所有設定均恢復為最初檢測到的狀態； 若休眠失敗，則將所有設定原樣還原。喚醒時會將鏈路恢復至休眠前狀態（包含傳輸速率上限、節能模式及封包延遲 `GevSCPD`——當傳輸速率上限啟用時，攝影機會將此值維持為 0；相關記錄會保留在此電腦上， 因此後端重新啟動後的喚醒操作也能找到這些設定），或者，對於這台電腦未將其置於睡眠狀態但看似已進入睡眠的攝影機（其限制值設於下限），在關閉限制且將 `GevSCPD` 維持在 0 並將限制值設為 80 （出廠設定）；若限制值設定在底限以上，則維持原狀。探針與喚醒操作會將其開啟時寫入的內容還原。 |

> **像素合併模式始終為「平均」。** `lattice resolution --binning N` 設定「平均」像素合併模式，且
> 不提供模式切換選項。 「總和」分桶會將每個動態數值（DN）乘以 N²，每步分桶大約會消耗兩檔的
> 高光餘裕；Chloros會在
> 輻射測量鏈之前將該係數除回，但被「總和」模式驅動至感光元件上限的像素會遭剪裁，且無法
> 恢復。 這並未帶來「平均」模式所沒有的優勢——指數是尺度不變的，且輻照度
> 標量已在「平均」尺度上進行校準。若相機預設為「總和」模式（由
> 其他工具設定，或因工廠重置所致）， Chloros 會透過節點讀回偵測到此狀態並
> 進行補償；執行 `lattice resolution --binning N` 即可將其恢復為「平均」模式。1.2.0
> `--binning-mode` 標誌仍被接受，因此舊版腳本仍可執行，但僅會顯示警告：
> 像素合併仍維持「平均」模式。

> **若在相機上變更分桶、像素格式及觸發設定，任何設定皆無法維持。** 當相機的
> 後端保持開啟並進行串流時（`cam-connect`，即 GUI），`lattice resolution --binning`
> 均無法生效（系統會顯示目前仍生效的像素合併模式，`Binning: 1x1`），且
> `lattice format --set` 與 `lattice trigger --mode` 會被相機拒絕
>（`SC_ERR_ERROR -1001`）。 請先釋放該設定（`lattice cam-disconnect -s <SN>`），或於
> 連線時設定（`cam-connect --pixel-format …`、`array-connect --binning …`）。

##### 在自動模式下讀取曝光與增益

當攝影機的 `ExposureAuto` (`GainAuto`) 正在執行時，其 `ExposureTime` (`Gain`) 節點會保留 **上次寫入的值**，而非相機當前的運作狀態。 對於後端保持開啟的攝影機（`cam-connect`、`array-connect`、GUI），`lattice configure --get`、預設視圖 `lattice configure` 以及 `lattice exposure`（無標誌）因此會輸出該節點的值，並在其下方顯示一則註記，內容為攝影機的最新畫面：

```

ExposureTime = 2000.448
  note: ExposureAuto is Continuous: ExposureTime holds the last value written (2000.448); the camera's newest frame was at 4900 us.
```

請從註解中讀取幀值；節點值僅為最後一次寫入的內容。透過 HTTP，相同的 `POST /api/camera/<serial>/read-nodes` 回應會保持 `nodes` 不變，並新增 `frame_values` （即 `{"ExposureTime": <us>, "Gain": <dB>}`，最新幀的值）以及 `notes`（即上文所述的值）。這兩者僅在自動模式非 `Off` 且已知有幀時才會出現； 僅為讀取而開啟的攝影機沒有幀，因此不會取得這兩者。`lattice gain`（無標誌）僅輸出節點——若需輸出註解，請使用 `lattice configure --get Gain`。

##### 陣列成員的曝光

當陣列進行串流時，每個成員皆以 **`AcquisitionFrameRateEnable`**運行，幀率略高於陣列的觸發率：其週期為觸發週期減去 15%（至少 3 毫秒），或當觸發率已接近相機自身最大幀率時，則採用該最大幀率。 因此，陣列成員上的**`configure --get`**在此設定下會讀取**`AcquisitionFrameRateEnable`**為**`True`**及**`AcquisitionFrameRate`**；當陣列 停止時，啟用狀態會重新關閉。**請勿在陣列成員上寫入這兩個節點** — 陣列會使其與觸發訊號保持同步，並重新寫入這些值。

該設定同時也是成員的曝光上限：在 10 fps 觸發（100 ms 週期）下，無論**任何 ROI 或像素合併模式**，約為 85 ms下皆然。在 1.3.0 版本之前，若設定小型 ROI，則會將成員的曝光時間限制在感測器的自由運行週期內（768 × 304 解析度下為 4.9 毫秒），無論陣列觸發速度多慢。**若手動設定的 `ExposureTime` 時間超過此上限，系統會將觸發速率調低以配合該設定**，當該設定透過 `lattice exposure --set` 或 `configure --set`（或從 GUI）寫入時。陣列會先為該成員的上限值開啟該範圍，並讓較慢的速率運行幾個觸發週期，因此寫入操作可能會多花幾秒鐘；隨後觸發速度將不會快於曝光所允許的範圍。在自動速率陣列中，當每個曾因手動曝光而減慢速率的成員恢復為自動曝光後，陣列便會恢復先前運行時的速率。 若數值超出相機本身可接受範圍，系統仍會拒絕該數值，並在錯誤訊息中標示其可接受範圍；若該成員的自動曝光僅因該次寫入而被關閉，則會恢復其自動曝光設定。配方 `apply` 的曝光不會減慢觸發速度：觸發速度將維持在上限值。 Chloros 絕不會為了配合更快的觸發頻率而縮短手動曝光時間；若固定速率（`array-connect --target-fps F`）快於曝光所允許的範圍，該成員的觸發間隔將維持足夠長的時間以配合其曝光，且該成員的觸發頻率不會快於其曝光所允許的範圍。

#### 校準與感測器

| 子指令 | 用途 |
| --- | --- |
| `lattice calibrate [--filter NAME] [--attempts N] [--save PATH]` | 根據反射率標靶進行校準。系統會搜尋最多 `--attempts` 幀（預設為 `30`）； 不完整的幀（GVSP 封包遺失）會消耗一次嘗試機會，而非整個執行次數。若幀未完整送達，則在 `no target detected in N attempts`、 若無完整幀抵達，則在 `no complete frame in N attempts: <last error>` 時失敗。|
| `lattice dls {--connect \| --spectrum \| --irradiance} --mac MAC [--filter NAME] [--camera SERIAL] [--json]` | DAQ-M 的頻譜與頻帶幅度， 透過後端以藍牙 (BLE) 讀取。`--mac` 是感測器本身的 BLE MAC 位址，執行以下三項操作時皆需提供：此處無預設值，因每台裝置皆有其專屬值（`daq pool-discover --only ble` 會列出這些值）； 若未提供此參數，指令將退出（`1`）。若未指定任何操作，則顯示其說明。`--camera SERIAL` 會透過該相機自身測得的光譜響應（而非濾光片模型的標準曲線）來整合已列印的波段幅度 ——RGB 相機，且該相機的校準包已載入（已連接）或快取於本電腦中。標題會標示使用的是哪條曲線：`Band magnitudes (camera <SERIAL>):`，或當相機無曲線時（非 RGB 相機、 或未載入／快取校正套件）時，則會顯示「`Band magnitudes (camera <SERIAL>):`」；此時將改用預設曲線。透過 HTTP 傳輸時，`POST /api/dls/cli-read` 會接收 `camera_serial` 並回傳 `band_magnitudes_source`。 |
| `lattice vignette --input DIR --output DIR [--lens-model KEY]` | 對現有影像套用暗角校正，每張影像皆採用其鏡頭的平場 （取自其 EXIF 型號）。`--lens-model` 適用於型號未指定相機的影像；若影像的型號與此設定相牴觸，則會跳過該影像。 |

#### 多相機（暫存工作階段）

暫存命令會在單次執行期間開啟相機，並於執行結束時關閉：此處的 `multi-capture` 未連接持久性陣列，以及 `multi-stream`、`multi-test`、 以及下方的 `align-calibrate`、`align-apply`、`align-stream` 和 `index --live`。以下規則適用於所有上述指令：

- **`--serials SN1,SN2,…` 負責選擇攝影機。** 僅會開啟這些攝影機，且第一台開啟的攝影機將觸發扳機 （若指定了 `align-calibrate`、`--reference` 則適用此規則）。若序列號未在網路中，則在任何攝影機開啟前，執行程序即告失敗。若未指定 `--serials`，網路中的所有攝影機都會被開啟。 若Chloros已持有某台攝影機（GUI、`cam-connect`、`array-connect`），無論`--serials`是否指定該攝影機，該攝影機皆會拒絕執行： 因為該攝影機與執行任務所針對的網路相同。（若未啟用 `--serials`，`index --live` 則會開啟其設定檔中已啟用的攝影機：請參閱 [索引 / 植被數學](#index--vegetation-maths)。）
- **這些攝影機共享網路的方式，與 `array-connect` 共享網路的方式相同。** 每台攝影機的傳輸速率皆依主連線承載量的分配比例而定（此為封包間延遲，而非攝影機的吞吐量上限——在 1 GbE 網路上，四台攝影機的吞吐量早已超過 31.25 MB/s 的下限）， 因此四台攝影機可運行於 1 Gbps 鏈路上。當攝影機無法以各自達到的最慢速率同時傳輸時，它們會依次觸發，每台攝影機皆獨佔鏈路， 此時指令會輸出 `Serial emit: the wire cannot carry these cameras at once, so each is triggered <ms> ms after the one before; their frames are not exposed together`。若任何時刻即使僅有一台攝影機無法符合傳輸條件，系統會在任何攝影機開始串流前即拒絕執行： `array over-subscribes the wire: <n> cameras demand <x> MB/s but the collision-safe ceiling is <y> MB/s. …`，HTTP 409，退出 `1`。
- **每台攝影機都必須傳輸畫面，否則執行即告失敗。** 未傳輸任何畫面的攝影機即屬錯誤： 該指令會輸出 `Error: … no frame from <n> of <total> cameras: SN:<serial> (<why>), …`，其中 `<why>` 代表該攝影機的擷取錯誤，接著顯示錯誤原因，並以 `1` 退出。 若接收到伴隨 GVSP 封包遺失（`IncompleteBufferError: GVSP packet loss …`）的幀，表示網路無法承載所有攝影機的總合頻寬（`Every missing frame arrived with GVSP packet loss: …`）：檢查主機及交換機上行鏈路的連線速度（`lattice network`），啟用巨型幀， 或減少開啟的攝影機數量（`--serials`）。若某台攝影機未傳回任何資料，通常是因為未接收到主控器的觸發訊號：請檢查其 M8 同步線纜，然後執行 `lattice multi-detect`。當幀組中缺少某台攝影機時，各指令的處理方式詳見其對應行。
- **每台攝影機將恢復至原始狀態。** 執行結束時，每台攝影機都會恢復至執行開始時的狀態：`TriggerMode`、`TriggerSource`、 `TriggerActivation`、`TriggerOverlap`、`TriggerDelay`、 以及位於 `Line2` 和 `Line3` 上的 `LineMode` / `LineSource` （以及陣列對從機線路提供的觸發輸入去毛刺功能，`LineFilterSelector` / `LineFilterWidth`）； 其 `PtpEnable`（以及與之配對的 `PtpSlaveOnly` / `PtpDomainNumber`）； `ExposureAuto` / `GainAuto` （以及 `ExposureTime` / `Gain`，其中採用手動設定值）；`TargetBrightness`、`ExposureAutoLimitAuto` / `ExposureAutoUpperLimit` / `ExposureAutoLowerLimit`， `GainAutoUpperLimit`、`ExposureAutoDamping`；`BalanceWhiteAuto`（以及所有白平衡設定為手動的 `BalanceRatio`）；`BlackLevelAuto` / `BlackLevel`；`PixelFormat`（採用像素合併技術），`Width` / `Height` / `OffsetX` / `OffsetY`；`GammaEnable`（以及啟用伽瑪校正的 `Gamma`）， `LUTEnable`、`ColorTransformationEnable`；`ReverseX` / `ReverseY`；`UserSetSelector`； 封包大小（`GevSCPSPacketSize` / `DeviceStreamChannelPacketSize`）；傳輸節奏，`DeviceLinkThroughputLimitMode`（以及當模式為 `On` 時，`DeviceLinkThroughputLimit`），各攝影機的封包間隔延遲 `GevSCPD` 以及起始錯開時間 `GevSCFTD`，陣列會為每台攝影機提供；每個串流啟用的區塊資料 (`ChunkModeActive`，每個 `ChunkEnable` 皆位於其對應的 `ChunkSelector` 之下)；`AcquisitionFrameRateEnable` （以及 `AcquisitionFrameRate`，當啟用狀態為開啟時——若關閉，該速率即為唯讀，且讀取值以節奏允許的最高值為限，因此並非設定項）。 僅寫入不同之處，相機先停止運作，並在需要幀週期的曝光前設定步進頻率與幀率（相機將 `ExposureTime` 限制為一個幀週期）。 在 `--keep-settings` 模式下，觸發與同步線連接將恢復原狀，相機處於自由運行狀態，其餘設定將依照運行設定儲存至相機自身的記憶體中；`--clear-settings` 則會將工廠預設值儲存至該處。
- **曝光設定取自保留首幀前的場景。** 拍攝過程會擷取同步的幀組並將其捨棄，同時根據這些幀設定每台相機的曝光值與增益。目標是使相機目標區域的平均亮度不超過 1% 的幀發生剪裁；若場景中存在明亮的高光區域，則會略低於目標值。 此過程通常需 2 至 6 組幀，最多耗時 5 秒。
  - 每台相機皆透過其濾鏡，對準由相連陣列所設定的起始目標：多光譜相機（RGN、OCN、NGB）的目標亮度設定為 **40**，RGB 相機則設定為**60**。 這相當於曝光目標值 80 的二分之一（RGB 則為四分之三），因此高亮度反射板發生剪裁的可能性較低。`--brightness N` (0-255) 則會分別設定每台相機的目標值（`multi-test` 沒有 `--brightness`：其自動曝光功能已關閉）。 後端日誌會為每台相機的目標命名：`[transient-ae] AE target per camera (filter default): SN:<serial> <target>, …`。
  - 曝光值優先，上限為相機自身的自動曝光極限。增益僅用於補足剩餘部分。
  - 計算出的曝光值與增益值會保留整個運行期間，因此 `multi-stream` 的每一幀皆以相同的曝光值拍攝。若需跟隨光線變化進行擷取，應使用相連的陣列（`array-connect`）， 其自動曝光會持續調整。
  - `--exposure` 和 `--gain` 將完全維持原設定。 若兩者皆已指定，則不會捨棄任何一幀畫面。
  - 當拍攝結束時，每台相機的自動曝光功能將重新啟用。
  - 後端日誌會回報結果：`[array-ae] exposure solved in <s> s (<n> warm-up groups): SN:<serial> <ms> ms <dB> dB mean <m>/<target>, …`，或附帶原因的 `exposure NOT solved after …`，以及未成功每台相機的 `(not settled)`。在此情況下，畫面仍會被拍攝。
  - `CHLOROS_TRANSIENT_AE_SETTLE_S` 會變更 5 秒的限制；`0` 則會關閉此功能（請參閱 [環境變數](#environment-variables)）。

**當 Chloros 持有任何攝影機**（包括 GUI、`cam-connect` 或已連接的陣列）時，`multi-stream`、`multi-test`、 `multi-detect`、`align-apply`、`align-stream` 以及 `index --live` 將拒絕執行，因為它們會開啟網路上的所有攝影機，而若嘗試開啟 已佔用的攝影機，可能會導致後端當機。（當 `index --live` 欲開啟的攝影機已被佔用時，它會拒絕執行：無論是其 `--serials` 自身、其設定檔中的攝影機，或是任何未指定任何攝影機的名稱之設定檔所對應的攝影機。） 後端會回傳 HTTP 409 錯誤（`CAMERA_HELD`），列出每台被佔用的攝影機及其釋放指令 （`chloros-cli lattice cam-disconnect --all` / `chloros-cli lattice array-disconnect --all`），並終止該指令 `1`。 若已連接陣列，`array-capture --continuous` 會將其記錄在 `multi-stream` 的位置，而 `array-capture` （預設為對齊狀態）則會將一個對齊的集合儲存至 `align-apply` 的位置；`multi-capture` 和 `align-calibrate` 會使用一個連接的陣列來存放其相機，而非另行開啟相機。

| 子指令 | 用途 |
| --- | --- |
| `lattice multi-info [--serials SN1,SN2,…]` | 列出所有具有同步角色的攝影機；若搭配 `--serials`，則僅列出該類攝影機，而 `MASTER` 則僅列出第一台。 |
| `lattice multi-capture [--serials SN1,SN2,…] [--format FMT] [--jpeg-quality N] [--processing LEVEL] [--brightness N] [--no-daq] [--daq-tolerance-ms MS]` | 從每台攝影機擷取一幀同步畫面。當連接了持久性陣列時，預設會儲存 **所有匯出類型**，並保留其所使用的 DAQ 讀數（[擷取畫面所儲存的 DAQ 讀數](#the-daq-reading-a-capture-saves)）； 而無陣列的暫時性備用模式則**僅進行去拜爾處理**（若需處理其餘內容，請先執行 `array-connect`）。 `--brightness` 僅適用於備用模式：已連接的陣列將保留其自身的自動曝光設定。備用模式不會儲存 `.daq` （`--no-daq` / `--daq-tolerance-ms` 在此處無效），且 `multi-capture` 不採用 `--force-daq`。在備用模式下， 若攝影機未傳送任何幀，則擷取失敗（`Error: multi-capture failed: no frame from …`，退出 `1`），且該幀組的檔案將不會被儲存。 |
| `lattice multi-stream [--serials SN1,SN2,…] [--fps F] [--count N] [--format FMT] [--jpeg-quality N] [--brightness N]` | 串流同步幀（暫時性）， `--fps` 預設 `3`，直至按下 Ctrl+C 或 `--count` 群組為止。若某台攝影機未傳送任何畫面至該幀群組，則該群組不會被寫入， 且連續三個幀組將終止串流。若任何幀組被跳過，執行程序將以輸出 `Incomplete: <n> frame group(s) not written, a camera delivered nothing (SN:<serial> x<count>, …)` 結束，並在終止串流時輸出 `Error: the stream stopped: …` 標示該攝影機名稱，隨後退出 `1`X。當儲存裝置滿載時會自動停止，輸出 `Failed:` / `Disk full:` 並以 `1` 退出； 若遇任何無法寫入的檔案，亦會退出 `1`。 |
| `lattice multi-test [--serials SN1,SN2,…] [--count N] [--ptp-settle SECONDS]` | GPIO 同步時序測試（瞬態），`--count` 預設 `10` 觸發條件。輸出網路共享方式 （`Wire: each camera paced to <x> MB/s (<n> on a <y> MB/s ceiling, <link> Mbps link)`，以及當攝影機依序觸發時顯示的 `Serial emit:` 行，此時時間戳記會包含其錯開的間隔），每個同步群組對應一行 `Sync group: master SN:<serial> -> SN:<serial>, …`（`(no slaves: not synced)` 代表未連接同步線纜且獨立觸發的攝影機）， 接著每一個觸發動作對應一行：`[  0] SN:<serial>=<timestamp> \| …`，若某台攝影機在該次觸發中未傳回任何資料，則對應 `SN:<serial>=(no frame)`。 接著是 `Offsets from master SN:<serial> (<PTP / device (PTP off)> clock, <n> trigger(s)):`，以及每台攝影機各一行的 `SN:<serial>: mean <±x> us, jitter <y> us`，每台攝影機在一個時鐘週期內與主機對應：若某次觸發的標記中混雜了攝影機的裝置時鐘與 PTP （PTP 關閉或仍在鎖定中）的觸發器將被排除，並由 `Note:` 進行計數。 任何遺漏的幀都會導致測試失敗（`Error: multi-test failed: <k> of <n> frame groups incomplete (SN:<serial> missed <k>); …`，退出 `1`）；連續三個不完整的觸發信號會導致測試提前結束。 此外，若少於兩個觸發器能讓所有攝影機在同一時鐘上同步（`… the clocks cannot be compared`），或時鐘從未一致： PTP 時鐘與主時鐘的偏差超過 1 毫秒（加上任何串列層的偏移），或裝置時鐘在觸發之間偏移量超過該值時。 若 PTP 總主時鐘尚未運行，則啟動該總主時鐘；開啟每台攝影機的 PTP 功能（如同 `array-connect` 所做，無論網路狀況如何）；並在首次觸發前，等待每台攝影機的 PTP 時鐘鎖定至總主時鐘並穩定下來： `PtpStatus` 從屬時鐘需鎖定於單一主時鐘，與其誤差在 100 微秒以內且穩定維持 3 秒，且每台攝影機均需以這種狀態連續鎖定 120 秒——自通電起約 2 分鐘。剛鎖定的時鐘在讀取「從屬」狀態後約 2 分鐘內，誤差可能高達 ~200 微秒，而相機自身的 `PtpOffsetFromMaster` 並不會顯示此狀況；經過 120 秒後，相機的時鐘誤差將收斂至 ~10 微秒以內。某台相機 若自先前測試結束後便持續保持 PTP 啟用狀態（依此後端所見）， 會從該次鎖定狀態開始計時，因此連續進行測試時無需再次等待；若測試中才開啟的攝影機，或是被偵測到 PTP 已開啟但此後端尚未見其鎖定（可能在其他處開啟，或於後端重新啟動前開啟），則需等待 120 秒；而在包含這兩類攝影機的陣列中，剛鎖定的攝影機仍需等待其預定時間。 若某台攝影機離開從屬狀態、計時超過 100 微秒，或在等待期間連續跟隨另一台總主時鐘進行 3 次讀取（1.5 秒），則會重新開始計時；此類讀取會記錄一至兩次 (`[array-sync] PTP SN:<serial> off lock at <s> s (<n> of 3): <status> <±x> us gm <id> -- locked <s> s, counting on`)，計數將繼續進行；若無法讀取主時鐘 ID，則視為未知狀態，而非變更。 若所有攝影機的計數在同一時間（一秒內）同時重啟，表示主節點的時間在移動，而非攝影機的時間：後端日誌顯示 `[array-sync] PTP grandmaster step: all <n> cameras restarted at <s> s …`， `ptp_lock` 將其列於 `grandmaster_steps` 中，隨後超時的等待以 `every camera's count restarted together at <s> s -- the PTP grandmaster's time moved under them …` 開始產生錯誤。 其時間限制並未因此延長。`--ptp-settle SECONDS` 設定該執行程序的穩定時間（0-600；若無穩定等待則為 `0`）。此等待時間的上限為穩定時間加上 90 秒（預設為 210 秒，至少 90 秒），且該指令會等待後端 240 秒加上穩定時間 （預設為 360 秒）。它會輸出 `PTP: locked after <s> s (each camera locked <settle> s, already locked: SN:<serial>, …; from the grandmaster: SN:<serial> <±x> us, …)`，且後端日誌每 5 秒會記錄一行，包含每台攝影機的狀態與偏移量、已鎖定時間以及剩餘時間： `[array-sync] PTP at <s> s: SN:<serial> Slave <±x> us for <s> s (<s> s left), … -- <s> s left, the wait gives up at <s> s`。每台攝影機開機時 PTP 皆處於關閉狀態； 隨後時鐘約在 10 秒內完成鎖定。之後，每台相機將恢復至測試發現時的狀態——其 PTP、自動曝光及增益（測試期間被設為關閉），其餘設定（除非出現 `--keep-settings`） ——因此，若相機保持 PTP 開啟狀態（`lattice configure -s <serial> --set PtpEnable=True`，或執行 `--keep-settings` 的測試），其時鐘將維持鎖定狀態；一旦此後端偵測到該相機已鎖定達穩定時間，後續測試便不會再等待它。 若時鐘在等待期間未能穩定，則會在觸發前失敗（`PTP: NOT locked …`，退出 `1`） 並說明應採取的措施：從未鎖定的相機（`PtpStatus Listening`）表示主節點無法與其建立連線（`time-sync status`）；持續斷開連線的相機則會標示其鎖定時間長短 （`… had not been locked for the 120 s settle time … (SN:<serial> locked <s> s, <s> s to go)`）；時鐘仍在收斂中：請檢查 `time-sync status`， 並在測試前讓它們鎖定，請開啟 PTP 並保持開啟狀態（每台攝影機皆需設定為 `lattice configure -s <serial> --set PtpEnable=True`，或使用 `--keep-settings` 執行測試；該測試會將偵測到的設定還原，因此會保持開啟），然後執行測試。 |
| `lattice multi-detect [--line LINE] [--json]` | 找出 M8 同步線上的攝影機，以及預設的主機：若已連接線纜，則為主機為 RGB 攝影機（型號以 `-FRGB` 結尾）；否則，依序列號為首的已連接線纜攝影機。 由於 M8 線纜匯聚於單一接點，因此 **任何已連接線纜的攝影機皆可擔任主機** ——印製的 `MASTER` 僅為預設值，並非實際接線狀態。每台攝影機會依序驅動傳輸線，並檢測其他攝影機是否接收到其脈衝；若某台攝影機已在接線群組中被偵測到，則不會再次進行探測， 因此，一個完全有線連接的陣列只需一輪掃描。若未偵測到 `--line`，則會掃描 `Line2`， 接著是 `Line3`。輸出 `MASTER: SN:…`，每台其他已連線的攝影機各輸出一個 `SLAVE: SN:…`，以及 `(probed on <line>)`；`--json` 則會將完整結果加入（`master`、`slaves`、 `no_cable` — 脈衝未觸及的攝影機 — `line`，以及每台攝影機的 `results`)。處理完成後，每台攝影機將恢復其原先的觸發與同步線設定（與上述相同的節點）。 |

#### 對齊

| 子指令 | 用途 |
| --- | --- |
| `lattice align-calibrate [--serials SN1,SN2,…] [--method orb\|akaze\|phase\|checkerboard\|manual] [--model translation\|rigid\|affine\|homography] [--frames N] [--checkerboard RxC] [--points PATH] [--reference SN] [--save PATH] [--preview] [--vignette] [--prefilter none\|gradient\|clahe\|blur\|hist_match] [--rms-threshold-px N] [--brightness N]` — 加上偵測器／匹配器控制項 `[--max-features N] [--ratio-threshold F] [--matcher bf\|flann] [--knn-k N]`， RANSAC 參數 `[--ransac-threshold-px F] [--ransac-iters N] [--ransac-confidence F]`、多幀組合 `[--averaging mean\|median\|inlier_weighted]`、幾何約束 `[--lock-rotation] [--lock-scale] [--lock-axis x\|y]`、空間限制 `[--roi X0,Y0,X1,Y1] [--mask PATH]`，以及從屬節點覆寫設定 `[--per-cam-override SN:KEY=VALUE]`（可重複） | 根據即時攝影機畫面計算對齊輪廓。預設值：`--method orb`、`--model rigid`、 `--frames 3`、`--save alignment_profile.json`（位於執行該指令的資料夾中）、 `--max-features 2000`、`--ratio-threshold 0.75`、`--matcher bf`、`--knn-k 2`、`--ransac-threshold-px 3.0`、 `--ransac-iters 2000`、`--ransac-confidence 0.99`、`--averaging mean`。（`project align calibrate` 有其專屬的預設值；請參閱 [其表格](#project-align-calibrate-options)。) `--prefilter` 的預設值為 `gradient` （邊緣地圖；與 GUI／陣列對齊器相符——邊緣可跨越光譜波段而保留）。`--matcher flann` 在特徵數超過 ~5000 時效果顯著； `--averaging median` 對單次不良擷取具有強健性，`inlier_weighted` 根據匹配次數進行加權；`--lock-scale` 投影至最近的旋轉（無縮放）， `--lock-axis` 將其中一個平移分量設為零； `--mask` 適用於所有攝影機（若需針對每台攝影機進行設定，請使用 `--per-cam-override`，例如 `--per-cam-override 214701292:method=phase`）。 `--rms-threshold-px` 會拒絕儲存重投影均方根誤差（RMS）超過閾值的校正資料。`--method manual` 從 `--points PATH` 取得其點資料： 一個 JSON 物件 `{"<serial>": [[x, y], ...], ...}`，其中包含每台相機對相同特徵的像素座標（包含參考相機，每台相機至少 2 個）；每個 `--frames` 皆由這些點解算得出。 當一個連線陣列包含多台相機時，校準會使用該陣列的即時幀，且該配置檔也會成為該陣列擷取影像的對齊基準。否則，它會開啟一個 [暫存陣列](#multi-camera-transient-sessions)： 若任何 `--frames` 中的相機未傳回幀，則校準失敗（退出 `1` 並為其命名），且不會儲存任何配置檔。 |
| `lattice align-apply --profile PATH [--serials SN1,SN2,…] [--format tiff\|png] [--bit-depth 8\|12\|16] [--bands NAMES] [--order NAMES] [--gpu\|--no-gpu] [--no-crop] [--per-camera] [--per-band] [--vignette] [--interpolation nearest\|linear\|cubic\|lanczos] [--border-mode constant\|replicate\|reflect\|wrap] [--border-value N] [--brightness N]` | 擷取一幀對齊的多波段影像。`--bit-depth` 預設會與相機匹配； `--no-crop` 保留完整畫面（以黑色填充邊緣）；`--interpolation`（預設為 `linear`）與 `--border-mode`/`--border-value`（預設為 `constant`/0）控制 CPU 變形——無論如何，GPU 路徑皆採用雙線性插值。開啟一個 [暫存陣列](#multi-camera-transient-sessions)：若某台攝影機未傳送任何畫面，則擷取失敗（退出 `1` 並標記其名稱），且不會寫入任何資料。 |
| `lattice align-stream --profile PATH [--serials SN1,SN2,…] [--fps F] [--count N] [--bit-depth 8\|12\|16] [--bands NAMES] [--order NAMES] [--gpu\|--no-gpu] [--no-crop] [--per-band] [--vignette] [--interpolation nearest\|linear\|cubic\|lanczos] [--border-mode MODE] [--border-value N] [--brightness N]` | 流對齊的多頻段幀（與 `align-apply` 具有相同的 Warp 控制參數）、`--fps` 預設值 `3`。開啟一個 [暫存陣列](#multi-camera-transient-sessions)：若某台攝影機未傳送任何幀，該幀群組將不被儲存；連續三次未傳送則會終止串流。 若任何群組被跳過，執行程序將輸出 `Incomplete: <n> frame group(s) not saved, a camera delivered nothing (SN:<serial> x<count>, …)` 和 `Error: …` 後結束， 並以 `1` 結束。若串流因其他錯誤而終止，亦會輸出 `Error: …` 並以 `1` 結束。 |
| `lattice align-info --profile PATH [--json]` | 顯示配置檔詳細資訊。 |
| `lattice align-reorder --profile PATH [--order NAMES] [--enable SERIALS] [--disable SERIALS]` | 變更圖層順序。 |

#### 索引 / 植被數學

```bash
# Offline: compute NDVI from an aligned multi-band TIFF (an RGN camera's bands)
chloros-cli lattice index --input aligned.tif --preset NDVI \
  --channel red=Red_660_RGN --channel nir=NIR_850_RGN \
  --output ndvi.tif --colorize --gradient RdYlGn

# Live: open the cameras, capture one aligned set with a profile made by
# `lattice align-calibrate`, and compute the index from it
chloros-cli lattice index --live --profile align.json --preset NDVI \
  --channel red=Red_660_RGN --channel nir=NIR_850_RGN \
  --save-multiband -o output/
```

**`--channel` 將預設值的符號映射至波段** （NDVI：`red`、`nir`），各一個 `--channel SYMBOL=BAND`。

- **`--live`：僅適用於堆疊無法回應的符號。** 與 `--input` 相同，若您省略某個預設值的符號，系統將採用堆疊中（該配置檔已啟用相機的波段）名稱以此符號開頭的單一波段。 此檢查會在任何攝影機開啟前進行：若某個符號沒有任何波段（或多個波段）能對應，則會被拒絕，並列出其名稱後退出 `1`。 自訂的 `--formula` 需針對每個符號配對一個 `--channel`。
- **`--live` 產品會標示其所属對象。** `index.tif` 與 `index_LUT.png` 攜帶堆疊中的相機資訊（品牌、型號、序號）、其擷取時間（參考相機的曝光時間，UTC） 及鏡頭資訊，且兩者皆將索引名稱視為其單一頻段（`XMP-Camera:BandName` `NDVI`；彩色 PNG 檔案以索引名稱而非其色彩通道命名），這與 `--input` 產品的做法相同。 若索引來自多台相機的波段，則會分別標示每個波段，其 `Model` 格式為 `LATTICE` — 詳見 [由多台相機製成的檔案](#files-made-from-several-cameras)。
- **`--input`：僅適用於檔案無法回應的符號。** 若您省略某個符號，系統將採用檔案中名稱以此符號開頭的單一波段： `red` 會採用 `Red_660` 或 `Red_660_RGN`，而 `nir` 則會採用 `NIR_850`。 這些名稱取自檔案本身：堆疊自身的波段清單，或 在Chloros輻度／反射率擷取檔或 `process` 產品中，其 `Chloros:Band{R,G,B}Name`／`XMP-Camera:BandName` 標籤。因此 `lattice index --input <reflectance capture> --preset NDVI` 無需 `--channel`。
- **不會推測兩個對應同一符號的波段**（9 台相機的堆疊包含兩個NIR波段）： 請為該波段命名。若某個符號對應的波段為空，則以 `channel_map missing entries` 拒絕該符號，並退出至 `1`。**堆疊中的波段名稱** 包含相機的濾光片資訊：M3C 相機的波段為 `<band>_<filter>` — `NIR_850_RGN`、`Green_550_RGN`、 RGN 上的 `Red_660_RGN`；`NIR_808_OCN`、`Cyan_490_OCN`、 `Orange_615_OCN` 位於OCN上；`Blue_475_RGB`、`Green_550_RGB`、`Red_625_RGB` 位於RGB上；`Blue_475_NGB`、`Green_550_NGB`、`NIR_850_NGB` 位於NGB上——而 M3M 相機的的單波段為 `Mono_<filter>`（`Mono_660`、`Mono_850`）。`lattice align-info --profile PATH` 列出了某個配置檔的波段。

完整標誌集：`--input PATH | --live --profile PATH`、`--preset NAME`（NDVI／NDRE／EVI／SAVI／GNDVI／…）， `--formula EXPR`、`--channel SYM=BAND`（可重複）、`--capture-level raw|debayered|radiance|reflectance|unknown`（覆寫來源 TIFF 檔案中記錄的擷取等級；預設： 從 TIFF 元資料讀取，若無則回退至 Chloros 檔案的 `Chloros:ProcessingLevel`，因此反射率擷取會回報 `reflectance`），`--output PATH` / `-o`（離線模式：路徑；若帶有圖像副檔名，檔案名稱將根據其主幹命名 — `-o ndvi_FRGN.png` 會寫入 `ndvi_FRGN.tif` 和 `ndvi_FRGN_LUT.png` — 若無副檔名則視為前綴，例如 `-o ndvi` → `ndvi.tif`、`ndvi_LUT.png`；若未指定，則預設與輸入相同， 例如 `<input name>_<preset>`；若包含 `--live` 卻沒有 `-o`，則會在執行命令的資料夾下建立一個名為 `output/` 的資料夾）， `--output-format all|raw|tif|colorized|lut|png`、`--gradient NAME|JSON`、`--vmin/--vmax/--percentile LO,HI`、 `--bg-mode clip|transparent|indexColor|backgroundColor`、`--colorize`、`--list-presets`、 `--list-gradients`。至於 `--live`，對齊變形旋鈕同樣適用於：`--save-multiband`、`--gpu/--no-gpu`、`--no-crop`、 `--bit-depth 8|12|16`、`--vignette`、`--interpolation nearest|linear|cubic|lanczos`、`--border-mode constant|replicate|reflect|wrap`、`--border-value N`、 `--brightness N`（攝影機的自動曝光目標，請參閱瞬態規則），`--serials SN1,SN2,…`。`--live` 開啟一個 [瞬態陣列]（#多攝影機暫存會話）：若某台攝影機未傳送任何畫面，則執行失敗（退出 `1`，並為其命名），且不會寫入任何堆疊或索引。它會開啟 `--serials`（第一個觸發器），否則則依序開啟該配置檔中 `layer_order` 條目已啟用的相機，並以參考相機為優先——即使該參考相機的波段已停用，因為其他相機已對齊至 該相機，且由其觸發扳機。頻段已停用的相機永遠不會進行校準，因此配置檔中停用的RGB相機將不再導致反射率指數（`no-absolute-radiance-scale`）失敗。未指定任何相機的配置檔將如以往般開啟網路上的所有相機。

每個表單皆在後端執行，因此後端必須處於運行狀態： 請從 Chloros GUI 啟動後端，或使用 CLI 在無法連通後端時所顯示的指令。離線時，後端會讀取 `--input`，並自行寫入浮點數 TIFF 及上色 PNG 檔案，因此相對的 `--input` 或 `-o` 會根據您執行命令的資料夾進行解析（參見 [命名慣例](#conventions)）。 這兩種離線產出皆包含輸入檔案的相機記錄，如同 `process` 索引匯出所呈現的：品牌／型號／序號、GPS、鏡頭標籤、拍攝時間（取自輸入檔案本身；若其 EXIF 未記載，則採用其記錄的曝光瞬間或檔案名稱中的時間戳記， 以 UTC 為單位）以及 MAPIR 校準識別碼——絕非原始 DN 校準（因其不適用於索引）——外加 `Chloros:ProcessingLevel=index` 及索引表達式；浮點數 TIFF 代表一個波段，即預設值。 `--list-presets` 和 `--list-gradients` 會輸出後端的預設值與漸變表。 若後端拒絕某個公式，或 TIFF 檔案不具備 `--channel` 所指的頻帶，則會以後端所提供的理由退出 `1`。

> **`--channel` 符號區分大小寫。** 符號一端必須與預設值的通道名稱完全一致（預設值使用小寫，例如 NDVI = `red`,`nir` — 請檢查 `--list-presets`），且波段一端必須與對齊堆疊中的波段名稱相符（或在離線模式下為以 0 為起點的波段索引）。`--channel red=Red_660_RGN --channel nir=NIR_850_RGN` 有效；`--channel RED=660` 會因 `channel_map missing entries` 錯誤。

#### `lattice index` — 多波段堆疊遮罩

在 `--capture-level radiance` 或 `reflectance` 處，每台相機的 3×3 解混運算可將某個波段鉗位為 0 （參見 [解混鉗位遮罩](#unmix-clamp-mask-which-pixels-a-band-was-clamped-on))。多波段堆疊包含多台相機的波段，且每台相機皆會對其自身波段進行鉗位，因此 `--live --save-multiband` 會在堆疊旁寫入 **每台來源相機對應一個遮罩**：

```
output/
├── index_multiband.tif
└── clamp_masks/
    ├── index_multiband_SN<serial A>-<filter A>_clampmask.tif
    └── index_multiband_SN<serial B>-<filter B>_clampmask.tif
```

- 每個遮罩皆為單波段 `uint8`，位於 **堆疊** 的像素網格上（經過相同的對齊變形與裁切），並包含該相機自身的位元：位元 0 = 其 R 通道的波段，位元 1 = G， 位元 2 = B。取自某台相機第 *c* 通道的堆疊頻帶，會被限制在該相機的位元 *c*（R = 0、G = 1、B = 2）被設定的位置。
- 每個頻帶的內嵌來源資訊會標明其相機的遮罩 （`unmix_clamp_mask`，以堆疊的資料夾為基準，或 `none`）以及待讀取的位元（`unmix_clamp_mask_bit`）。
- 若相機的「解混」處理未對任何通道進行裁剪，則不會產生檔案；單色或 RGB 相機則不執行「解混」處理，同樣不會產生檔案。CLI 會在圖層堆疊下方列印每個遮罩（`clamp mask SN…: …`）；回應中則將其列為 `clamp_masks`。
- 同一執行程序的索引 TIFF / PNG 檔案是根據同一堆疊、在同一網格上計算得出的，因此這些遮罩也適用於該檔案。`--capture-level raw` 不會執行解混，也不會寫入遮罩。
- 堆疊檔與索引檔直接存放於 `--output` 中，因此遮罩檔則存放於 `--output` 自身的 `clamp_masks/` 內 — 絕不會位於您所指定資料夾的上層。 該資料夾混合了不同類型的產出（堆疊、索引、上色渲染）：請從中挑選所需檔案，切勿將整個資料夾直接交給攝影測量軟體處理。

#### 持久連線（Smart-Prep、GUI 等效流程）

這些指令可在多次 CLI 呼叫之間，將相機保持開啟狀態於後端池中。

| 子命令 | 用途 |
| --- | --- |
| `lattice cam-connect [--serial SN]` | 向彙集新增一台相機（單一相機，非陣列）。[通用選項](#common-options-shared-by-most-subcommands) `--exposure`、`--gain`、`--preset`、 `--pixel-format`、`--width` / `--height` 及 `--packet-size`（`standard` = 1500、`jumbo` = 9000，或位元組；`auto` 探測）於連接時 （`--exposure` 或 `--gain` 會自動關閉此模式）。 當攝影機的串流開始後，程式即會返回（等待約 15 秒，接著再等待 10 秒以取得第一個完整的緩衝區），並輸出 `stream: streaming`；若串流未傳送任何資料 ——此情況適用於使用巨型封包的攝影機，並建議使用 `--packet-size standard` 重新連線。當其中一台攝影機正在連線時（首次連線會下載其校準資料，約需 70 秒），另一項需要該攝影機的指令最多會等待 5 秒， 隨後會因 HTTP 503 錯誤（代碼 `CAMERA_POOL_BUSY`）而失敗，並標示出正在連線的攝影機名稱；請稍後重新執行。`lattice status` 及攝影機清單從不等待：正在連線中的攝影機會顯示為「連線中」，並標示其階段。 |
| `lattice cam-disconnect [--serial SN] [--all] [--keep-settings \| --clear-settings]` | 釋放。 攝影機將恢復至連線時被偵測到的狀態（觸發器接線、PTP、設定）；`--keep-settings` 會將其設定儲存至自身記憶體中，`--clear-settings` 則將該記憶體重置為出廠設定 （參見 [攝影機記憶體](#camera-memory----keep-settings----clear-settings)）。身為陣列成員的攝影機會釋放整個陣列。 |
| `lattice cam-list` | 列出池中的攝影機。 |
| **`lattice array-connect`**|**連接一個持久性同步陣列（推薦的入口點）。** 執行完整的 GUI 智慧預備流程。處理 9 台攝影機約需一分鐘； 相機剛開機時可能需要數分鐘。在等待期間，它會輸出後端達到的每個步驟，例如 `    <s> s: <step>`（例如 `    41 s: open SN:214000535 (4 of 9)`）， 且後端日誌中每一步驟會有一行時間記錄（`[array-connect <id>] <step>: <s> s …`），最後一行則列出耗時最長的步驟。 等待時間最長為 600 秒。若屆時仍未建立連線，後端會撤銷操作：所有已開啟的攝影機皆會恢復至初始狀態並關閉， 該指令會輸出 `Error:` 並標示已達到的步驟，隨後退出 `1`，此時沒有任何陣列已連線（HTTP 504 `CLIENT_DEADLINE_PASSED`）。 按下 Ctrl+C 亦會以相同方式取消連線（`Interrupted: …`）；若陣列已建立連線，該行會標示此狀態並列出釋放連線的指令 `array-disconnect`。若在等待期間被終止的指令 (`kill -9`)，當後端在 30 秒內未收到該命令的回應時，此連線將被撤銷。已啟動但尚未擁有同步幀群的陣列，仍會被報告為已連線，其資料流標示為 `no_frames`。 |
| `lattice array-disconnect [--array-id ID] [--all] [--keep-settings \| --clear-settings]` | 釋放一個陣列。 每台攝影機均保持自由運行狀態（`TriggerMode` `Off`），其餘設定均維持連線建立時的狀態——包括觸發器與同步線設定（主機的的 `TriggerSource` 恢復至原始狀態，其同步線恢復為輸入模式）、其 PTP 設定，以及該會話所變更的攝影機設定——因此對成員發出的單一攝影機指令可立即生效。`--keep-settings` 將每台攝影機的的設定儲存至相機自身的記憶體中；`--clear-settings` 則會將該記憶體重置為出廠設定。請參閱 [相機記憶體](#camera-memory----keep-settings----clear-settings)。 |
| `lattice array-list` | 列出已連線的陣列。 |
| `lattice array-status [--array-id ID]` | 即時幀率 (fps)、PTP、 最後一次錯誤。`ptp_enabled` 僅在所有成員皆啟用 PTP 讀取時等同於 `True`；`ptp_off:` 則指明未啟用 PTP 讀取的成員。當 `state:` 行會在設定重新啟動時讀取 `restarting (applying settings)`，在失去電源的成員重新開啟時讀取 `recovering (…)`（參見下方註解）， `stalled (…)` 則發生於抓取迴圈執行中但未發佈任何幀群時，而 `stopped (…)` 則發生於抓取迴圈已放棄（重新連線陣列）之後。 |
| `lattice array-capture [--processing LEVEL\|all] [--levels L1,L2,…] [--aligned\|--no-aligned] [--orient] [--index\|--no-index] [--no-daq] [--daq-tolerance-ms MS] [--smart] [--fastest] [--compression deflate\|none] [--continuous\|--interval S] [--count N] [--duration S]` | 來自即時陣列的一次同步擷取 — 單次 / 連續 / 間隔 / 最快。**預設為 `all`**（每台攝影機針對適用之匯出類型各產生一個檔案），**對齊方式**與應用程式的擷取設定一致（原生檔案請參閱 `--no-aligned`）。被跳過的攝影機（例如 RGB 將從輻照度／反射率中排除）會以 `Skipped: SN:<serial> (<reason>)` 報告；用於讀取該擷取資料的 DAQ 讀數會一併儲存，並包含僅含原始資料的擷取檔，且以 `DAQ: <path>` 報告（[DAQ 讀取擷取資料時會儲存](#the-daq-reading-a-capture-saves))。請參閱 [擷取模式、記錄器與離線重新處理](#capture-modes-recorders--offline-reprocess)。 |
| `lattice array-record [--fps F] [--duration S] [--gif] [--gif-only] [--expr EXPR]` | 將即時複合索引視圖錄製為影片／GIF（監控級；當未開啟任何 GUI 視圖時，後端會渲染陣列的 NDVI，或 `--expr`）。 |
| `lattice array-burst [--duration S] [--max-frames N] [--orient] [--build] [--products …]` | 高幀率原始拜耳連拍 （分析級；離線重新處理）。與 `array-record` 及 `array-capture --continuous` 相同，此任務會在每次輪詢時向後端回報狀態：若未停止即遭終止（`kill -9`），後端會在最後一次回報後 30 秒終止該任務（`stop_reason` `client_gone`；此時連拍的清單已定案，並開始建立合併索引，此過程與終止操作相同），而陣列仍保持連線狀態。 |
| `lattice array-build-video --burst-dir DIR [--products …] [--fps F] [--save-tiffs] [--gif]` | 將已儲存的原始連拍資料重新處理為經校準的影片。 |

> **若相機因斷電或連線中斷而失去回應，系統會自動恢復其連線。** 當某個成員停止回應時（交換機重啟電源會重新啟動其供電的所有相機），陣列會等待該成員，並在其再次回應時重新建立連線， 並將 `array-connect` 所設定的參數還原給該攝影機：PTP、觸發與同步訊號、網路步調、像素格式與幀大小，以及自動曝光。隨後，該攝影機將繼續串流傳輸，且不會有新的 `array-connect` 設定。 與此同時，`array-status` 顯示 `state: recovering (...)`，而 `last_error` 則標示正在等待的攝影機；隨後會輸出 `re-opened:`。 重新啟動的相機恢復連線時 PTP 處於關閉狀態，因此在重新開啟之前，`ptp_enabled` 會顯示為 `False`，而 `ptp_off:` 則會標示其名稱。

> **被佔用的攝影機絕不會被第二次開啟。** 第二次開啟會停止佔用者的串流並覆寫其設定，甚至可能導致後端當機。 因此，對於由池（GUI 或 `cam-connect`）持有的攝影機、陣列成員，或是由背景工作持有的攝影機（正在執行的 `lattice continuous`、 `multi-stream` 或 `align-stream`，無論是自身或另一位客戶端所持有），`lattice continuous`、 `lattice capture`、`cam-connect`、`power --sleep`、`calibrate`、單次嘗試的 `white-balance` 和 `presets --apply` 會以 HTTP 409 狀態碼拒絕 `CAMERA_HELD`，並指明持有者及釋放指令 （`chloros-cli lattice cam-disconnect -s <SN>`、`chloros-cli lattice array-disconnect --array-id <ID>`，或針對工作則為其 ID 及啟動者），並退出 `1`。 目前正被其他指令使用的攝影機也是如此（例如 `multi-test` 或 `multi-capture` 正在開啟其攝影機，或是另一個單次指令），針對此處的每個指令：錯誤訊息會提示等待該指令完成。 若沒有 `--sleep`，則 `lattice power` 會透過持有者自身的句柄（若存在的話，例如池、陣列或工作已開啟的攝影機）進行讀取，並因該攝影機正被其他指令使用而遭拒絕。 每個 `CAMERA_HELD` 回應皆包含 `code` 與 `error_code` 中的代碼。關於開啟所有攝影機的指令，請參閱 [多攝影機（暫存會話）](#multi-camera-transient-sessions)。

##### `array-connect` 選項

| 標誌 | 預設值 | 說明 |
| --- | --- | --- |
| `--serials SN1,SN2,…` | 自動偵測所有 LATTICE 攝影機（需 ≥2 台） | 第一個序列號的攝影機為主機（MASTER）。若未指定，偵測時將篩選 LATTICE（M3C / M3M）型號並連接所有該型號的攝影機。 |
| `--line {Line0,Line2,Line3}` | `Line2` | GPIO 同步線。 |
| `--target-fps F` | 自動 | 主控器觸發射擊率。 |
| `--force-tier {sim-capture-sim-emit, sim-capture-ftd-stagger, slip-emit-and-capture}` | 自動 | 覆寫層級選擇器。 |
| `--wire-ceiling-mbps MB_PER_S` | 自動偵測 | **主機的持續線路預算（單位：MB/s）——整個陣列的資源分配皆以此數值為依據。** 當陣列回報 GVSP 損毀幀時，請降低此數值：自動設定值是根據網卡通報的鏈路速率推算得出，但此數值會高估 USB 適配器、窄頻 PCIe 通道及繁忙的共用匯流排的實際速率。此數值會儲存於專案的陣列擷取區塊中，因此重新開啟 / 透過 CLI / SDK 重新連線時，該數值將恢復原狀。 請參閱 [陣列健康狀態](#array-health--which-subsystem-is-losing-frames)。 |
| `--binning {1,2,4}` | 無（全感測器） | 硬體像素合併（始終為平均值）。 |
| `--width N` / `--height N` / `--pixel-format FMT` | 全感測器；每台相機格式 | 為每台相機指定明確的幀大小或像素格式，而非採用全感測器與每台相機格式選項（Mono12 / BayerRG12）。 |
| `--no-recommend` | 關閉 | 跳過網路分析步驟 （此時每台攝影機將保留其當前尺寸設定，而非被設定為全感光元件）。 |
| `--no-ptp` | 關閉 | 停用 PTP（此時跨攝影機的時間戳記將**無法**相互比較）。 |
| `--packed {auto,on,off}` | `auto` | 在本電腦上以打包形式傳輸 12 位元幀並進行解包 — 在全感測器模式下可獲得更高的每秒幀數，且資料內容完全相同。`auto` 僅在 且本電腦處理能力足以支撐時才進行封裝；`off` 從不進行封裝。請參閱 [封裝式 12 位元傳輸](#packed-12-bit-transport---packed)。 |
| `--packet-size {auto,jumbo,standard,N}` | `auto` | 陣列中每台攝影機的 GVSP 封包大小。 `auto` 允許每台攝影機探測網路；若 `jumbo` = 9000、`standard` = 1500，或指定以位元組為單位的大小，則會鎖定此設定。 在 1.3.0 版本之前，`array-connect` 會接受此標誌但予以忽略。 |

> **預設為全感測器模式。** 若未設定 `--width` / `--height` / `--binning` /
> `--pixel-format`、`array-connect` 參數時，會將每台攝影機設定為全感測器模式——不
> 進行像素合併、 不進行 ROI 裁切——並讓幀率降至這台電腦與
> 網路所能提供的水準，而非透過縮小畫面來維持幀率。
> 網路分析行顯示預估幀率，下一行則列出
> 較快的選項（像素合併或較小的 ROI）及其預估幀率；若要採用
> `--binning` 或 `--width`/`--height` 來選擇其中一種。Connect 絕不會自行進行像素合併或裁切
> 操作。

##### 壓縮式 12 位元傳輸 (`--packed`)

在感測器全速運作時，陣列的幀率取決於每台攝影機傳送
幀的速度。 12 位元格式（`BayerRG12`、`Mono12`）通常以 16 位元
字元傳輸；**壓縮**傳輸則將相同的 12 位元以每像素 1.5 位元組的格式傳送，並
並在該電腦上進行解壓縮，因此每幀的傳輸時間減少 25% ——
在全感測器模式下，**每秒幀數約增加 33%**（實測：10.7 → 14.0 fps
於 9 台相機陣列上）。 **所節省的資料完全相同**：數值相同、
16 位元檔案相同、元資料相同。

- `auto`（預設）僅在有助益且**且**這台電腦能負擔
  解壓縮作業時才會進行壓縮。 連線時，後端會測量自身的解壓縮速度、CPU 核心數、
  當前 CPU 負載及網路頻寬，因此核心餘裕的桌上型電腦會在
  全感測器模式下進行壓縮，而繁忙或低功耗的電腦則會以未壓縮格式串流。小型 ROI 及二進位幀的
  的效益微乎其微。
- `off` 絕不進行封裝 — 可用於在診斷時排除封裝，或應用於
  必須將所有 CPU 資源保留給其他工作的電腦上。
- `on` 始終對 12 位元格式進行封裝。

connect 輸出會顯示決策結果及其原因（`封裝 12 位元：啟用 (自動：
預估 11.0 → 14.4 fps，…))`); `/api/camera/array/狀態` 報告
`packed_transport` 和 `packed_decision`。若要設定全系統預設值
而非每次都傳入旗標，請在後端環境中設定 `CHLOROS_ARRAY_PACKED12=off` （或
`on`）——但若明確指定 `--packed on|off` 仍會優先。
8 位元與 16 位元格式絕不會進行壓縮。

### 單一專案記錄，適用所有模式

一個專案資料夾包含所有裝置的設定：`cameras.json` — 每台攝影機的設定（`cameras[].settings`， 適用於單一相機及每個陣列成員）以及每個陣列的全陣列設定（`arrays[].capture`：格式、ROI、像素合併、觸發率、線路預算）——還有 `sensors.json`， 每個光感測器的設定（積分時間、幀平均值、上限）。應用程式、CLI、SDK 和 MCP 伺服器皆以相同方式讀寫相同的檔案：

- **連接至專案下的裝置會儲存於該專案中，並將每項變更記錄至其中。** 「隸屬於某個專案」意指後端（應用程式端）已開啟該專案，或——透過SDK——經由 `ChlorosProject` 連接，該連線會指定其專屬資料夾。當專案尚未包含的攝影機、陣列或光感測器連線時，其資料將被新增至該專案的檔案中 (`lattice cam-connect`、`array-connect`、`daq pool-connect`、應用程式、SDK、`chloros-mcp`)，並包含其設定。 無論從何種模式（`lattice exposure`、`gain`、`configure --set`、`resolution`、陣列的的設定與觸發率、光感測器的上限、應用程式的控制項）所做的任何變更，都會在片刻後儲存至該專案的檔案中。 在相機上設定的曝光、增益或自動曝光目標會以「保留」狀態儲存，因此重新開啟時會再次保留該設定。
- **開啟專案時，系統會根據該專案設定裝置**，無論處於何種模式：若相機、陣列或光感測器已連接但未設定自身參數，則會採用專案中儲存的設定 （`lattice cam-connect` / `array-connect` 在開啟專案時，`chloros-cli project connect`、`open_project().connect_all()`，以及應用程式重新開啟時）。 若以新 ID 連接的陣列，會更新專案中已儲存的、包含相同攝影機的陣列；若陣列包含其他攝影機，則會並列新增。
- **應用程式的儲存功能絕不會以較舊的副本** 覆寫裝置的即時設定，並會保留CLI或SDK呼叫者所儲存的各成員覆寫設定。
- **您所設定的像素格式即為專案所保留的格式。** 無論是在「陣列設定」中、透過 `--pixel-format` 或 `connect_array(pixel_format=)` 設定的格式（包含 8 位元格式），都會被精確儲存並在重新連線時完全復原。 若未設定，每台相機將以 12 位元預設值連線（單色感測器上為 `BayerRG12`、`Mono12`）。格式名稱代表其位元深度；格式家族取自感測器，因此彩色攝影機絕不會被設定為單色格式，單色攝影機也不會被設定為拜耳格式，且當格式被映射時，行 `[PixelFormat <serial>] ... written` 會明確標示此點。8位元擷取的輸出資料為 8 位元，包含反射率：經重新縮放並標記其 `PixelScale` 值。
- 未隸屬於任何專案的裝置不會進行任何記錄。
- **錄製作業會遵循其後建立的專案；若專案已開啟，則保留原有專案。**當建立專案時——無論是透過 SDK 的 `create_project()`、`chloros-mcp` 的 `create_project`，或是應用程式的**新專案**——或複製（應用程式的**複製**功能，此操作會開啟副本）時，若錄製仍持續寫入先前開啟的專案中，則錄製會繼續進行，其寫入的內容最終會存入新專案中，位於該專案內的相同位置（下方）。開啟現有專案 — `chloros-cli process`、`project devices --reconnect`、 `ChlorosProject.process()`，即應用程式中的**開啟專案** —— 此時正在錄製的影片仍會繼續進行，並寫入其原本錄製的專案中：錄製完成後，相關檔案將加入該專案的檔案清單（應用程式會在開啟專案前停止所有錄製）。
  - `array-record` 和 `array-burst`：當錄製停止時，影片或包含所有幀的連拍資料夾會整批移至該處（其 `--duration`， Ctrl+C）。程式輸出的 `Burst dir` 為新檔案，而 `--build` 和 `array-build-video` 則以此為基礎運作。若磁碟拒絕移動檔案，錄製檔案將保留在原始寫入位置；末端的日誌會說明原因，停止時的 HTTP 回應（`relocation_error`）亦會顯示原因。已停止的連拍所產生的原始幀與 `.daq` 會加入其最終歸屬之專案的檔案清單中； 影片（即 `array-record`，為一組連拍所建構的影片）則不會被加入專案的檔案清單中。
  - `array-capture --continuous`：變更後的所有擷取內容都會儲存至新專案中；變更前儲存的內容則維持原位。
  - `daq pool-record` 和 `project sensor log`（以及應用程式自身的光感測器錄製功能）：當錄製停止時，`.daq` 會移入新專案的的 `light_sensor` 資料夾中，並加入該專案的檔案清單。

  若錄製至開啟專案以外的資料夾（例如位於其他位置的 `-o`） 則會保留在原處，而 `lattice continuous` 和 `multi-stream` 則會保留其被指定的資料夾。
- **在應用程式中，建立新專案後的擷取資料會存入該專案。** 建立或複製專案時，所有正在執行的擷取作業與錄製過程都會持續進行，而其後進行的每項擷取（「全數擷取」、「開始擷取」）都會儲存至新專案並加入其檔案清單。在應用程式中開啟現有專案時，會先停止所有這些作業。
- **專案在錄製期間不會被移動。**將開啟的專案移動至另一個工作目錄（應用程式中的**設定 → 工作目錄**； 沒有任何 CLI 指令可移動專案，但透過 HTTP 客戶端向 `/api/move-project` 發送請求則可）將因 HTTP 409 `PROJECT_RECORDING` 錯誤而被拒絕，只要陣列錄製或連拍、陣列連續擷取、 `lattice continuous` / `multi-stream` 工作或光感測器錄製正在寫入該專案時：`Project '<name>' is recording (<what>), so moving the project now is refused. Stop the recording, then try again.` 應用程式會以相同方式拒絕， 在要求選擇資料夾之前即會以相同方式拒絕，且此時其自身的「全數擷取」、「開始擷取」或光感測器錄製功能正在執行中。

這與相機在斷開連接時留下的狀態無關（請參閱 [相機記憶體](#camera-memory----keep-settings----clear-settings)）：專案在開啟時會自行設定其相機。

### 攝影機記憶體 — `--keep-settings` / `--clear-settings`

當陣列連接時，其相機將採用陣列的設定，您之後可進行變更。當 Chloros 釋放相機時 — `array-disconnect`，即暫時性指令的結束 （`multi-capture` 且無陣列、`multi-stream`、`multi-test`、`align-calibrate`、 `align-apply`、`align-stream`、`index --live`)， `lattice continuous` 的停止，以及某台相機所屬的 `lattice capture` 的結束 —— 它會將每個項目恢復至連接前被發現時的狀態： 其觸發器與同步線接線、PTP 及相機設定（完整清單請參閱 [多相機（暫存會話）](#multi-camera-transient-sessions)）。

若要在專案外使用相機，上述任一指令中的兩個標誌會將設定寫入相機的 **自身記憶體** —— 無論在何種軟體環境下，這都是相機開機時的初始狀態：

- **`--keep-settings`** 會將每台相機在工作階段結束時的設定，以原始狀態儲存至其記憶體中（UserSet2，即開機預設設定）。 攝影機仍保持自由運行狀態（`TriggerMode` `Off`），同步線接線維持原狀。若某台攝影機沒有 UserSet2，則會將設定儲存至 UserSet1；本電腦會記住此狀態，因此 Chloros 的陣列自癒功能 ——該機制會重置已偏離Chloros基準值的UserSet1——則不會干預該設定。
- **`--clear-settings`** 將每台攝影機的記憶體重置為出廠設定：即Chloros的基準值，此為其陣列自我修復機制所整合的 — 工廠預設值，包含 `TriggerMode`、`Off`、`TriggerSource` 及 `Line2`， 自動曝光，RGB攝影機的校準白平衡將保留 — 存於 UserSet1 中，並設為開機預設集。
- 這兩者無法結合使用。 每台相機各佔一行：`SN:…: settings saved to the camera's memory (UserSet2); it powers up with them.` 或 `SN:…: camera memory reset to the factory settings.`；回應中的 `camera_memory` 針對每台相機亦同。

該應用程式不提供這些標誌。

**任何退出方式都會將攝影機恢復原狀。**相同的「恢復原狀」操作會在每個退出路徑上執行，而不僅限於上述指令：`cam-disconnect`（以及應用程式中的攝影機**X**）； 應用程式中的**X** 若作用於陣列或任何陣列成員的列上（陣列若無成員便無法繼續執行，因此整個陣列將被釋放）；關閉應用程式（系統會給予後端最多 20 秒的時間來將每台攝影機、 陣列以及正在執行的 `lattice continuous` 恢復正常運作後，才會終止該應用程式；若此時仍有指令正在執行——例如暫時性指令正在開啟其攝影機或等待其 PTP 時鐘、單次執行的 `lattice capture`、 `power`、`white-balance` 或 `presets --apply` —— 將在下一安全點停止，優先將其相機恢復原狀，此過程最長可達 8 秒， 並回應 HTTP 503 `SHUTTING_DOWN`；一個仍在執行的陣列連線 — `lattice array-connect`（即該應用程式的連線） — 會在抵達下一台攝影機之前停止，並關閉已開啟的攝影機，持續時間最長達 10 秒， 並回傳 HTTP 503 `SHUTTING_DOWN`；而在關機過程中抵達的連線則不會開啟任何連接：HTTP 503 `SHUTTING_DOWN`）；此外，若 CLI 指令遭終止或其終端關閉（SIGTERM / SIGHUP 會停止其工作 ——並將其開啟的攝影機恢復原狀——然後才退出 `143` / `129`，類似於 Ctrl+C）。 若 `lattice continuous`、`multi-stream` 或 `align-stream` 被直接終止（SIGKILL）——此情況無法停止任何操作——後端會在最後一次檢查完成後 30 秒 。若後端當機或遭強制終止，則無法還原任何設定；重新開機會將每台攝影機恢復至其記憶體狀態，而下一次Chloros連線時，會重新配置其偵測到已啟用的攝影機。**專案會自行設定其攝影機。**當陣列在專案開啟時連線，每個成員將取得專案為其儲存的設定——包括曝光值、增益或自動曝光目標（以您的設定為基準，因此自動曝光功能不會干預）及其白平衡模式（Chloros 的校正；相機自身的白平衡功能將保持關閉）——無論相機記憶體中儲存的是什麼。

### 智慧自動曝光 / 智慧擷取

LATTICE 陣列一經連接，便會在背景中持續執行自動曝光，但新對準的場景需要片刻時間才能收斂。 `array-capture --smart` 提供 **一站式便利功能**：它會等待陣列中每台相機的自動曝光穩定下來，然後才觸發拍攝。當您在拍攝過程中更換場景時，請使用此功能。

```bash
# Connect once, then take settled captures whenever you re-point the rig
chloros-cli lattice array-connect --serials SN1,SN2,SN3,SN4
chloros-cli lattice array-capture --smart --processing reflectance -o pose_a/
# (move the rig)
chloros-cli lattice array-capture --smart --processing reflectance -o pose_b/
```

預設的穩定化政策較為保守：5 秒超時、1.5 秒穩定性窗口、±5 % 曝光波動容差。若需自動曝光功能呈現不同行為，請透過 SDK（`ArrayHandle.capture_smart(settle_timeout_s=…, stability_window_s=…, exposure_tolerance_pct=…)`）進行調整。

#### 自動曝光的運作方式

- **先設定曝光，後調整增益。** 當增益設為「自動」時，增益將維持在 0 dB，直到曝光時間達到允許的最長值為止。
- **手動設定曝光時，增益仍維持在「自動」模式**，直到您變更增益或將其設為手動為止。切回自動模式後，增益的上限將恢復為先前設定的值。
- **微小的亮部不會使畫面變暗。** 佔畫面不超過 1% 的鏡面高光將被忽略；而在已知校準標靶的面板中，則不超過該面板面積的 0.5%。
- **DAQ 曝光上限僅在校準標靶處於視野內時生效。** 此設定可防止白色面板發生剪切；若未偵測到面板， 相機將根據場景進行曝光。
- **恆定光線保持穩定，光線變化則在數秒內趨於穩定。** 在恆定光線下，曝光值及其上限保持不變；光線發生大幅變化後，畫面將在數秒內趨於穩定。剪裁現象不再永久限制曝光，且當感測區域 (ROI) 變更時，系統會重新調整曝光值。
- **亮度目標始終在可控範圍內。**當攝影機處於曝光極限且增益無法提升時，亮度不會持續攀升，因此光線恢復時也不會導致畫面過曝。
- **當相機的自動曝光停止響應時，會維持在當前光線條件下的曝光水準。** 在強烈的陽光下，相機可能會降至最短曝光時間並維持該狀態，畫面呈現黑暗， 而所有設定數值皆顯示正常。Chloros 會監控畫面，將相機維持在該光源下的曝光水準，並在光線變化時放開控制。即使光感測器被遮蔽或處於黑暗中，也不會干擾此機制。

### 擷取輸出等級（`all` 預設值）

自本版本（`lattice capture`）起， `lattice multi-capture` 以及 `lattice array-capture` **預設為 `--processing all`** — 每種匯出類型對應一個儲存檔案，適用於每台相機，與 GUI 的「擷取全部」行為一致。各級別如下：

| 級別 | 輸出 | 適用於 |
| --- | --- | --- |
| `raw` | 直接來自感測器的單通道拜耳（單色相機：單一波段）影像。 | 所有攝影機。 |
| `debayered` | 3 通道彩色，以 **RGB**通道順序儲存（TIFF 檔案採用 RGB 光度解讀；PNG/JPEG 檔案同樣以 RGB 格式開啟）：從原始資料中扣除校準套件的暗電偏移（DSNU），接著進行 OpenCV 的邊緣感知去馬賽克處理——未使用線性 LUT、平場校正或解混色（單色相機：1 通道灰階）。Python SDK 返回的內存中去拜耳化幀（`grab()`、`frame_stream()`）為**BGR** 格式，即 OpenCV 的順序； 僅儲存的檔案為RGB格式。 | 所有相機。 |
| `radiance` | 透過完整的輻測鏈獲得的 float32 W/m²/sr/nm 值。 | 僅限多光譜（M3C/M3M）—— **對於 RGB 濾鏡相機**。 |
| `reflectance` | uint16 ρ（`32768` = 1.0），支援 Pix4D。 | 僅限多光譜，且 **僅當已綁定 DAQ 且相機已校準時** 才適用；否則將被跳過。 |
| `display` | 完整的 GUI 預覽鏈 （根據相機設定檔進行色彩管理 + 白平衡 + gamma 校正）。儲存於 `preview/` 資料夾中。`lattice capture` 亦接受名稱 `preview` 來指稱該資料夾； `array-capture` / `multi-capture` 僅接受 `display`。 | 所有攝影機。 |

傳入單一層級以僅儲存該層級（`--processing debayered`）。當您查詢 `all` 時，不適用於特定凸輪的層級會被跳過 （並會進行回報），而非產生錯誤 — 未連接或未校準的攝影機仍會取得 `raw` / `debayered` / `preview`。

#### 擷取所儲存的 DAQ 讀數

每當 DAQ 與相機或陣列綁定，或被納入資源池時，擷取所使用的下行輻射讀數會寫入 **`.daq`** 檔案中，並標註其 `daq/` （讀數與曝光時間的間隔）及其 `.cameras.json`（該讀數所屬的對象），並於 `DAQ:` 行中報告。 此規則適用於所有層級，包含原始檔或去拜耳化後的擷取檔，因此任何擷取檔日後皆可重新處理為輻照度、反射率或相關指標。 無論擷取來源是 GUI、CLI 還是 SDK，檔案名稱和附屬檔案都相同。若完全沒有 DAQ 讀數，則不會有 `.daq`；而一個未連接陣列的 `multi-capture` （瞬態備用方案）則不會儲存任何資料。

| 標誌 | 效果 |
| --- | --- |
| `--no-daq` | 若擷取的級別未使用該讀數（`raw`、`debayered`），則不儲存該讀數。但 `radiance`、`reflectance` 或 `display` 級別仍會寫入該讀數，因為重新匯入需要該產品製作時所使用的讀數。 |
| `--daq-tolerance-ms MS` | 讀數與曝光值的偏差範圍內仍被視為在公差範圍內（預設值 500）。僅判定結果會有所變化 （`.match.json` 的 `in_tolerance` 和 `tolerance_ms`，以及反射率的 `DlsInTolerance`），絕不會改變實際使用的讀數。 請參閱 [遠離曝光值的 DAQ 讀數](#a-daq-reading-far-from-the-exposure)。 |
| `--force-daq` | 在僅原始檔擷取模式下，用於查詢 `.daq` 的舊式方法。此現為預設方式，而 `lattice capture` 和 `array-capture` 仍接受此標誌（`multi-capture` 則不接受）。`--no-daq` 會優先於此標誌。 |

使用哪個 DAQ：與相機或陣列綁定的那個（GUI 中的光感測器選項、專案中儲存的綁定，或 `chloros-cli daq pool-bind`），否則則使用後端中最新的那個。原始連拍 (`array-burst`) 會像以往一樣，針對每筆獨立讀數在連拍資料夾中保留各自的 `.daq`。

**PNG 和 JPEG 每通道儲存 8 位元，因此僅採用 `display` / `preview` 及索引產物。** 若 `--format png` 或 `jpg` 包含其無法容納的亮度級別，則會在擷取任何內容前被拒絕，並標示該亮度級別及可容納該級別的格式（TIFF）：

- `radiance`（float32）與 `reflectance`（uint16，`32768` = 1.0） — 此情況始終成立，因此 `--processing all`（預設值）搭配 `--format png`/`jpg` 亦會遭拒： 請使用 `--format tiff` 或 `--levels display`。
- 來自位元深度超過 8 位元的攝影機所傳輸的 `raw` 和 `debayered` （12 位元的 M3C / Mono12 M3M 幀）——CLI 無法偵測到攝影機的位元深度，因此後端會拒絕這些幀（HTTP 400）。單一幀的 `lattice capture --processing debayered` 會擷取 8 位元幀供 PNG／JPEG 使用，因此不會被拒絕；同樣地，未包含持久性陣列的 `multi-capture`（8 位元去拜耳化）也不會被拒絕。

每條擷取路徑的回應方式皆相同 （`error_code` `FORMAT_CANNOT_HOLD_LEVEL`）：`lattice capture`、`multi-capture`、`array-capture`、 `project capture` / `burst`，以及其後方的 HTTP 路由。

### 擷取資料夾的樣貌

每種匯出類型都會存放於 `-o` 下的 **專屬子資料夾**中，因此多層級擷取絕不會混淆不同類型。 下圖展示**由 `lattice capture` 對單一攝影機進行的預設 `all` 擷取** — 每台攝影機包含多個層級，因此每個檔案都帶有 `_<level>` 的副檔名：

```
output/
├── raw/           capture_<ts>_SN<serial>-<filter>_raw.tiff
├── debayered/     capture_<ts>_SN<serial>-<filter>_debayered.tiff
├── radiance/      capture_<ts>_SN<serial>-<filter>_radiance.tiff
├── reflectance/   capture_<ts>_SN<serial>-<filter>_reflectance.tiff
├── preview/       capture_<ts>_SN<serial>-<filter>_display.tiff
├── index/         per-camera vegetation-index (LUT) render, when --index is on
├── composite/     array foreground/background live-view composite, when produced
├── clamp_masks/   radiance/…_radiance_clampmask.tif, reflectance/…, index/… — the unmix clamp
│                  masks, one subfolder per product folder, only when a band clamped
└── daq/           daq_data_<sensor id>_<ts>.daq: the downwelling reading matched to the capture
                   (+ .daq.cameras.json: whose)
```

檔案名稱的變化方式：

- **相機欄位 — 序列號，接著是濾鏡：**每個擷取檔案皆以拍攝該檔案的相機名稱作為檔案名稱，格式為 `SN<serial>-<filter>`，其中 `<filter>` 為相機型號中的濾鏡部分： `SN261902142-F450` 對應於 `M3M-L87-F450`， `SN213800234-FRGN` 對應於 `M3C-L87-FRGN`，`SN214701292-FRGB` 則對應於 RGB 相機。濾鏡部分將序列號與 `-` 結合， 而非 `_`，因此名稱中的 `_` 分隔字段與先前相同（前綴、日期、時間、毫秒、攝影機、等級）。 若無法讀取相機型號（DeviceUserID 已被清除），則僅以 `SN<serial>` 命名。早期版本的擷取檔名稱中不包含篩選器；Chloros 會讀取這兩種格式。
- **前綴：** `lattice capture` 會將檔案命名為 `capture_…`；若為連線陣列，則命名為 `array-capture`、`multi-capture`，而 `project capture` / `burst` / `stream` 等陣列名稱，則命名為 `sync_<ts>_SN<serial>-<filter>…`。
- **層級後綴僅適用於包含多個層級的情況。**若攝影機在單次擷取中儲存**多個**層級（例如 `all` 或 `--levels a,b`），則會追加 `_<level>`。若攝影機僅儲存**一個** 層級的相機則保留原始名稱，不附加後綴 — `--processing reflectance` 會寫入 `reflectance/sync_<ts>_SN<serial>-<filter>.tiff` — 因此針對單層級擷取所編寫的腳本仍可正常運作。 層級仍由資料夾名稱決定。
- **裁切遮罩跟在產品名稱之後：** `<product name>_clampmask.tif`。單層反射率擷取會寫入 `clamp_masks/reflectance/sync_<ts>_SN<serial>-<filter>_clampmask.tif`；而 `all` 的擷取則會寫入 `clamp_masks/reflectance/sync_<ts>_SN<serial>-<filter>_reflectance_clampmask.tif`。
- **副檔名** 跟在 `--format` 之後：預設的 `tiff` 會產生 `.tiff`，`--format tif` 則會產生 `.tif`（鉗位遮罩始終為 `.tif`）。
- **唯一例外：** `lattice capture --processing debayered`（或 `--levels debayered`）會取一個快速的單一幀路徑，並將 `capture_<ts>_SN<serial>-<filter>.tiff` 直接寫入 `-o` — 沒有子資料夾。
- **串流與突發傳輸** 攜帶相同的欄位： `lattice continuous` 會寫入 `frame_<seq>_SN<serial>-<filter>.tiff`、`lattice multi-stream`、`stream_<seq>_SN<serial>-<filter>_<frame id>.tiff`，以及一個原始連拍檔 `<burst>_f<frame>_SN<serial>-<filter>_raw.tiff`。

`<ts>` 為擷取時間戳記，而 `<serial>` 為相機序號，因此一個同步群組在各台相機間共享同一
時間戳記。**請注意一處不對稱之處：** `display` 層級的檔案儲存於名為
`preview/` 的資料夾中，而多層級檔案僅在該層級的名稱 — 僅此層級的資料夾名稱與後綴會有所不同
。未知層級則會回退至以其自身名稱命名的資料夾，若無法建立子資料夾
，檔案將寫入輸出根目錄，而非遺失。

**重新處理「captures」資料夾：**將 `chloros-cli process` 指向**擷取根目錄**
(`output/`)。`process` 通常僅匯入您指定的資料夾，但當該資料夾內沒有
且確實存在子資料夾時，它會自動向下遍歷 — 因此該層級的子資料夾以及
`daq/` 子資料夾都會一次全部被擷取。擷取的每個層級都會以單一影像的形式匯入，
其餘層級則可作為模式使用，而非每層級各為一幅影像。

直接命名 **層級子資料夾**（例如 `output/raw/`）同樣有效。如此操作會遺留 `daq/`
，因此當您重新從 `raw/` 衍生輻射測量
產品時，請一併複製或指向該 DAQ 讀取資料 — 否則時間戳記的匹配將無對應對象可供解析。

**處理程序總是從 `raw` 開始。** 在每個擷取中，原始幀是處理管線的來源；
`debayered`、`radiance`、`reflectance` 及 `preview` 雖作為可檢視模式存在，但絕不會
被送回處理管線。重新處理衍生產品將再次套用已烘焙至其像素中的暗角、CCM 及
輻射度運算，因此 Chloros 會選擇拒絕處理，而非
進行重複處理。有兩點值得注意的後果：

- `index/` 和 `composite/` 的渲染結果 **絕不會** 經過處理。它們是輸出檔，而非擷取檔——
  NDVI LUT 渲染結果不具備有意義的輻照度詮釋。
- **未**包含 `raw`（例如 `array-capture --processing reflectance`）而匯出的擷取資料夾
  沒有有效的處理流程來源。這些擷取檔雖能正常匯入並顯示，但 `process` 會跳過
  它們並顯示此訊息：

  ```
  [IMPORT-LEVEL] Skipping 4 already-processed file(s) with no raw source: capture_…_reflectance.tif
  [IMPORT-LEVEL] Processing starts from raw. Re-capture with --processing raw, or force an entry
                 point with --input-level.
  ```

  若您確實需要將衍生產品傳送通過——例如啟用
  `demosaic` 啟用狀態下擷取的匯集點會話，或是舊版資料夾 — `--input-level {raw,debayered,processed}` 會強制指定
  入口點並覆寫跳過設定。該標誌是刻意設計的應急方案；`auto` （預設值）
  絕不會處理沒有原始資料的擷取檔。

**校準標籤：僅限 `raw` 適用。** 擷取資料中的 `raw` 檔案包含相機的校準封包：`CalibrationJSON` 有效載荷以及與 MicaSense 相容的標籤（`BlackLevel`、 `RadiometricCalibration`、`Band{R,G,B}RadiometricCalibration`、`MicaSenseVignetting*`），這些標籤用於描述感測器的原始數值。 其他各層級的檔案（`debayered`、`display`、`radiance`、 `reflectance`、`index/` 渲染檔）以及每個 `lattice continuous` / `multi-stream` 幀均攜帶相機的識別資訊（`XMP-mapir:CameraSerial`、`CameraModel`、`FilterType`、 `LensModel`、`CalBundleSha256`、……）以及該幀本身的記錄檔 `XMP-mapir:FrameJSON`（曝光、增益、溫度、像素格式、擷取時間、 DAQ 匹配），因為每個 `process` 產品都攜帶此識別碼。`process` 會根據其序號在 Chloros 的校準快取中查找該檔案的校準資料，若該 電腦尚未擁有該套件時，僅需下載一次。

#### 由多台相機產生的檔案

由多台相機的波段組合而成的檔案——多波段疊加圖（`align-apply`、`align-stream`、`index --live --save-multiband`）， 由多台相機生成的即時索引檔（`index.tif`、`index_LUT.png`），以及陣列擷取的 `composite/` 影像——會列出所有參與生成的相機名稱：

- **按頻帶順序排列的各攝影機清單**（合成圖：背景層，接著是前景層；索引層，以及其頻帶來源的攝影機）：`XMP-mapir:SourceCameraSerials`, `SourceCameraModels`、`SourceSensorProfiles`、`SourceProductLines`、`SourceFilterTypes`、`SourceLensModels`、 `SourceCalDateUTCs`、`SourceCalBundleSha256s`（各相機波段所經過的校正套件，否則 — 一個 `align-apply` / `align-stream` 堆疊，其波段未經任何校正 — 該相機的Chloros套件已快取，即其擷取檔案所命名的套件）， `SourceLightboxCertIds`、`SourceHalogenLampCertIds`、`SourcePackTiers` ——並列清單，其中每個清單的第 *n* 項皆描述同一台相機；`-` 則表示該相機無相關值（無校準套件，或鏡頭型號未知）。 每台相機佔用 **一個** 項目：若其封裝中某個欄位有多個值 — 例如封裝中的鹵素燈認證 — 則以 `;` (`F-1479-1;F-1480-1;F-1481-1`) 連接，絕不使用逗號， 因此，無論列印方式為何，清單中每台相機皆對應一個項目。
- **EXIF `SerialNumber`** 則包含相同的序列號，以逗號分隔（`214000535,214000551`）。
- 它**不**包含單一相機的識別碼（`XMP-mapir:CameraSerial`、`CalBundleSha256`、……）： 這些識別碼分別代表一台相機及其感測器組，而 `process` 則會根據這些資訊解析檔案的校正參數。由多台相機的波段資料計算出的索引（`index.tif`、 `index_LUT.png`）所計算出的索引，其 EXIF 資料中會標示為 `Model` `LATTICE` （即攝影架）的資料，而非單一相機的 `LATT-…` 型號。

由 **單一** 相機的波段所組成的檔案——無論是每波段 TIFF 檔案（`--per-band`）、`align-apply --per-camera` 單幀影像、單一相機的堆疊或即時索引——皆會攜帶該相機自身的識別標籤，如同其他所有擷取檔案一樣。 `align-apply` 的堆疊檔、各波段檔及各相機檔，以及每個 `align-stream` 堆疊檔及其各波段檔，皆包含其群組的擷取時間（參考相機的曝光時間，UTC）， 而單色相機的 `--per-camera` 幀僅包含一個平面，其格式與堆疊檔案相同（如 `--bit-depth` 所示）。**相機方向（翻轉／旋轉）。** 相機的方向設定（GUI 中的「鏡像水平／
垂直鏡像／旋轉）會影響擷取內容的儲存方式：

- **`raw` 與 `debayered` 會完全依照感光元件讀取的原始狀態儲存**，無論設定為何。若
  若拜耳馬賽克被旋轉或翻轉，將形成不同的色彩濾鏡圖案，且校正
  映射（暗電偏移、平場）皆基於感光元件自身的網格。
- **`preview`/`display`、`radiance`、 `reflectance` 以及 `index/` 的渲染檔均已校正方向** —— 在完成所有校正運算後
  旋轉了一圈，因此開啟時呈現的畫面與相機取景時的方向一致。這些
  `clamp_masks/…_clampmask.tif` 檔案位於同一網格上。
- **每個擷取檔案都會在 XMP 中記錄設定**： `XMP-Camera:ChlorosFlipHorizontal` /
  `ChlorosFlipVertical` (`true`/`false`)、`ChlorosRotation`（0 / 90 / 180 / 270，順時針方向，在
  翻轉操作之後），以及 `ChlorosOrientationApplied` — `false` 針對原始檔與去拜耳化（原生像素）的
  定向產品。`true` 則列出針對該檔案的
  `true` 則適用於已定向的產出檔。`ChlorosProcessing` 列出對該檔案
  像素所施加的操作（若為原生像素則為 `none`）。這包含 `lattice continuous` 串流的每一幀（
  相機本身的設定，未套用）以及 `lattice multi-stream` 的每一幀（單位元組：其
  成員幀為原生像素，如同陣列擷取一般）。 `lattice align-stream` / `align-apply`
  堆疊記錄主相機的設定（未套用）：其波段會被變形至主相機的
  幀中，但不會旋轉。
- **`process` 在處理原始影像時會套用記錄的方位**：首先在感測器座標系中進行暗電流偏移、平場校正、
  去馬賽克及解混，接著進行陣列對齊變形，最後
  最後才進行方向校正——因此預覽、輻照度、反射率及指數產出結果與
  即時擷取的完全一致，並記錄為 `ChlorosOrientationApplied=true`。未經校正的備用
  產出（`Sensor_Response_Images`、`Vignette_Corrected_Images`）在
  平場校正後，其方向與原始影像相同。去拜耳化產品則維持原始狀態。
- **Chloros影像檢視器會根據記錄資訊旋轉原始檔或去拜耳化檔案**（僅限顯示 —
  檔案本身不會被修改），因此其外觀與產品一致； 縮圖、全尺寸檢視、
  游標讀數以及索引/LUT 沙盒均採用相同的旋轉角度。未包含記錄的檔案，以及
  （已校正方向的）產品，則會以儲存時的原始狀態顯示。 沙盒匯出會記錄其
  像素所攜帶的方位（`ChlorosOrientationApplied=true`；此為無記錄層的標識）。
- **未包含記錄的擷取檔案將以原生格式處理**（行為未變更）。在 1.3.0 版本之前，
  若單鏡頭擷取設定了翻轉或旋轉，系統會自行旋轉其原始檔與去拜耳化檔案；
  Chloros 無法辨識此類已-旋轉過的原始檔與原生檔區分，也不會進行推測，因此會
  直接按原始儲存狀態進行處理。校準圖是基於感光元件自身的網格，因此對於此類
  擷取，若其校準結果至關重要，請改用以 1.3.0 版本拍攝的校準圖。
- **陣列預設不會旋轉任何成員檔案**（對齊變形與 `Alignment*` XMP 皆位於
  相機自身的、未旋轉幀中），並記錄其原始狀態。 `array-capture --orient` 會將每個
  成員的處理結果，在對齊變形後：依據 **主影像** 在
  變形至其座標系時的方位進行旋轉（以確保成員保持共註冊），否則則依據其自身相機的方位。 每個
  檔案皆記錄其自身像素所處座標系的定向——原始資料採用其自身相機的座標系——
  而攜帶對齊變換的原始資料亦會記錄 `ChlorosAlignedFlipHorizontal` /
  `ChlorosAlignedFlipVertical` / `ChlorosAlignedRotation`：主影像的變換參數，適用於已變形
  至主相機的畫面中。`process` 會根據其像素所屬的變形量旋轉每個產品：
  若在變形原始檔時已進行對齊（`--array-alignment`，預設值——即即時對齊
  擷取的產出），若未對齊則採用相機本身的（`--no-array-alignment` ——即即時
  未對齊擷取的產出）。`array-burst`、`project capture` / `burst` /
  `stream`、配方以及SDK中皆提供此選項。

### 混合濾光片陣列中的跳過擷取

當您在同一個陣列中混合使用RGB和多光譜相機時，`array-capture --processing radiance`（或 `reflectance`）會儲存多光譜幀，並**跳過** RGB 相機的擷取——對於寬頻感測器而言，每格拜耳陣列的輻度值並無實質意義。 CLI 會明確列出每個已儲存的檔案（連同其匯出等級）、每個寫入的 `.daq`，以及每個跳過的情況，因此檔案數量並不出人意料：

```
  Saved: …/output/reflectance/sync_…_SN213800234-FRGN.tiff [reflectance] (SN:213800234, fid:1)
  Saved: …/output/reflectance/sync_…_SN214000533-FOCN.tiff [reflectance] (SN:214000533, fid:1)
  Saved: …/output/reflectance/sync_…_SN214701288-FNGB.tiff [reflectance] (SN:214701288, fid:1)
  DAQ:   …/output/daq/daq_data_daq-e-54b5e0_….daq
  Skipped: SN:214701292 (reflectance-not-applicable-to-rgb-cam filter=RGB)

  3 synchronized frames captured. (1 skipped)
```

跳過原因代碼遵循 `<level>-not-applicable-to-rgb-cam` 的模式。若相機的校準套件未載入，則會跳過 `radiance` / `reflectance`（當 `<level>-needs-calibration (connect the camera so its cal pack loads)` 跳過；若無可用的 DAQ 反射率讀數，則以 `reflectance-skipped-no-dls` 跳過。反射率亦可能以 `reflectance-skipped-no-fresh-dls` / `reflectance-skipped-bound-daq-unavailable (…)` 跳過， 以及當波段主要位於 DAQ 光感測器的輻測校準範圍（約 374–974 nm）之外時，`dls-uncalibrated-band-<nm>` 也會出現此現象——在現有出貨 SKU 中，僅有 F988 支援此路徑，其支援的工作流程為反射率面板工作流程。

若要包含所有相機（不論濾光片類型），請使用 `--processing debayered`（或 `display`）；若要一次取得每台相機的所有適用級別，則請使用預設的 `all`。

### 與曝光值相距甚遠的 DAQ 讀數

每次擷取所儲存的 `.daq` 皆為與該曝光值對應的 DAQ 讀數，而其旁的 `.match.json` 則記錄兩者之間的差距 (`gap_ms`)，以及該偏差是否在容差範圍內 (`in_tolerance`，預設為 500 毫秒，除非擷取設定中另有指定 `--daq-tolerance-ms`)。 超出容差範圍的讀數仍會被採用，因此不會遺失任何幀，且會加以標記：反射率檔案的 XMP 中會包含 `DlsInTolerance=False` 和 `DlsMatchGapS`。 `lattice capture`、`multi-capture` 和 `array-capture` 也會在相關的 `.daq` 下輸出此資訊，並在說明時標示為 DAQ'資料擷取（DAQ）串流故障：

**讀數如何與曝光時間配對。** 擷取指令會等待曝光結束後 DAQ 的第一個讀數，以便將該讀數前後的讀數進行融合（`method` `interpolated`）。DAQ-E 在日光下每 0.4 秒傳送一次讀數，室內則每 1-2 秒傳送一次，因此等待時間會使指令總計增加約一個半的間隔，而非曝光時間。若 DAQ 已停止傳送，則不會等待。DAQ-E 的讀數代表自上次讀數以來該間隔內的平均光照值，因此會置於該間隔的中點：`gap_ms` 是以曝光時間為起點計算至該中點，而 `bracket_stamps_ns` 則顯示讀數本身的時間戳記。 `array-capture --continuous` 會進行等待，但不會減慢擷取速度：其原始檔會立即寫入，而 `.daq` 則在讀數送達後隨即寫入。 透過 `--interval`，每個循環的等待時間皆等同於單次擷取。

```
  DAQ:   …/output/daq/daq_data_daq-e-54b774_….daq
         DAQ reading 114.9 s from the exposure (tolerance 500 ms): reflectance uses it, labelled DlsInTolerance=False
         DAQ stream fault: no frame accepted from daq-e-54b774.local in 25 s, after 1932 good frame(s). …
```

若該讀數未寫入任何反射率檔案，則第一行將以 `saved anyway; its .match.json says in_tolerance: false` 結尾。

該擷取仍以 `0` 結束：所有檔案均已寫入，且標示其內容。若資料擷取裝置（DAQ）的資料流已停滯，它仍會持續傳送最後一次讀數，因此在依賴該擷取的反射率之前，請先透過 `chloros-cli daq pool-list` 進行檢查。 後端會重新連線至已停止傳輸資料的彙總型 DAQ-E （參見 [停止傳輸資料的 DAQ-E](#a-daq-e-that-stops-delivering)）。超出容差範圍的讀數僅會輸出其 `DAQ:` 行，除非該讀數屬於保留讀數（詳見下文）。

**DAQ 離線後仍被保留的讀數。** 當沒有任何 DAQ 能為某次曝光提供讀數時（綁定的 DAQ 不可用，或沒有任何彙總的 DAQ 持有讀數，例如在 `daq pool-disconnect` 之後），擷取程序將重複使用 Chloros 最後擁有的讀數，其 `.match.json` 會針對此曝光描述該讀數：`method`、`retained`、`gap_ms` 從本次曝光對應至產生該讀數的樣本、`in_tolerance` 來自該間隙，以及 `retained_from_epoch_ns`——即該讀數最初被配對的曝光。因此，在 DAQ 停止運作很久之後擷取的影像，會輸出上述行，並在 下方則會多印出一行，即使讀數仍在容差範圍內，該行仍會被印出：

```
         This is the DAQ's last reading: the DAQ is no longer reporting, so each capture reuses it until a new one arrives
```

其 `.daq` 仍會標示出擷取該讀數的 DAQ，以及該讀數所採用的校準參數與蓋帽，因此重新匯入時便能得知是哪個 DAQ、校準參數和蓋帽產生的。

### 無法寫入的檔案

若某台攝影機的檔案無法寫入（例如硬碟已滿或權限錯誤），並不會阻止其他攝影機的檔案儲存。每個擷取指令（`lattice capture`、`multi-capture`、 `array-capture`；`project capture`、`burst`、`stream`； 以及配方中的 `capture`、`burst` 和 `stream`）都會將每個失敗的檔案輸出至 stderr， 隨後 **退出 `1`**，即使其餘檔案已成功儲存：

```
  Failed: SN:214701292 [reflectance] OSError: [Errno 28] No space left on device
  Disk full: the drive the backend captures to has no space left. Free space on it (or capture to another drive) before capturing again.
```

每個擷取任務應產生的檔案均已涵蓋，其中包含單一單幀去拜爾化處理的 `lattice capture --levels debayered` 亦包含在內。若某個 `.daq` 讀取作業的 `.match.json` 來源副檔無法寫入，則會被列為 `[match]`（該 `.daq` 檔案本身已儲存）。

在磁碟空間已滿的情況下，重複的擷取（`array-capture --continuous` / `--interval`、`project burst` / `stream`，以及一個配方 `burst` / `stream`) **在該輪次停止**，而非在後續執行過程中對同一磁碟機完全不寫入任何資料。任何其他錯誤都會在發生當下的每輪次中被報告，而執行程序會繼續進行至結束，然後退出 `1`。`array-capture --continuous` （後端的連續擷取）不會列出任何檔案（包含失敗的檔案）：它會在進度與摘要行中統計這些檔案 （`N failed`、`N file(s) failed to write.`）中進行計數，並退出 `1` — 請參閱 [連續模式](#continuous-mode---continuous)。

記錄器 — `array-burst` 和 `array-record` — 會回報自身的失敗狀況。 當磁碟機空間已滿，或即將滿時，`array-burst` 會自行停止運作。這兩者都會在發生其他寫入失敗的情況下繼續運作。當其中任一者回報錯誤時，該錯誤訊息會顯示在計數器之後，且指令 **退出 `1`**：

```
  Recorder error: disk-full: OSError: [Errno 28] No space left on device
  Disk full: the drive the backend captures to has no space left. Free space on it (or capture to another drive) before capturing again.
```

`Disk low:` 取代 `Disk full:` 表示該批次在填滿 。停止前已錄製的幀將被保留。當空間用盡時，`array-burst --build` 會跳過此建置，因為影片將寫入同一顆硬碟。請先釋放空間，然後執行 `array-build-video --burst-dir …`。發生任何其他錯誤後，建置仍會繼續執行。

`array-record` 若完全未寫入任何幀，也屬於錄製失敗 （`Recorder error: no-frames: the combined view published no frame to record …`，退出 `1`）— 請參閱 [`array-record`](#array-record--combined-index-videogif-monitoring-grade)。

---

## 擷取模式、錄影機與離線重新處理

這些功能皆在 **持久性陣列** 上運作（請先執行 `array-connect`）。它們與 GUI 擷取面板的功能相呼應。

### `array-capture` 模式

`array-capture` 是一項單一指令，包含四種快門模式以及一組匯出切換選項：

| 模式 | 標誌 | 行為 |
| --- | --- | --- |
| **單次** *(預設)* | (無) | 執行一個同步擷取群組，然後退出。 |
| **連續** | `--continuous` | 連續進行擷取，直到 `Ctrl+C`、`--count N` 擷取， `--duration S`，或磁碟已滿為止。執行後端自身的連續擷取（即 GUI 的「連續模式」）——請參閱下方的 [連續模式](#continuous-mode---continuous)。 |
| **間隔** | `--interval S` | 每 `S` 秒執行一次（從每次循環開始計時），範圍相同。 |
| **最快** | `--fastest` | 僅原始模式 + 指派的 DAQ 讀數 + 組合指數複合值；跳過輻照度／反射率／顯示運算，使畫面快速呈現。此模式隱含 `--processing raw --force-daq`。稍後將儲存的 `.daq` 重新處理為校正後的產品。 |

匯出切換選項（可與任何模式結合使用；皆共用同一 GUI／SDK 端點）：

| 標誌 | 效果 |
| --- | --- |
| `--array-id ID` | 擷取來源陣列（僅連接一個時請省略）。 |
| `-o, --output DIR` | 輸出資料夾（預設為 `output`，位於執行指令的資料夾之下）。 |
| `--format {tiff,tif,png,jpg,jpeg}` | 檔案格式（預設為 `tiff`）。 PNG / JPEG 僅支援 `display` 及索引渲染；請參閱 [擷取匯出層級](#capture-export-levels-the-all-default)。 |
| `--processing LEVEL` | 單一匯出層級 (`raw`、`debayered`、`radiance`、`reflectance`、`display`)，或 `all` （預設）。 |
| `--levels L1,L2,…` | 匯出類型的明確子集（例如 `raw,radiance,reflectance`）；**會覆寫 `--processing`**。 |
| `--aligned` / `--no-aligned` | **預設：對齊，如同 Chloros 應用程式對同一陣列的擷取方式**（其每組陣列的**對齊** 擷取設定，預設為開啟，除非您將其關閉）。每個成員的非原始資料匯出（`debayered`、`display`、`radiance`、`reflectance`，即索引渲染） 都會被變形至主相機的幀內，並裁切至各成員的共同重疊區域，因此這些檔案在像素上已對齊，且輸出尺寸、裁切範圍及 `Alignment*` 標籤均與應用程式所產出的相同——比感光元件 （或 ROI）小；若採用 ROI，變換會重新基於該視窗進行。`XMP-mapir:AlignmentJSON` 表示 `"applied": true` 並產生 `crop_roi`。 原始檔將維持原生尺寸且未經變形，但會在元資料中攜帶變換資訊 (`"applied": false`)。`--no-aligned` 會將每個檔案以原生尺寸儲存，且未經變形。 `--aligned` 即為預設設定。無論是否套用變形，一旦陣列擁有對齊設定檔，每個成員檔案都會記錄其相機的 `Alignment*` 變換 ——但若某台相機無法與配置檔對齊（被排除在外，或因解算失敗而被迫設為單位矩陣），以及在無其他相機對齊時作為參考的相機則除外：這些相機不會攜帶 `Alignment*` 標籤。若陣列上無配置檔，擷取資料將以未對齊狀態儲存，且 CLI 會於 `Alignment:` 行中列印原因，此舉亦適用於其他無法套用變形的情況（例如：成員解析度不同、陣列未透過纜線同步，或輪廓無法映射至幀上）。陣列的自動對齊功能會在曝光穩定後生成輪廓； `align-calibrate` 則會根據需求建立對齊設定。 |
| `--orient` | 將每個成員的 **產出**— `display`、`radiance`、 `reflectance`、`index/` 渲染檔以及 `composite/` —— 根據相機方向，在對齊變形後：將產品變形至主幀（預設）中，並以**主幀**的朝向進行變形，使各成員保持對齊；未對齊的擷取（`--no-aligned`，或無法套用變形的擷取）則採用其自身相機的朝向。**預設：關閉 — 每個成員檔案均保持原生格式**，如同陣列擷取始終如此。 無論哪種情況，`raw` 和 `debayered` 均保持原生狀態。每個檔案都會記錄其所包含的產出檔案所採用的方向（參見 [相機方向](#what-a-captures-folder-looks-like))；`Alignment*` 的 XMP 檔案保留原生座標系變換（在方向設定之前套用），因此 `process` （先變形，再進行方向轉換）可重現原始產出。 |
| `--index` / `--no-index` | 若已設定，則儲存／跳過每台相機的植被指數疊加層。預設：進行渲染。 |
| `--no-daq` | 當未選取的層級未使用 DAQ（僅擷取原始資料）時，跳過 `.daq` 輔助檔案。 預設情況下會儲存讀數，以便日後離線將幀重新處理為反射率／植被指數。`--force-daq` 仍被接受（此為預設值）。 |
| `--daq-tolerance-ms MS` | 這些擷取的 DAQ 匹配容差 （預設 500 毫秒）。 |
| `--smart` | 在觸發前等待所有相機的自動曝光（AE）穩定（參見 [Smart-AE / Smart-Capture](#smart-ae--smart-capture)）。 |
| `--compression {deflate,none}` | TIFF 像素壓縮。`none`（自 1.3.0 起為預設值；1.2.0 版本寫入 `deflate`）= 未壓縮 — 感光元件的像素完全依照讀取結果呈現，每張全解析度畫面約 6.3 MB，且具備最快的持續寫入速率；`deflate` = 無損 zlib L1 + 水平預測器，每幀約 4.1 MB，速度較慢。兩者皆為無損格式，且在匯入時顯示結果完全相同。 |

> **單次寫入 TIFF + 持續傳輸率模型。**每個檔案僅寫入**一次**：像素、 XMP 封包、IFD0 中的製造商/型號/相機序號，以及 EXIF `ExposureTime` / `SerialNumber` 標籤，皆在單次寫入過程中完成，之後不會有任何內容覆寫該檔案。`--continuous` 會擷取連續且同步的群組 — 每次擷取都會接續前一個群組，包括在該次擷取仍在儲存時已發布的群組 — 因此其運作速度與陣列的速率一致。DEFLATE 會佔用 Python 的 GIL，因此壓縮寫入**不會**在各相機寫入執行緒間並行處理； 若要以感測器速率持續進行全解析度擷取，則需要未壓縮寫入（預設設定）**以及** NVMe 級別的硬碟（9 台相機陣列在約 10 fps 時，持續傳輸速率約為 600 MB/s）。此設定選項在 `POST /api/camera/array/capture` 上亦以 `compression` 形式提供。

```bash
# Interval timelapse: one reflectance pass every 10 s for 5 minutes
chloros-cli lattice array-capture --interval 10 --duration 300 \
  --processing reflectance -o timelapse/

# Fastest grab for a moving rig — raw + .daq now, calibrate later
chloros-cli lattice array-capture --fastest -o flightline/

# Co-registered multi-band export (aligned is the default; drop the index overlay)
chloros-cli lattice array-capture --processing reflectance --no-index -o out/

# Each member at its native size, unwarped
chloros-cli lattice array-capture --processing reflectance --no-aligned -o native/

# Continuous raw capture for 60 s, as fast as the array publishes
chloros-cli lattice array-capture --continuous --duration 60 --fastest -o flightline/
```

#### 連續模式 (`--continuous`)

`array-capture --continuous`（不包含 `--interval` 或 `--smart`）會將執行權交給 **後端自身的連續擷取功能** —— 也就是 GUI「連續模式」所執行的會話。 後端會自行儲存每個同步群組，同時保持兩個擷取任務正在執行，且群組之間不會有來自CLI的請求，因此能跟上高速陣列的節奏——在每擷取一次才處理一個請求的情況下，後者會落後（在 9 台相機的陣列上，以 512 像素 ROI 測量，速度為 73–77 次擷取/秒）。 上述所有匯出設定均適用於每個擷取。

- **邊界：** `--count N` 將在 N 次擷取後停止（一次擷取 = 一個同步群組：該群組中所有成員的檔案）；`--duration S` 則在 S 秒後停止。若未設定上述任一條件，程式將持續運行直至 `Ctrl+C` 觸發、硬碟空間已滿、發生錯誤，或陣列離線為止。`Ctrl+C` 會終止後端的會話， 隨後輸出摘要。
- **進度** 每 2 秒更新一次，當數值不為零時會附加 `, N failed` 與 `, N groups skipped`（被跳過的小組是指陣列已發布但未被任何擷取處理的小組）：

  ```
     12.0s  880 captures (73.3/s), 7920 files
  ```

- **不列出單一檔案清單。** 執行結束時會顯示一行摘要——包含擷取數量、速率、檔案數及輸出資料夾——若非零值，則後接 `N file(s) failed to write.` / `N group(s) skipped.`：

  ```
  Done — 880 captures (73.3/s), 7920 files in /home/user/flightline.
  ```

- **停止** 情況：當磁碟空間已滿時顯示 `Continuous capture stopped: the drive is full.`；發生錯誤或陣列消失時顯示 `Continuous capture stopped: <reason>.`（兩者皆輸出至 stderr）。
- **退出狀態：** 當已儲存擷取內容且所有檔案皆已寫入（包含 `Ctrl+C`）時，輸出 `0`；若任何檔案寫入失敗、 磁碟空間已滿、發生錯誤、陣列消失，或完全沒有擷取資料時。
- **Microsoft Defender：**當 Defender 耗用一個或多個 CPU 核心來掃描擷取程式寫入的檔案時，執行過程會在標準錯誤輸出 (stderr) 上顯示一次：`Note: Microsoft Defender is using N.N CPU cores scanning the capture files, CPU the capture needs. Re-run the Chloros installer with "Exclude the Chloros backend from Microsoft Defender scanning" checked.`。請參閱安裝程式的 [系統設定](#windows-exe)。
- 每個陣列一次僅能進行一項連續擷取；若嘗試進行第二項擷取，將顯示錯誤訊息 `a continuous capture is already running for this array`。 透過 HTTP 建立的連線為 `POST /api/camera/array/continuous/start`（包含 `/api/camera/array/capture` 主體以及 `max_captures` / `duration_s`）， `GET /api/camera/array/continuous/status?array_id=ID` 以及 `POST /api/camera/array/continuous/stop`。

**Interval**（`--interval S`，無論是否包含 `--continuous`）以及**`--continuous --smart`** 保留每請求循環：每輪一次擷取請求 — 對於「間隔」模式，在每個排程時刻執行；對於 `--smart`，則在 AE 穩定後執行 ——每輪的檔案清單格式與單一模式相同，末尾有一行 `Done — N capture pass(es).`，並以 `--count N` 計數輪次。

### `array-record` — 視訊／GIF 合併索引（監控級）

將陣列的 **即時複合索引視圖** 記錄至 `.avi`（並可選地記錄至 `.gif`）。當 GUI 的複合視圖開啟時，它會將該視圖以顯示的樣貌進行記錄。 若未開啟任何視圖，後端將自行渲染視圖：即您透過 `--expr` 傳入的索引，或預設情況下，在陣列對齊後，陣列的 NDVI。 預設的 NDVI 會使用最接近 660 nm 和 850 nm 的紅光與 NIR 波段，與即時檢視相同。在錄製期間開啟的 GUI 檢視，只要保持開啟狀態，就會被記錄下來。它每 2 秒輪詢一次進度，並在接收到 `--duration`、 `Ctrl+C`，或當錄影器自行結束時。

若錄製過程中未寫入任何幀，則錄製失敗：錄製器會回報 `no-frames: …`（陣列可能仍在對齊或穩定曝光），且指令 **將退出 `1`**。

```bash
# 30-second combined-index clip at 10 fps, plus a GIF (no GUI needed)
chloros-cli lattice array-record --duration 30 --fps 10 --gif -o monitoring/

# Record a chosen index instead of the default NDVI
chloros-cli lattice array-record --duration 30 \
  --expr "(NIR_850_RGN-Red_660_RGN)/(NIR_850_RGN+Red_660_RGN)" -o monitoring/
```

| 標誌 | 預設值 | 說明 |
| --- | --- | --- |
| `--array-id ID` | 僅陣列 | 目標陣列（若僅連接一個則省略）。 |
| `-o, --output DIR` | `output` | 輸出目錄。相對路徑以您 執行指令的資料夾之下。 |
| `--fps F` | `10` | 錄製幀率。 |
| `--duration S` | 直到按下 Ctrl+C | `S` 秒後自動停止。 |
| `--gif` | 關閉 | 同時寫入動畫 GIF 檔。 |
| `--gif-only` | 關閉 | 僅寫入 GIF 檔（不包含 `.avi`）。 |
| `--expr EXPR` | 陣列的NDVI | 當未開啟任何 GUI 視圖時，用於記錄索引表達式的陣列波段名稱（例如 `'(NIR_850_RGN-Red_660_RGN)/(NIR_850_RGN+Red_660_RGN)'`； 參見 [波段名稱](#index--vegetation-maths))。簡短的顏色名稱（`NIR`、`Red`）僅在陣列中有一波段具備該名稱時才有效。若 GUI 視圖處於開啟狀態，則會記錄其顯示內容。 |

### `array-burst` — 原始拜耳高幀率連拍（分析級）

直接讀取抓取迴圈的同步群組緩衝區 — **無需校準鏈、無需 exiftool、無需即時預覽** — 因此可達到相機的完整抓取速率。寫入原始影像幀 + 每幀清單 + 每個獨立 DLS 讀數對應一個 `.daq` 檔案，並存放於 `<output>/bursts/<base>/` 目錄下。 可離線重新處理（下一個指令），或傳入 `--build` 以便在停止時立即執行。

```bash
# 5-second raw burst, then build the combined index video in one shot
chloros-cli lattice array-burst --duration 5 --build \
  --products combined:index --fps 10 -o capture/
```

| 標誌 | 預設值 | 說明 |
| --- | --- | --- |
| `--array-id ID` | 僅限陣列 | 目標陣列。 |
| `--array-id ID` | 僅限陣列 | 目標陣列。 || `-o, --output DIR` | `output` | 輸出目錄（批次處理結果存放在 `<DIR>/bursts/<base>/`）。 |
| `--duration S` | 直到按下 Ctrl+C | 在 `S` 秒後自動停止。 |
| `--max-frames N` | 無限制 | 在處理完 `N` 個原始幀後自動-停止 |
| `--orient` | 關閉 | 建構從此批次產生的結果會根據相機方向進行旋轉——單一相機的結果依其自身相機方向旋轉， 合併索引則依主相機的方位角旋轉——如同 `array-capture --orient` 所示。原始影像仍保持原始格式； 該連拍的 `manifest.json` 會記錄每個成員的方位與選項。 |
| `--build` | 關閉 | 停止後，立即重新處理該連拍（與 `array-build-video` 相同）。 |
| `--products …` | `combined:index` | 搭配 `--build` 使用時：要建立哪些影片（參見下方）。 |
| `--fps F` | `10` | 搭配 `--build` 時：輸出影片的每秒幀數 (fps)。 |
| `--save-tiffs` | 關閉 | 與 `--build` 搭配使用時：同時儲存每幀經校準的 TIFF 檔案（每個檔案皆包含其 [解混鉗位遮罩](#unmix-clamp-mask-which-pixels-a-band-was-clamped-on)；建置結果中將其列為 `clamp_masks`)。 |
| `--gif` | 關閉 | 啟用 `--build` 時：亦會寫入動態 GIF 檔案。 |

### `array-build-video` — 離線重新處理已儲存的連拍

將每個原始幀與最接近的已儲存 `.daq` 讀數進行時間對齊，並使其通過 **與匯入管線相同的輻照度／反射率／指數處理鏈**，渲染一個或多個影片。

`--products` 是由 `kind:level` 項目組成的逗號分隔清單，其中 `kind` ∈ `per_cam` | `combined`，且 `level` ∈ `radiance` | `reflectance` | `index`。 若僅指定 `level`（未指定 `kind:`），則預設值為 `per_cam`。預設值為 `combined:index`。

```bash
# Per-cam reflectance video for every member + one combined NDVI video
chloros-cli lattice array-build-video \
  --burst-dir "capture/bursts/2026-06-24_141500" \
  --products per_cam:reflectance,combined:index \
  --fps 10 --save-tiffs
```

| 標誌 | 預設值 | 說明 |
| --- | --- | --- |
| `--burst-dir DIR` | （必填） | 批次處理資料夾的路徑 (`…/bursts/<base>/`)。 |
| `--products …` | `combined:index` | `kind:level` 清單，如上所述。 |
| `--fps F` | `10` | 輸出影片的每秒幀數 (fps)。 |
| `--save-tiffs` | 關閉 | 同時將每幀校正後的 TIFF 檔案與影片一併儲存。 |
| `--gif` | 關閉 | 同時寫入動畫 GIF 檔案。 |

**影片不包含任何夾緊遮罩**（單一影像流，8 位元）。取而代之的是，建置結果會包含 `clamp_fractions`：針對每個影片產出， 該影片產出中，每幀在 R / G / B 通道所屬頻帶內經去混色後被夾緊的像素比例 — 即該幀輻照度／反射率乘積應被標記的 `Chloros:Band{R,G,B}UnmixClampFraction` 值（針對合併索引，包含每台參與相機的數據）。 執行過程會在 `Clamp:` 行中輸出每項產品中最差的一幀畫面。`--save-tiffs` 會為 TIFF 檔案新增每像素遮罩， 並將其存放在 TIFF 檔案的 `<level>/` 資料夾旁的 `clamp_masks/<level>/` 中。**若建置過程中途中止，不會產生損壞的影片。** 每個影片皆以 `<name>.partial.avi` 格式寫入，並僅在完成後才取得最終名稱；因此，同名的舊影片會保留至其替代檔完整生成為止。請重新執行建置以替換遺留的 `.partial.avi` 檔案。

執行過程會列印所有已寫入的影片，接著列出任何 `Skipped:` 產出項目（即所請求的相機／層級——或合併索引——但未產生影片的項目，並附上原因——例如：無校準套件、 無反射率 DLS 讀數，或組合索引缺乏紅光／NIR 波段對）以及任何 `Note:` 項目。若 RGB 相機被要求輸出輻照度或反射率，則會改以顯示影片取代，並在相機的校準套件可用時進行白平衡； 若相機在缺乏校準包的情況下進行渲染，則會於 `Note:` 中列出其名稱。當請求的產品被跳過或未按要求建置時（已配置的指標表達式不會在離線狀態下進行評估），該程式會 **以非零值退出**；已建置的影片仍會被寫入並列出。

> **選擇合適的錄影器。** `array-record` 屬於 *監控級* — 它會錄製渲染後的即時複合影像：即 GUI 的合併視圖，或若未開啟 GUI，則為後端自身的 NDVI（或 `--expr`）視圖。`array-burst` → `array-build-video` 屬於 *分析級* ——它以全速率儲存原始感測器資料，並在事後重建經校正的輻照度／反射率／指數影片，無需即時預覽。

### 單色 (M3M) 單波段攝影機

**M3M**系列是拜耳**M3C**的單色版本：每台相機配備一個窄帶干涉濾光片（例如 `M3M-<lens>-F<wavelength>`、`M3M-L87-F685`）， 因此感光元件輸出的是**單一灰階波段**，不帶拜耳馬賽克。無需進行去馬賽克處理、無需分離通道間的串音，也無需設定白平衡——整個 RGB 顯示色彩處理流程根本不適用。

這在 CLI 中的具體表現為：

- **`lattice white-balance`、`lattice color-profile`、`lattice color`**偵測到單色攝影機時，會**跳過並顯示一行訊息**，而非推送無意義的設定。 在同一執行階段中，若遇到 RGB/Bayer 格式的 M3C 相機，這些指令仍會正常運作。
- **Bayer 像素格式即代表其色深。** M3M 僅提供單色格式，因此來自 `--pixel-format` 或 `--preset` 的拜耳格式會設定相同深度的單色格式：`BayerRG8` 和 `--preset high_speed` 會產生 `Mono8`；`BayerRG12`、`high_quality` 及 `triggered` 會產生 `Mono12`； 壓縮名稱仍保持壓縮狀態（`BayerRG12p` 產生 `Mono12p`）。檔案及 `lattice format` 均採用 Mono 格式。混合使用 M3C 與 M3M 相機的陣列採用相同格式 ：每個 M3C 採用拜耳（Bayer）格式，每個 M3M 則採用其單色（Mono）深度。名稱即代表其位元深度：反之亦然， 因此 M3C 絕不會被置於 Mono 格式中（無論陣列的主相機是哪一台，`--pixel-format Mono12` 在 M3C 上即為 `BayerRG12`）， 而 `lattice format --set` 亦以相同方式映射，並輸出其設定的格式。
- **`lattice calibrate` / `process --reflectance` / `array-capture --processing radiance`** 仍可正常運作——輻照度與反射率是 *每波段* 的輻測圖，且對於單一波段而言定義明確。單色影像攜帶 **單位** 感測器響應矩陣（無 3×3 解混），因此該平面能完全不經校正運算直接通過。
- **單一單色相機無法產生植被指數。**NDVI／NDRE／等指數至少需要兩個波段（例如 Red + NIR）。若要從單色硬體中獲得指數，需將**數台** 不同波長的 M3M 相機對準目標，將其對齊整合為一個多波段堆疊，並對 *該* 堆疊進行指數運算：

```bash
# Red (660) + NIR (850) mono pair -> aligned 2-band stack -> NDVI
chloros-cli lattice index --live --profile align.json \
  --preset NDVI --channel red=Mono_660 --channel nir=Mono_850 \
  --save-multiband -o output/
```

`index --live` 會自行開啟相機（一個 [暫時性會話](#multi-camera-transient-sessions)），因此請先勿連接任何設備：`--serials`，否則該設定檔中已啟用的相機 （參見 [索引 / 植被數學](#index--vegetation-maths)）。當 Chloros 佔用任何攝影機（GUI、`cam-connect`、`array-connect`）時 將因 HTTP 409 錯誤 `CAMERA_HELD` 而遭拒絕，並退出 `1`；請透過 `chloros-cli lattice cam-disconnect --all` / `chloros-cli lattice array-disconnect --all` 釋放它們。

`--channel` 符號必須與預設值的通道名稱**完全**匹配（區分大小寫；NDVI 通道為小寫 `red`、`nir` — 參見 `--list-presets`)，且頻段名稱須對應對齊堆疊中的特定頻段（離線模式亦接受以 0 為起點的頻段索引，例如 `--channel red=0 --channel nir=1`）。

整個堆疊中的區分標誌是模型字串中的 `M3M` 標記（該標記絕不會出現在 `M3C` 字串中），並以 `is_mono` 的形式呈現於 GUI／SDK 中。

---

## 主機網路介面卡設定與調校（LATTICE 陣列）

LATTICE 攝影機透過主機的乙太網路介面卡串流傳輸 GVSP，因此對於多攝影機陣列而言，介面卡的 **驅動程式**與**接收環大小** 至關重要，其重要性不亞於鏈路速率。錯誤的設定會導致「陣列設定」面板中出現 `FRAMES WILL DROP` / `Reduce ROI to enable` 警示（位於「陣列設定」面板中，以及 `lattice network-analysis` / SDK 的 `analyze_array_network()`），即使攝影機本身運作正常亦然。

### USB 10GbE 介面卡 — Realtek RTL8157（「Realtek USB 10GbE 系列控制器」）

以下設定為 Windows 驅動程式屬性。在 Linux 系統中，USB 介面卡同樣設有傳輸速率上限（x86_64 架構下為 400 MB/s， NVIDIA Jetson 上為 200 MB/s）的上限（參見 [陣列 fps 與突發模型](#array-fps--burst-model)），且核心的套接字接收緩衝區對每個適配器都至關重要 — 詳見 [Linux 主機](#linux-hosts--socket-receive-buffers)。

| 項目 | 所需值 | 為何重要 |
| --- | --- | --- |
| **驅動程式版本**|**≥ v10.67（2026 年 1 月）**，INF `rtump64x64sta.inf` | 舊版**2016**驅動程式（v10.65， `rtump64x64.inf`）在關機／重新啟動／休眠時，會錯誤處理電源關閉及**`DRIVER_POWER_STATE_FAILURE`（藍屏 `0x9F`）**引發的系統錯誤檢查。 系統轉換過程會卡住（約 5 分鐘超時），使用者被迫強制關機，而反覆發生不當關機將**損毀 WMI 儲存庫**（PowerShell／工具開始因 `Invalid class` 錯誤而失效），並**導致 USB 堆疊卡死** 。請先從 realtek.com（或 USB 轉接器廠商）更新驅動程式，再嘗試乾淨重啟。 |
| **接收緩衝區**— 關鍵字 `ReceiveBufferLen` |**256**（驅動程式最大值） | 網卡的 RX 環。驅動程式的預設值**32**僅留下約 0.26 MB 的可用環空間 —— 對於多鏡頭突發傳輸而言遠遠不足 —— 因此陣列面板會回報 `Sim-emit burst … exceeds NIC RX ring usable capacity 0.26 MB` 並阻斷連線。當**256**時，環容量較大（**實驗室 10GbE 主機測量值約為 13.5 MB**），為接收管線提供了處理多鏡頭 GVSP 突發傳輸的實際餘裕。（特定配置是否實際能 *建立連線*，取決於兩項檢查 — **排水感知**通量檢查與**聚合超額訂閱** 檢查 — 並非單純的突發流量與環狀緩衝區容量之比對；詳見 [陣列 fps 與突發流量模型](#array-fps--burst-model)。) |
| **接收 URB**— 關鍵字 `PendingReceives` |**64**（最大） | 傳輸中的 USB 請求區塊；應與接收緩衝區同步增加以吸收突發流量。 |
| **巨型幀** — 關鍵字 `*JumboPacket` | **9014** | 用於 9000 位元組的 GVSP 封包（每幀封包數比 1500 位元組封包少 6 倍）。 |

> ⚠️ **網卡驅動程式更新會將這些進階屬性重設為預設值。**更新或更換網卡驅動程式後，請**重新套用** `ReceiveBufferLen=256` 和 `PendingReceives=64`，否則即使「硬體未作任何變更」，陣列面板仍會再次進行流量閘控。這是先前運作正常的系統突然無法連線的首要原因。

請透過 **提升權限** 的 PowerShell 執行（請替換為您的網卡名稱，例如 `"Ethernet 5"`）：

```powershell
Set-NetAdapterAdvancedProperty -Name "Ethernet 5" -RegistryKeyword ReceiveBufferLen -RegistryValue 256
Set-NetAdapterAdvancedProperty -Name "Ethernet 5" -RegistryKeyword PendingReceives  -RegistryValue 64
Get-NetAdapterAdvancedProperty  -Name "Ethernet 5" -RegistryKeyword ReceiveBufferLen,PendingReceives   # verify
```

> **`lattice network --fix` 適用於 USB 10GbE 網卡。**系統現已能偵測網卡類型並調整正確的接收環鍵字：PCIe 網卡（Intel I219、 等），或針對 Realtek**USB** 10GbE 控制器（該控制器未公開 `*ReceiveBuffers`）設定為 `ReceiveBufferLen`→256 + `PendingReceives`→64。目標值會被限制在各驅動程式回報的最大值（`NumericParameterMaxValue`）內，因此絕不會寫入超出範圍的數值。 請從 **提升權限** 的終端機執行此操作；如同任何基於登錄檔的調整，變更需在網卡重新啟動或系統重新開機後才會生效。上述的手動 `Set-NetAdapterAdvancedProperty` 指令仍是很好的替代方案——它們可在不重新啟動的情況下即時生效（重新綁定網卡）。

### Linux 主機 — 套接字接收緩衝區

在 Linux 系統（包括 NVIDIA Jetson）上，核心的套接字接收緩衝區與網路介面卡（NIC）的接收環同樣重要。 在內核預設值（`net.core.rmem_max` / `net.core.rmem_default` = 212,992 位元組）下，攝影機發送的 GVSP 封包洪流會在 Chloros 讀取之前就導致流套接字溢出。 這些封包會在核心中被丟棄，儘管網路介面卡本身並未報告任何丟失，而影像陣列會因此遺失幀群組並降低幀率。 經測試，在連接 USB 10 GbE 適配器的 4 台攝影機陣列上，將兩者均調升至 32 MB 後，陣列在連線後的幀率從 4.96 fps 提升至 7.52 fps，連續原始擷取速率則從 4.7 次/秒提升至 7.6 次/秒，且內核中未丟失任何封包。

- **.deb 套件會設定這些參數。** 安裝套件時會寫入 `/etc/sysctl.d/60-chloros-gige.conf`（兩者皆設為 `33554432`），並立即套用。 若該值已設定為更高，則不予變更。
- **`lattice network` 負責檢查這些設定。** 當任一數值較低時，系統會在 *發現的問題* 下列出 `socket receive buffers too small for GigE Vision (…)`，且後端日誌會在第一台攝影機或陣列連線時顯示相同的警告。
- **若要手動提高這些值**（效果將持續至下次重新開機；套件的檔案會在每次開機時套用這些設定）：

  ```bash
  sudo sysctl -w net.core.rmem_max=33554432 net.core.rmem_default=33554432
  ```

  `lattice network --fix` 同樣會提高並維持這些設定值，但僅限於後端以 root 身分運行時。若後端以您的使用者身分運行（即以 `chloros-backend` 或 [`chloros-backend@<user>`X](#running-the-backend-as-a-service-headless)) 則無法變更核心設定，並會改為輸出 `sudo sysctl` 這行。

**自動網卡調校功能僅針對已連線的網卡。** 當第一台攝影機連線時，以 root 身分執行的後端會調校那些接收環大小過小的有線網卡。 現在，它僅針對已建立連線的網卡進行調整，因此未連接的內建埠會保留其 MTU，且不會更動接收緩衝區。`lattice network` 不再將未連接網卡的的 MTU 列為問題。`lattice network`、`--fix`、`lattice probe` 以及網卡調校功能僅考慮有線乙太網路埠， 不包含 Jetson 的 CAN 匯流排（`can0`）或 USB 裝置模式連線 （`usb0`、`usb1`）——這些連結是 Jetson 提供給主機電腦的；但用於攝影機的 USB 以太網適配器仍被計入。

### 網路基礎（所有 LATTICE 連結）

- **位址設定：** 鏈路本位址 `169.254.0.0/16`（GigE Vision LLA）。主機採用靜態位址 `169.254.x.x/16`；攝影機與 DAQ-E 會在同一範圍內自動分配位址。 無需 DHCP 或閘道。
- **封包大小：**優先採用巨型封包（9000），但讓自動探測功能自行判定 — 它會透過 GVSP 探測繞過攝影機 1500 位元組的 ICMP 限制，因此無論線路實際能傳輸多大封包，最終都會採用巨型封包。**每台攝影機在每個後端會話中會被測量一次，當其連線狀態發生變化時（例如更換網卡、連線速度變更，或因切換交換機而導致連線中斷）會再次測量；重新連線時將沿用先前結果。若後端交換機保持連線狀態但內部配置變更，請重新啟動後端以進行重新測量。**首次連線進行此項檢查時，偶爾可能需要幾分鐘，若發生此情況，日誌中會明確記載。僅在您比探測器更了解情況時，才使用 `CHLOROS_GVSP_PACKET_SIZE_FORCE=9000` 進行固定，且應優先採用「按指令」而非「永久」設定：固定會跳過探測器， 因此若路徑實際上無法傳輸 9000，則**每次** 擷取都會因 `SC_ERR_TIMEOUT -1011` 而超時（參見 [環境變數](#environment-variables)）。
- **RX 環區大小隨 `ReceiveBufferLen` 調整：**在預設值 `32` 下，可用環路容量約為 0.26 MB（對於任何多鏡頭突發傳輸而言都過小）；在最大值 `256` 下，容量則較大 （在實驗室 10GbE 主機上測量約為 13.5 MB），提供了實質的餘裕空間。配置能否建立連線，取決於「考量流量消耗的准入檢查」**以及**下文所述的「聚合超訂閱檢查」——而非單純比較原始突發流量與環狀頻寬的大小。

### 陣列 fps 與突發傳輸模型

如何解讀「陣列設定」面板（以及 `lattice analyze-array` / SDK 的 `analyze_array_network`）：

- **突發速率是根據每台攝影機的實際像素格式進行累計的。**單色**M3M**攝影機傳輸**Mono12（2 B/px）**，或透過 `--pixel-format` 傳輸其深度圖的單色格式 （`BayerRG8` 即為 `Mono8`， 1 B/px)；**M3C**拜耳相機則傳輸 8 位元或 12 位元（即使請求的是 BayerRG8，M3C 韌體仍會靜默輸出 BayerRG12）。因此，若四台相機皆為 8 位元，**全解析度畫面約為 12.6 MB；但若三台為 12 位元單色相機，則約為 25 MB**。 投影機制會根據型號（識別快取）解析各攝影機的格式，因此傳輸幀內容與線路實際傳輸的資料相符——而非預設為統一的 BayerRG8 格式。
- **USB 以太網轉接器的傳輸速率上限為 200 MB/s，無論其標示規格為何。** 將鏈路速率轉換為持續傳輸速率的效率表是基於 PCIe 推導而來；USB 網路卡雖宣稱其 *乙太網路* 鏈路速率，但實際上受限於 USB 匯流排及其驅動程式。一款 USB 10GbE 轉接器曾測得約 1063 MB/s 「持續」傳輸速率——這個數值從未被實際驗證過——而由此產生的速率調控導致 6–18 % 的幀資料損壞，同時仍報告正常的目標 fps 數值。目前，透過 USB 連接的網路介面卡已設定絕對上限：在 Windows 及 NVIDIA Jetson 上為 **200 MB/s**， 在 Linux x86_64 系統上則為 **400 MB/s**（在 Linux 系統中，連接至 USB 2.0 埠的網卡預設配額為 **30 MB/s**，且功能說明建議將其插入 USB 3.0 埠） （此限制取決於主機的 USB 匯流排與驅動程式，因此不會隨標稱規格線性提升；USB 1 GbE 網卡可達到約 80 MB/s，且不受此限制影響）。在 Linux 系統中，此轉接器可透過 sysfs 識別（若無法識別則回退至 `ethtool -i` 的 `bus-info`）。各項數據均經實測：在 Windows 系統上，同一組 4 台攝影機陣列在傳輸速率超過 200 MB/s 時，會有 6–10 % 的幀出現損壞；在 Linux x86_64 系統上，同一款轉接卡在最高達 480 MB/s 的所有傳輸速率下均能穩定運作（實際傳輸速率為 273 MB/s，此為攝影機本身的極限）。在 NVIDIA Jetson 上，同一張適配卡僅在 200 MB/s 時運作正常，超過此速度後會間歇性地丟失幀。能力記錄中的 `wire_ceiling_source` 以文字說明了這一點，而 `nic_is_usb` 則以標記標示。無論哪種方式，皆可透過 `--wire-ceiling-mbps` 即可。
- **通量控制會考量匯出通道負載，而非以「整組突發」對比「環形緩衝區」。** 同時發生的突發傳輸只需符合 *瞬態積壓* = `max(0, Σ per-cam arrival − host drain) × emit_window`，而非整個突發傳輸。在高速主機／低速攝像機的架構中（例如 **PCIe**10G 主機 + 4× 1 GbE CAM：傳入速率 ≈ 320 MB/s，排出速率 ≈ 1063 MB/s）的環境下，主機的排空速度快於攝影機的填充速度，積壓量 ≈ 0，因此全解析度的模擬發射模式仍會**允許**傳輸，即使 25 MB 的突發流量超過了 13.5 MB 的環形容量。 若將這四台攝影機連接至 Windows 系統上的 **USB**10GbE 轉接器後端，資料排出速率為 200 MB/s，而非 1063 —— 資料抵達速率超過排出速率，且資料遺失會以損壞幀的形式呈現，而非幀率降低。 在 1 GbE 主機上，攝影機 31.25 MB/s 的 DLThr 下限會導致傳入流量超過傳出流量 → 系統會正確地**阻擋**（針對 *此類* 阻擋情況，請降低 ROI 或使用 ≥ 2 的像素合併）。 通量是 **兩個** 連接閘門之一——另一個是下方的聚合超訂閱檢查。
- **預估 fps 取自最慢相機的週期。**在同步發射模式下，每台相機皆於同一時間傳送其幀，因此陣列的週期取決於其最慢成員的感測器讀出時間與發射時間兩者中較長者（幀位元組 ÷ 其發射速率：即相機的**存取鏈路**，1 GbE ≈ 80 MB/s，或當主機線路上限較低時，其佔用的比例）。每次曝光都會與前一幀的讀出時間重疊，因此曝光過程不計入週期： 自動曝光會填滿由傳輸速率決定的週期，隨後提高增益，且昏暗場景不再降低傳輸速率（在 1.2.0 版本中會降低）。只有手動曝光時間長於該週期時，才會減慢觸發速度（參見 [陣列成員上的曝光](#exposure-on-an-array-member))。增加相機數量並不會減慢陣列速度，除非其線路共享空間縮減。在串列發射模式（`slip-emit-and-capture`）中，各相機的發射時間會累積，且每個成員的曝光時間會被納入交錯時序的計算中。 在 9 台相機陣列上，以全感測器、12 位元模式，搭配 PCIe 10 GbE 主機進行測量：10.7 fps，若採用 [壓縮傳輸](#packed-12-bit-transport---packed) 則為 14.0 fps。
- **連續擷取結束後，預估值會顯示其維持的幀率。** 若以自動幀率進行 30 秒或更長時間的擷取，系統會記錄該陣列尺寸在此電腦上最終穩定的幀率；幀率控制器短暫嘗試的較高幀率則不被計入。 此後，當該幀率低於模型預設值時，`array-connect` 的行以及「陣列設定」面板會顯示該幀率，並註明限制因素：`clock-bound`（陣列以穩定後的幀率運行， 抓取迴路正在等待其觸發時鐘)、`loop-bound`（抓取迴路時間已耗盡）、`capture-bound`（擷取落後並跳過群組）或 `measured` （未明確指出具體原因）。該模型的數值見於 `wire_fps_max`。陣列的幀率仍會攀升至線路所能承載的上限。
- **陣列可在任何網路環境下運作——幀率才是決定性因素。**每台攝影機均能均等分配**防衝突線路上限**（`sustained × sim_emit_factor`），並配合 GevSCPD 進行節拍控制，且所有攝影機仍能同步觸發曝光。 在低速線路上，分配比例較小，幀率也會隨之下降：8 台全解析度 12 位元相機連接 1 GbE（未啟用巨型幀）時，每台可獲得 52 / 8 = 6.5 MB/s，約**1 fps**（壓縮後約 1.3 fps），而 10G 主機運行時則可達到 11-14 fps。若無法將相機的同步速度調至如此緩慢——請透過讀取各相機的 GevSCPD 範圍來確認——陣列將切換至**串列**發射模式 (`slip-emit-and-capture`)：相機將以全鏈路速率輪流使用傳輸線，依序觸發，因此每個幀仍會送達，但各成員的資料**不會同時曝光**。 `analyze-array` 以及 connect 指令的輸出會標示該模式；將 `--force-tier sim-capture-sim-emit` 設為固定值可拒絕此退避機制。幀大小不影響此共享機制，因此**像素合併與較小的 ROI 雖能提高幀率，但不會改變陣列是否符合**。 只有當一條線路連以串列方式都無法傳輸陣列時，才會發生**超額訂閱**—— 例如：相機數量遠超過一條線路所能處理的數量，或該線路上有太多其他相機正在串流 —— 此時出現的症狀是 GVSP 封包遺失，而非傳輸速率降低： `analyze-array` 會將可達幀率數值設為零並輸出 `** OVER-SUBSCRIBED**`，連線會伴隨明顯警告繼續進行，而出現 `pin_resolution: true`（嚴格模式）的 HTTP 連線則會 **會被拒絕**。解決方案包括減少攝影機數量、端到端啟用巨型幀，或使用更快的網路介面卡。`CHLOROS_ARRAY_ALLOW_OVERSUBSCRIBED=1` 會將拒絕狀態降級為「強烈警告」，以便進行基準測試 — 請參閱 [環境變數](#environment-variables)。

### 陣列健康狀態 — 哪個子系統正在丟失幀

已連線陣列的 `GET /api/camera/array/<array_id>/capability` 攜帶一個即時
`health` 區塊，並在 **10 秒** 的滾動視窗內重新評估。 它將幀丟失
細分為兩種需要相反解決方案的原因，而非僅報告一個未指明原因的「不完整」
率：

| 欄位 | 含義 | 涉及子系統 |
| --- | --- | --- |
| `gvsp_corrupt_rate_pct`（按序列號） | 幀 **已抵達但結構錯誤**— GVSP 封包遺失。 |**網路**：線路預算、節奏控制、NIC 接收環、MTU |
| `never_arrived_rate_pct`（每組序列號） | 幀**完全未送達**— 攝影機未觸發，或未傳出任何資料。 |**觸發／同步**：M8 纜線、`--line`、`TriggerMode` |
| `worst_gvsp_corrupt_pct` / `worst_never_arrived_pct` | 每種情況下最差的攝影機傳輸率。 | — |
| `per_cam_rate_pct` | 每台攝影機的綜合不完整傳輸率（兩種原因合計）。 | — |
| `stable_for_seconds` | 每台攝影機維持在 0.01 % 以下的時間長度。 | — |

當比率高於 5% 時，後端會記錄一條 `[array-health <id>] WARN` 行，並標明分界點 — 在
首次觸發時、嚴重性區間變更時、持續期間每分鐘一次，以及
恢復正常時各記錄一次。 損壞的那一半會在後端日誌檔案中寫入
`[gvsp-corrupt <SN>]`，時間點為每台攝影機及每種原因的
首次觸發，之後每 60 秒匯總一次；而應用程式的日誌面板則會顯示
`[array-health <id>]` 這行記錄。每次評估結果仍會寫入後端日誌檔案；
無論輸出內容為何，計數器都會隨著每個緩衝區而更新。

同一筆記錄會報告整個分配所依附的數字：

| 欄位 | 代表意義 |
| --- | --- |
| `wire_ceiling_mbps` | 主機當前生效的持續線路預算，單位為 MB/s。 |
| `wire_ceiling_source` | 該數值的來源（文字說明）——例如 `USB-capped 200 MB/s (was theoretical 1062; PnPDeviceID=USB\VID_0BDA&PID_815A)` （在 Linux x86_64 系統上，為 `USB-capped 400 MB/s (…; sysfs: bus=usb (usb2/2-1/2-1:1.0))`）或 `user override 120 MB/s (auto said 200)`。 |
| `wire_ceiling_is_user_set` | 當由 `--wire-ceiling-mbps`（或 GUI 中的 **線路預算** 欄位）設定時，則為 `true`。 |
| `nic_is_usb` | 針對 USB 以太網路轉接器，此值為 `true` — 請參閱上方的 USB 上限說明。 |

**解讀方式：** 當 `gvsp_corrupt_rate_pct` 值不為零，且 `never_arrived_rate_pct` 為 0 時
表示觸發與纜線同步狀態完美，且 100% 的損失發生在網路
路徑上 — 請降低 `--wire-ceiling-mbps` 並重新連線。反向模式則指向
同步纜線或觸發線的問題。

> **`--target-fps` 並非解決幀損壞問題的關鍵。** GevSCPD 的節拍設定
> 僅在連線時寫入一次，因此降低觸發率會改變占空比，而非
> 同時發射的突發速率。經測量，將需求削減 5 倍並未帶來改善；
> 將線路上限從 240 MB/s 降至 200 MB/s 後，同一套設備的
> 損壞率從 10.4 % 降至 0.00 %。

> **在 LATTICE (M3C / M3M) 韌體下，運行中的陣列無法自行重新規劃。** 它
> 無法自行修復此問題；請斷開並重新連接，以便連線時間選擇器能
> 根據新的上限進行重新規劃。

### 症狀 → 解決方法

| 症狀（陣列設定 / 連線 / `analyze_array_network`） | 原因 | 解決方法 |
| --- | --- | --- |
| `FRAMES WILL DROP … exceeds NIC RX ring usable capacity 0.26 MB`, `Reduce ROI to enable` | `ReceiveBufferLen` 重設為 32（通常發生於驅動程式更新後） | 將 `ReceiveBufferLen` 設為 256，`PendingReceives`→64；重新開啟控制面板（若後端已快取舊的環形大小，則重新啟動後端） |
| 重新啟動／關機卡死；隨後出現 `Invalid class` WMI 錯誤， 網卡無法啟用、USB 裝置遺失 | 舊版 2016 年 Realtek USB 10GbE 驅動程式 → 藍屏死機 `0x9F` → 強制關機 | 將網卡驅動程式更新至 ≥ v10.67 (2026)，然後重新套用上述的接收環設定 |
| 陣列雖以全感測器速度連線，但幀率偏低 | 自動模式會維持全感測器狀態，並讓幀率降至線路所能承載的水平 | 選擇「連線輸出」清單中較快的選項之一（`--binning 2` / `4`，或較小的 `--width`/`--height`），啟用巨型幀，或升級鏈路 |
| Array 回報目標 fps 狀態正常，但實際傳輸量僅為其一部分；`health.gvsp_corrupt_rate_pct` 值不為零， `never_arrived_rate_pct` 0 | 主機推斷的線路頻寬預算高於其實際可維持的值（常見於 USB 以太網適配器、窄頻 PCIe 通道或共用架構） | 請以較低的 `--wire-ceiling-mbps` 值重新連線，並再次檢查健康狀態區塊。 **非** `--target-fps` — GevSCPD 調速機制在連線時即已固定 |
| Linux：某個陣列會遺失幀群組並降低幀率，而網路介面卡卻報告未發生丟包； `lattice network` 列出 `socket receive buffers too small` | 核心的套接字接收緩衝區設定為預設值，導致攝影機封包在核心中遭丟棄 | 將 `net.core.rmem_max` / `rmem_default` 至 32 MB — 參見 [Linux 主機](#linux-hosts--socket-receive-buffers)。.deb 套件會在安裝時自動執行此設定 |
| 已發佈的群組中缺少攝影機；`health.never_arrived_rate_pct` 值不為零， `gvsp_corrupt_rate_pct` 為 0 | 觸發／同步路徑 — 攝影機未啟動，非網路問題 | 檢查 M8 同步線纜及 `--line`；確認每個成員皆已啟用 (`TriggerMode=On`) |
| `**OVER-SUBSCRIBED**` / `Wire budget` 在 `analyze-array` 中超限，或嚴格模式下發生連線拒絕 (`pin_resolution`；`array over-subscribes the wire`) | 無論攝影機以何種頻率運作，該線路皆無法同時或輪流傳輸該陣列——攝影機數量遠超線路可分攤的負荷，或線路上已有其他攝影機正在傳輸 | 減少攝影機數量、端到端使用巨型幀，或使用更快的網路介面卡。 **ROI/像素合併無助於解決問題**（資源共享與幀大小無關）。在測試平台上，`CHLOROS_ARRAY_ALLOW_OVERSUBSCRIBED=1` 會覆蓋此設定（接受封包遺失） |
| 陣列透過 `slip-emit-and-capture`（序列埠）連接至低速網路 | 相機無法以足夠慢的節奏運作以同時共用該線路，因此需輪流運作；幀資料不會在同一瞬間傳輸 | 使用巨型幀或更快的網路介面卡可恢復同時擷取； `--force-tier sim-capture-sim-emit` 堅持此模式（並因此導致封包遺失） |

---

## `chloros-cli daq`

光譜感測器指令。 `pool-*` 指令是輕量級的 HTTP 客戶端，透過後端的持久化連接池驅動感測器，因此需要一個正在運行的後端（參見 [啟動後端](#start-the-backend))。後端擁有傳輸通道，因此 CLI 和 SDK 腳本共享一個活躍的處理程序，而非爭奪串列埠。`pool-discover` 是獲取 DAQ-M 的 BLE MAC 位址的唯一方法。

> **`chloros-cli daq --help`**（或 `-h`、或 `daq help`）會列出 `pool-*` 的子指令（在 Linux 上，還會一併列出建置時包含的直接硬體指令）， 而 `chloros-cli daq <subcommand> --help`（或 `daq help <subcommand>`）則列出各項指令的選項。這些指令涵蓋連線、 串流、記錄經校準的 `.daq` 檔案、交換電容曲線檔，以及將 DAQ 綁定至攝影機；對於 DAQ-E 裝置，還包含其網路設定、PTP、重新啟動及韌體更新。 此外，亦可透過 `chloros_sdk.connect_daq_sensor()` 從 Python 驅動 DAQ，該指令使用相同的彙總路徑。
>
> **請在所有平台上使用 `pool-*` 指令。** 這些指令在 Windows、Linux 和 Jetson 上的運作方式完全相同。在 Windows 上， `daq discover` 和 `daq list` 是 `pool-discover` 的別名，而任何其他非  的 `daq` 指令2567 以外的任何其他 `daq` 指令，都會以 `1` 搭配 `DAQ direct-hardware commands require the full daq package` 退出。在 Linux 和 Jetson 上， `daq discover`、`daq list` 以及其他非 `pool-*` 的 `daq` 指令，是 DAQ 工具的直接非硬體指令：它們會在CLI 的自有程序中開啟感測器， 位於後端資源池之外（因此並非由後端、GUI 或腳本所持有的感測器），且該處接受 `daq discover`、`--only` 及 `--timeout`，但不接受 `--json`。本參考文件僅記錄 `pool-*` 指令。

### DAQ 感測器首次連線工作流程

```bash
# 1. Scan every transport, showing the address to connect with.
#    This is how you find a DAQ-M's BLE MAC — unlike a DAQ-E hostname or a
#    DAQ-U serial port, a MAC isn't printed on the device or listed by the OS.
chloros-cli daq pool-discover
chloros-cli daq pool-discover --only ble           # BLE only
chloros-cli daq pool-discover --json               # machine-readable

# 2. Open a persistent pool session (handle stays alive across CLI calls)
chloros-cli daq pool-connect                                   # smart-detect
chloros-cli daq pool-connect --port /dev/ttyUSB0               # DAQ-U on a serial port (Windows: COM3)
chloros-cli daq pool-connect --mac AA:BB:CC:DD:EE:FF           # DAQ-M by BLE MAC
chloros-cli daq pool-connect --eth-host daq-e-xxx.local        # DAQ-E by hostname

# 3. List what's in the pool, including the sensor_id you'll use next
#    (DAQ-U ids look like 'CB-7C-A8-2E-5F'; DAQ-E ids like 'daq-e-def330')
chloros-cli daq pool-list

# 4. Read the latest spectrum frame
chloros-cli daq pool-latest --sensor-id CB-7C-A8-2E-5F

#    Bind it to a camera, so that camera's captures use and save its readings
chloros-cli daq pool-bind --camera 213800234 CB-7C-A8-2E-5F

# 5. Record a calibrated .daq file for 60s
chloros-cli daq pool-record --sensor-id CB-7C-A8-2E-5F --duration 60 \
  -o ~/Documents/spectra --device-name "field-A"

# 6. Release
chloros-cli daq pool-disconnect --sensor-id CB-7C-A8-2E-5F
```

### `pool-*` 參考說明

| 子指令 | 用途 |
| --- | --- |
| `daq pool-connect` (smart-detect) | 在後端池中開啟一個感測器。 |
| `daq pool-connect --port PORT` (`-p`) | 透過特定序列埠啟用 DAQ-U。 |
| `daq pool-connect --ble` | 透過 BLE 進行 DAQ-M，自動掃描 MAC 位址。 |
| `daq pool-connect --mac MAC` (`-m`) | 透過 BLE 連線至已知 MAC 位址的 DAQ-M（隱含 `--ble`）。 |
| `daq pool-connect --eth-host HOST` | 透過乙太網路進行 DAQ-E，使用已知主機名稱或 IP 位址（隱含 `--eth`）。 |
| `daq pool-connect --eth` | 透過乙太網路進行 DAQ-E，自動偵測主機（mDNS + ARP 備用機制；在 Windows 和 Linux 的冷 ARP 快取環境下仍可運作）。 |
| `daq pool-connect --integration-time MS (-t) --frame-avg N (-f) --no-ae` | 調整積分視窗／AE 狀態。 |
| `daq pool-connect --no-stream` | 建立連線但尚未開始串流 （請以 `pool-stream --start` 繼續）。 |
| `daq pool-connect --cap-id {none, fov_15, fov_30, fov_45, fov_60, fov_90, sunshine_cosine}` | 上限校正設定檔。後端預設值為 `sunshine_cosine`； `none` 為裸感測器。[DAQ-E-S](#daq-e-s--fixed-diffuser-and-tilt-sensor) 會忽略此感測器（固定擴散片）。**感測器出貨時為裸感測器，因此請為您安裝的蓋子命名：** 系統會預設記錄為 (`cap_id_source=auto_default`)，且處理程序會就此發出警告，因為未加裝擴散片的 DAQ-U 在陽光校正下讀數會高出約 30 倍。 |
| `daq pool-discover [--only usb,ble,eth] [--timeout SEC] [--json]` | 掃描每個傳輸通道，尋找可連接的感測器，但不會實際連接。 **這是查找 DAQ-M 的 BLE MAC 位址的方法。** `--timeout` 適用於每個傳輸通道（預設 5 秒，範圍 1-20）。傳輸池中已開啟的感測器不會列出——已連線的 DAQ-M 會停止廣播——因此請使用 `pool-list` 來查詢這些感測器。 `usb` 僅列出 USB 序列裝置：電腦自身的序列埠（`COM1`、Linux 的 `/dev/ttyS*`）無法作為感測器，因此不會被列入。 |
| `daq pool-list` | 顯示後端池中的所有感測器。 |
| `daq pool-bind (--camera SERIAL \| --array ARRAY_ID) {SENSOR \| --clear}` | 將相機或陣列綁定至 DAQ，如同 GUI 中的光感測器選取功能： 其擷取操作會使用、比對並儲存該 DAQ 的讀數。`SENSOR` 為來自 `pool-list` 的感測器 ID，或是用以存取該 DAQ 的位址。`--clear` （無 `SENSOR`）將解除綁定， 且擷取作業將回退至最新的 DAQ。此綁定關係將持續至後端重新啟動，或該攝影機、陣列或 DAQ 斷開連接為止：任何斷開連接（此處指來自 SDK 或應用程式內的斷開）都會解除所斷開硬體的所有綁定，無論由誰設定，陣列成員的綁定亦隨陣列一併解除。 該相機必須已連接、位於開啟中的專案中，或先前已在此電腦上開啟過；且該陣列必須已連接或位於開啟中的專案中；任何其他序列號或識別碼將因名稱不符而被拒絕，且不會建立任何綁定。 |
| `daq pool-disconnect {--sensor-id ID \| --all}` | 釋放 — 無論由誰開啟該感測器，因此也會釋放因腳本當機而殘留在感測器池中的感測器（但會先完成該感測器上的任何錄製作業）。若感測器不在感測器池中，則無需釋放任何內容： 它會輸出 `ID: already released (not in the pool)` 並退出 `0`；若後端拒絕釋放或未確認釋放，則會輸出 `1`。 SDK：`DAQSensorSession.force_disconnect()`。 |
| `daq pool-latest --sensor-id ID [--recent N] [--json]` | 最近的 N 個頻譜幀（預設為 1）。 |
| `daq pool-stream --sensor-id ID [--start \| --stop]` | 恢復／暫停串流。 |
| `daq pool-record --sensor-id ID [--duration SEC] [-o DIR] [--device-name NAME] [--confirm-after S] [--heartbeat S]` / `--stop` | 開始／停止 .daq 錄製。 `-o` 預設值為後端電腦上的 `~/Documents/DAQ Live View`（相對路徑位於執行命令的資料夾之下）；`--duration`（`-d`，預設值為 `0`，直至 `--stop`）。 `--confirm-after S` 會在開始後 S 秒內未收到讀數時發出警告（預設值為 20；設定為 0 則關閉此功能）；而 `--duration` 則 `--heartbeat S` 會每 S 秒輸出讀數計數，並在計數停止增加時發出警告（預設值為 30；設為 0 則關閉此功能）。當 DAQ-E 上的記錄若出現靜默，一旦後端重新連線，便會繼續寫入同一檔案；而 `--stop` 會列出資料缺口（[停止傳輸的 DAQ-E](#a-daq-e-that-stops-delivering))。按下 Ctrl+C 可提前終止 `--duration` 的記錄並回報此狀況； 若未啟用 `--duration`，按下 Ctrl+C 後（退出 `130`）錄製仍會持續運行，直至 `--stop`。 |
| `daq pool-set-cap --sensor-id ID --cap-id CAP` | 於執行時切換電容校正配置檔。必須設定 `--cap-id`；`--cap-id none` 會移除電容校正 （裸感測器）。DAQ-E-S 會將 `as_recorded` 維持為當前設定值。 |
| `daq pool-enable-ptp --sensor-id ID` | 針對 PTP 已關閉的 DAQ-E，重新啟用 PTP。此設定會儲存於裝置中， 因此若裝置處於關閉狀態，將保持關閉——其時間戳記不會與 PTP 同步——直到執行此指令為止。 `pool-list` 會標記此類裝置（`!! PTP is switched OFF`）並為其輸出此指令。Chloros 絕不會自行將裝置重新啟用，因為關閉狀態可能是刻意設定的。 |
| `daq pool-info --sensor-id ID [--json]` | DAQ-E 本身的設定：其韌體（以及此Chloros 版本是否包含更新版本）、其位址及取得方式 （DHCP／鏈路本地／靜態），以及在感測器執行該指令前儲存的網路設定（包含由 `--no-reboot` 儲存的設定）、開機時的資料串流，以及 PTP。若感測器的資料串流未能傳送至這台電腦，系統會在建立新連線時讀取感測器狀態並發出提示。 |
| `daq pool-set-network --sensor-id ID {--auto \| --static IP [--mask M] [--gateway G]} [--no-reboot] [--force] [--timeout S]` | 將 DAQ-E 設定為 DHCP (`--auto`) 或靜態位址 (`--mask` 預設為 `255.255.255.0`)。感測器會重新啟動以套用設定，並以原先的設定重新連線，最多會等待 `--timeout` 秒（預設 45 秒）； `--no-reboot` 僅會儲存該設定。請參閱 [DAQ-E 裝置設定](#daq-e-device-settings)。 |
| `daq pool-set-auto-stream --sensor-id ID {--on \| --off}` | DAQ-E 在開機時若未連接任何軟體，是否開始串流（儲存於感測器中）。無論如何，Chloros 都會自行啟動串流。 |
| `daq pool-reboot --sensor-id ID [--timeout S]` | 重新啟動 DAQ-E 並以其先前設定重新連線 （最多等待 `--timeout` 時間，預設 45 秒）。 |
| `daq pool-update-firmware --sensor-id ID [--timeout S]` | 將此 Chloros 版本隨附的 DAQ-E 韌體刷寫至運行舊版本的感測器，然後重新連接 （約需一分鐘；請勿關機；`--timeout` 預設等待 45 秒使其恢復運作）。已為最新版本的感測器則不予處理。 |

### 停止傳輸資料的 DAQ-E

若泳池中的 DAQ-E 連續 20 秒未傳輸光譜資料，後端系統將重新連接該裝置。此操作透過重啟供電該感測器的交換機來實現：感測器隨之重新啟動，並在收到指令前不會再次傳輸資料 （除非設定為 `pool-set-auto-stream --on`）。

- **首先會驗證是否為同一台感測器。** 系統會以感測器原先的位址（不做任何變更）查詢感測器的名稱、MAC 位址及序列號。 它也會以感測器的 `daq-e-<id>.local` 名稱進行查詢，但僅在回應來自感測器自身網路介面卡（MAC）時才會信任該回應。其他任何回應均不會被重新連線或寫入資料。
- **它會使用該連線的原始設定重新連線：** 曝光時間、幀平均值、自動曝光及快門曲線設定均維持原狀，而非採用感測器開機時的預設值。若感測器先前已啟用 PTP 功能，但重新連線時發現該功能已關閉，則會重新啟用。隨後，它會重新啟動影像串流，並重新啟動電腦端的接收程式。
- **它會以逐漸延長的間隔重新嘗試：** 首先立即嘗試，接著在 5、10、20 和 40 秒後嘗試，之後在感測器離線期間每 60 秒嘗試一次。 每次嘗試僅向感測器查詢其身分。若感測器雖有回應，但在三次重新連線後仍未傳送任何資料，則不再干預並回報此狀況（`auto-reconnect GAVE UP`）。`pool-stream --start` 將再次嘗試。
- **`pool-list` 顯示當前狀態。** 當感測器保持靜默時，其 `STREAM FAULT` 行會說明重新連線的執行狀況。 該行在第一個光譜出現時消失，隨後感測器讀數顯示為 `reconnected automatically N time(s)`。
- **記錄仍以相同的 `.daq` 格式持續進行。** 檔案會以兩行標記每個間隙：`event_type` 間隙前最後一個光譜處標記為 5，間隙後第一個光譜處標記為 6，每行僅包含時間戳記。`pool-record --duration` 會在感測器離線時輸出 `!! STREAM LOST …`，感測器回線時則輸出 `stream resumed after …`。`pool-record --stop` 會列出每個間隙並發出警告，如同針對 （若記錄檔為空——即某筆記錄未包含任何讀數——則會退出並顯示 `1`）。
- **每筆記錄僅保留一套校準參數。** 若在記錄執行期間，感測器恢復連線時帶有不同的校準參數，該筆記錄的剩餘部分仍會維持最初開始時的校準參數，且停止警告訊息會明確指出此情況。 `pool-disconnect` + `pool-connect` 才會採用新的校準值。
- **您的指令優先。** 若您對感測器發出 `pool-disconnect` 指令，或使用 `pool-stream --stop` 暫停，該感測器絕不會重新連線，即使重新連線程序已正在進行中亦然。
- **對靜默感測器執行 `pool-info`** 會在建立新連線時讀取感測器，並新增 `!! stream NOT reaching this computer`。 若感測器未回應，系統會立即回報。對於已停止回應的感測器，`pool-list` 將不再顯示其 PTP 狀態。
- 本文未涵蓋 DAQ-U 和 DAQ-M。DAQ-M 會自行透過藍牙重新連線，而從 USB 拔除的 DAQ-U 則需再次執行 `pool-connect`。

### DAQ-E-S — 固定式漫射器與傾斜感測器

**DAQ-E-S** 是一款 DAQ-E，但有兩項對軟體而言至關重要的差異。其餘所有功能——包括裝置偵測、連線、PTP、記錄以及下方的裝置設定——皆與 DAQ-E 的運作方式完全相同。**其漫射器為固定式，因此無需進行蓋帽校正。** 此款 Sunshine 風格的漫射器為內建設計，且在進行工廠校準測量時已安裝於裝置上，因此該校準值已能準確讀取向下輻射照度。Chloros 會將其蓋帽設定為 `as_recorded`（無配置檔），無論系統要求為何： 若指定以該裝置命名的上限值（`sunshine_cosine`、`fov_*`、`none`），系統會顯示訊息並忽略該設定，且其記錄資料會儲存為 `cap_id_source=model`，因此針對該感測器的「假設上限」警告永遠不會觸發。請勿手動對其資料套用上限：若在頂部放置 `sunshine_cosine`，讀數將高出約 11 倍。

**此感測器配備三軸加速度計。** 陽光漫射器的餘弦響應僅在水平狀態下有效，因此傾斜的感測器會回報錯誤的輻照度——且反射率會繼承此誤差——且無其他症狀。因此，每個 DAQ-E-S 讀數皆包含其姿態資訊：

| 欄位 | 意義 |
| --- | --- |
| `tilt_deg`, `roll_deg`, `pitch_deg` | 以度為單位的角度。當韌體標記角度為未定義時（自由落體、死軸）時，則為 `None`）。 |
| `x_mg`、`y_mg`、`z_mg`、 `mag_mg` | 加速度單位為毫 g。若 `mag_mg` 遠離 ~1000，表示裝置當時正在移動，該樣本的角值無實質意義。 |
| `cal_applied` | 設定：座標軸與角度以擴散器座標系為基準（已套用裝置的安裝校正）。清除：原始電路板座標系，此模式可能顯示高達 ~2.3° 的傾斜角度，但該傾斜並非真實存在。 |
| `sample_age_ms` | 傳送頻譜時加速度計樣本的採樣時間。 |
| `temp_raw` | 未經校準的晶片溫度 **趨勢** —— 僅顯示差異值，絕非溫度計讀數。 |

`daq pool-latest --json` 會將這些值回傳至各幀的 `imu` 鍵下（在 DAQ-U、DAQ-M 或標準 DAQ-E 上則為 `null`）。從 DAQ-E-S 記錄的 `.daq` 會將相同數值儲存於其 `imu_*` 欄位中（`imu_tilt_deg`、`imu_roll_deg`、`imu_pitch_deg`、`imu_mag_mg`、 `imu_cal_applied`, …）中儲存相同的數值，因此讀者可以區分雲層與三腳架被撞倒的情況。在其他所有型號中，這些欄位皆為空。

姿態資料流會單獨報告：當 `daq pool-list` 處於停機狀態時，會輸出 `!! ATTITUDE STREAM FAULT (spectra unaffected)`，並顯示其後恢復了多少次故障。該處的故障絕不會在光譜記錄中造成缺口。 `daq pool-discover` 會將該裝置列為 `DAQ-E-S`（若未在約 2 秒內回應身分驗證，則列為 `DAQ-E`）。

### DAQ-E 裝置設定

```bash
chloros-cli daq pool-info --sensor-id daq-e-def330

# A fixed address on the network this computer is on (192.168.1.x)
chloros-cli daq pool-set-network --sensor-id daq-e-def330 \
  --static 192.168.1.50 --gateway 192.168.1.1

# Back to the factory setting: DHCP, falling back to a 169.254.x.x
# link-local address when no DHCP server answers
chloros-cli daq pool-set-network --sensor-id daq-e-def330 --auto

chloros-cli daq pool-update-firmware --sensor-id daq-e-def330
```

- **若未設定 `--gateway`，則 `--static` 不會儲存任何閘道**（先前設定的任何閘道皆會被捨棄），這對於沒有路由器的攝影機網路而言是正確的。
- **靜態 IP 位址必須位於此電腦的網路中。** 若感測器網路中，Chloros 會拒絕子網中未包含本電腦位址的設定：重新啟動後，將無法連線至感測器以將其改回原設定。僅當本電腦將被重新分配位址，或透過路由轉至新子網時，才應傳遞 `--force`。
- **感測器正在錄製時，系統將拒絕重新啟動。** 請先停止錄製（`pool-record --stop`）。在重新啟動前一刻開始的錄製，將因重新啟動而結束， 且 `pool-set-network`、`pool-reboot` 及 `pool-update-firmware` 將以 `Warning:` 行結尾，該行會標示錄製檔案名稱 （即使感測器隨後未重新連線亦然）。
- **重新啟動後，Chloros 僅會重新連線感測器本身。** 它會在感測器原先所在的位置——以及在 `--auto` 之後，亦在其 `daq-e-<id>.local` 名稱下——等待包含感測器名稱、 主機名稱及型號，並根據其原先擁有的網路介面卡（MAC）位址，在回應的位址上重新連線。 僅接受來自該 MAC 位址的回應，因此若透過路由器連線至感測器的電腦，或處於預設 NAT 網路環境中的 WSL2，當感測器因 DHCP 被移至新位址時，將無法重新連線： 該指令會顯示此訊息，並輸出 `pool-connect --eth-host <its address> …` 指令，以感測器原先的快門速度與曝光設定將其恢復連線。
- **`pool-update-firmware` 會拒絕無法讀取其韌體版本的感測器。** 若感測器在閃光後花費的時間超過等待時間，並不表示其已故障：一旦其回應，系統便會以新韌體的設定重新連線，且該指令會警告其回應速度較慢。
- **韌體更新需要一個傳入的 TCP 連線。** 感測器會重新連線至這台電腦以擷取影像。Chloros 會監聽路由至感測器的網卡（若無法辨識具體網卡，則會監聽所有網卡），且僅服務該感測器本身的位址； 任何其他連線都會在未處理的情況下被關閉。在 Windows 系統中，安裝程式設定的 `Chloros Backend (TCP-In)` 防火牆規則會允許其連線（參見 [Windows (.exe)](#windows-exe))；若防火牆阻擋此連線，更新將因 `… device never connected back on TCP <port> …` 錯誤而失敗。
- **在 WSL2 環境下，請使用鏡像網路**（`networkingMode=mirrored` 及 `%UserProfile%\.wslconfig`）。 在預設的 NAT 模式下，這台電腦在感測器網路上的位址是 WSL 自身的位址，因此所有靜態位址都會被拒絕（除非出現 `--force` 錯誤），即時串流無法傳送到 WSL，且韌體更新無法完成，因為感測器會連回電腦以接收映像檔。
- **感測器 ID 和主機名稱不會改變。** 無論其位址為何，DAQ-E 仍為 `daq-e-<id>` 和 `daq-e-<id>.local`，因此 `--eth-host daq-e-<id>.local` 仍能持續找到它。
- **GUI 的設定保持不變**：在「光感測器」分頁中，點選感測器的設定（齒輪圖示），然後選擇「網路」。GUI 會維持與感測器的獨立連線，因此若要變更 GUI 已開啟的感測器設定，請透過 GUI 進行操作： 若透過 CLI 重新啟動，將終止 GUI 與該感測器的連線。
- 對應的 SDK 型號為 `DAQSensorSession.device_info()`、`set_network()`、`set_auto_stream()`、 `reboot()` 以及 `update_firmware()`。

---

## `chloros-cli project`

開啟、連線並執行已儲存的 Chloros 專案 （內含 `cameras.json`、`sensors.json` 及 `project.json` 的資料夾）。所有操作皆透過後端進行，因此圖形介面（GUI）與命令列介面（CLI）所呈現的硬體狀態完全一致。

### 子指令參考

| 子指令 | 用途 |
| --- | --- |
| `project open PATH [--relink FOLDER]` | 列印專案的裝置清單（攝影機、陣列、感測器），從其檔案中讀取資料，並檢查其原始檔案（參見 [已移動的專案](#a-moved-project---relink)）。 此指令不會在後端開啟該專案：該專案在後端的開啟狀態，以及其中的裝置與錄影檔，均維持原狀。若無法找到該專案的任何原始檔，則退出 `1`。 |
| `project devices PATH [--reconnect]` | 列出或重新執行偵測。 |
| `project connect PATH [--cameras-only] [--sensors-only] [--packed {auto,on,off}]` | 連線所有已儲存的攝影機／陣列／感測器。每個陣列的連線方式與 `lattice array-connect` 相同： 最長 600 秒，其步驟會輸出至 stderr（`[array <id>]    <s> s: <step>`；標準輸出為 JSON 報告）， 若陣列未能在該時間內連線成功，則會撤銷該陣列的連線設定——其攝影機將恢復為初始偵測狀態——並於 `errors` 下進行報告，絕不會保持連線狀態。`--packed` 會為每個陣列設定 12 位元壓縮傳輸模式 （預設值為 `auto`；專案不會儲存各陣列的個別設定；參見 [12 位元壓縮傳輸](#packed-12-bit-transport---packed))。每個光感測器都會在後端的 DAQ 池中，以其儲存的位址、積分時間、幀平均、自動曝光及蓋帽設定開啟： 未選取上限的感測器將採用 `sunshine_cosine`；已儲存為「裸感測器」（`none`）的感測器將維持「裸感測器」狀態；而 DAQ-E-S 感測器則永遠不會設定上限。 GUI 的 DAQ 面板不屬於此池的一部分：請先在該處關閉感測器。 |
| `project capture PATH NAME [-o DIR] [--format FMT] [--exposure US] [--gain DB] [--prefix P] [--orient]` | 來自命名相機或陣列的單次擷取（預設為 `--prefix`，`capture`）。`--orient` （陣列）：陣列儲存原始資料，並保持原生格式——這表示由 `process` 從中產生的產出需進行方向調整，如 `array-capture --orient` 所示。預設值：原生。 `project burst` / `stream` 以及配方中的 `capture` / `burst` / `stream`（`orient: true`）皆採用相同的選項。 |
| `project burst PATH NAME [-n N] [-i S] [-o DIR] [--format FMT] [--exposure US] [--gain DB] [--prefix P] [--orient]` | 來自指定攝影機或陣列的 N 幀連拍（`-n/--count` 預設值為 5；`-i/--interval` 幀間間隔（以秒為單位），預設值為 0）。 陣列連拍會對重複的同步群組進行去重（過期監控機制），因此部分循環的陣列無法回傳同一幀的 N 份副本； 並輸出每次迭代的結果。若擷取過程中途失敗，將停止連拍並退出 `1`，同時顯示失敗前已儲存的幀數。 |
| `project stream PATH NAME [-n N] [--fps F] [-o DIR] [--format FMT] [--exposure US] [--gain DB] [--orient]` | 循環將擷取資料寫入磁碟：`--fps F` 每秒擷取一次 （預設為 3），每次擷取來自一臺相機的一幀畫面，以及來自陣列的一組同步幀（採用與 `project burst` 相同的過期監控機制），直到 `-n N` 擷取 （預設 `0` = 直到按下 Ctrl+C 為止，此時會顯示已儲存的張數並退出 `0`；若未儲存任何畫面或檔案傳輸失敗，則退出 `1`； 若在 `-n N` 擷取完成前按下 Ctrl+C，則會退出 `130`)。若擷取過程中途失敗，將停止串流並退出 `1`，同時顯示在此之前已儲存的幀數。`--poll-interval` 已廢棄且被忽略： 但系統仍會接受該指令，因此舊版腳本仍可執行，並會在標準錯誤輸出（stderr）中顯示相關提示。 |
| `project sensor read PATH NAME [--json]` | 最新光譜幀。 |
| `project sensor log PATH NAME [--seconds SEC] [-o DIR] [--device-name NAME]` | 為 `--seconds` 錄製 .daq 檔案 （預設 10）。若記錄檔中未包含任何讀數，系統將在標準錯誤輸出 (stderr) 中報告此狀況並退出 `1`。 |
| `project run PATH RECIPE.yaml` | 執行 YAML／JSON 擷取配方。 `--dry-run` 進行驗證而不執行。 |
| `project align calibrate PATH NAME [--method M] [--model M] [--frames N] [--reference SN] [--max-features N] [--ratio-threshold F] [--ransac-threshold-px F] [--min-matches N] [--max-reproj-err-px F] [--checkerboard RxC] [--name PROFILE]` | 計算陣列的對齊 — 請參閱 [下方的旗標表](#project-align-calibrate-options)。 |
| `project align status PATH NAME [--json]` | 列印當前對齊設定檔。 |
| `project align clear PATH NAME` | 清除快取中的設定檔。 |
| `project align tweak PATH NAME --serial SN [--dx N] [--dy N] [--rotation-deg N] [--scale N]` | 微調一個從屬節點的變換 （預設為 `0`、`0`、`0`、`1.0`：不作變更）。 |
| `project align export PATH NAME --to FILE` | 將設定檔儲存為 JSON。 |
| `project align import PATH NAME --from FILE [--no-validate]` | 載入已儲存的設定檔。 |

#### `project align calibrate` 選項

| 標誌 | 預設值 | 說明 |
| --- | --- | --- |
| `--method {feature_orb, feature_akaze, phase_correlation, checkerboard, manual}` | `feature_orb` | 對齊方法。**此處的拼寫與 `lattice align-calibrate`** 不同，後者接受簡寫形式 `orb` / `akaze` / `phase`；在此旗標下，這兩項指令不可互換。 |
| `--model {translation, rigid, affine, homography}` | `affine` | 變換模型以貼合。 |
| `--frames N` | `1` | 將同步幀快照與 平均值。 |
| `--reference SN` | 主相機 | 參考相機序列號；其餘所有成員皆會變形對應至該主相機。 |
| `--max-features N` | `5000` | ORB 特徵點數量上限。 |
| `--ratio-threshold F` | `0.75` | Lowe'比率測試。 |
| `--ransac-threshold-px F` | `3.0` | RANSAC 內點閾值。 |
| `--min-matches N` | `15` | **品質門檻** — 若符合的內點數低於此數值，則拒絕該解。 |
| `--max-reproj-err-px F` | `4.0` | **品質門檻** — 若重投影均方根誤差超過此值，則拒絕該求解結果。 |
| `--checkerboard RxC` | — | `--method checkerboard` 的板級幾何結構，例如 `9x6`。 |
| `--name PROFILE` | 空 | 嵌入至儲存之 JSON 中的輪廓名稱。 **非陣列名稱** — 該陣列名稱指的是位置參數 `NAME`。 |

這兩項品質門檻是校準過程雖能成功解算，卻仍
拒絕儲存的原因：若配置檔未能通過其中任何一項，將會在後續
每次擷取時靜默地產生對位錯誤，因此系統會拒絕儲存而非保留該配置檔。

### 已移動的專案 (`--relink`)

專案的 `project.json` 會以完整路徑為每個來源檔案（每張影像及光感測器
記錄）命名。若將專案資料夾，或整個專案資料夾
移至另一顆磁碟機，這些路徑仍會指向舊位置。

開啟專案時，系統會自行修正可修正的部分：原本位於專案
資料夾內的檔案，會被搜尋至專案目前所在的資料夾中，並將該處找到的
每個路徑重新指向新位置並儲存 （`project.json` 會先備份為
`project.json.before-relink-<date>-<time>`）。此過程會在每次開啟時執行——無論是應用程式本身、
`chloros-cli project open`， `chloros-cli process` 匯入現有專案時，SDK 的
以及 `chloros-mcp` 的。路徑僅會重新指向該處存在的檔案。

原本位於專案資料夾外的檔案——例如透過 Hub 會話或「新增資料夾」匯入——
將無法透過此方式找到。請指定它們目前所在的資料夾名稱：

```bash
chloros-cli project open "N:/Chloros Projects/Field_A" --relink "E:/backup/Hub Projects"
```

`--relink` 會以檔案所處的資料夾，或其上層的任何資料夾為基準：系統會根據每個遺失
檔案的資料夾名稱後綴，依序在其下層搜尋
（例如 `Hub Projects/<session>/raw/<serial>`、接著是 `<session>/raw/<serial>`、……），而每個
資料夾的檔案，將從能找到最多檔案的單一位置中擷取。`project open` 會輸出
重新指向的內容、找到的原始檔數量，以及其餘檔案所在的
資料夾。 仍有部分檔案缺失：顯示警告，並退出 `0` —— 專案會連同其餘檔案一併開啟。未
找到任何檔案：顯示錯誤訊息，列出那些資料夾名稱及 `--relink`，並退出 `1`；後端同樣
亦拒絕開啟此類專案，例如 `process` 或 `project devices --reconnect`（HTTP 409 `PROJECT_SOURCES_MISSING`）。

### 範例

```bash
# Open a project and see what it knows about
chloros-cli project open "/home/user/Chloros Projects/Field_A"

# Its hub captures moved to another drive: point it at them
chloros-cli project open "/home/user/Chloros Projects/Field_A" --relink "/media/backup/Hub Projects"

# Connect everything saved in the project
chloros-cli project connect "/home/user/Chloros Projects/Field_A"

# Capture from a named camera (defined in cameras.json)
chloros-cli project capture "/home/user/Chloros Projects/Field_A" FrontLeft \
  -o output/ --format tiff

# Capture from a named array
chloros-cli project capture "/home/user/Chloros Projects/Field_A" main_rig \
  -o output/ --format tiff

# Capture with overrides
chloros-cli project capture "/home/user/Chloros Projects/Field_A" main_rig \
  --exposure 5000

# Read a spectrum
chloros-cli project sensor read "/home/user/Chloros Projects/Field_A" Sky --json

# Record a DAQ log
chloros-cli project sensor log "/home/user/Chloros Projects/Field_A" Sky \
  --seconds 120 -o ~/Documents/spectra/

# Align an array (live)
chloros-cli project align calibrate "/home/user/Chloros Projects/Field_A" main_rig
chloros-cli project align status "/home/user/Chloros Projects/Field_A" main_rig

# Run a recipe
chloros-cli project run "/home/user/Chloros Projects/Field_A" recipe.yaml
```

### 配方 DSL

`project run RECIPE.yaml` 接受 YAML 或 JSON 檔案，用以描述一連串的操作：

```yaml
# recipe.yaml
overrides:
  cameras:
    FrontLeft:
      exposure_us: 5000
      target_brightness: 80

stop_on_error: true
actions:
  - apply:
      name: FrontLeft
      settings:
        auto_exposure: "Off"
        gain_db: 6.0
        auto_gain: "Off"
  - wait: 2s
  - capture:
      name: FrontLeft
      output: pose_a/
      format: tiff
  - stream:
      name: main_rig
      count: 60
      fps: 5
      output: stream/
  - burst:
      name: main_rig
      count: 10
      interval: 0.5
      output: burst_a/
      format: tiff
  - sensor:
      name: Sky
      action: read
```

支援的動作：`apply`、`wait`、`capture`、`stream`、 `burst`、`sensor`。 `apply` 會取用 `cameras.json`、`settings` 及 `exposure_us` 這些金鑰 — `exposure_us`、 `auto_exposure`、`gain_db`、`auto_gain`、`target_brightness`、…… （舊名稱 `exposure_time`、`exposure_auto`、`gain`、`gain_auto` 皆可作為別名使用）。 以此方式設定的曝光、增益或目標值將被視為您的設定：Chloros 的自動曝光控制（包括 DAQ 測光曝光）不會對其進行調整。 手動曝光會維持增益不變：在自動模式下，相機會持續調整增益（在其完整範圍內）以趨近目標亮度，直到您設定 `gain_db` 或 `auto_gain: "Off"` 為止。 後續的 `auto_exposure: "Continuous"` 會將曝光控制交還給自動模式，而 `target_brightness_auto: true` 則會將已鎖定的目標交還給 Chloros 的自動目標設定。 `burst` 動作會接受 `name`（必填）、`count`（預設值 5）、 `interval`（秒，預設值 0）、`output`、`format`、 以及 `settings`（與 `apply` 具有相同的每台相機設定結構）；陣列連拍使用與 `project burst` 相同的「新同步群組」看門狗。

至於 `capture`、 `burst` 或 `stream` 步驟中的 `output` 預設值為 `output`。 相對路徑 `output` （預設即為此）會根據您執行 `chloros-cli project run` 的資料夾進行解析，而非根據配方檔案的資料夾或專案資料夾（在 1.2.0 版本中，它位於後端的安裝資料夾內）。

執行它：

```bash
chloros-cli project run "/path/to/project" recipe.yaml

# Dry-run to validate without firing hardware
chloros-cli project run "/path/to/project" recipe.yaml --dry-run
```

**每次執行前，系統都會先對照專案進行檢查。**每個動作的鍵值必須精確對應上述動詞之一 — 例如 `- capture: {name: FrontLeft}`，而非 `- action: capture` 或 `- type: capture` ——若使用 `name`，該專案的 `cameras.json` / `sensors.json` 會包含一個相機或陣列，而 `apply` / `capture` / `burst` / `stream` 以及一個用於 `sensor` 的光感測器； `stream` 需要 `count` > 0，而 `wait` 則需要一個持續時間。若食譜中存在任何此類錯誤，則**不會**執行任何動作 （`[recipe] error: …`、`[recipe] not run: …`、退出 `1`），即使包含 `stop_on_error: false` 亦然。 步驟未使用的鍵會觸發 `[recipe] warning:`，凡是該配方所驅動但目前未連接的陣列或光感測器，亦會觸發此錯誤。 配方絕不會直接連接裝置：請先執行 `chloros-cli project connect PATH`。`--dry-run` 會輸出配方內容，接著顯示相同的錯誤與警告，並在配方發生錯誤時執行 `[recipe] valid: …` / `NOT valid: …`，若配方出現錯誤，則不執行任何操作並退出 `1`。

---

## 環境變數

| 變數 | 效果 |
| --- | --- |
| `CHLOROS_BACKEND_URL` | 覆寫後端URL（預設為 `http://127.0.0.1:5000`） — **僅由 `lattice`、`project` 及 `daq pool-*` 命令家族支援** （`lattice network-analysis` 則採用 `--backend-url`）。 核心指令（`process`、`login`、`logout`、`status`、 `export-status`、`time-sync`、`selftest`) 會鎖定 `http://127.0.0.1:<port>` 並忽略此變數 （IPv4 字面值可避開 Windows 的 `localhost`→`::1` 每請求約 2 秒的懲罰），因此它們總是鎖定本機。 |
| `CHLOROS_ARRAY_ALLOW_OVERSUBSCRIBED` | 設定於後端環境中。 `1` 會降低超額訂閱拒絕的嚴重程度 — 將與 `pin_resolution` 連線的陣列，或 [暫時性多鏡頭指令](#multi-camera-transient-sessions) (`error_code` `ARRAY_OVERSUBSCRIBED`) — 降級為強烈警告，並接受 GVSP 封包遺失。 僅供測試使用 — 請參閱 [陣列 fps 與突發模型](#array-fps--burst-model)。 |
| `CHLOROS_ARRAY_PACKED12` | 在後端環境中設定： `on` 或 `off` 會為每個陣列設定 [壓縮 12 位元傳輸](#packed-12-bit-transport---packed) 為預設值 （預設為 `auto`）。若明確指定 `--packed`，則仍以該設定為準。 |
| `CHLOROS_DISABLE_UPDATE_CHECK` | `1` （或 `true` / `yes`）可於啟動時停用 Linux CLI 的 [自動更新檢查](#chloros-cli-update)。 |
| `CHLOROS_CLI_MODE` | 由CLI本身設定；指示後端啟用並行處理。 |
| `CHLOROS_GVSP_PROBE_FALLBACK` | `0` 跳過 GVSP 備用探測 （僅限 ICMP 結果）。**此設定會停用巨封包，並非僅是抑制日誌輸出** — 攝影機在每條路徑上僅會回應最大 1500 位元的 DF 探測，因此此探測是唯一能偵測巨封包的方式。 每台攝影機每次連線可節省約 1 秒；若網路 *原本* 能傳輸巨型封包，則代價約為線路上限的 1.45 倍。設定此選項時，SDK 會發出警告。 |
| `CHLOROS_GVSP_SESSION_REUSE` | 在後端環境中設定：`0` 會在每次連線時測量封包大小，而非每台攝影機在每個後端會話中僅測量一次。 |
| `CHLOROS_GVSP_PACKET_SIZE_FORCE` | 將 GVSP 封包大小固定為 N 位元組；完全跳過探測步驟。建議採用「每項指令設定」（`CHLOROS_GVSP_PACKET_SIZE_FORCE=9000 chloros-cli …`）而非永久設定： 固定大小將無法適應前端網路的變化，若在無法傳輸巨型封包的路徑上將大小固定為 9000，則會導致使用 `SC_ERR_TIMEOUT -1011` 時 **所有** 擷取操作皆超時。 |
| `CHLOROS_TRANSIENT_AE_SETTLE_S` | 在後端環境中設定：[暫時性多鏡頭指令]（#multi-camera-transient-sessions），或某台攝影機因無任何內容可擒獲而觸發的 `lattice capture`，在保存畫面之前，用於設定每台攝影機曝光時間的最長秒數（預設值為 `5`）。 `0` 會關閉此功能：此時各攝影機將從第一幀開始，以其曝光極限為起點，自行執行自動曝光。 |
| `CHLOROS_LOG_DIR` | 在後端環境中設定：詳細日誌的儲存資料夾。 預設值：Windows 為 `%LOCALAPPDATA%\Chloros\logs`，Linux 為 `~/.cache/chloros/logs`（若已設定，則為 `$XDG_CACHE_HOME/chloros/logs`）。 .deb 套件的根服務會設定 `/var/log/chloros`，該設定可由 `adm` 群組讀取 — 請參閱 [將後端作為服務執行](#running-the-backend-as-a-service-headless)。 |
| `TMPDIR`（Linux） | 暫存檔的儲存位置。若該目錄存在，CLI 會自動使用 `/mnt/ssd/tmp`。 |

---

## 退出代碼

| 代碼 | 含義 |
| --- | --- |
| `0` | 成功。 |
| `1` | 一般性失敗（多數子指令錯誤）。 當擷取指令被要求寫入的任何檔案未能成功寫入時，即使其他檔案已儲存，該指令仍會以 `1` 退出，而當記錄器回報錯誤時，則會以 `array-burst` / `array-record` 會導致 `1` 退出，當記錄器回報錯誤時 — 請參閱 [寫入失敗的檔案](#files-that-fail-to-write)。 當無法驗證執行程序的匯出內容，或遺失影像時，`process` 也會退出 `1`，儘管其他產品仍會被寫入。 |
| `2` | 參數錯誤（未知或缺失的標誌；亦包含未指定優先級的 `time-sync set-priority`）。 |
| `130` | 遭 Ctrl+C 中斷：任何被其中斷的指令 — `lattice`、`project`、 `daq` 及 `process` 皆然（當後端已停止執行後，則為 `process`，詳見 [`process` 說明](#notes))。若某個命令的正常中止方式為 Ctrl+C — `lattice array-capture --continuous` / `--interval`、`lattice viewer`、`lattice continuous`、`multi-stream`、`align-stream`、 `array-record`、`array-burst` 以及 `project stream` 搭配 `-n 0` — 則會根據實際執行的操作來決定何時退出： 當所有資料寫入完成時為 `0`，當任何操作失敗時為 `1`。 |
| `143` / `129` | 因 SIGTERM / SIGHUP（128 加上該訊號）被強制終止，此發生於指令的清理程序執行完畢之後：正在執行的任務已停止且其攝影機已歸位，後端處理程序亦已停止。 `lattice continuous` 或其他通常透過 Ctrl+C 停止的錄製任務，也會退出 `143`，而非 `0`。 |

當判定結果為 `needs_force_slip` 和 `error` 時，`lattice network-analysis` 會退出 `1`， 而 `selftest` 會退出 `1`，除非所有七項檢查均通過。

---

## 疑難排解提示

- **「需要登入」** → 請在此電腦上執行一次 `chloros-cli login EMAIL PASSWORD`。
- **「此電腦的專案資料夾層級過深」** → 此電腦已停用 Windows 長路徑功能（此為 Windows 預設設定），因此任何路徑長度不得超過 260 個字元，且產品路徑位於專案資料夾下方最多約 130 個字元的深度內。 未處理任何項目。該訊息會顯示符合條件的最長專案資料夾名稱：請改用較短的專案資料夾（例如 `-o` 或 `chloros-cli set-project-folder`），或啟用 Windows 長路徑並重新啟動 Chloros。 Windows 安裝程式會提供「啟用 Windows 長路徑」選項，預設為勾選狀態；否則請使用群組原則：*電腦設定 > 管理範本 > 系統 > 檔案系統 > 啟用 Win32 長路徑*。
- **「無法連線至後端」** → 啟動 Chloros 桌面應用程式（Windows），或手動啟動後端：Windows 上的 `"C:\Program Files\Chloros\backend\chloros-backend.exe"`（位於安裝資料夾內的 `backend` 資料夾中； 該檔案不在 `PATH` 中），Linux 系統上則為 `chloros-backend`。若為遠端連線，請檢查 `CHLOROS_BACKEND_URL`。
- **`lattice` 指令執行失敗並顯示「找不到 LATTICE 攝影機驅動程式」** → 後端無法載入其安裝時附帶的 LATTICE 攝影機執行階段。 所有 `lattice` 指令皆在後端執行；CLI 本身並不包含任何攝影機驅動程式。請重新安裝 Chloros：Windows 安裝程式和 .deb 套件會將執行時環境安裝在後端旁。
- **「Array connect」／「Array Settings」顯示「FRAMES WILL DROP」或「請縮小 ROI 以啟用」** → 主機網路介面卡 (NIC) 的接收環大小過小（通常在更新 NIC 驅動程式後會重設為 32）。 請參閱 [主機網路介面卡設定與調校](#host-nic-setup--tuning-lattice-arrays) — 設定 `ReceiveBufferLen=256`、`PendingReceives=64`。
- **系統在重新啟動／關機時當機，隨後出現 WMI 錯誤 `Invalid class`／網路介面卡無法啟用／USB 裝置遺失** → 過時的 USB 10GbE 適配器驅動程式導致 `DRIVER_POWER_STATE_FAILURE` （藍屏錯誤 `0x9F`）。請更新網卡驅動程式 — 請參閱 [主機網卡設定與調校](#host-nic-setup--tuning-lattice-arrays)。
- **Linux：擷取資料歸 root 所有，因此無法刪除，且後端日誌不在您的 `~/.cache`** → 後端透過 `chloros-backend.service` 以 root 身分運行。 請切換至 `chloros-backend@<user>.service`，該裝置會以您的身分執行後端 — 請參閱 [將後端作為服務執行](#running-the-backend-as-a-service-headless)。 在此之前，root 服務的日誌位於 `/var/log/chloros`（`adm` 群組可讀取）。
- **在 Linux 中：若後端單元為 `failed` 且顯示「已有另一個實例正在執行」（退出狀態 98）** → 表示該埠已由另一個後端佔用：每次僅能執行一個後端。 若啟用了兩個後端單元，請保留其中一個：將未使用的單元設為 `sudo systemctl disable --now <unit>` — 請參閱 [將後端作為服務執行](#running-the-backend-as-a-service-headless)。 若僅啟用此單元，則表示有在 systemd 外部啟動的後端（透過 `chloros-backend`、`chloros-cli` 或 SDK 腳本）佔用了該埠：請先停止該後端，然後執行 `sudo systemctl start <unit>`。
- **Jetson 交換空間警告** → 新增基於檔案的交換空間；CLI 會輸出確切的 `fallocate` / `swapon` 指令。
- **來自 `lattice`、`project` 或 `daq` 指令的「無法連線至後端」** → 沒有後端正在運行：這些指令從不會啟動後端。 請執行 `chloros-cli backend status`，並參閱 [啟動後端](#start-the-backend)。
- **「Chloros 中無法使用 `daq net ...`」**（Linux 和 Jetson）→ DAQ-E 部分內建設定已預留給 MAPIR 服務使用。 網路設定、重新啟動及韌體更新分別由 `pool-set-network`、`pool-reboot` 及 `pool-update-firmware` 處理；其餘事項請聯繫 MAPIR 技術支援。
- **「DAQ 直接硬體指令需要完整的 daq 套件」**（Windows）→ 請使用 `pool-*`（該指令會透過後端驅動相同的感測器），或使用 Python 中的 `chloros_sdk.connect_daq_sensor()`。

---

## 參見

- [Python SDK 參考手冊](sdk-reference.md) — 每個 CLI 指令的程式化對應方式。
- [DAQ 光感測器](https://mapir.gitbook.io/chloros/daq-light-sensors) 詳見Chloros手冊 — 感測器、其上限與校準範圍、記錄功能、DAQ-E 網路連線。
- [LATTICE 攝影機](https://mapir.gitbook.io/chloros/lattice-cameras/lattice)（見於 Chloros 手冊）— 連接方式、攝影機設定、擷取模式、陣列。
- 線上文件：`https://mapir.gitbook.io/chloros/cli`
