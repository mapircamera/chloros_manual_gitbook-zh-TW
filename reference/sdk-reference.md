# Chloros Python SDK 參考手冊

**版本：**

1.3.0**生成日期：**2026-07-29 19:19 ·**修訂日期：** 2026-11-06**套件：** `chloros-sdk` (PyPI)**適用對象：** 針對大型語言模型（LLM）使用進行優化；人類可讀。**範圍：** `import chloros_sdk` 所公開的所有公開類別、函式及輔助函式，並附有可直接複製貼上的範例，涵蓋影像處理、單鏡頭控制、同步陣列、 DAQ 感測器及專案自動化等領域，可直接複製貼上的範例，以及用於 AI 助理的 `chloros-mcp` 伺服器。

若您僅需重點內容，請跳至：
- [安裝與快速入門](#安裝)
- [適用於 LATTICE 陣列的 Smart-Connect](#smart-connect-for-lattice-cameras)
- [DAQ 感測器會話](#daq-sensor-sessions)
- [相機公用程式](#camera-utilities)
- [瞬態陣列、對齊配置檔與索引](#transient-arrays-alignment-profiles--indices)
- [專案資料夾與配方](#project-folder--recipes)
- [專案自動化](#project-automation--chlorosproject)
- [Smart-AE / Smart-Capture](#smart-ae--smart-capture)
- [在您自己的處理流程中使用即時幀](#live-frames-in-your-own-pipeline)
- [MCP 伺服器（AI 助理）](#mcp-server-ai-assistants)

> **

1.3.0 版本的變更**（現有 1.2.0 腳本會偵測到的內容）。
>
> **

1.2.0 腳本可能需要處理的變更：**
>
> - **後端未確認的發行版本現在會觸發錯誤。** `CameraSession.disconnect()`、`ArraySession.disconnect()` / `force_disconnect()` 發生於該會話開啟的陣列上；`DAQSensorSession.disconnect()` / `force_disconnect()` 發生於該會話開啟的感測器上，以及 `ChlorosProject.disconnect_all()` 會觸發 `ChlorosConnectError`，當後端拒絕釋放或未能及時確認時，因為裝置可能仍被佔用。 在 1.2.0 版本中，這些錯誤會靜默返回。`with` 區塊的退出仍不會觸發任何錯誤。
> - **已借用的裝置絕不會被釋放。** `with attach_array(...)` 以及針對該裝置的顯式 `disconnect()` 會讓陣列持續運行（在 1.2.0 版本中，離開該區塊會釋放裝置，即使 GUI 仍持有該裝置）；若要終止其運作，請呼叫 `force_disconnect()`。同樣地，當 `connect_daq_sensor()` 偵測到感測器已存在於池中（包含零旗標智慧偵測）時，會使其在 `disconnect()` 時，以及離開其 `with` 區塊時，仍會保持該感測器開啟狀態。
> - **PNG 和 JPEG 僅接受 `display` 及索引產品。** `capture()` 搭配 `format="png"` / `"jpg"` (`ext=".png"` / `".jpg"`) 以及 `radiance` 或 `reflectance` 等級， 或 `processing="all"`，會在任何請求之前觸發 `ValueError`； 12 位元的 `raw` / `debayered` 將被後端拒絕。請參閱 [PNG / JPEG](#png--jpeg-take-the-display-and-index-products-only)。
> - **`connect_daq_sensor(sensor_id=...)` 會開啟該感測器或觸發** `ChlorosConnectError` 並列出其偵測結果；它絕不會回退至其他感測器。請參閱 [固定傳輸](#pinned-transport)。
> - **`frame_stream()` 觸發錯誤而非保持靜默：** `FrameFetchError` 在遭拒後，下一個時標會重複 （執行反射率讀取且完全不包含 DAQ），在 60 秒內未接收到幀，以及在 60 秒內未出現新觸發信號時。請參閱 [每次讀取一個觸發信號](#one-trigger-per-read--group_id)。
> - **讀取超時或連線中斷會觸發 `ChlorosConnectError`** 錯誤，此錯誤源自每個 smart-connect 呼叫，以及 `analyze_array_network()`，其中 1.2.0 版本會觸發底層的超時或連線錯誤。
> - **`connect_array()` 預設等待 600 秒（`timeout`，1.2.0 版本中為 180 秒），若其放棄的連線將被撤銷。** 該呼叫會將其 `timeout` 傳送至後端，後端會將每台攝影機恢復至最初被偵測到的狀態，並在連線未及時完成時不進行任何連線；該呼叫會觸發 `ChlorosConnectError` （HTTP 504 `CLIENT_DEADLINE_PASSED`）。在 1.2.0 版本中，若在攝影機通電後立即進行連線，可能會持續時間超過該呼叫，並在呼叫拋出錯誤後仍成功連線陣列。`connect_all()` 會以相同方式等待每個已儲存的陣列 （1.2.0 版本中為 120 秒）。請參閱 [`connect_array()` 簽名](#connect_array-signature)。
> - **`connect_array()` 會保留完整的感測器。** 若未啟用 `width` / `height` / `binning`，每台攝影機皆會設定為全感測器模式，幀率將降至主機與網路所能提供的水準；1.2.0 會縮小或合併幀以維持幀率。請參閱 [同步陣列](#synchronized-array--arraysession-smart-prep)。
> - **M3M（單色）攝影機上的拜耳像素格式即為其深度格式。** M3M 僅提供單色格式，而預設名稱中標示為拜耳格式的： 在 1.2.0 版本中，相機拒絕此格式並維持其原有設定，而 `settings.pixel_format` 則命名為拜耳格式。`LatticeCamera(preset=...)`、`apply_preset()`、`set_format()`、`CameraSettings(pixel_format=...)`、`connect_camera(preset=...)` 以及一個陣列'共享格式，現於 M3M 上設定相同深度的 Mono 格式：`"BayerRG8"` 與 `high_speed` 產生 `Mono8`；`"BayerRG12"`、`high_quality` 和 `triggered` 會產生 `Mono12`；`cam.settings.pixel_format` 則為其命名。 反之亦然：M3C 絕不會以 Mono 格式儲存，因此 `"Mono12"` —— 無論源自上述任何一種， `connect_array(pixel_format=...)`、`set_settings(pixel_format=...)` 或陣列的儲存格式——在 M3C 上都會產生 `BayerRG12`，無論該陣列的主相機是哪一台。 您傳入的 `CameraSettings` 會保留其名稱，因此同一物件仍會向每個相機查詢其自身家族中的該深度。`connect_camera(preset=...)` 會保留預設值的格式 （1.2.0 版本在套用預設值後會將相機設定為 12 位元），且透過預設值開啟的相機將不再將其擷取的內容寫入 `PRESETS`。若相機拒絕任何其他格式 （其 `PixelFormat` 清單中未包含的格式，例如 M3M 上的 `RGB8` 或未支援該格式的韌體上的 `BayerRG16`），也不會被命名為獨立項目：當 `LatticeCamera` 連線後，無論是 `apply_settings()` 或 `set_format()`， `cam.settings.pixel_format` 和 `packed_transport` 會標示攝影機串流的格式，而其中一行列印內容則會標示其拒絕的格式。
> - **`analyze_array_network()` 顯示該連線適用之設定：** `pixel_format=None`（原為 `"BayerRG8"`）表示連線所設定的格式 (Mono12 / BayerRG12)，而 `packed="auto"` 則代表封裝傳輸的設定，因此其報出的 fps 可能與 1.2.0'版本有所不同。若傳入的參數為 `pixel_format`，則表示該深度下的 M3M 格式。請參閱 [網路分析輔助工具](#network-analysis-helper)。
> - **`CameraSession.capture()` 會傳回一個 `CaptureResult`** （仍為一組幀字典清單），其中包含 `.skipped`、`.failed`、`.daq_files` 及 `.daq_matches`。`ArraySession.capture()` 的結果中，亦包含 `.alignment`（標示是否已套用對齊變形，以及未套用的原因）與 `.oriented`。
> - **陣列擷取預設會進行對齊，與應用程式的處理方式相同。** `ArraySession.capture()`、`continuous()`、 `capture_repeated()` 以及 `ArrayHandle.capture()` / `capture_smart()` 會將每個成員的 `debayered`、`display`、`display`、`reflectance` 以及索引檔案，並將其裁切至成員的疊加 （比感光元件小，幅度等於相機的偏移量，`XMP-mapir:AlignmentJSON` `"applied": true`），因此這些檔案與Chloros應用程式擷取的同一場景所產生的檔案相同； `raw` 則維持原始格式。 先前，若未使用 `aligned=True` 進行擷取，所有檔案皆會以原始尺寸儲存且未經變形處理。`aligned=False` 維持此設定。`chloros-mcp`'的 `capture_array` 和 `start_capture_run` 採用相同的預設值。請參閱 [`ArraySession` 方法](#arraysession-methods)。
> - **`connect_all()` 指稱一個拒絕 SDK 的後端。** 已登出或使用低於 Copper 方案的運行中後端，將以 `errors` 報告，並附上其專屬原因 （HTTP 401 / 403），而非以「沒有 Chloros 後端正在運行」的形式報告；且每次 `connect_all()` 呼叫都會重新檢查後端，因此於登入後呼叫該函式，在同一個專案物件上仍可正常運作。
> - **相對路徑的 `output_dir` 適用於本機後端。** 相對路徑在傳送前會轉換為以您的腳本工作目錄為基準的絕對路徑。在 1.2.0 版本中，後端會嘗試在其自身的安裝資料夾內解析這些路徑，導致出現「存取被拒絕」的錯誤。
> - **`numpy` 現已成為** SDK 的必備條件（live-frame 呼叫會傳回 numpy 陣列）。
> - **若匯出層級名稱不為「one」，則會被拒絕。** `processing=` 和 `levels=` 會處理 `raw`、`debayered`、`radiance`、 `reflectance` 或 `display` （或其別名，例如 `preview`；`processing=` 以及 `"all"`）。 在觸碰相機之前，任何其他名稱均會被拒絕（HTTP 400 `UNKNOWN_LEVEL`：`capture()` 會觸發 `ChlorosConnectError`、`grab()` / `frame_stream()` `FrameFetchError`）；1.2.0 版本會為此儲存或傳回 `raw`。
> - **LATTICE 擷取檔不支援「Texture Aware」功能。** 在包含 LATTICE 擷取影像的專案中，`configure(debayer="Texture Aware (Slow, Highest Quality)")` 會觸發 `ChlorosConfigurationError`；同樣地，當在匯入前曾透過 `configure()` 要求 。專案先前儲存的「Texture Aware」值會被設定為「Standard」，並觸發 `UserWarning` 錯誤。由於 LATTICE 擷取檔始終採用標準去馬賽克處理，因此 1.2.0 版本接受此設定並予以忽略；Chloros 應用程式對於此類擷取檔亦不提供此設定選項。 請參閱 [支援的值](#supported-values)。
> - **相機將恢復至初始狀態 — 除非發生 `keep_settings=True`。** `ArraySession.disconnect()` / `force_disconnect()`，每個暫態函數的結尾（`capture_cameras_transient()`, `start_multi_camera_stream()`、`test_array_sync()`、`calibrate_alignment_profile()`、`capture_aligned_multiband()`、`start_aligned_stream()`、 `capture_live_index()`）的結束，以及 `start_camera_continuous()` 的停止，會將每台相機恢復至原始狀態：包括其觸發器與同步線接線、PTP 設定，以及本次會話中變更的所有相機設定（曝光與增益、 自動曝光目標、限制與阻尼、白平衡、黑電平、像素格式、像素合併與感測範圍、翻轉設定、幀率）。 `keep_settings=True` 則會將每台攝影機的設定儲存至攝影機自身的記憶體中，以便開機時直接採用這些設定；`clear_settings=True` 則會將該記憶體重置為出廠設定。 無論採用何種方式，攝影機都會保持自由運行狀態。專案在連接攝影機時，會根據自身儲存的設定來設定這些攝影機。請參閱 [攝影機記憶體](#camera-memory--keep_settings--clear_settings)。
> - **所有模式皆以相同方式儲存專案設定。** 針對專案下所連接的攝影機、陣列或光感測器所做的任何變更——無論是透過 SDK、應用程式、CLI 或 MCP 伺服器進行——都會儲存至該專案的 `cameras.json` / `sensors.json` 目錄中；在任何模式下開啟該專案時，系統會根據該專案設定裝置；而 `ChlorosProject` 會將其專屬資料夾名稱傳送至後端。先前，僅有應用程式會儲存專案。請參閱 [單一專案記錄，適用所有模式](#one-project-record-every-mode)。
> - **由多台相機產生的檔案會標示每台相機的資訊。** 多波段疊加影像、由多台相機波段組成的即時索引，以及陣列擷取的 `composite/` 影像，均以清單形式攜帶每台來源相機的序列號、型號及校準包（`XMP-mapir:SourceCameraSerials`、 `SourceCameraModels`、`SourceCalBundleSha256s`、……)，而 EXIF 資料中則以逗號分隔列出序列號清單。過去這些檔案通常不包含相機資訊，或僅包含第一台相機的序列號。 請參閱 [由多台相機產生的檔案](#files-made-from-several-cameras)。
> - **僅有 `raw` 檔案會包含相機的校準標籤。** 相容於 MicaSense 的套組（`BlackLevel`、`RadiometricCalibration`、各-通道的 `Band{R,G,B}RadiometricCalibration`、`MicaSenseVignetting*` 套件）以及 `CalibrationJSON` 有效載荷皆描述感測器的原始數值，因此它們僅載於 `raw` 擷取檔案中傳輸。其餘所有檔案——即擷取`debayered`、`display`、`radiance`、`reflectance` 以及索引檔案、連續錄製與多鏡頭串流幀，以及所有 `process()` 產品 ——皆攜帶攝影機的識別碼（`XMP-mapir:CameraSerial`、`CameraModel`、`FilterType`、`LensModel`、 `CalBundleSha256`, ...)，並在擷取時包含該幀本身的記錄（`XMP-mapir:FrameJSON`：曝光、增益、溫度、像素格式、擷取時間）。在 1.2.0 版本中，這些檔案也包含原始資料集，而應用該資料集的工具（如 Pix4D、Metashape）會將其套用至非原始值的像素上。Processing 會根據檔案的序號及 `CalBundleSha256` 來查找此類檔案的校準資訊。
>
> **其他變更：**
>
> - **陣列的自動幀率將根據陣列的實際輸出進行調整。** 若某幀率因觸發次數減少超過 1% 而降級（此判斷基於多次長時間觀察而非單次），系統會不定期重新嘗試其先前降級前的幀率。您手動設定的幀率則維持不變。
> - **自動曝光更穩定。** 它在光線恆定時保持不變，光線變化後數秒內穩定下來，忽略微小的亮部細節，並能恢復在強陽光下停止回應的相機。請參閱 [自動曝光的運作方式](#how-auto-exposure-behaves)。
> - **`import_images(recursive=True)` 僅統計您的影像與錄影。** 若資料夾內包含專案的 `camera_cal/` 檔案，該資料夾中的校準檔案將不再被計入影像與錄影總數；而已將這些檔案納入的專案則會將其移除。
> - **`configure(radiometric_output=...)` 適用。** 現在它會設定專案的 `Export radiance` / `Export reflectance` 產出設定；先前，每個專案自身的產出設定會優先於此設定， 因此 `radiance`、`sensor-response` 和 `all` 皆會匯出輻照度與反射率。
> - **`test_array_sync()` 根據攝影機的開機狀態進行處理。** 它會在運行期間開啟相機的 PTP 功能，並在首次觸發（`ptp_lock`）前，等待相機時鐘鎖定並穩定下來——每台相機從開機起需 120 秒（約 2 分鐘）才能鎖定 （`ptp_settle_s=` 設定此參數），因為剛鎖定的時鐘在約 2 分鐘內誤差可能高達 ~200 微秒 ——並將每台攝影機的 PTP 恢復至原始狀態。若某台攝影機仍維持先前測試時的鎖定狀態，則不再等待其鎖定。過去在單一交換機網路中，若無 DAQ-E 回應，系統會保持 PTP 關閉狀態，並將每台攝影機的自由運行時鐘視為無法鎖定的 PTP 時鐘 （「PTP 未鎖定」，每次執行皆然）。
> - **`.daq` 錄製檔會標示其封頂值是經選擇還是預設而來。** DAQ-U、 DAQ-M 和 DAQ-E 出廠時未附蓋帽，且 Chloros 無法偵測是否已安裝蓋帽，因此若感測器連接時未指定蓋帽名稱，預設會被標記為 `sunshine_cosine`，這屬於 *假設*。每個 Chloros 記錄（模式 1.30）現在會在 `cap_id` 旁儲存 `cap_id_source`：當您為端子命名時（包含 `none`），則為 `operator`， 若為預設值則為 `auto_default`。當專案的 DAQ 電容設定保留為「自動」，且處理的記錄中電容為預設值時，系統會輸出一個命名為該檔案的 `[daq-dls] WARNING`： 在陽光校正下，未加裝電容的 DAQ-U 讀數約高出 30 倍。**請為您安裝的電容命名**（例如 `connect_daq_sensor(cap_id=...)` 或 `DAQSensorSession.set_cap()`、`chloros-cli daq pool-connect --cap-id` / `daq pool-set-cap`，或 GUI 中的電容下拉選單），或設定專案的 DAQ 電容，警告訊息便會消失。
> - **Windows 系統上不再出現 pip 控制台視窗。** 在已安裝 Python 的電腦上，後端及其每個影像處理程序在啟動時會各執行一次 `pip`，並各自開啟一個控制台視窗。現在已不再如此。
> - **`.daq` 的即時錄製資料可在斷電後存續。**錄製資料會在每次提交時，透過日誌記錄並進行完整的磁碟同步來寫入（此過程發生在感測器後端），因此即使磁碟速度較慢，也只會延遲檔案寫入，絕不會延遲讀數。 若錄製在提交過程中被中斷（斷電、強制終止），可能會在原始錄製檔旁留下 `<name>.daq-journal` 檔案：**請將這兩者合併。**Chloros 在首次讀取檔案時會回滾日誌，因此記錄檔案開啟時會包含停止前已提交的所有資料（約最後 3 秒的資料會遺失）；在 1.2.0 版本中，若在錯誤的時機發生斷電，可能會導致整個檔案無法讀取（「資料庫磁碟映像格式錯誤」）。
> - **位於繁忙磁碟上的 `DAQSensorSession.record_start()` 不再導致感測器停滯**： 讀取資料會排入佇列並在獨立的執行緒中寫入，而無法寫入的資料會計入 `status()` 的 `record_errors` 中（連同 `record_last_error`）。
> - **直接 `LatticeCamera`：**若幀中未包含區塊曝光/增益，現在會回報相機的**實際** 曝光值與增益（備用機制會讀取快取值，而該值可能在整個自動曝光運行期間保持不變），而 `apply_settings(trigger_source='LineN')` / `set_trigger_source('LineN')` 則採用與 Chloros 線纜從機相同的 5 µs 輸入去毛刺處理（未經濾波，即從機在未接收到觸發訊號的情況下仍會觸發幀）。
> - **軟體觸發的相機不再遺漏每兩個觸發訊號中的其中一個。** 處於軟體觸發模式的相機現在會將讀出期間收到的觸發訊號排入佇列（或由 Chloros 等待其準備就緒），這與線纜同步相機的運作方式相同。
> - **`discover_daq_sensors()` 將 DAQ-E-S 命名為 `DAQ-E-S`。** 其 `model` 對每個乙太網路感測器而言皆為 `DAQ-E`；若感測器在約 2 秒內未回應身分驗證，仍會以 `DAQ-E` 列出。
> - **新增：`chloros-mcp`，一款用於 AI 助理的 MCP 伺服器。** Claude Desktop、Claude Code、Cursor 及其他 MCP 客戶端可處理影像（建立專案、匯入擷取畫面、選擇設定、進行處理、匯出執行結果報告），並操控 LATTICE 相機、陣列及 DAQ 光感測器：連線、預覽、設定曝光與色彩、擷取畫面、錄製。 其工具涵蓋了此 SDK 的功能。它會隨 SDK 一併安裝，而 Linux 的 .deb 套件會將其放置於 `/usr/bin` 目錄中。請參閱 [MCP 伺服器（AI 助理）](#mcp-server-ai-assistants)。
> - **新增：相機實用工具**，`chloros-cli lattice` 指令作為 SDK 呼叫：色彩配置檔、飽和度／對比度、高光、白平衡（單次、 自動、固定比例）白平衡、預設值、濾鏡模型、韌體狀態與更新、電源（偵測、休眠、喚醒）、目標校準、網路介面卡（檢查、估算、修復）、檔案邊角暗補正、DLS 讀數，以及主機偵測。詳見 [相機實用工具](#camera-utilities)。
> - **新增：其餘 CLI 功能現已作為 SDK 呼叫**，因此 CLI、SDK 及 `chloros-mcp` 提供相同的功能（基於安全考量，CLI 僅保留登入、DAQ-E 的配置以及直接 DAQ 伺服器功能）：透過單一指令開啟的相機（`start_camera_continuous()`、`start_multi_camera_stream()`、`capture_cameras_transient()`、`test_array_sync()`、`detect_array_master()`、`analyze_array_models()`），對齊配置檔（`calibrate_alignment_profile()`、`capture_aligned_multiband()`、`start_aligned_stream()`、 `alignment_profile_info()`、`reorder_alignment_profile()`）、索引檔（`index_catalog()`、 `compute_index()`、`capture_live_index()`）、PTP 主節點（`time_sync_status()` 及其相關檔案）、專案資料夾（`project_folder()`、 `set_project_folder()`、`reset_project_folder()`）以及配方（`run_project_recipe()`，CLI 專屬的執行程式）。針對光感測器：`connect_daq_sensor(cap_id=...)`、`DAQSensorSession.set_cap()`、`daq_bindings()` / `set_daq_bindings()`、`daq_band_presets()` 及 `light_sensor_reflectance()`（兩個感測器的環境光 + 物體反射率及指數）；針對相機： `CameraSession.status()` 和 `capture_repeated()`；針對陣列：`ArraySession.capability()` 和 `set_member_settings()`（單一元件的曝光值、增益等）。請參閱 [瞬態陣列、對齊輪廓與指標](#transient-arrays-alignment-profiles--indices)、[專案資料夾與配方](#project-folder--recipes) 以及 [DAQ 感測器工作階段](#daq-sensor-sessions)。
> - **改進的 LATTICE 指數產品**（NDVI 及其他指數）源自 `process()`，適用於所有處理模式。
> - **LATTICE OCN 相機：在陽光下，輻照度、反射率及指數產品已獲得改善。** 頻段分離後會套用機隊校正 — 適用於即時處理、擷取作業及 `process()`；RGN 相機則維持不變。 請參閱 [輻量輸出](#radiometric-output-lattice-multispectral-pipeline)。
> - **每個擷取檔都會儲存其使用的 DAQ 讀數，此功能與 GUI 相同。** `CameraSession.capture()` 及 `ArraySession.capture()` / `continuous()` 會寫入 `.daq` 及其 `.match.json`；在 1.2 版本中，0 版本中，原始或去拜耳化擷取僅會透過 `force_daq=True` 儲存其中一項。`daq=False` 會關閉此功能，而 `daq_tolerance_ms=` 則設定匹配容差；兩項結果皆會標示 `.daq_files` 和 `.daq_matches` 中儲存了什麼。現在，擷取會等待 DAQ 的下一次讀取，以便能將曝光前後兩側的讀數進行混合。請參閱 [擷取所儲存的 DAQ 讀取值](#the-daq-reading-a-capture-saves--daq_files-and-daq_matches)。
> - **`ArraySession` 包含即時幀及伺服器端的連續擷取。** `grab()` 和 `frame_stream()` 會從單一觸發器返回 numpy 幀 （與專案 API 的 `ArrayHandle.grab()` / `frame_stream()` 讀取方式相同），而 `continuous()` 會將每個同步群組儲存至後端，並傳回一個 [`ContinuousHandle`](#continuoushandle)。請參閱 [在您自己的處理流程中使用即時幀](#live-frames-in-your-own-pipeline)。
> - **一個陣列 `grab()` 被固定至一個觸發器**（`ArrayHandle.grab()`，以及新的 `ArraySession.grab()`）。在 1.2.0 版本中，20 個四- 讀取中，有 12 次橫跨兩個觸發器。現在每個幀都攜帶 `group_id`，而陣列 `frame_stream()` 會讓每個觸發器各產生一次。 請參閱 [每次讀取一個觸發器](#one-trigger-per-read--group_id)。
> - **各波段與對齊幀：** `grab()` / `frame_stream()` 會讀取 `band=`， `align=`、`bit_depth=` 以及 `refl_max=` 作為相機和陣列的句柄。請參閱 [匯出單一波段](#exporting-a-single-band--band)。
> - **新的 `ArraySession` 方法與參數：** `set_target_fps()`、`force_disconnect()`、`capture(after_seq=..., orient=...)` （搭配 `CaptureResult.group_seq` 和 `.failed`）、`continuous()`、`connect_array(packed=...)`；`capture_repeated()` 會串聯連續的群組，並在磁碟空間已滿時停止。
> - **`ArraySession.record()` 在未開啟 GUI 的情況下進行記錄。** 當 GUI 的合併檢視關閉時，後端會自行渲染該檢視：陣列的NDVI，或是您作為 `expr=` 傳入的索引。當錄製失敗（即未寫入任何幀）時，`RecorderHandle.stop()` 會觸發 `ChlorosConnectError`，並將停止結果保留在 `.result` 上保留停止結果（若留下 `with` 區塊則不會觸發：請檢查 `rec.result`）。請參閱 [`RecorderHandle`](#recorderhandle)。
> - **新的 `DAQSensorSession` 方法：** `enable_ptp()`、`force_disconnect()`，以及用於 DAQ-E 自身設定的 `device_info()`、`set_network()`、`set_auto_stream()`、`reboot()` 以及 `update_firmware()`。 `record_stop()` 會回傳 `warning` 和 `gaps`。若重新啟動時關閉了某個錄製，則會在回覆中為其命名；若重新啟動失敗，則 則會於 `ChlorosConnectError`（`recording_closed`；請參閱 [DAQ-E 裝置設定](#daq-e-device-settings)）中標示該錄製名稱。 請參閱 [`DAQSensorSession` 方法](#daqsensorsession-methods)。
> - **若一個已停止傳輸的彙總型 DAQ-E 由後端重新連線**，將保留其原有設定，且該裝置上的記錄將在同一個 `.daq` 中繼續進行，並在檔案中標記出間隙； `record_stop()` 會將中斷段落以 `gaps` 的形式回傳。在 1.2.0 版本中，感測器會保持靜默狀態，直到被斷開並重新連線，此時記錄才會結束。請參閱 CLI 參考手冊中的 [停止傳輸資料的 DAQ-E](cli-reference.md#a-daq-e-that-stops-delivering)。
> - **`open_project()` 需透過已安裝的 SDK 運作。** 安裝程式（及 PyPI）現已內含所需的專案載入程式；先前，`open_project()` 會觸發 `ImportError` 錯誤，並指明 `project_cameras` / `project_sensors` 的錯誤。`connect_all()` 會透過正在運行的後端 （光感測器則透過其 DAQ 池）連接已儲存的攝影機、陣列與光感測器；而對於無法搜尋到的已儲存裝置，系統會將其列於 `errors` 下並說明原因，而非不作任何說明便直接跳過。若後端（GUI、 或 `chloros-cli project connect`）中已啟動的陣列，現可連接並保持運行；而先前此情況會被報告為錯誤（`Camera <sn> is already in array <id>`）。請參閱 [`ChlorosProject` 方法](#chlorosproject-methods)。
> - **`ChlorosLocal()`、`process_folder()` 和 `process_lattice_capture()` 可在 Windows 上找到 Chloros。** 它們會搜尋安裝程式放置後端的位置（預設為 `<install folder>\backend\`、`C:\Program Files\Chloros\backend\`， 安裝程式自身記錄中的自訂資料夾），並直接連線至已執行的後端，完全不會進行搜尋。在 1.2.0 版本中，除非您傳入 `backend_exe=`，否則它們會在已安裝的機器上拋出「找不到 Chloros 後端」的錯誤。
> - **若後端在啟動時發生異常終止，系統會立即回報。** 當 `ChlorosLocal()` 或智慧連線呼叫啟動的後端在回應前即退出時， 該呼叫將在數秒內觸發錯誤：`ChlorosBackendError` 來自 `ChlorosLocal`，而 `ChlorosConnectError` 則來自 smart-connect 呼叫中，並附上退出代碼、其執行的指令以及後端輸出最後幾行內容。1.2.0 版本會等待 60 秒的啟動超時，但不會說明原因。請參閱 [後端自動啟動](#backend-auto-start)。
> - **後端在腳本之間持續運行。** 離開 `with ChlorosLocal(...)` 區塊時， `process_folder()` / `process_lattice_capture()` 返回，以及 `chloros-mcp` 退出，已不再會停止由 SDK 啟動的後端； 下一個腳本會重新使用該後端。它會在 `logout()` 時停止（除非 Chloros 應用程式處於開啟狀態或正在執行任務），或在 `shutdown_backend()` 時停止，或者在後端環境中無人使用 30 分鐘後自動停止 （後端環境中的 `CHLOROS_BACKEND_IDLE_MIN` 會改變此行為；`0`：絕不）。
> - **SDK 啟動的後端是被要求關閉，而非被強制終止。** `shutdown_backend()` 會讓後端釋放其相機和光感測器，並完成任何錄製作業；只有在 10 秒後仍持續運行時才會終止該後端。1.2.0 版本會立即終止它 （在 Windows 上，這屬於強制終止），這可能會導致連接至該後端處理程式的 DAQ 光感測器無法回應。SDK 偵測到正在運行的後端處理程式則仍會保持原狀。
> - **在 Linux 上，自動啟動功能絕不會從當前資料夾執行 `backend_server.py`。** 先前若未安裝後端程式，但在包含後端程式的資料夾中執行腳本時，會自動執行該後端；現在僅會啟動已安裝的後端程式（或對於 `ChlorosLocal`，則執行 `backend_exe=` 所指定的後端）。
> - **`configure(indices=...)`**會將每個預設值新增至該相機波段的專案中，並包含一個新專案；若遇到無法新增的名稱，則會發出警告（`UserWarning`）。**`import_images()`** 會將 `.daq` 的錄影報告為 `scan_count`，且不再針對 LATTICE 擷取資料夾發出警告。
> - **`read_image_audit_tags()` 可透過 pip 安裝運行。** 它會自行讀取影像的元資料；在 1.2.0 版本中，當 Python 路徑上沒有桌面安裝版的內部元件時，它會返回 `{}`。
> - **緊接在 `connect_array()` 之後執行的 `capture()`** 會等待最多 20 秒以取得陣列的第一個幀組，而非因 HTTP 503 錯誤「陣列仍在預熱中」而失敗。
> - **`status()` 回報系統停滯。** 新的 `stalled` 欄位在擷取迴圈執行期間為 True，但不會發布任何內容，而 `actual_fps` 隨後會變成 0。在 1.2.0 版本中，`actual_fps` 在 停滯期間會保留其最後一個值，因此停滯的陣列會顯示 `actual_fps > 0`，而擷取則會因 503 錯誤「無新的同步幀組」而失敗。
> - **不完整的幀組數量減少，且擷取 503 錯誤減少：** 每台從屬攝影機的觸發輸入現已進行去抖動處理。
> - **壓縮 12 位元傳輸**（`connect_array(packed="auto")`）將全感測器陣列的幀率提升了約三分之一。詳見 [壓縮 12 位元傳輸](#packed-12-bit-transport-packed)。
> - **區域內 (ROI) 的陣列曝光：** 狹小 ROI 不再將成員的曝光時間限制在感測器的自由運行週期內（768 × 304 解析度下為 4.9 毫秒）。請參閱 [陣列成員的曝光](#exposure-on-array-members)。
> - **自動曝光不再降低陣列的自動幀率。** 透過纜線同步的陣列會以網路和攝影機讀出速度所允許的速率運行；自動曝光會先填滿該速率所提供的週期，然後提高增益。若手動曝光時間長於該週期，系統會放慢觸發速度以配合曝光時間，而非拒絕該曝光。請參閱 [陣列成員的曝光](#exposure-on-array-members)。
> - **`read_nodes()`**在自動曝光運行期間新增 `frame_values` 和 `notes`（請參閱 [自動曝光下的 `read_nodes()`](#read_nodes-under-auto-exposure--frame_values-and-notes))。當相機拒絕某次曝光或增益時，**`set_settings()`** 會標示其可接受的範圍，並將先前針對該值關閉的自動模式恢復； `set_settings(target_brightness_auto=True)` 為新增項目。
> - **`orient=`** 關於 `ArraySession.capture()` / `capture_fastest()` / `continuous()` / `burst()` 及 `ArrayHandle.capture()` / `capture_smart()` 會根據相機的朝向旋轉產品。請參閱 [`ArraySession` 方法](#arraysession-methods)。
> - **`DLS()` 需要感測器本身的 `mac_address`。** 已不再有預設的 MAC 位址（它原本是某個單元的位址）；若未指定 MAC 位址，`DLS()` 會觸發 `DLSError`。 請參閱 [DAQ-M 頻段幅度](#daq-m-band-magnitudes--dlsband_magnitudes)。
> - **Windows 安裝程式** 提供「啟用 Windows 長路徑」和「將 Chloros 後端排除於 Microsoft Defender 掃描之外」兩項選項，兩者預設皆已勾選。請參閱 [步驟 1](#step-1--install-the-chloros-platform-package)。
> - **Linux .deb 套件** 會為系統安裝 Python SDK（`python3` （先前僅將 wheel 套件放置於磁碟上），並會增加相機串流所需的內核套接字接收緩衝區，同時新增一個以您的使用者身分執行的後端服務。Jetson 套件為 `chloros_1.3.0_arm64_jp6.deb`，適用於 JetPack 6， 並搭配 JetPack 的 CUDA PyTorch。請參閱 [步驟 1](#step-1--install-the-chloros-platform-package) 及 [步驟 2](#linux-deb)。

---

## 60 秒速懂架構

此 SDK 是在 Chloros 後端（與桌面 GUI 和 CLI 所使用的相同 Flask 伺服器）之上的一層輕量級 Python 介面。若要進行自動化，請匯入 `chloros_sdk` 並呼叫高階方法；在底層機制中， 每個呼叫都會轉化為對本地後端 5000 埠的 HTTP 請求 — `http://127.0.0.1:5000/api/...` （刻意不使用 `localhost`，因為在 Windows 上該名稱會先解析為 `::1`，且針對僅支援 IPv4 的後端，每次請求耗時約 2 秒）。 後端管理著硬體資源池——包括相機、數據擷取（DAQ）感測器、對齊設定檔及幀緩衝區——因此 SDK 腳本可與 GUI 和諧共存，無需爭奪串列埠或網路介面卡（NIC）頻寬。

您將使用以下三個介面：

1. **`ChlorosLocal` + 自由函式** (`process_folder`、`process_lattice_capture`) — 影像處理管線。透過單一Python呼叫，即可對整個資料夾執行校正／去拜爾演算法／索引匯出。
2. **智慧連接處理程式** (`connect_camera`、`connect_array`、`connect_daq_sensor`) — 為即時硬體開啟持久性後端連線。與 GUI 相同的「智慧預備」流程：網路探測、層級自動選擇、PTP、AE 初始化、GPIO 觸發設定。
3. **`ChlorosProject` / `open_project`** — 載入已儲存的專案（包含 `cameras.json` + `sensors.json` + `project.json` 的資料夾），一次連接其中的攝影機、陣列與光感測器，並透過命名處理常式驅動擷取作業。

若尚未有後端處於監聽狀態，介面 1 和 2 將 **自動啟動本地後端**（即 GUI／CLI 所啟動的相同內建二進位檔）—— 因此，即使在全新終端機中，無需事先啟動後端，單純執行腳本即可運作：`ChlorosLocal` （以及 `process_folder()` / `process_lattice_capture()`）、`connect_camera()`、 `connect_array()`、`connect_daq_sensor()`、 `discover_lattice_cameras()` 以及 `discover_daq_sensors()`。`list_cameras()`、`list_arrays()`、`list_daq_sensors()`、 `attach_array()` 和 `analyze_array_network()` 絕不會自行啟動：它們需要一個正在運行的後端。傳入 `auto_start_backend=False` 可選擇不啟動 （例如：當指向一個永遠不會被啟動的遠端後端時）。請參閱 [後端自動啟動](#backend-auto-start)。Surface 3 的行為有所不同：`open_project()` 不接受 `auto_start_backend` 參數，且 `connect_all()` 絕不會啟動後端 ——它會對 `http://127.0.0.1:5000` 進行一次偵測，若無回應，則回退至直接（無後端）的 `lattice_sdk` 裝置控制。此模式需要 Arena SDK 執行時環境以及 OpenCV （`cv2`）可於您的 Python 中導入，而 SDK wheel 並未安裝這些元件； 若缺少這些元件，每個已儲存的相機和陣列都會被報告為錯誤，並附上原因。專案中的光感測器沒有無後端的路徑：若沒有後端在運行，每個已儲存的感測器都會被報告為錯誤，並附上原因（參見 [`SensorHandle`](#sensorhandle))。僅有 `proj.process()` 與 `stream(..., overlays=True)` 會延遲建構一個 `ChlorosLocal()`（該物件會自動啟動）。

這三者皆需：在該機器上執行一次 `chloros-cli login`，或透過桌面 GUI 登入。 若在沒有有效工作階段的情況下進行呼叫，所觸發的錯誤取決於介面（參見 [例外情況](#exceptions)）：來自 `ChlorosLocal`（以及 `process_folder()` / `process_lattice_capture()`)、來自智慧型-connect 呼叫及其會話所引發的 `ChlorosLocal`（以及 `process_folder()` /`process_lattice_capture()`），來自 smart-connect 的 `ChlorosConnectError` 及其會話，由 `analyze_array_network()` 引發的 `status: "error"` 與 `error_code`，以及 `connect_all()` 報告中的一筆記錄。

需求：
- Python 3.7 以上（依套件聲明；於 3.10 版本上開發/測試）
- 本地已安裝 Chloros Desktop（後端二進位檔隨安裝程式一併提供）
- 持有 **任何付費 Chloros+ 方案（Copper 或更高）**的有效 Chloros+ 帳戶；免費的**Iron**方案不提供 SDK/CLI 存取權限。此限制由**伺服器端**強制執行：**每個標記為 SDK/CLI 的請求，必須同時具備有效會話及付費方案。若方案等級低於「Copper」，則會收到 `403` 錯誤碼，並伴隨 `error_code: PLAN_UPGRADE_REQUIRED`（來自 `ChlorosLicenseError` 的 `ChlorosLocal`X000535，`ChlorosConnectError` 來自 smart-connect 呼叫）。已登出狀態的呼叫者則會獲得 `401` / `AUTH_REQUIRED`，而非 （`ChlorosAuthenticationError` 來自 `ChlorosLocal`，`ChlorosConnectError` 來自 smart-connect 通話）——這兩者有所不同，因為重新執行 `chloros-cli login` 可修復 `401`，但無法修復 `403`，後者需要升級方案。
- 在方案的寬限期內支援離線使用：服務層級會從伺服器驗證快取（5 分鐘）或已簽署且與機器綁定的授權快取（月費方案為 30 天，年費方案則至訂閱到期日）中讀取。當寬限期結束後，方案將轉為免費方案，且 SDK/CLI 存取將中斷，直至該裝置能成功連線至伺服器一次為止。`chloros-cli status`（`GET /api/license-status`）在免費方案下仍可存取，因此可查看錯誤原因。
- Windows 10/11 64 位元、**Ubuntu 22.04 LTS 或更新版本**，或 Jetson（JetPack 6）。Ubuntu 20.04**不**受支援： `.deb` 的依賴項源自後端所連結的組件（包括 `libc6 (>= 2.35)`），而 Focal 系統預設搭載 glibc 2.31。

---

## 安裝

Python SDK 是在 Chloros 後端之上的一層輕量級 Python 介面。除非您在另一台機器上驅動後端（請參閱 [遠端後端模式](#remote-backend-mode-pip-only-host-via-tunnel)），否則您需要 **在本地端安裝 Chloros 桌面套件** （Windows 安裝程式或 Linux `.deb`），其中包含 DAQ 光感測器。此套件提供後端二進位檔、適用於 LATTICE 相機的 Arena SDK 執行環境，以及校準套件。

最新下載：[`https://mapir.gitbook.io/chloros/download`](https://mapir.gitbook.io/chloros/download)

### 步驟 1 — 安裝 Chloros 平台套件

#### Windows (.exe)

1. 從下載頁面下載 `Chloros Installer 1.3.0.exe`。
2. 執行安裝程式並依照精靈指示操作。預設安裝路徑為 `C:\Program Files\Chloros\`。
3. 至少啟動一次 Chloros，並使用您的 Chloros+ 帳戶登入。

安裝程式會讓後端程式（`chloros-backend.exe`， 僅此而已）在每個網路設定檔中通過 Windows 防火牆：UDP 傳入（用於攝影機串流、PTP、mDNS 及 DAQ-E 串流），以及僅限本地子網的 TCP 傳入（用於 DAQ-E 在韌體更新期間回連至電腦的連線） ([`update_firmware()`](#daq-e-device-settings))。解除安裝程式會移除這兩項規則。無論如何，後端的 HTTP API 皆維持在 `127.0.0.1` 上。

精靈的 **系統設定** 群組中有兩個選項，預設皆已勾選：

| 選項 | 功能說明 | 選擇不進行靜默安裝 |
| --- | --- | --- |
| **啟用 Windows 長路徑** | 允許專案資料夾位於超過 Windows 260 個字元路徑限制的層級。若關閉長路徑功能（Windows 預設設定），當專案資料夾層級過深而超出其產品支援範圍時，系統會在處理任何內容之前即拒絕該專案資料夾 — 請參閱 [疑難排解](#疑難排解要點)）。 | `/MERGETASKS="!longpaths"` |
| **將 Chloros 後端排除於 Microsoft Defender 掃描範圍外** | 停止 Defender 對即時掃描。在高擷取率下，掃描會佔用約 30% 的擷取率：於 9 台相機陣列且 ROI 為 512 的情況下測量，持續 RAW 擷取時，開啟掃描的速率為 52.7 幀/秒，而排除後端時則為 77.7 幀/秒。 | `/MERGETASKS="!defenderexclusion"` |

連續擷取會將 Defender 的負載報告為 `defender_cores`（參見 [`ContinuousHandle`](#continuoushandle)）。

#### Linux amd64 (.deb)

```bash
sudo apt install ./chloros_1.3.0_amd64.deb
chloros-cli --version
chloros-cli login user@example.com 'YourPassword'
```

請使用 `apt install ./…` 進行安裝，而非 `dpkg -i`：apt 還會一併安裝 Python SDK 的依賴項，而該套件僅建議而非強制要求這些依賴項（參見 [步驟 2](#linux-deb)）。

#### Linux arm64 — Jetson (JetPack 6)

```bash
sudo apt install ./chloros_1.3.0_arm64_jp6.deb
chloros-cli --version
chloros-cli login user@example.com 'YourPassword'
```

適用於運行 JetPack 6（L4T R36）的 Jetson Orin / Orin NX / Orin Nano。 其 PyTorch 為 JetPack 6 的 CUDA 版本，因此 GPU 處理會在 Orin 的 GPU 上執行；它使用 JetPack 6 安裝的 CUDA 和 cuDNN 函式庫（若映像檔缺少這些函式庫，請參閱 `sudo apt install nvidia-jetpack`）。

#### Linux：接收緩衝區與後端服務

這兩套 .deb 套件都會將 LATTICE 攝影機串流所需的內核套接字接收緩衝區大小提升至 32 MB（`/etc/sysctl.d/60-chloros-gige.conf`）。若維持內核預設值， 攝影機封包會在核心中被丟棄，且陣列會因此遺失幀數並導致幀率下降；當緩衝區空間不足時，`chloros-cli lattice network` 會發出警示。請參閱 CLI 參考手冊中的 [Linux 主機 — 套接字接收緩衝區](cli-reference.md#linux-hosts--socket-receive-buffers)。

在無顯示器的機器上，請以 **您的使用者身分** 將後端作為服務執行，以便腳本能偵測到正在運行的後端（`connect_all()` 絕不會啟動後端）：

```bash
sudo systemctl enable --now chloros-backend@$USER.service    # $USER expands to your user name
```

此時後端將以您的身分運行：其寫入的擷取資料屬於您，且會使用您的 `chloros-cli login`。對於已啟用此功能的機器，root 的 `chloros-backend.service` 仍會安裝，但 root 後端寫入的每個檔案均歸 root 所有。 請參閱 CLI 參考手冊中的 [將後端作為服務執行](cli-reference.md#running-the-backend-as-a-service-headless)。

由服務執行的後端是共享的，絕不會被客戶端停止：SDK 腳本和 `chloros-cli` 會直接使用現有的後端 （`ChlorosLocal.shutdown_backend()` 僅會停止由 `ChlorosLocal` 自行啟動的後端），而 `GET /api/status` 將該單元命名為 `service`。 請使用 `sudo systemctl restart chloros-backend@$USER.service` 重新啟動它。升級套件時，後端服務會在安裝過程中停止運作， 因此正在執行的腳本會失去其攝影機和感測器，隨後會重新啟動所有已啟用的裝置；移除套件會停止這些服務並停用已啟用的裝置。由於每次僅能運行一個後端，因此請僅啟用一個裝置：若嘗試啟動第二個裝置，該裝置將無法啟動並維持在 `failed` （退出狀態 98，未重新啟動），其 `systemctl status` 會說明原因。若要切換單元，請停用您不使用的那個。

### 步驟 2 — 安裝 Python SDK

**Chloros 安裝程式會隨附一個相容的 SDK wheel 檔案。** 每個 Windows 安裝程式和 Linux .deb 套件都會在磁碟上放置一個名為 `chloros_sdk-X.Y.Z-py3-none-any.whl` 的檔案，其版本與 GUI / CLI / 後端版本完全一致。您無需特別追蹤 PyPI 以保持版本同步。

#### Windows

安裝程式會自動使用您的系統 Python 執行 `pip install` 來處理內建的 wheel 檔案（優先使用 `py.exe` 啟動程式，若無法執行則回退至 `python -m pip`）。無需任何操作——安裝成功後，`import chloros_sdk` 即可在您的 Python 環境中運作。若系統上未安裝 Python，安裝程式會靜默跳過此步驟，且 GUI 與 CLI 仍可正常運作。

#### Linux (.deb)

兩個 .deb 套件（x86_64 和 Jetson）皆會安裝適用於系統 `python3` 的 SDK，因此 `import chloros_sdk` 在安裝完成後即可運作，無需 pip 亦無需網路連線。這些套件 （`chloros_sdk`、 `lattice_sdk`、`lattice_cam`、`arena_api` 以及專案載入器）會被整合至 `/usr/lib/python3/dist-packages` 中， 而 SDK 的 `chloros-mcp` 指令（即 [MCP 伺服器](#mcp-server-ai-assistants)）則位於 `/usr/bin` 中，所有這些皆隸屬於該套件：升級時會替換這些檔案，移除套件時則會一併移除。

SDK 本身的依賴項 `python3-numpy` 和 `python3-requests` 僅為 **建議安裝，並非必需**（GUI、CLI 和後端皆為自包含式）： `sudo apt install ./chloros_<version>_amd64.deb`（在 Jetson 上為 `…_arm64_jp6.deb`）會安裝這些套件，而 `dpkg -i` 則不會。 安裝輸出會報告 `python3` 所導入的 SDK 版本、任何缺失的依賴項， 以及位於 `python3` 路徑上其他位置、會覆蓋該套件副本的 `chloros_sdk`（例如先前透過 `pip install --user` 安裝的版本）。

在無法偵測到系統套件的 virtualenv 中，請將捆綁的 wheel 安裝至其中：

```bash
pip install /usr/lib/chloros/sdk/chloros_sdk-*.whl
```

對於隔離式 Jetson 部署，此過程完全離線進行 — SDK 和 wheel 已存在於磁碟上。

#### 公開 PyPI

對於僅使用 pip 的主機（未安裝 Chloros 桌面套件；此時 SDK 會驅動另一台機器上的後端，請參閱 [遠端後端模式](#remote-backend-mode-pip-only-host-via-tunnel)）：

```bash
pip install chloros-sdk
```

PyPI 提供最新的穩定版本。請確保 SDK 與其驅動的 Chloros 後端保持相同版本（參見 [版本控制與相容性](#versioning--compatibility)）。 DAQ-M 輔助程式 `DLS` 亦需 `bleak`：`pip install "chloros-sdk[camera]"`（或 `pip install bleak`）。

#### 驗證

```python
import chloros_sdk
print(chloros_sdk.__version__)
print("CAMERA_AVAILABLE =", chloros_sdk.CAMERA_AVAILABLE)
print("PROJECT_AVAILABLE =", chloros_sdk.PROJECT_AVAILABLE)
```

> **需具備 Chloros+ 訂閱資格。** 所有 SDK 呼叫皆需有效的 Chloros+ 登入帳戶。請在每台機器上執行一次 `chloros-cli login user@example.com 'YourPassword'`；憑證會快取於 `~/.chloros/` 中。

### 我需要桌面套件嗎？

對於大多數工作流程而言，僅靠 pip 套件是**不足**的。以下是各 SDK 介面所需的內容：

| SDK 介面 | 需要桌面套件嗎？ | 原因 |
| --- | --- | --- |
| `ChlorosLocal`、`process_folder`、`process_lattice_capture` | **是**（用於啟動一個）**／否** （後端已可連線） | 使用正在運行的後端（桌面應用程式的，或由其他腳本啟動的）；否則自動啟動已安裝的後端：`/usr/lib/chloros/chloros-backend`（Linux）或 `<install folder>\backend\chloros-backend.exe` （Windows；預設為 `C:\Program Files\Chloros\backend\`，自訂安裝資料夾會從安裝程式的記錄中讀取）。傳入 `backend_exe=` 即可自行命名。 |
| `chloros-mcp` ([MCP 伺服器](#mcp-server-ai-assistants)) | **是**（要啟動一個）**／ 否**（後端已可連線） | 基於 `ChlorosLocal` 建置，因此與上一行相同。 |
| `read_image_audit_tags` | **否** | 讀取已處理影像本身的元資料（僅限 Python 標準函式庫）。 |
| `connect_camera`、 `connect_array`, `connect_daq_sensor`, `analyze_array_network`, `list_*`, `discover_*` | **是**（本地）**／ 否**（遠端） | 透過後端運作的純HTTP客戶端。本地後端 → 需安裝桌面套件。遠端後端 → `backend_url=`**透過隧道**（參見「遠端後端模式」——隨附的後端僅綁定迴路介面）。 |
| `ChlorosProject` / `open_project` | **是** | 透過後端驅動已儲存的專案。 |
| 直接 LATTICE 類別（`LatticeCamera`、`CameraPool`、`Calibration`、`DLS`、…） | **是** | 需要 Arena SDK 執行時環境（已包含於桌面套件中），以及 Python 環境中的 OpenCV (`cv2`)；而桌面套件與 SDK 皆未安裝 (`pip install opencv-python`)。否則，`CAMERA_AVAILABLE` 在匯入時即為 `False`。 `DLS` 還需要 `bleak`。 |

### 遠端後端模式（僅 pip 主機，透過隧道）

> **無法透過區域網路 (LAN) 連線至已安裝的後端。** 它僅在
> 迴路埠上監聽（`127.0.0.1` / `::1`），因此 `backend_url="http://<lan-ip>:5000"`
> **無法連接到已安裝的 Chloros**。 若要驅動位於另一台
> 機器上的後端，請自行轉發其迴路埠，並將 SDK 指向
> 隧道：

```bash
# on the pip-only host: forward local 5000 to the Chloros machine's loopback
ssh -N -L 5000:127.0.0.1:5000 user@chloros-host
```

```python
import chloros_sdk

BACKEND = "http://127.0.0.1:5000"     # the tunnel endpoint
serials = ["213800234", "214000533"]   # master first

# auto_start_backend=False: the tunnel URL looks local, so without it a tunnel
# that is down would start this machine's own backend, if one is installed.
arr = chloros_sdk.connect_array(serials, backend_url=BACKEND,
                                auto_start_backend=False)
daq = chloros_sdk.connect_daq_sensor(eth_host="daq-e-1.local",
                                     backend_url=BACKEND,
                                     auto_start_backend=False)

# The backend opens every path on ITS machine: pass absolute paths valid there.
# (A relative path would be made absolute against this machine's working folder.)
arr.capture("/home/chloros/captures/run1", processing="raw")
```

無顯示器／持續整合／機器人主機可保留一台安裝完整桌面環境的機器作為「Chloros 伺服器」，其餘位置則使用 `pip install chloros-sdk` —— 但它們之間的傳輸通道是上述由使用者自行安排的隧道，而非直接的區域網路 URL。

> **位於僅支援 pip 的主機上的 `ChlorosLocal`。** `ChlorosLocal(backend_url=BACKEND)` 會連線至無需安裝桌面環境即可存取的後端：它僅在必須啟動後端時才會搜尋本機後端二進位檔（無回應時會觸發 `auto_start_backend=True`）。在 1.2.0 版本之前，系統會先搜尋二進位檔，即使後端可達，在僅安裝 pip 的主機上仍會觸發 `ChlorosBackendError` 錯誤（「找不到 Chloros 後端…」）。您傳入的資料夾 (`process_folder`, `import_images`) 會在此機器上進行檢查並由後端讀取，因此若使用遠端後端，兩者必須指向同一個資料夾（例如共用磁碟機）。

### 僅 DAQ 工作流程（僅 pip 主機）

僅靠 `pip install chloros-sdk` 無法驅動 DAQ 感測器：DAQ 感測器必須透過配備 `connect_daq_sensor()` 的 Chloros 後端來驅動：

- **本機上的感測器：** 請在此處安裝 Chloros 桌面套件。若尚未有後端執行，`connect_daq_sensor()` 會啟動其後端。
- **另一台機器上的感測器：**該機器需要安裝桌面套件；透過 [遠端後端模式](#remote-backend-mode-pip-only-host-via-tunnel) 中的隧道連接到其後端。後端會在自身機器上開啟感測器， 因此 DAQ-U 需連接至該機器，DAQ-M 需在其藍牙範圍內，而 DAQ-E 則需位於該機器可連通的網路中。

```python
import chloros_sdk

# http://127.0.0.1:5000 is the local desktop install's backend, or the tunnel.
with chloros_sdk.connect_daq_sensor() as daq:   # smart-detect USB / BLE / ETH
    print(daq.model, daq.transport, daq.address)
    for frame in daq.latest(n=5):
        print(frame["spectrum"][:10])
```

無論採用何種方式，後端皆會強制執行「Chloros+」方案：請在運行後端的機器上（`chloros-cli login` 或桌面應用程式）使用任何付費方案（Copper 或更高階）登入。若後端已登出，或方案等級低於「Copper」，都會觸發 `ChlorosConnectError` 錯誤，並攜帶後端的錯誤原因（`AUTH_REQUIRED` / `PLAN_UPGRADE_REQUIRED`）。

---

## 快速入門

```python
import chloros_sdk

# === Image processing ===
results = chloros_sdk.process_folder(
    "C:/DroneImages/Flight001",
    indices=["NDVI", "GNDVI", "OSAVI"],
)

# === Live LATTICE single-cam ===
with chloros_sdk.connect_camera("213800234") as cam:
    cam.set_settings(exposure_time=10000, gain=0.0)
    cam.capture("output/")

# === Live LATTICE synchronized array (GUI smart-prep flow) ===
with chloros_sdk.connect_array(
        ["213800234", "214000533", "214701288", "214701292"]) as arr:
    arr.capture("output/", processing="reflectance")

# === Live DAQ spectral sensor ===
with chloros_sdk.connect_daq_sensor() as daq:    # smart-detect USB / BLE / ETH
    for frame in daq.latest(n=5):
        print(frame["spectrum"][:10])

# === Drive a saved project end-to-end ===
proj = chloros_sdk.open_project("/path/to/project")
proj.connect_all()
proj.arrays["main_rig"].capture("output/", processing="reflectance")
proj.disconnect_all()
```

---

## 頂層 API 索引

```python
import chloros_sdk

# === Image processing (full pipeline) ===
chloros_sdk.ChlorosLocal                          # class
chloros_sdk.process_folder(...)                   # one-shot helper
chloros_sdk.process_lattice_capture(...)          # LATTICE-friendly defaults
chloros_sdk.read_image_audit_tags(path)           # post-run audit

# === Live cameras (persistent backend pool) ===
chloros_sdk.connect_camera(serial, ...)           # → CameraSession
chloros_sdk.connect_array(serials, ...)           # → ArraySession (smart-prep)
chloros_sdk.attach_array(serials_or_id, ...)      # → ArraySession (attach without re-connecting)
chloros_sdk.list_cameras()
chloros_sdk.list_arrays()
chloros_sdk.discover_lattice_cameras()
chloros_sdk.analyze_array_network(...)            # network capability + recommendation
chloros_sdk.CaptureResult                         # list subclass returned by CameraSession.capture / ArraySession.capture
chloros_sdk.RecorderHandle                        # handle for an array record()/burst() job
chloros_sdk.ContinuousHandle                      # handle for an array continuous() capture
chloros_sdk.GrabResult                            # dict returned by grab() / yielded by frame_stream()
chloros_sdk.FrameFetchError                       # raised by grab()/frame_stream() on a refusal

# === Live DAQ sensors (persistent backend pool) ===
chloros_sdk.connect_daq_sensor(...)               # → DAQSensorSession
chloros_sdk.discover_daq_sensors()                # scan USB/BLE/ETH (finds a DAQ-M MAC)
chloros_sdk.list_daq_sensors()
chloros_sdk.daq_bindings() / set_daq_bindings(cameras=, arrays=)   # which DAQ feeds which camera
chloros_sdk.daq_band_presets(filter_code=None)    # band sets for Ambient + Object reflectance
chloros_sdk.light_sensor_reflectance(ambient_id, object_id, ...)   # two DAQs -> reflectance + indices
chloros_sdk.time_sync_status() / time_sync_peers() / time_sync_cameras()   # PTP grandmaster
chloros_sdk.restart_time_sync() / set_time_sync_priority(priority1=, priority2=)

# === Camera utilities (the CLI's lattice commands; by serial) ===
chloros_sdk.camera_color_profile(serial) / set_camera_color_profile(serial, ...)
chloros_sdk.camera_color_adjust(serial) / set_camera_color_adjust(serial, ...)
chloros_sdk.camera_highlights(serial) / set_camera_highlights(serial, value)
chloros_sdk.camera_white_balance_once(serial) / set_camera_white_balance(serial, ...)
chloros_sdk.camera_presets() / apply_camera_preset(serial, preset)
chloros_sdk.camera_filter_models() / camera_filter_model_info(name)
chloros_sdk.camera_firmware_status() / update_camera_firmware(serial)
chloros_sdk.camera_power_probe(serial) / sleep_camera(serial) / wake_camera(serial)
chloros_sdk.calibrate_camera_with_target(serial, ...)
chloros_sdk.network_adapters(...) / fix_network_settings()
chloros_sdk.apply_vignette_correction(input_dir, output_dir, ...)
chloros_sdk.read_dls(mac, ...) / probe_system(...) / system_info()

# === Transient cameras, alignment profiles, indices (cameras opened for one command) ===
chloros_sdk.start_camera_continuous(serial, ..., daq=None, daq_tolerance_ms=None)  # → CliJob; a DAQ bound to the camera records one .daq per frame (daq=False: none)
chloros_sdk.start_multi_camera_stream(serials, ...)   # → CliJob
chloros_sdk.capture_cameras_transient(serials, ...)
chloros_sdk.test_array_sync(serials, count, ptp_settle_s=None) / detect_array_master(line) / analyze_array_models(models)
chloros_sdk.calibrate_alignment_profile(serials, save_path, ...)
chloros_sdk.capture_aligned_multiband(profile_path, ...) / start_aligned_stream(profile_path, ...)  # → dict / CliJob
chloros_sdk.alignment_profile_info(path) / reorder_alignment_profile(path, ...)
chloros_sdk.index_catalog() / compute_index(input_path, ...) / capture_live_index(profile_path, ...)
chloros_sdk.CliJob                                # stats() / stop() / wait()

# === Project lifecycle ===
chloros_sdk.open_project(path, relink=None)       # → ChlorosProject
chloros_sdk.project_folder() / set_project_folder(path) / reset_project_folder()
chloros_sdk.run_project_recipe(recipe, project_path)  # `chloros-cli project run`
chloros_sdk.ChlorosProject                        # class
chloros_sdk.AlignmentSpec                         # dataclass
chloros_sdk.ArrayHandle, CameraHandle, SensorHandle

# === Direct-hardware (no-backend) classes (from lattice_sdk) ===
chloros_sdk.LatticeCamera, CameraSettings, PRESETS, CameraPool
chloros_sdk.Calibration, CalibrationCoefficients, FilterModel, list_filters
chloros_sdk.DLS, NetworkDiagnostics               # DLS: DAQ-M over Bluetooth; needs bleak

# === Exceptions ===
chloros_sdk.ChlorosError                          # base
chloros_sdk.ChlorosBackendError
chloros_sdk.ChlorosLicenseError
chloros_sdk.ChlorosConnectionError
chloros_sdk.ChlorosProcessingError
chloros_sdk.ChlorosProjectFilesMissingError      # a ChlorosProcessingError
chloros_sdk.ChlorosAuthenticationError
chloros_sdk.ChlorosConfigurationError
chloros_sdk.ChlorosConnectError                   # raised by smart-connect surface
chloros_sdk.LatticeError, CameraNotFoundError, ...  # from lattice_sdk

# === Availability flags ===
chloros_sdk.CAMERA_AVAILABLE     # True iff lattice_sdk imported cleanly
chloros_sdk.PROJECT_AVAILABLE    # True when chloros_sdk.project imported; open_project()
                                 # still raises ImportError without the project loaders
```

---

## 影像處理 — `ChlorosLocal`

主要處理管線類別。在建構時啟動後端（若無後端回應且為 `auto_start_backend=True` 時），建立／配置專案、監控進度，並回傳執行後摘要。

### 建構函式

```python
ChlorosLocal(
    api_url="http://127.0.0.1:5000",   # backend URL (also: backend_url=)
    auto_start_backend=True,            # spawn backend if not running
    backend_exe=None,                   # override backend binary path
    timeout=30,                         # request timeout seconds
    backend_startup_timeout=60,         # backend boot timeout
    processing_timeout=14400,           # hard cap on process() (4 h)
    processing_stuck_timeout=1800,      # no-progress threshold (30 min)
)
```

### 方法

| 方法 | 說明 |
| --- | --- |
| `create_project(project_name, camera=None)` | 建立新專案（可選配相機範本，例如 `"Survey3N_RGN"`）。 |
| `open_project(project_path, relink=None)` | 開啟已儲存的專案（其資料夾，或專案資料夾中的專案名稱），並將其設為當前開啟的專案。 若專案已移動，其路徑將重新指向目前所在的資料夾；`relink` 會標示其他遺失檔案目前所在的資料夾。透過 `source_files` 傳回後端的回應； 若無法找到任何檔案，則拋出 `ChlorosProjectFilesMissingError` 錯誤。請參閱下方的 **已移動的專案**。 |
| `import_images(folder_path, recursive=False)` | 匯入 RAW/TIF/JPG/DNG 影像 **以及 `.daq` 光感測器記錄**。傳回 `count` （影像）及 `scan_count`（記錄檔）。僅當資料夾中既無影像亦無記錄檔時才會發出警告。 |
| `export_light_sensor(daq=True, csv=True)` | 針對專案中的每筆光感測器記錄，寫入經校準的 `.daq` + `.csv`， 並儲存至 `<project>/Light Sensor/`。請參閱 [光感測器記錄](#light-sensor-recordings--calibrated-daq--csv)。 |
| `configure(debayer=..., vignette_correction=..., reflectance_calibration=..., indices=[...], export_format=..., ppk=..., daq_log_path=..., input_level=..., radiometric_output=..., array_alignment=..., array_alignment_crop=..., array_alignment_interpolation=..., custom_settings=None)` | 設定處理參數。 `indices` 為預設名稱（`NDVI`、`GNDVI`、 `OSAVI`, …)，每個名稱皆根據其相機的波段新增至專案中；若名稱並非預設值，或該波段組合無法構成（例如 `NDRE` 需要紅光（RE）與近紅外線（NIR），而單一相機無法提供這兩者），則會以 `UserWarning` 為名，且不包含索引影像。`daq_log_path` 則指代一個光感測器錄製檔案 （即 `.daq` 或 `.csv` 檔案，或其資料夾），每幀影像皆會與此進行比對，無論是 LATTICE 還是 Survey3，而非使用隨影像一併匯入的記錄檔； 若指定路徑不存在或未包含任何記錄，則會使用已匯入的記錄。 |
| `process(mode="parallel", wait=True, progress_callback=None, poll_interval=2.0)` | 執行處理流程。返回 `{"status": "complete", "async": False}`， 若後端提供，則會附加一個 `summary` 鍵 — 請參閱 [執行後摘要與提示](#post-run-summary--hints) ——若無法驗證執行結果的輸出，則返回 `export_check_failure`；若遺失影像，則返回 `image_failure`。`wait=False` 會一次返回 `{"status": "started", "async": True}`。 |
| `get_config()` / `get_status()` / `status()` | 檢查後端狀態。 |
| `logout()` | 登出：清除快取的憑證並釋放此機器的裝置槽位（結果中為 `device_deregistration`）。 正在進行的處理執行不會被中斷：它將以啟動時的模式完成（結果中顯示為 `run_in_progress: true` 和 `run_mode`），而下一次執行將採用標準處理流程。 |
| `shutdown_backend()` | 若此執行個體曾啟動後端，則停止該後端；若發現後端正在運行，則不予干預。即使留下 `with` 區塊，或該執行個體正被垃圾回收，也不會停止後端： 後端會繼續為下一個腳本運行，並在無人使用其服務 30 分鐘後自行停止。系統會要求後端關閉，因此它會先釋放其攝影機和光感測器，並完成任何錄製作業； 僅當 10 秒後仍持續運行時，才會終止其運作。 |
| `discover_cameras()` | **透過此實例的後端** 偵測 LATTICE 攝影機 (`/api/camera/discover`)。回傳一組字典清單（`serial`、`model`、 `ip`, …) — 其結構與 GUI／CLI 所見相同。若未找到任何相機或無法連線至後端，則返回空清單。 |
| `camera_capture(output_dir, format="tiff", **settings)` |**透過後端**擷取單一幀影像（由此處理程序自動啟動），使其獲得與 GUI／CLI 相同的預處理設定（預設 12 位元、池資源重複使用、嵌入式校準元資料）。請使用 `serial=` 或 `device_index=` 解析目標； 將 `exposure`/`gain`/`pixel_format`/`preset` 作為 `**settings` 傳入。回傳舊版元資料字典（`filepath`、`width`、 `height`、`pixel_format`、`exposure_time`、`gain`、`timestamp`). |
| `camera_stream(serial, *, fps=10.0, overlay=None, decode=True, connect_timeout=10.0, read_timeout=15.0)` | 從彙總式攝影機產生疊加合成預覽幀 — 透過後端的 `/api/camera/<serial>/stream-annotated` 路徑運作的輕量級 MJPEG 客戶端 （斑馬線／網格／十字線／直方圖／峰值／點狀圖皆由伺服器端繪製）。`decode=True` 產出 BGR 陣列，並需要 OpenCV (`cv2`)，而SDK並未安裝該套件；`False` 產出原始 JPEG 位元組。亦可透過專案層級存取 `ChlorosProject.stream(overlays=True)`。 |

**執行中的處理任務會保留其專案。** 當處理執行中時——無論是此腳本、應用程式、`chloros-cli process` 或 `chloros-mcp` 的執行——後端將拒絕建立、開啟、匯入或變更專案的處理設定 （HTTP 409 `PROCESSING_IN_PROGRESS`）：`create_project()` 和 `import_images()` 會觸發 `ChlorosProcessingError`， 而 `configure()` 會觸發 `ChlorosConfigurationError`；此外，若 `process()` 的模式與當前執行模式不同，則會觸發 `ChlorosProcessingError` （執行程序會保留其啟動時的模式），並各自附帶後端傳來的訊息。儲存至專案中的擷取資料會在執行結束時加入該專案（其回覆中的 `project_add` 如此表示）。請等待執行完成（`get_status()`），然後繼續。**已移動的專案。** 專案的 `project.json` 會以完整路徑為每個原始檔（每張影像和光感測器記錄）命名，因此若將專案資料夾——或整個專案資料夾——移至另一顆磁碟機，這些檔案仍會保留舊位置的路徑名稱。 每次開啟時，系統會自行修正可修正的部分：原本位於專案資料夾內的檔案，會被搜尋至專案目前所在的資料夾中，並將其中找到的每個路徑重新指向正確位置後儲存 （`project.json` 會先備份為 `project.json.before-relink-<date>-<time>`）。路徑重定向僅適用於該位置確實存在的檔案。 原本位於專案資料夾外的檔案——例如 Hub 工作階段或「新增資料夾」匯入的檔案——僅會透過 `relink=<the folder they are in now>`（或其上層的任何資料夾）被找到。`open_project()` 會返回 `source_files`： `total`、`found`、`missing`、`missing_roots`（缺失檔案原先所在的資料夾）， `missing_outside_project` 和 `relink`（重新指向的對象，從何處指向何處）。若專案中部分檔案遺失，其餘檔案仍會正常開啟；若完全找不到任何檔案，則無法開啟該專案， 而 `open_project()` 會觸發 `ChlorosProjectFilesMissingError`（`.missing_roots`、`.audit`； HTTP 409 `PROJECT_SOURCES_MISSING`）。CLI 的 `project open --relink` 和 `chloros-mcp`'`open_project`（`relink_folder`）亦會產生相同行為。

用作上下文管理器以確保清理：

```python
with chloros_sdk.ChlorosLocal() as cl:
    cl.create_project("FieldA_2026-05-26", camera="Survey3N_RGN")
    cl.import_images("C:/DroneImages/Flight001")
    cl.configure(
        vignette_correction=True,
        reflectance_calibration=True,
        indices=["NDVI", "GNDVI", "OSAVI"],
        export_format="TIFF (16-bit)",
    )
    results = cl.process(mode="parallel", wait=True)
print(results["summary"])
```

### 光感測器記錄 — 已校準 `.daq` + `.csv`

DAQ-U / DAQ-M / DAQ-E 可在 **未** 搭配其校準套件的情況下進行記錄。這正是
公開版 [`chloros_scripts`](https://github.com/mapircamera/chloros_scripts)
記錄器 (`record_daq.py`) 預設的運作方式：它們寫入原始感測器計數值，並在
檔案中標記時間戳記，以便 Chloros 先從 **序列號** 擷取該感測器的工廠校準值 — 首先從本地快取
，接著是 MAPIR 雲端 —— 並在匯入時套用校正。

Chloros 會將結果寫回，每筆記錄產生兩個產品，存放於
`<project>/Light Sensor/` 下：

| 產品 | 內容說明 |
| --- | --- |
| `<name>_calibrated.daq` | 可重新處理的存檔 — 與即時記錄採用相同結構，現已標示產生該存檔的套件。重新匯入時，**不會**進行第二次校正。 |
| `<name>_calibrated.csv` | 感測器自身波長網格上的光譜輻照度（單位：W/m²/nm），每筆讀數佔一行，並附有光度學欄位（總功率、明視/暗視勒克斯、PPFD 及其藍/綠/紅分量、峰值波長）。 |
| `<name>_raw.daq` / `<name>_raw.csv` | **僅限無套件感測器（DAQ-A）.** 感測器的原始光譜計數值 — *非*輻照度。詳見下文。 |

`<name>` 為記錄檔本身的檔案名稱。位於不同資料夾的兩份記錄可共用
同一個 — 由兩台相機在
同一毫秒內擷取的讀數（`<capture>/daq/daq_data_<sensor id>_<ts>.daq`） — 且各自仍會產生獨立的結果：第一個保留
`<name>`，而下一筆則在名稱**之後**加上區分兩者的資料夾名稱
(`<name>__<capture>_daq_calibrated.daq`)，因此記錄的日期與時間仍居首位。

`process()` 將此匯出作業作為其階段之一執行。它**無需**影像：
單獨搭載的光感測器即構成一級工作流程，而此類專案在設計上
本就沒有任何影像。

**DAQ-A 記錄會以原始計數形式匯出。** DAQ-A 系列早於「依序列號」
捆綁系統問世，因此無需擷取任何捆綁檔——它改以在現場對照
反射率標靶進行校準，這正是它從未需要捆綁檔的原因。 這些記錄會以
`_raw` 為前綴進行匯出，而非 `_calibrated`：這是採用不同的檔案名稱而非檔案內的標記，
因為該標示必須能在僅以裸名透過電子郵件傳送時仍能保留。
`.csv` 標頭中標示為 `raw spectral sensor counts (NOT irradiance)`，並警告這些
數值僅在**單一**檔案內可比——這正是基於標靶的校準所
的用途——而非跨感測器比較。與功率相關的光度學欄位（總功率、
明視／暗視勒克斯、PPFD）會返回 **NULL**，而非根據計數值進行積分。

若 DAQ-U / DAQ-M / DAQ-E 的套件根本無法取得，系統仍會 **跳過** 該裝置，
而非寫入原始資料：此情況下套件確實存在，因此「重新連線並重新處理」是實際可行的建議。

舊版 **v1.01 / v1.02** 記錄（由 DAQ-A-SD 寫入）不包含每筆讀數的紀元，
僅有檔案的寫入時間。影像↔下行光匹配器仍會拒絕這些記錄——將
幀與寫入時間進行配對，將導致隱性錯誤——但匯出工具會讀取這些記錄，且
生成的 CSV 檔案會輸出 `clock=daq_created_on`，因此該產品會標示其所採用的時鐘類型。

```python
import chloros_sdk

with chloros_sdk.ChlorosLocal() as cl:
    cl.create_project("DAQ-U_2026-08-26")
    cl.import_images("C:/Flights/raw_daq")     # .daq only — no camera involved
    result = cl.export_light_sensor()          # or just cl.process()

for rec in result["exported"]:
    print(rec["csv"])
for rec in result["skipped"]:
    print("skipped", rec["source"], "--", rec["reason"])
```

若無法取得校準包的記錄（離線狀態，或感測器無
檔案中的校準資料），系統會以 `skipped` 格式回報，並附上**原因**。 該記錄絕不會
以包含原始計數值的「已校準」檔案形式寫入 — 請連線至網際網路並
重新執行，匯出作業即會完成。

### 進度回呼

```python
def show_progress(percent, message):
    print(f"[{percent:3d}%] {message}")

with chloros_sdk.ChlorosLocal() as cl:
    cl.create_project("FieldA")
    cl.import_images("C:/DroneImages/Flight001")
    cl.configure(indices=["NDVI"])
    cl.process(progress_callback=show_progress, poll_interval=1.0)
```

### 執行後摘要與提示

完成後，`process()` 會擷取 `GET /api/processing-summary`，並將其內容附加為 `result["summary"]`。此擷取操作採「盡力而為」原則，絕不會阻擋成功的回傳 — 若摘要不可用，`process()` 將回退至純粹的 `{"status": "complete", "async": False}` 結構。`summary["hints"]` 中的每個條目 — 包含建議修正措施的完整句子， 例如為何某次執行產生零輸出 — 也會重新發送為 Python 格式的 `UserWarning`，因此即使您從未檢查該字典，零輸出的執行也具有自我診斷功能：

```python
result = cl.process()
for hint in result.get("summary", {}).get("hints", []):
    print("HINT:", hint)
# hints also arrive on the warnings channel:
#   python -W always::UserWarning your_script.py
```

`summary["totals"]` 則是機器可讀的部分：

| 鍵 | 統計內容 |
| --- | --- |
| `models` | 該執行中的相機群組。 |
| `images_in_groups` | 這些群組中的原始影像。 |
| `targets_found` | 偵測到的反射率目標。 |
| `images_calibrated` | 該執行任務所產出反射率產品的影像。 |
| `images_failed` | 處理失敗且未匯出的影像。 |
| `exported_files` | **該執行任務所產出的影像產品檔案**：該運行所產生的所有產品（去拜耳化、預覽、輻照度、反射率、索引、目標渲染），包含所有格式。 |
| `reflectance_files` | 其中反射率產品 (`Reflectance_Calibrated_Images`)，無論是由目標或光感測器（DAQ）的輻照度進行校正。若某次執行未設定目標但仍寫入反射率，系統會於提示中標示；僅當執行未寫入任何反射率產出時，系統才會提示其反射率輸出缺失。 |
| `clamp_masks_written` | [解混鉗位遮罩](#unmix-clamp-mask--which-pixels-a-band-was-clamped-on) ——這些是輔助資料，而非產出結果，因此不會出現在 `exported_files` 中（CLI 亦以相同方式將其區分計算）。 |
| `captures_left_out` | 專案中位於 `captures` 資料夾內，但本次執行未擷取的原始影像幀。 每次執行開始時會等待前一次擷取的資料，因此此數值為 0，除非在執行期間儲存了擷取資料，或等待時間已到期。`summary["captures_left_out_note"]` 說明了應如何處理這些資料。 |
| `daq_recordings_exported` / `daq_recordings_skipped` | 光感測器記錄，刻意分開計數 — 它們來自不同的階段，且存在於完全沒有影像的執行中，因此若將其併入，會使僅進行資料擷取（DAQ）的執行看起來像是已匯出影像。 |

與此同時：`summary["output_dirs"]`（寫入過的每個目錄）、
`summary["light_sensor_export"]`、`summary["stopped"]`（當使用者中斷
執行時為真， 因此部分計數不會被視為產出不足的已完成運行），以及
`summary["groups"]`（各組別的細項分項）。

`exported_files` 是由管線在 **寫入時** 記錄的，並非事後從
專案的映像物件中掃描而來。平行與 GPU 策略會建立各自的映像
物件（在 GPU 路徑的工作子程序中），因此舊版掃描會針對
針對每個此類執行都回報了`0 file(s) written`，隨後發出零輸出提示——這發生在
所有環節皆運作正常的執行中。若您根據此數字編寫腳本，現在正常的平行執行
將回報非零計數。

輕量感測器跳過報告會顯示讀取器實際針對每個檔案確定的原因——例如
無法讀取的架構、缺失的套件、寫入錯誤——並已**去重**，因此因單一原因被跳過的二十個檔案，
會被視為單一原因，而非該原因的二十次重複。

> **當執行結果未產生任何影像時，`process()` 不會觸發。** 這是 SDK 與
> CLI 刻意設計不同的兩個地方之一：`chloros-cli process` 將「已請求產品，但未
> 寫入任何產品」視為失敗並以非零狀態退出，而 SDK 則正常返回並透過
> 透過 `summary` / hints 回報該狀況。若您的處理流程應在空運行時停止，請自行檢查
> — 檢視 `summary` （或統計專案資料夾下的檔案數量），而非僅依賴
> 是否出現例外狀況。常見原因包括輸入資料夾未被識別為
> 擷取來源，以及因不適用於現有攝影機而跳過某些產出（例如僅具 RGB 的
> 攝影機所產生的輻射度）。
>
> 當執行任務的輸出無法驗證時，`process()` 也不會觸發：系統會發出警告並設定
> `result["export_check_failure"]`。產出檔仍會被寫入。若要在此情況下停止執行，請檢查該關鍵值。 一個
> 遺失影像的執行程序亦同：系統會發出警告並設定 `result["image_failure"]`，而 `chloros-cli process`
> 會在此狀態下以非零狀態退出。

### 便利函式

```python
# One-call process: project + import + configure + process
results = chloros_sdk.process_folder(
    folder_path="C:/DroneImages/Flight001",
    project_name="FieldA_2026-05-26",
    camera="Survey3N_RGN",
    indices=["NDVI", "GNDVI", "OSAVI"],
    vignette_correction=True,
    reflectance_calibration=True,
    export_format="TIFF (16-bit)",
    mode="parallel",
    debayer="High Quality (Faster)",      # or "Texture Aware (Slow, Highest Quality)"
    ppk=False,
    recursive=False,
    processing_timeout=14400,
)

# LATTICE-friendly defaults (no panel-target detection, standard debayer)
results = chloros_sdk.process_lattice_capture(
    folder_path="C:/Captures/2026-05-13_Field",
    indices=["NDVI"],
)

# process_folder() / process_lattice_capture() also take input_level=,
# array_alignment=, array_alignment_crop=, array_alignment_interpolation=,
# progress_callback=, poll_interval=, timeout=, processing_stuck_timeout=,
# backend_exe= and api_url= (NOT backend_url=). indices defaults to ["NDVI"].

# Audit which calibration sources were applied to a processed image
tags = chloros_sdk.read_image_audit_tags("output/Reflectance_Calibrated/x.tif")
print(tags["CalibrationSource"])   # 'per_serial' / 'legacy_lookup' / 'none'
print(tags["VignetteSource"])      # 'per_serial' / 'legacy_polynomial' / 'none'
```

每個執行程序都在其自身的 `with ChlorosLocal(...)` 內運行；當它返回時，其必須啟動的後端會繼續運行， 以便進行下一次呼叫，並在 30 分鐘內無人使用時自行停止。這兩者均不接受 `auto_start_backend`、`radiometric_output` 或 `daq_log_path`；請使用 `ChlorosLocal` 及 `configure()` 來處理上述情況。 `process_lattice_capture()` 不接受 `camera`、`debayer` 或 `ppk`：它會跳過面板偵測，並使用標準的去拜爾演算法。

### 支援的數值

```python
# export_format
"TIFF (16-bit)"           # default, recommended
"TIFF (32-bit, Percent)"  # reflectance percentage as float32
"TIFF (8-bit)"            # all-8-bit images are written as this; never raised
"PNG (8-bit)"
"JPG (8-bit)"

# debayer
"High Quality (Faster)"               # standard, default
"Texture Aware (Slow, Highest Quality)"  # neural debayer for Survey3 imagery, Chloros+ only.
                                         # Survey3 only: on a project holding LATTICE captures
                                         # (which always use the standard demosaic) configure()
                                         # or process() raises ChlorosConfigurationError; a
                                         # Texture Aware value the project saved earlier is set
                                         # to Standard with a UserWarning, as the app does. Needs a
                                         # CUDA GPU the backend's PyTorch can use (an NVIDIA
                                         # GPU, or a Jetson Orin's). Without one, process()
                                         # uses the standard debayer, and the backend prints
                                         # one "[DEBAYER] ... needs a CUDA GPU" line saying why.
"Standard (Fast, Medium Quality)"      # alias used internally for LATTICE

# input_level (LATTICE only; Survey3 .raw ignores)
"auto"        # default — infers from each file's XMP ProcessingLevel tag
"raw"         # force-treat as raw Bayer
"debayered"   # force-treat as Chloros's saved debayered (3-channel) product
"processed"   # force-treat as already-calibrated radiance

# array_alignment / array_alignment_crop (LATTICE arrays; None = keep saved setting)
True          # backend default — apply the module-to-module transform stamped
              # in each capture's Chloros:Alignment* XMP to every product
False         # export in native sensor geometry / skip the common-overlap crop

# array_alignment_interpolation (alignment warp resampling)
"bilinear"    # backend default
"nearest"     # preserves exact source DNs (no inter-pixel value mixing)
"cubic"
```

#### 輻測輸出（LATTICE 多光譜處理流程）

`process` 處理流程的 LATTICE 多光譜（M3C/M3M）匯出層級 — `reflectance`（預設值）、 `radiance`、`sensor-response` 或 `all`（每張影像的每個適用模式）—— 設定專案的 **「匯出輻度」**及**「匯出反射率」** 產品：`radiance` 僅匯出輻照度，`reflectance` 僅匯出反射率，`sensor-response` 兩者皆不匯出（僅提供去拜耳化與預覽產品），`all` 則同時輸出兩者。去拜耳化與預覽產出檔並非此層級可變更的項目。`configure()` 對此設有專用關鍵字：

```python
with chloros_sdk.ChlorosLocal() as cl:
    cl.create_project("Field_A")
    cl.import_images("C:/Captures/lattice_flight")
    cl.configure(
        radiometric_output="radiance",   # reflectance (default) / radiance / sensor-response / all
        export_format="TIFF (32-bit, Percent)",
    )
    cl.process()
```

進階的應急方案——透過 `custom_settings` 寫入產出——仍然有效，但請記住這會替換整個設定區塊 （參見下方的警告）。寫入 `"Export <product>"` 鍵：專案自身的 `"Export radiance"` / `"Export reflectance"` 設定會優先於較舊的 `"Radiometric output"` 鍵，因此僅使用該鍵並不會產生任何變更。

```python
cl.configure(custom_settings={
    "Project Settings": {
        "Processing": {"Export radiance": True, "Export reflectance": False},
        "Export": {"Calibrated image format": "TIFF (32-bit, Percent)"},
    }
})
```

`reflectance` （預設值）會將相機輻度除以 **與時間戳記匹配的 DAQ 下行輻度**，該值會自動從記錄的 `.daq`（DAQ-U/M/E）**或與影像一同發現的 DAQ-M 原生 `.csv`**；若本地端缺少任何單一相機或 DAQ 校正套件，將於首次使用時**自動從 AWS 擷取**。CLI 會依照`chloros-cli process`：`--radiance`/`--no-radiance`、`--reflectance`/`--no-reflectance`、`--debayered`、`--preview`。**這是誰的讀數？**一個 `.daq` 擷取檔所儲存的，是其所描述的相機（或多台相機）的讀數——該相機名稱載於其旁的一個 `<file>.daq.cameras.json` 附檔中 （單張、同步及陣列擷取；原始連拍則在其清單中列出各成員的讀取值）——且**僅與來自該等相機的影像**配對。因此，沒有自身讀取值的獨立相機將不再借用其他相機的 DAQ；請將 DAQ 綁定至該相機，或進行錄製。DAQ 自身的錄製 （`chloros-cli daq pool-record`，或在 GUI 中啟動的錄製），或是任何未包含該 sidecar 的 `.daq`（包含較舊的擷取檔），仍可服務所有相機。**陽光下的 OCN 相機。** 每個 OCN 校正包皆在窄帶光源下建構（每波段一顆 LED），而陽光會混合各波段。Chloros 1.3.0 在波段分離後會套用基於陽光的校正，這能改善 OCN 在陽光下的輻照度、反射率及折射率產出結果。 只要進行光譜分離之處，此修正皆會生效——包括 `process()`、擷取影像、`grab()` / `frame_stream()` 以及即時預覽——適用於支援此功能的 OCN 校準包；RGN 相機則維持不變。 若使用較早版本拍攝的 OCN 影像，在具備該影像拍攝時所用校準資料的電腦上進行處理時，亦會套用此修正（Chloros 會在相機連接後保留其校準資料）；否則則依原有方式處理。因此，OCN 的輻度、反射率及指數值將與 1.2.0 版本有所不同。

> `custom_settings` **取代** 整個計算設定區塊 （其設計上會繞過 `configure()` 的其他關鍵字與驗證機制）。使用時，請包含您所關心的每個 `Project Settings` 關鍵字，如上例所示。

#### 讀取指數像素 （NDVI 及其他植被指數）

**整數指數柵格是針對每張影像進行的對比度拉伸，而非 −1…1 的映射。** `TIFF (16-bit)`、`TIFF (8-bit)`、 `PNG (8-bit)` 以及 `JPG (8-bit)` 等指數產品——包括 8 位元擷取影像所寫入的 8 位元 TIFF 檔案——皆儲存

```

DN = (value − IndexMin) / (IndexMax − IndexMin) × full_scale
```

其中 `IndexMin` / `IndexMax` 分別是 **該幀本身的最低與最高有限索引值**。 DN 0 並非 −1，且滿量程並非 +1。

Chloros 會在兩個端點都加蓋標記，以確保映射保持可逆：

| 標籤 | 含義 |
| --- | --- |
| `Chloros:IndexScaling` | `linear_minmax` — 如下所示的慣例 |
| `Chloros:IndexMin` | 儲存為 DN 0 的索引值 |
| `Chloros:IndexMax` | 儲存為全量程的索引值 |

讀取這些資料並進行反轉，首先根據儲存的資料型別 (dtype) 的最大值，將儲存的 DN 歸一化為 0…1，以便單一表達式能涵蓋所有整數格式：

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

讀取時無需 `-config pix4d.config` — exiftool 能自行讀取這些 XMP 屬性。此設定僅在 *寫入* 時才需要。

> **`IndexMin == IndexMax`** 表示該幀的每個像素都具有相同的索引值。 光柵以中點值填充，且無需除以任何數值——直接取端點值即可。

> **若無 `Chloros:Index*` 標籤，則表示未指定映射關係。** LUT 匯出檔是一種上色的渲染結果，其像素為根據您的 LUT 閾值視窗進行鍵控的漸變色；而沒有有限索引像素的幀則無範圍可言。無論哪種情況，請勿假設存在映射關係——此規則與缺少 `PixelScale` 標籤的情況相同。**由於拉伸是針對每張影像進行的，因此同一航次的兩幀影像在數位數值（DN）上無法相互比較**—— 每幀都有其自身的端點，因此相同的NDVI值在不同影像中會對應到不同的DN。若需進行跨航次的定量分析（如分區統計、固定閾值、時間序列），請匯出**`TIFF (32-bit, Percent)`**：索引柵格將以 float32 格式寫入其中，內容為**索引值本身**，無需重新縮放亦無需標籤。儘管格式名稱如此，索引產品並非 ×100 —— 該縮放僅適用於整數資料型的反射率。

#### 解混剪裁遮罩 — 某波段在哪些像素上被剪裁

3×3 光譜解混可能會導致某波段呈現負值 — 例如當光線幾乎不包含該波段，或場景超出相機解混矩陣所描述的範圍時。這並非真實值，因此匯出時會將其固定為 0 ——即時預覽畫面亦顯示相同的 0。某波段可能僅在少數像素上被限制，也可能幾乎涵蓋所有像素；限制遮罩會記錄這兩種情況。`Chloros:Band{R,G,B}UnmixClampFraction` 說明此現象發生於幀中的多大範圍；而限制遮罩則說明 **哪些像素**：

- **`clamp_masks/<product folder>/<product name>_clampmask.tif`**— 位於產品資料夾**旁邊**的 `clamp_masks/` 資料夾中， 絕不會位於其內部（`radiance/x_radiance.tif` → `clamp_masks/radiance/x_radiance_clampmask.tif`）——適用於每個輻照度、反射率及指數產品，其中至少有一個像素發生了截斷。 單波段 `uint8`，位於該產品自身的像素網格上（包含方向、 ROI、像素合併及陣列對齊變形均已包含在內）。每個遮罩會標示其產品所屬的相機（`Make` / `Model` / 序列號）、擷取時間及鏡頭標籤，並列出其位元所代表的波段 （`XMP-Camera:BandName`，以位元 0 為首）；其 `ImageDescription` 和 `Software` 標記其為夾緊遮罩，且不包含任何校正資訊。
- **位元**：位元 0 = 產品 R 通道中的波段 (`Chloros:BandRName`)，位元 1 = G (`BandGName`)，位元 2 = B (`BandBName`)。若位元被設定為 1，表示該頻段中該像素的值已被鉗位為 0 — 請將其（以及由此建立的任何索引）視為可信度較低。
- **`Chloros:UnmixClampMask`** 會標記在每個執行過分離處理的產品上：此為遮罩相對於產品自身資料夾的相對路徑（`../clamp_masks/<product folder>/<product name>_clampmask.tif`，始終以 `/`分隔 — 將其與影像資料夾名稱結合即可開啟遮罩），若無像素被限制則標記為 `none`。若無標記，表示該產品從未執行過解混程（RGB 或單色相機、raw/去拜耳化級別）。
- 擷取結果會將其作為幀的 **`clamp_mask`** 傳遞（即遮罩的路徑；若無像素被限制，則為 `None`） — 在 `CameraSession.capture()` 結果中，`ArraySession.capture()`'`CaptureResult` 幀以及 HTTP 擷取回應中。離線建置會將其遮罩列為 `clamp_masks`。
- 即時索引的多頻段堆疊（`/api/camera/cli-array-index` 與 `save_multiband`）會針對每台來源攝影機產生一個遮罩，並存放於輸出資料夾內的 `clamp_masks/` 中 — 請參閱 CLI 參考手冊'中的 `lattice index`。影片則沒有遮罩：離線建置的結果會改為報告每幀的夾緊比例。

由於任何層級的產品資料夾都不包含遮罩，因此可將產品資料夾（包含子資料夾）直接傳遞給攝影測量軟體——該軟體僅會讀取影像。請將產品資料夾傳遞給該軟體， 而非其上層的資料夾。Chloros 自身的匯入功能會跳過遮罩。

```python
import os, numpy as np, tifffile

mask = tifffile.imread(
    "clamp_masks/radiance/capture_<ts>_SN<serial>-<filter>_radiance_clampmask.tif")
# or, from the image: os.path.join(os.path.dirname(image_path), tag), with
# tag = the image's Chloros:UnmixClampMask
r_clamped = (mask & 1) != 0      # the band Chloros:BandRName names
g_clamped = (mask & 2) != 0
b_clamped = (mask & 4) != 0
```

---

## 適用於 LATTICE 相機的 Smart-Connect

針對即時硬體的持久後端連線。使用與 GUI 相同的端點，因此 SDK／CLI／GUI 間的行為完全一致。

### 單一相機 — `CameraSession`

```python
import chloros_sdk

# Open by serial; reuses existing pool entry if one exists
with chloros_sdk.connect_camera("213800234") as cam:
    # cam is a CameraSession; supports context manager + manual disconnect
    cam.set_settings(
        exposure_time=10000,    # microseconds
        gain=0.0,               # dB
        pixel_format="BayerRG12",
        target_brightness=80,
        ae_damping=8.0,
    )
    cam.capture("output/", ext=".tiff")
```

此處設定的曝光、增益和目標值將被視為您的設定：Chloros 的自動曝光控制（包括 DAQ 測量的曝光）不會干預這些設定。`cam.set_settings(exposure_auto="Continuous")`（或 `gain_auto=`）將曝光（增益）交還， 而 `cam.set_settings(target_brightness_auto=True)` 則將目標交還給 Chloros 的自動目標設定。

#### `connect_camera()` 簽名

```python
connect_camera(
    serial,
    *,
    preset=None,                       # "default" | "high_quality" | "high_speed" | "triggered"
    settings=None,                     # dict overlaid on the preset
    backend_url="http://127.0.0.1:5000",  # deliberately not 'localhost' (::1-first on Windows ≈ 2 s/request)
    timeout=60.0,
    auto_start_backend=True,           # spawn a local backend if none is running
) -> CameraSession
```

- **它會在串流啟動後返回。** 後端會進行等待，且有時間上限（約 15 秒等待串流啟動，接著再等 10 秒以取得第一個完整的緩衝區——此處並非指第一個預覽幀，該幀約在 1.5 秒後出現）， 因此緊接在 `status()` 之後便出現 `streaming`。該會話的 `connect_stream` 即為回覆中的 `stream` 區塊： `state` `streaming`、`no_frames`（資料流已啟動但未傳送任何資料 ——`note` 已明確指出這一點，且針對使用巨型封包的攝影機，需以 1500 位元組的封包大小重新連線），`starting` 或 `start_failed`，其中 `waited_s`。`None` 適用於已連線的攝影機。預覽 （`get_camera_preview`、`/api/camera/jpeg/<serial>`）在等待結束後立即被請求，系統會等待（上限 4 秒）相機傳回首個預覽幀，而非回應 「尚無畫面」；而其串流尚未傳送任何資料的攝影機（`no_frames`，或已啟用硬體觸發）仍會立即回應。
- **封包大小可由您自行設定。** `settings={"gev_packet_size": 1500, "gev_packet_size_auto": False}` 以 1500 位元組封包開啟攝影機，未進行探測（`chloros-cli lattice cam-connect --packet-size standard` 傳送相同封包）。
- **每次僅能建立一個連線，以維持攝影機資源池**：包含開啟連線、封包大小探測、首次校準資料下載（對這台電腦而言，新攝影機約需 70 秒），以及串流啟動。若此時有其他請求需要使用該攝影機（`read_nodes`， `set_settings`、擷取、第二次連線）將最多等待 5 秒，隨後因 HTTP 503 錯誤而失敗（`CAMERA_POOL_BUSY`），並標示正在連線的攝影機名稱（`ChlorosConnectError`）；稍後 稍後再試。清單項目從不等待：`list_cameras()` (`GET /api/camera/pool`)、`list_arrays()` 和 `CameraSession.status()` 會立即回應。 一臺正在連線中的攝影機會被列在彙總回應的 `connecting` (`serial`、`phase` — `opening the camera`、`rebalancing the pool bandwidth`、 `loading its calibration`、`starting its stream`、`waiting for its stream` — 以及 `elapsed_s`)，且其 `status()` 會傳回 `connecting: True`，並附帶其 `phase`； 其餘攝影機在幀池空閒時會即時讀取，否則則採用上次讀取的值（`live: False`）。

#### `CameraSession` 方法

| 方法 | 說明 |
| --- | --- |
| `status(timeout=15.0)` | 攝影機的即時狀態，由 `chloros-cli lattice status` 讀取： `streaming`、`stream_state`（`starting`、 `no_frames`、`streaming`、`start_failed` 或 `stopped`）、`device`（型號、序號、韌體、溫度、`is_mono`）及其當前設定。 |
| `read_nodes(names, enum_names=(), timeout=30.0)` | 讀取 GenICam 節點；在自動曝光執行期間（詳見下文），會傳回 `{nodes, errors, enums, device, was_in_pool, in_array_id}`、 以及 `frame_values` 和 `notes`（在自動曝光執行期間，詳見下文）。 |
| `set_settings(**kwargs)` | 根據友善名稱寫入節點（`exposure_time`、`gain`、 `pixel_format`、`width`、`height`、 `target_brightness`、`target_brightness_auto`、`ae_damping`、`ae_upper_limit`、`trigger_mode`、 `trigger_source`, …)。`exposure_time`（`gain`）為手動曝光（增益）：若未標註 `exposure_auto` (`gain_auto`) 時，需先將自動模式關閉，以免該數值被正在運行的自動模式拒絕。結果 `applied` 保存了相機所拍攝的內容； 相機拒絕的寫入值則存於 `errors` 中，且不會被保留。錯誤訊息會標明相機當前可接受的範圍（`ExposureTime` 的最大值取決於相機的幀率）， 且僅針對被拒絕的數值而關閉的自動模式將被恢復； `restored` 會列出該範圍（例如 `{"ExposureAuto": "Continuous"}`）。 |
| `capture(output_dir="output", ext=".tiff", jpeg_quality=95, processing=None, levels=None, force_daq=None, daq=None, daq_tolerance_ms=None, settings=None, timeout=None)` | 擷取 **單一** 幀。回傳一個 `CaptureResult`：包含幀元數據字典的清單（每儲存一個層級一個），其中包含 `.skipped`、`.failed`、 `.daq_files` 及 `.daq_matches`。（連拍／多幀擷取功能已移除 — 若需擷取系列畫面，請在迴圈中呼叫 `capture()`。） |
| `capture_repeated(output_dir="output", *, count=None, duration_s=None, interval_s=0.0, on_capture=None, **capture_kwargs)` | 從這台攝影機進行連續（`interval_s=0`）或間隔擷取， 針對 `count` 擷取及／或 `duration_s`（至少需執行一次，方能結束），其運作方式與 `ArraySession.capture_repeated()` 針對陣列的處理方式相同。 返回 `CaptureResult` 的清單；當有擷取報告硬碟已滿時即停止。`on_capture(i, result)` 會在每次擷取後執行；其他關鍵字則轉至 `capture()`。 |
| `disconnect(keep_settings=False, clear_settings=False)` | 從池中釋放，將攝影機恢復至連線時發現的狀態（觸發線路、PTP、設定）；`keep_settings=True` 則將設定儲存至相機自身的記憶體中，`clear_settings=True` 則重置該記憶體（參見 [相機記憶體](#camera-memory--keep_settings--clear_settings))；當任一項被要求時，即返回 `camera_memory`。若相機為陣列成員，則會釋放整個陣列。若已連接至已開啟的會話，則不執行任何操作。 當後端確認釋放完成，或無需釋放的項目時（攝影機已不存在，或無後端正在運行），即返回；若釋放操作超時或遭後端拒絕，則會拋出 `ChlorosConnectError` 錯誤，因為該攝影機可能仍被佔用，而下一次連線將與之發生衝突。`with` 區塊的退出程序絕不會拋出錯誤。 |

屬性：`serial`、`device`（攝影機的發現記錄）以及 `backend_url`。

`capture()` 匯出控制（與陣列 + GUI 相同模型）：

- `processing` / `levels` — `processing="all"` 會儲存所有適用的匯出類型；`levels=["raw","radiance"]` 僅儲存其中部分（會覆寫 `processing`）。若要採用後端預設值，請省略這兩項。
- `daq` — 用於讀取擷取資料的 DAQ 預設會以 `.daq` 附檔形式儲存，其中包含僅限原始資料的擷取內容，以便日後可將該幀重新處理為反射率／折射率。 當沒有任何級別使用 `daq=False` 時，系統會跳過它。`force_daq=True` 是預設值的舊拼寫。若無 DAQ， 亦無 `.daq`。
- `daq_tolerance_ms` — 讀數與曝光時間的間隔範圍內仍可處於容差範圍內（預設值 500）。僅判定結果會隨之改變。

#### 自動曝光下的 `read_nodes()` — `frame_values` 和 `notes`

當 `ExposureAuto` (`GainAuto`) 執行時，`ExposureTime` (`Gain`) 節點讀回的是 **最後寫入的值**，而非相機當下的實際狀態。`nodes` 仍保留該節點的值。 當您對一臺已連接且自動模式非 `Off` 的攝影機查詢 `ExposureTime` 或 `Gain` 時，回覆中會額外包含：

- `frame_values` — 攝影機最新幀所攜帶的數值，以該節點名稱標示（`ExposureTime` 以微秒為單位，`Gain` 以分貝為單位）；
- `notes` — 針對每個此類節點，各有一句包含上述兩個數值的訊息。

```python
r = cam.read_nodes(["ExposureTime", "Gain"])
print(r["nodes"]["ExposureTime"])                  # last value written, e.g. 2000.448
print(r.get("frame_values", {}).get("ExposureTime"))  # what the newest frame used, e.g. 19600.0
for name, note in r.get("notes", {}).items():
    print(note)
# ExposureAuto is Continuous: ExposureTime holds the last value written (2000.448);
# the camera's newest frame was at 19600 us.
```

當自動模式為 `Off` 時，這兩項鍵皆不存在（此時該節點即為攝影機的的實際數值），或當相機未連接至後端時，這兩項鍵均不存在。對於陣列，`ArraySession.status()["member_ae"]` 會一次提供每個元素的幀值。

#### DAQ 測光曝光上限（HTTP）

當連接 DAQ 光感測器時，Chloros 會根據測得的光線強度，對相機的曝光 × 增益設定上限，以確保白色參考板不會過曝。 此上限僅在該相機視野內有校準目標時生效（即過去 10 秒內曾出現在其畫面中）；若視野內無目標，則改由相機的普通自動曝光（附帶其自身的亮部保護功能）運作，以避免場景其餘部分被不必要地壓暗。 此功能無專用SDK封裝函式；請透過 `GET` / `POST /api/camera/<serial>/daq-exposure-cap` 針對各相機讀取或切換此設定（POST 請求正文中的 `{"enabled": false}` 可為該相機關閉此功能）。 回應中包含 `enabled`、`cap_us`（即時上限值；當功能處於非活躍狀態時，包含未偵測到校準目標時，則為 `None`）， `k_exp_g`（用於計算上限的常數）以及：

| 欄位 | 含義 |
| --- | --- |
| `k_exp_g_source` | 正在使用的常數來源：`camera` — 此相機自身的常數，由其校準包中的實驗室點計算得出 （RGB 相機）；`fleet` — 機隊常數，用於校準包中無實驗室點的情況；`override` — 在後端設定的欄位覆寫值。 |
| `k_exp_g_fleet` | 機隊常數，供比較之用。 |

### 同步陣列 — `ArraySession` (Smart-Prep)

`connect_array` 是多攝影機配置的 **建議起始點**。它在後台執行完整的 GUI Smart-Prep 流程：

1. **網路分析**(`/api/camera/array/recommend`) — 設定**完整感測器** （無像素合併、無 ROI 裁切），並讓幀率降至該主機與網路所能提供的水準，此數值會根據 Connect 將使用的像素格式與封裝方式進行預估。系統絕不會為了維持幀率而縮小影像幀：傳入較小的 `width` / `height` 或 `binning=2` / `4` 參數，以犧牲解析度換取幀率（該終點的的 `alternatives` 清單中列出了這些選項及其預估的 fps)。
2. **層級自動選擇** — `sim-capture-sim-emit`（所有攝影機在同一觸發信號下曝光，發射節奏經調控以共享傳輸線，當共享空間縮減時幀率隨之下降），適用於任何可透過節奏調控使攝影機共享傳輸線的網路； `slip-emit-and-capture`（攝影機輪流使用通訊線路，不會同時進行曝光）僅在無法共享時啟用。
3. 預設 **啟用 PTP**— 跨相機的時間戳記會以**約 1 毫秒**的精度對齊至單一共享時鐘，後端 [PTP 主時鐘](#ptp-grandmaster--time_sync_status)。同步曝光源自 M8 硬體觸發器（**< 100 µs** 模組間延遲），而非來自 PTP：PTP 僅對齊 *時間戳記*，而非曝光時機。
4. **每台相機的像素格式自動選擇** — 單色 → `Mono12`，彩色 → `BayerRG12`。
5. **自動曝光（AE）初始化** — 快照各相機當前的 AE 狀態，以確保連接時不會在運作中重置曝光。
6. **GPIO 觸發設定** — `connect_array` 將每台相機設為就緒狀態（`TriggerMode=On`、`TriggerSource=Line2`），以便主機的的脈衝能透過 M8 纜線驅動從機。此步驟僅適用於陣列模式：若僅開啟單一攝影機並執行 `LatticeCamera`，該攝影機則會改為自由運行模式。釋放陣列（`disconnect()` / `force_disconnect()`、`chloros-cli lattice array-disconnect`）將所有攝影機重設為自由運行模式（`TriggerMode=Off`），其餘設定則維持連接時所偵測到的狀態：觸發器與同步線設定 （`TriggerSource`、`TriggerActivation`、`TriggerOverlap`、`TriggerDelay`、 `LineMode` / `LineSource`；主機的同步線恢復為輸入模式），PTP，以及該工作階段所變更的相機設定——除非 `keep_settings=True` / `clear_settings=True`（參見 [相機記憶體](#camera-memory--keep_settings--clear_settings)）。

```python
import chloros_sdk

# First serial is the MASTER (fires the trigger pulse); rest are slaves.
with chloros_sdk.connect_array(
        ["213800234", "214000533", "214701288", "214701292"]) as arr:
    print(arr.array_id, arr.sync_mode, arr.ptp_enabled)
    arr.capture("output/", processing="reflectance")
```

屬性：`array_id`、`master_serial`、`slave_serials`、`ptp_enabled`、 `sync_mode`、`packed_transport` 及 `packed_reason` （最後四個為 `None`，其句柄源自 `attach_array`）、`connect_stream` 及 `backend_url`。 `connect_stream` 是連線回應的 `stream` 區塊，即單一連線的形狀加上 `members` (`{serial: {state, why?}}`)：當陣列中的第一個同步群組抵達時（後端會等待該群組，有上限，約 10 秒）， 或當每個成員的最後一次擷取失敗作為其 `why` 時，即為 `no_frames`；至於巨型封包， `note` 中的 1500 位元組備用值。`None` 位於 `attach_array` 處理程序上。

#### `connect_array()` 簽名

```python
connect_array(
    serials,                              # list[str]; serials[0] = master
    *,
    line="Line2",                         # GPIO sync line: Line0 | Line2 | Line3
    target_fps=None,                      # pin the master trigger rate (auto if None); see set_target_fps
    force_tier=None,                      # override tier picker; see below
    wire_ceiling_mbps=None,               # host sustained wire budget, MB/s (auto if None)
    width=None,                           # explicit frame size; skips network analysis
    height=None,
    pixel_format=None,
    binning=None,
    recommend=True,                       # set False to skip the recommend step
    ptp_enable=True,                      # set False to disable PTP
    packed="auto",                        # packed 12-bit transport: "auto" | "on" | "off"
    backend_url="http://127.0.0.1:5000",  # same IPv6-avoidance default as connect_camera
    timeout=600.0,                        # s; also the backend's deadline (see below)
    auto_start_backend=True,              # spawn a local backend if none is running
) -> ArraySession
```

`timeout` 表示呼叫等待連線的時間長度，並作為
截止時間傳送至後端。 若屆時仍未建立連線，則會撤銷該操作：所有已開啟的攝影機
將恢復至被發現時的狀態並關閉，所有陣列皆未建立連線，且
該呼叫會拋出 `ChlorosConnectError` （HTTP 504 `CLIENT_DEADLINE_PASSED`，標示
連線所達到的步驟）。因此，觸發此錯誤的呼叫絕不會留下
。一個包含 9 台攝影機的陣列約需一分鐘完成連線；在
攝影機開機後，此過程可能需要數分鐘。已啟動但尚未擁有
同步幀群的陣列會被返回，並拋出 `connect_stream["state"]`
`no_frames`。後端日誌中，每個連線步驟
（`[array-connect <id>] <step>: <s> s …`）各有一行時間記錄，最後一行則標示最慢的步驟。

`force_tier` 值：
- `"sim-capture-sim-emit"` — 真正同步（所有攝影機在同一時鐘邊沿觸發）。
- `"sim-capture-ftd-stagger"` — 彈性時域交錯（各凸輪在略微錯開的時間點發射，使封包在傳輸線路上串行化）。
- `"slip-emit-and-capture"` — 各凸輪依序擷取 （無時間同步）。僅當無法將攝影機速度調低至足以同時共用傳輸線時，才會自動選用此模式。

`wire_ceiling_mbps` 會覆寫 **主機的持續傳輸線預算**（單位為 MB/s）—— 這是
整個陣列資源分配所依循的單一數值。 保留為 `None` 即可使用自動偵測的
數值。當陣列回報 GVSP 損毀幀時，請降低此數值： 自動值是根據
網卡通報的鏈路速率推算而來，但此數值會高估 USB 轉接器、帶寬有限的 PCIe 通道以及
繁忙的共用互連結構的實際效能——而這種高估會以損壞幀的形式顯現，而非
明顯的鏈路速度變慢。 該值會儲存於專案的陣列擷取區塊中，因此
重新開啟專案或日後設定 `connect_array` 時，便會像其他陣列設定一樣恢復此值。
請參閱 [陣列健康狀態](#array-health--which-subsystem-is-losing-frames)。

#### 壓縮式 12 位元傳輸 (`packed=`)

當感測器全速運作時，陣列的幀率取決於每台攝影機傳送
幀的速度。12 位元格式（`BayerRG12`、 `Mono12`）通常以 16 位元
字元傳輸；緊湊傳輸則將相同的 12 位元資料以每像素 1.5 位元組傳送，並
在主機端進行解封裝，因此每個幀的傳輸時間減少 25% — 約
**在感測器全速運作時每秒可多傳輸**33% 的幀數**（實測：9 台相機陣列上從 10.7 → 14.0 fps）。**所節省的資料內容完全相同**：數值相同、16 位元
檔案相同、元資料相同。

- `packed="auto"`（預設）僅在壓縮有助益 **且** 主機能
  負擔解壓縮作業時才會進行壓縮：連線時，後端會測量自身的解壓縮速度、
  CPU 核心數、當前負載及網路頻寬預算。 若主機核心資源充裕，
  則在全感測器模式下進行封裝；若主機繁忙或功耗受限（如單板電腦、
  嵌入式控制器），則以未封裝形式串流傳輸。小型感測區域（ROI）及二進位幀的效益甚微。
- `packed="off"`（或 `False`）絕不進行封裝 — 用於在診斷時排除此選項，
  或適用於必須將 CPU 資源保留給其他工作的主機。
- `packed="on"`（或 `True`）會始終對 12 位元格式進行封裝。
- 任何其他值都會觸發 `ValueError`。8 位元和 16 位元格式絕不會
  被壓縮。

`analyze_array_network(..., packed="auto")` 採用相同的決策邏輯，
因此其 fps 即為陣列的執行速率（回應中包含
`packed_transport` 與 `packed_reason`）； 其 `pixel_format` 預設值為
「連接」所套用的設定（Mono12 / BayerRG12），而非 8 位元。

該決定及其理由記錄於工作階段中：

```python
arr = chloros_sdk.connect_array(serials)          # packed="auto"
print(arr.packed_transport, arr.packed_reason)
# True  auto: projected 11.0 -> 14.4 fps, unpack 0.16 of 1.50 budgeted cores

arr = chloros_sdk.connect_array(serials, packed="off")
```

`ChlorosProject.connect_all(packed=...)` 會將相同的覆寫設定套用至專案中的每個
陣列（預設為 `None`，這表示 `"auto"`：專案不會為每個陣列儲存
壓縮設定）。
若要設定全機台預設值，請在
後端環境中設定 `CHLOROS_ARRAY_PACKED12=off`（或 `on`）； 但顯式的 `packed="on"` / `"off"` 設定仍會優先。
`/api/camera/array/status` 會回報 `packed_transport` 及 `packed_decision`。

#### 預期幀率

測量到的陣列幀率（硬體同步，12 位元；完整感測器 = 2048 × 1536）。 請利用這些數值對 `status()["actual_fps"]` 進行合理性檢查；`analyze_array_network()` 則會根據您的主機與網路環境推估幀率。

| 設定 | 陣列幀率 |
| --- | --- |
| 全感測器、PCIe 10 GbE 主機網路介面卡、封裝傳輸（`packed="auto"` 於桌上型電腦） | ~14 fps |
| 全感測器、PCIe 10 GbE 主機網路卡、未封裝傳輸（`packed="off"`） | ~10.5 fps |
| 全感測器、8 台攝影機連接至 USB 10 GbE 轉接器 | ~3.8 fps |
| 全感測器模式，1 GbE 主機連結 | 1.2–1.7 fps |
| 2 × 2 像素合併 (`binning=2`)，10 GbE 主機網路介面卡 | ~30–33 fps |

- 在 PCIe 10 GbE 連線下，全感測器模式下的瓶頸在於 **攝影機** 本身，而非網路：4 台與 8 台攝影機的運作速率相同。
- **`target_fps` 為觸發速率，陣列運作速度絕不會超過此值。** `target_fps=1` 會將陣列運行速度設定為 1 fps；`None`（預設值）則會將其運行速度設定為主機與網路所能承載的最高速率。若某個引腳的速率高於此限，則會被調降至該限值 （參見 [`set_target_fps`](#interval-capture-for-drone-surveys)）。
- **`link_speed_mbps` 中的 `4294` 並非鏈路速率。** 這表示 Windows 無法讀取網卡的速度（驅動程式回報了最大的 32 位元值），這是 USB 以太網路網卡的常見現象。無論如何，USB 網卡在 Windows 上的可用速率上限為 200 MB/s（參見 [陣列健康狀態](#array-health--which-subsystem-is-losing-frames))；若要進行調校，請參閱 [主機 NIC 設定與調校](cli-reference.md#host-nic-setup--tuning-lattice-arrays) 中 CLI 參考手冊的「USB 10GbE 網卡」章節。
- **Linux 系統同樣受 USB 速率上限限制：x86_64 平台為 400 MB/s，NVIDIA Jetson 平台為 200 MB/s。** 網卡是透過 sysfs 識別的（若無法識別則回退至 `ethtool -i`），因此 USB 5 GbE 或 10 GbE 網卡的傳輸速率會以該上限為準，而非其標稱的 500 / ~1060 MB/s。 x86_64 平台的數值是在 Linux 上測得的：同一款在 Windows 上超過 200 MB/s 時會導致幀資料損毀的網卡，在 Linux 上運行 4 台攝影機陣列時，即使傳輸速率達到 480 MB/s 的上限，仍能保持資料完整無損。 在 NVIDIA Jetson 上，同一款網卡僅在 200 MB/s 時能保持無誤，超過此速度後會間歇性地丟失幀。若您的主機在未使用 `gvsp_corrupt` 幀的情況下能處理更高流量， 請透過 `wire_ceiling_mbps` 提高傳輸速率。在 Linux 系統中，連接至 USB 2.0 埠的介面卡預設傳輸速率上限為 30 MB/s；請將其插入 USB 3.0 埠。

#### 超額訂閱（每台攝影機的下限）

陣列可在任何網路環境下運作；幀率則取決於實際情況。 「模擬發射」調速機制會將碰撞安全的線路預算均分給每台攝影機，且所有攝影機仍會於同一觸發時刻輸出影像；因此，在低速線路上，每台攝影機的分配量——以及幀率——都會很低：8 台完整的解析度的 12 位元攝影機，在未啟用巨型幀的 1 GbE 環境下，每台可獲得 6.5 MB/s，約 **1 fps**（壓縮後約 1.3 fps）。此分配量設有下限，即攝影機可被調速至的最低速率 (`per_cam_floor_bps`：連線時從每台相機的 GevSCPD 範圍讀取，且絕不低於 1 MB/s)。 若線路要求傳輸速度低於此值，連線將回退至**串列**傳輸模式 (`slip-emit-and-capture`)：相機將輪流以全鏈路速率傳輸，因此每個幀皆能送達，但各成員的資料**不會同時曝光**— 回應的層級與阻塞器 (`SERIAL: ...`) 會明確標示此情況；傳遞 `force_tier="sim-capture-sim-emit"` 可拒絕此退而求其次的模式。幀大小不影響資源共享，因此**幀合併與 ROI 雖能提高幀率，但不會改變陣列是否能容納**。唯有當線路無論以何種傳輸節奏（同時或輪流）皆無法傳輸該陣列時，才會發生**超訂閱** —— 故障模式為 GVSP 封包遺失，而非幀率降低。解決方案：減少相機數量、端到端使用巨型幀，或採用更快的網路介面卡。

- `analyze_array_network()` 和 `/api/camera/array/connect` 的回應中包含 `oversubscribed`、`aggregate_demand_bps`、`collision_safe_ceiling_bps`、`max_cams_collision_safe` 以及 `per_cam_floor_bps`。當 `oversubscribed` 為真時，投影會 **將 fps 欄位設為零**（`achievable_fps_max` / `fps_bright` / `fps_dark`），而非回報一個誤導性的「速度緩慢但仍可運作」的幀率。
- `POST /api/camera/array/connect` 接受 `pin_resolution` 正文參數（**僅限 HTTP — 並非 SDK 關鍵字參數**；`connect_array` 未公開此參數）。此為嚴格模式：若設定了 `pin_resolution` 卻發生超額訂閱的連線，系統將**直接拒絕** 並顯示錯誤訊息，列出所有可行的解決方案。 若未設定此參數，連線將繼續進行，並發出陣列將丟失封包的警告。無論何種情況，連線都會保留所請求的幀大小 — 絕不會自行進行分桶或裁切。
- 僅供測試使用： 請在後端環境中設定 `CHLOROS_ARRAY_ALLOW_OVERSUBSCRIBED=1`，將拒絕行為降級為明顯的警告 — 您仍可建立連線，並接受封包遺失的情況。

#### 陣列健康狀態 — 哪個子系統正在遺失幀

`CHLOROS_ARRAY_ALLOW_OVERSUBSCRIBED=1` (`GET /api/camera/array/<array_id>/capability`) 在已連線的陣列上攜帶一個有效的 `health` 區塊，
並於 **10 秒** 的滾動 視窗內進行滾動重新評估。它將幀丟失
區分為兩種需要相反解決方案的原因，而非僅以一個未明確指明原因的「不完整」率
來概括：

| 欄位 | 含義 | 涉及子系統 |
| --- | --- | --- |
| `gvsp_corrupt_rate_pct`（按序列號） | 幀 **已送達但結構異常**— GVSP 封包遺失。 |**網路**：線路預算、步調控制、NIC 接收環、MTU |
| `never_arrived_rate_pct`（依序列號） | 幀**根本未送達**— 攝影機未觸發，或無資料傳出。 |**觸發／同步**：M8 纜線、`line=`、`TriggerMode` |
| `worst_gvsp_corrupt_pct` / `worst_never_arrived_pct` | 每台攝影機的最低傳輸速率 。 | — |
| `per_cam_rate_pct` | 每台攝影機的綜合不完整率（兩項原因合計）。 | — |
| `stable_for_seconds` | 每台攝影機維持在 0.01% 以下的時間長度。 | — |

與 `health` 並列，同一筆記錄還報告了整個配額的懸而未決數量：

| 欄位 | 含義 |
| --- | --- |
| `wire_ceiling_mbps` | 主機當前生效的持續線路預算，MB/s。 |
| `wire_ceiling_source` | 該數值的來源（文字說明）——例如 `USB-capped 200 MB/s (was theoretical 1062; …)` 或 `user override 120 MB/s (auto said 200)`。 |
| `wire_ceiling_is_user_set` | 當由 `wire_ceiling_mbps=` 設定時，此值為 `true`。 |
| `nic_is_usb` | 對於 USB 以太網轉接器，則為 `true`。 |

```python
import chloros_sdk

serials = ["213800234", "214000533"]           # master first
arr = chloros_sdk.attach_array(serials)
h = arr.capability()

health = h.get("health", {})
print("wire ceiling:", h["wire_ceiling_mbps"], "MB/s", h["wire_ceiling_source"])
print("corrupt (network) :", health.get("worst_gvsp_corrupt_pct"), "%")
print("absent  (trigger) :", health.get("worst_never_arrived_pct"), "%")

if (health.get("worst_gvsp_corrupt_pct") or 0) > 1.0:
    # Network path. Reconnect with a lower budget -- NOT a lower target_fps.
    # force_disconnect(), not disconnect(): this handle came from attach_array,
    # and disconnect() never tears down an array the session only attached to.
    # It returns only once the backend confirms the release; a teardown that
    # timed out or was refused raises ChlorosConnectError instead, so the
    # reconnect below never races a half-released array.
    arr.force_disconnect()
    arr = chloros_sdk.connect_array(serials, wire_ceiling_mbps=120)
```

**解讀方式：** 若 `gvsp_corrupt_rate_pct` 值不為零，且 `never_arrived_rate_pct` 為 0，表示
觸發與纜線同步狀態完美，100% 的訊號損失源自網路路徑 — 請將
`wire_ceiling_mbps` 並重新連線。反向模式則指向同步纜線或
觸發線。

> **`target_fps` 並非解決幀損壞問題的關鍵。** GevSCPD 節拍器參數僅在
> 連線時設定一次，因此降低觸發率會改變占空比，而非
> 同時發射的突發速率。經測量，將需求削減 5 倍並未帶來改善，而
> 將線路上限從 240 MB/s 降至 200 MB/s 後，同一套測試設備的損壞率從 10.4 % 降至
> 0.00 %。

> **在 LATTICE (M3C / M3M) 韌體下，正在運行的陣列無法自行重新規劃。** 它無法
> 自行解決此問題；請斷開並重新連接，以便連線時間選擇器根據
> 新的上限進行重新規劃。

**USB 以太網適配器** 會被探針限制在 200 MB/s，無論其
標示為何：將鏈路速率轉換為持續傳輸速率的效率表是
基於 PCIe 衍生而來，而 USB 網卡雖會廣播其乙太網路鏈路速率，但實際傳輸速率受限於
USB 匯流排及其驅動程式。 此上限為絕對值，而非比例值——USB 1 GbE 轉接器
可達到約 80 MB/s，且不受此限制影響。此限制適用於 Windows 和 Linux（包含 Jetson）；
請參閱 [預期幀率](#expected-frame-rates) 以了解測量位置。

#### `ArraySession` 方法

| 方法 | 說明 |
| --- | --- |
| `status(timeout=10.0)` | 執行中陣列的即時狀態：`target_fps` / `target_fps_auto`（觸發率及率控制器是否擁有該陣列）、`target_fps_clamped_reason`（為何固定率被維持在較低值，否則為 `None`）、 `actual_fps`（測量值；一旦抓取迴圈放棄或停滯，則為 0），`recommended_fps`，`streaming` （一旦抓取迴圈放棄即為 False — 請檢查此值，而非僅檢查 `last_error`； 在設定重啟後仍保持為 True，此狀況由 `restarting` 標示），`stalled`（當抓取迴圈正在運行，但約十個幀週期內未發布任何幀群組時為 True，時間約為 2–30 秒； `last_error` 僅在 60 秒後才會如此標示)，`last_frame_age_s`、`last_error`、`serials`， `ptp_enabled`（`True` 僅在陣列透過 PTP **且** 每個成員上次讀取 `PtpEnable` 時處於開啟狀態；這與「鎖定」狀態不同，後者由 `ptp_epoch` 進行測量），`ptp_configured`（連接請求所要求的狀態）、`ptp_off_members` / `ptp_unread_members`（最後讀取 `PtpEnable` 為關閉狀態 / 未回應的成員）、`member_recovery` （`None`，或已斷電或連線中斷的成員：`state`、`waiting` 雖未回應，`reopening`， 接著是 `recovered`，以及 `members` 和 `why` —— 當該節點再次回應時，陣列會重新開啟其連線，並恢復其連線時的 PTP、 觸發器與同步線接線、節奏控制及相機設定，隨後再次進行串流（無新的 `connect_array`），以及 `ptp_epoch`（測量值）、`sync_mode` / `sync_tier`、`packed_transport` / `packed_decision`、`per_cam_emit_mbps`、`link_speed_mbps`（`4294` 表示 Windows 無法讀取網卡的速度 — 請參閱 [預期幀率](#expected-frame-rates))，`trigger_pipeline` / `pipeline_tune`、`radiometric_chain`（表示各頻段輻照度鏈是否可用，以及是哪個成員導致其中斷），以及 `band_clamp` ——同時透過即時預覽渲染陣列：各成員中，解混過程被限制為 0 的受光像素比例 （`members`：依相機 R、G、B 波段順序排列的 `{serial: [band0, band1, band2]}`），並按各波段顯示複合影像 （`composite`：`{band name: share}`），其中 `warn` 為 GUI 標記為「低可信度波段資料」的比例 （約每秒測量一次；當未執行即時預覽時為 `None`），以及 `member_ae` — 針對每個成員，其最後發布的幀所攜帶的曝光值（`exposure_us`）與增益 （`gain_db`）——這些資料取自該幀影像本身（`read_nodes` 為 `ExposureTime` / `Gain` 會回傳自動曝光執行期間寫入的最後一個值，而非當前狀態）。 |
| `set_member_settings(serial, **kwargs)` | 寫入某台成員相機的設定，由 `CameraSession.set_settings()` 將其 （曝光、增益、目標亮度等）；該成員仍保留在陣列中。若序列非陣列成員，則觸發 `ChlorosConnectError` 錯誤。參見 [陣列成員的曝光設定](#exposure-on-array-members)。 |
| `capability(timeout=10.0)` | 智慧預設功能所確定的參數（擷取層級、幀大小、`pixel_format`）、即時 `health` 區塊以及主機的線路預算。 請參閱 [陣列健康狀態](#array-health--which-subsystem-is-losing-frames)。 |
| `set_target_fps(fps, timeout=10.0)` | 在陣列運行期間變更觸發率，無需重新連線。特定數值會鎖定該設定（率控制器不會干預）； `None` 將其恢復為自動模式。當傳輸速率降低至主機與網路可承載的範圍時，`{requested, target_fps, target_fps_auto, clamped}` — `clamped` 會返回 True。 請參閱 [無人機測量之間隔擷取](#interval-capture-for-drone-surveys)。 |
| `grab(processing="debayered", with_metadata=False, apply_calibration=True, apply_white_balance=True, apply_index=False, index_expression=None, bit_depth=None, band=None, align=False, refl_max=None)` | 從每台相機擷取一幀影像，**均由單一觸發訊號控制**，以 numpy 陣列形式輸出（不寫入檔案）。回傳 `GrabResult`（一個 `dict`）：`{serial: array}`，或 `{serial: (array, metadata)}` 搭配 `with_metadata=True`。 `.group_id` 是這些幀共用的觸發器；`.errors` 將每個被排除的攝影機映射至原因（無法在該觸發器下運作的攝影機將被省略，絕不會由其他攝影機填補）。 `processing` 等同於 `raw` / `debayered` / `radiance` / `reflectance` / `display`。此讀取方式與專案 API 的 [`ArrayHandle.grab()`](#one-trigger-per-read--group_id) —— 一個共用實作 —— 因此那裡的每個關鍵字（`band`、`align`、 `bit_depth`、`refl_max`) 皆可在此處運作。需使用 `numpy`。 |
| `frame_stream(*, processing="debayered", fps=5.0, duration=None, count=None, apply_calibration=True, apply_white_balance=True, apply_index=False, index_expression=None, with_metadata=False, bit_depth=None, band=None, align=False, refl_max=None)` | 每 tick 產出一個 `grab()` 結果的迭代器（僅限關鍵字參數），步進速度設定為 `fps`， 並受 `duration`（秒）和／或 `count` 限制（`None` 表示無限制）。**每個產出的集合皆為一個新的觸發器：** 若某個時標的最新觸發條件正是其已產出的集合，則該時標會等待下一個觸發條件而非重複執行，因此資料流將以 `fps` 與陣列自身幀率中的較慢者運行。此時會觸發 `FrameFetchError` 錯誤，而非保持靜默： 當所有攝影機因某種原因拒絕下一個 tick 會重複的情況時，在連續 60 秒的 tick 沒有幀的情況下，以及在 60 秒內沒有新觸發器（「60 秒內無新觸發器」）的情況下，會立即拋出 `FrameFetchError` 錯誤，而非保持靜默： 與 `ArrayHandle.frame_stream()` 實作相同。請參閱 [在您自己的處理流程中使用即時幀](#live-frames-in-your-own-pipeline)。 |
| `continuous(output_dir="output", *, format="tiff", processing="debayered", levels=None, aligned=None, render_index=None, force_daq=None, orient=None, max_captures=None, duration_s=None, daq=None, daq_tolerance_ms=None, timeout=30.0)` | 啟動後端自身的連續擷取（即 GUI 的「連續模式」）→ [`ContinuousHandle`](#continuoushandle)。會儲存陣列所發佈的 **每個** 同步群組，其儲存方式與 `capture()` 相同（採用相同的匯出控制），直到被停止，或直到 `max_captures` / `duration_s` （`None` = 直到停止為止）。若該陣列已有同名程序正在執行，則會拋出 `ChlorosConnectError` 錯誤。 |
| `capture(output_dir="output", format="tiff", processing="debayered", levels=None, aligned=None, render_index=None, force_daq=None, orient=None, smart=False, after_seq=None, daq=None, daq_tolerance_ms=None, timeout=300.0)` | 一個同步的擷取群組。回傳一個 `CaptureResult` （幀字典清單 + `.skipped` + `.failed` + `.group_seq` + `.daq_files` + `.daq_matches` + `.alignment` + `.oriented`)。將前一個結果`group_seq` 作為 `after_seq`，以取得其後接續的下一個同步群組，而非等待新的群組。 以下為匯出控制選項。 |
| `capture(..., smart=True)` | **智慧擷取** — 等待所有攝影機的 AE 值趨於穩定後，再觸發。 |
| `capture_fastest(output_dir="output", force_daq=True, render_index=True, orient=None, timeout=120.0)` | 最快擷取：僅原始資料 + 已指派的 DAQ 讀數 （+ 自由組合索引）。功能與 GUI 中的「最快擷取」按鈕相同。 |
| `capture_repeated(output_dir="output", count=None, duration_s=None, interval_s=0.0, on_capture=None, **capture_kwargs)` | 在單一有限迴圈中執行單次／連續／間隔擷取。 返回 `list[CaptureResult]`。**需搭配 `count` 及／或 `duration_s`** 方能終止（SDK 沒有 Ctrl+C 功能）。此外，若某次執行階段回報驅動器已滿（即出現 `.failed` 條目搭配 `disk_full`），則該執行階段將提前結束， 當某個處理階段回報磁碟已滿（即包含 `disk_full` 的 `.failed` 條目）時，該處理階段即為最後結果。 |
| `record(output_dir="output", fps=10.0, duration_s=None, video=True, gif=False, expr=None, timeout=30.0)` | 開始將即時組合索引檢視錄製為影片／GIF → `RecorderHandle`。影片為 Motion-JPEG 格式 `.avi` （無需 ffmpeg）。若未開啟任何 GUI 視圖，後端將渲染：`expr`（陣列頻道名稱中的索引表達式，例如 `"(NIR_850_RGN-Red_660_RGN)/(NIR_850_RGN+Red_660_RGN)"`），或預設為陣列的 NDVI。 每個陣列對應一個複合錄影器。 |
| `burst(output_dir="output", duration_s=None, max_frames=None, index_config=None, serial_index_config=None, orient=None, timeout=30.0)` | 啟動高幀率原始拜耳連拍 → `RecorderHandle`。使用 `build_video()` 進行離線重新處理。`orient=True`：`build_video()` 產生的產出會根據相機方向進行旋轉（各相機的產出依其自身方向旋轉，合併索引則依主相機方向旋轉）；原始幀保持原始格式，且連拍清單會記錄每個成員的方向。 |
| `build_video(burst_dir, products=None, fps=10.0, video=True, gif=False, save_tiffs=False, wait=True, poll_s=2.0, timeout=1800.0)` | 將已儲存的原始連拍檔離線重新處理為經校準的影片。 此作業將阻塞直至完成（`wait=True`），並回傳建置結果：`{success, outputs, errors, skipped, substitutions, notes, degraded, combined}` — 請參閱 [建置結果](#the-build_video-result) 。 |
| `build_video_status(job_id, timeout=15.0)` | 輪詢離線建置工作：`{running, result, error, burst_dir}`。 |
| `disconnect(keep_settings=False, clear_settings=False)` | 釋放整個陣列 — **若此會話曾開啟該陣列** (`connect_array`)：此時即為 `force_disconnect()`，且當釋放未獲確認時會拋出 `ChlorosConnectError` （`with` 區塊的退出從不拋出異常）。若處理來自 `attach_array` 的句柄，則讓陣列繼續運行，並在 stderr 輸出一行訊息說明此情況：該陣列可能屬於 GUI 或其他腳本。 每台攝影機均維持自由運行狀態，且其他設定維持連接時原狀；`keep_settings=True` 會將設定儲存至攝影機自身的記憶體中，`clear_settings=True` 則會將該記憶體重置為出廠設定（參見 [攝影機記憶體](#camera-memory--keep_settings--clear_settings))）。 |
| `force_disconnect(timeout=60.0, keep_settings=False, clear_settings=False)` | 釋放整個陣列，無論是由誰開啟的。若您意圖終止該陣列（例如為了以不同設定重新連線），請在已連接的處理程序上使用此函式。 當後端確認釋放完成，或無需釋放的項目時（陣列已不存在，或無後端正在運行），即會返回；若釋放超時或遭後端拒絕，則會觸發 `ChlorosConnectError`。無論使用哪個標誌，此函式皆會返回各攝影機記憶體所獲得的 (`camera_memory`)，否則返回 `None`。 |

`capture()` 匯出控制（與 GUI／CLI 使用的端點相同）：

- `processing` / `levels` — `processing="all"`（或 `levels=["raw","radiance",…]`）會針對每台攝影機儲存所有適用的匯出類型； 單一 `processing` 值僅儲存該層級的設定。
- `aligned` — **預設為啟用狀態（`None`），因為 Chloros 應用程式對同一陣列的擷取**（其「陣列級」**對齊**擷取設定）。每個成員的非原始匯出檔會根據陣列的 [對齊設定檔]（#array-alignment）變形至主相機的幀，並裁切至各成員的共同重疊區域，因此檔案已進行像素對齊，且輸出尺寸、裁切範圍及 `Alignment*` 標籤均與該應用程式的一致（在 ROI 範圍內，變換會重新基於視窗進行）；`XMP-mapir:AlignmentJSON` 表示 `"applied": true` 及其 `crop_roi`。 原始檔將保持原生尺寸且未經變形，但會在元資料中攜帶變換資訊。`aligned=False` 會將每個檔案以原生尺寸、未經變形的方式儲存；`aligned=True` 則是明文寫出的預設設定。`continuous()`、`capture(smart=True)`、`capture_repeated()` 以及專案層級的 `ArrayHandle.capture()` / `capture_smart()` 均採用相同的預設值。 無論是否套用變形，一旦陣列具有對齊配置檔，每個成員檔案都會記錄其相機的 `Alignment*` 變換——但若配置檔未對齊某台相機 （被省略，或因求解失敗而被迫設為單位變換），以及在沒有其他相機對齊時作為參考的相機：這些相機不會攜帶 `Alignment*` 標籤。 若陣列上沒有對齊配置檔，擷取資料將以未對齊狀態儲存，且結果會標示此狀況：`result.alignment` 即為 `{"requested": True, "applied": False, "warnings": [...]}`，每項導致變形無法套用的原因（無配置檔、成員解析度不同、陣列無法，或幀無法映射至輪廓）時，會各產生一則警告。僅當所有要求變形的匯出皆成功完成時，`applied` 才會等同於 `True`。 陣列的自動對齊功能會在曝光穩定後自動生成配置檔；`ArrayHandle.calibrate_alignment()`／CLI 的 `align-calibrate` 則可依需求手動生成。 `result.oriented` 用於標示 `orient=True` 是否已處理該產品。
- `render_index=False` — 跳過每台相機的植被指數疊加層；預設會在設定的位置進行渲染。
- `daq` / `daq_tolerance_ms` — DAQ 讀數預設會以 `.daq` 副檔形式儲存，即使所選層級並不需要此資料；此時 `daq=False` 會跳過此步驟， 而 `daq_tolerance_ms` 則設定匹配容差（預設為 500）。`force_daq=True` 是預設值的舊拼寫。請參閱 [擷取所儲存的 DAQ 讀數]（#the-daq-reading-a-capture-saves--daq_files-and-daq_matches）。
- `orient=True` — 將每個成員的 **乘積**（`display`、`radiance`、`reflectance`、各相機索引及複合影像）乘以相機方向，此操作在對齊變形後進行： 若在對齊變形時已採用**主相機**的方位進行變形（此為 `aligned` 的預設值 — 各成員仍保持共註冊），則以該方位為準；否則則採用各成員自身相機的方位。**預設：每個成員檔案皆為原生格式**，如同陣列擷取始終如此；無論何種情況，`raw` 與 `debayered` 皆維持原生格式。每個檔案皆記錄其像素所處幀的方位 (`XMP-Camera:ChlorosRotation` / `ChlorosFlip*` / `ChlorosOrientationApplied` — 原始檔，採用其自身相機的座標系），而 `Alignment*` 的 XMP 檔案則保留原始座標系變換，該變換在方向設定之前即已套用； 攜帶該變換的原始檔還會記錄 `ChlorosAligned*`（主座標系的），即變形至主座標系後產品的方位角。處理過程會根據像素所屬的變換來旋轉每個產品：當對原始檔進行變形時（陣列對齊開啟），則採用對齊後的變換；當 未對齊原始資料時，則採用相機自身的旋轉。 `capture_repeated()` 和 `capture_fastest()` 會直接傳遞該變換；專案層級的 `ArrayHandle.capture()` / `capture_smart()` 亦會採用該變換。

**TIFF 壓縮 （HTTP 專用控制項）：** `ArraySession.capture()` 不會傳送 `compression` 鍵， 因此將套用後端預設行為 — `POST /api/camera/array/capture` 讀取 `compression` 正文參數，預設情況下： 未壓縮，完全保留感光元件讀取的原始像素資料（每幀全解析度約 6.3 MB）。 `"deflate"`（無損 zlib L1 + 水平預測器，約 4.1 MB/幀）寫入的檔案較小但速度較慢 — 兩者皆為無損格式，且在匯入時讀取結果完全相同。 SDK 未為此提供任何關鍵字參數；解決方案是使用 `chloros-cli lattice array-capture --compression deflate` 或原始 HTTP。DEFLATE 會佔用 Python 的 GIL，因此壓縮寫入無法在各相機寫入執行緒間並行處理——若要以感測器速率持續進行全解析度擷取，則需使用未壓縮的預設設定。詳細資訊： [CLI 參考 → array-capture](cli-reference.md)。**連續群組（HTTP）：** 端點會傳回 `group_seq`；若在循環中將其作為 `after_seq` 傳回至下一次請求，即可取得下一個同步群組——包含在前一次擷取正在儲存時所發布的群組 ——而非等待請求後發布的群組，而 `skipped_groups` 會回報循環遺漏的任何群組。`capture_repeated`、 CLI 的 `--continuous` 以及 GUI 的「連續模式」皆採用此機制；間隔性傳輸在每個排程時刻皆保持最新狀態。**按成員進行的匯出覆寫（僅限 HTTP）：**同一端點亦接受 `exclude_serials`（清單 — 從儲存的集合中移除成員；該陣列仍會作為一個同步群組觸發，且被排除的成員會回傳至 `excluded`），以及 `serial_levels` （`{serial: [level tokens]}` 每台攝影機層級覆寫），以及 `serial_index`（`{serial: bool}` 每台攝影機索引疊加層級覆寫）。 這些是與 GUI 對應的主體參數，**目前還不是 SDK 關鍵字參數**；若映射中缺少的成員，則會回退至陣列層級的 `levels` / `render_index`。

##### 檢查被跳過的鏡頭 — `CaptureResult.skipped`

`ArraySession.capture()` 會傳回一個 `CaptureResult`，該物件是 `list` 的子類別：可對其進行迭代、索引， `len()` — 所有現有模式皆能正常運作。新程式碼可檢查 `.skipped` 屬性，以了解哪些凸輪被排除以及原因。 最常見的情況是，當您在混合濾光陣列中請求 `processing="radiance"` 或 `"reflectance"` 時，其中的 RGB 感光元件 時，混合濾光陣列中的 RGB 鏡頭便會被排除——對於寬頻感測器而言，每點拜耳（Bayer）的輻射度並無意義，因此後端會跳過這些鏡頭，而非產生無意義的資料。

```python
with chloros_sdk.connect_array(serials) as arr:
    result = arr.capture("output/", processing="reflectance")

    # Back-compat: iterate as a plain list
    for frame in result:
        print(frame["filepath"], frame["serial"])

    # New: see why N-1 cams were saved
    for skip in result.skipped:
        print(f"skipped SN:{skip['serial']} reason={skip['reason']}")
        # e.g. {'serial': '214701292', 'level': 'reflectance',
        #       'reason': 'reflectance-not-applicable-to-rgb-cam',
        #       'filter': 'RGB'}
```

原因代碼遵循 `<level>-not-applicable-to-rgb-cam` 的模式（每個被跳過的層級對應一個條目，每個條目包含 `level`）。 若相機的校準包未載入，則會以 `<level>-needs-calibration (connect the camera so its cal pack loads)` 跳過 `radiance` / `reflectance`， 若無可用的 DAQ 反射率讀數，則以 `reflectance-skipped-no-dls` 跳過該層級。其他與反射率相關的跳過情況包括：`reflectance-skipped-no-fresh-dls`（無新的下行輻射讀數可用）、`reflectance-skipped-bound-daq-unavailable (…)` （無法連通綁定的 DAQ）以及 `dls-uncalibrated-band-<nm>` — 該波段大部分位於 DAQ 光感測器的輻測校準範圍之外（約 374–974 nm），因此基於 DAQ 的絕對反射率分割被拒絕，該幀會明顯降級為感測器響應模式。在現有 SKU 中，僅 F988 會觸發此情況；該相機支援的工作流程為反射率面板工作流程。

##### 無法寫入的檔案 — `CaptureResult.failed`

即使某次擷取僅能寫入部分檔案，仍會回傳： 該清單包含所有**已**寫入的檔案，而 `.failed` 則列出其餘檔案，每個檔案對應一個字典，並以 `serial` （`None` 代表複合影像與陣列的 `.daq` 輔助檔案），`level` （`raw`、`radiance`、……、`index`、 `composite`、`daq`、 或 `match` —— 當 `.daq` 被寫入但其 sidecar 未被寫入時，其 `.match.json` 的來源）以及 `error`。在磁碟空間已滿的情況下，該條目還會包含 `disk_full: True`；此後對該磁碟的每次擷取都會以相同方式失敗，這正是 `capture_repeated()` 止步於此的原因。 部分擷取不會觸發例外狀況，且 `.failed` 中的檔案並未以半寫入狀態殘留於磁碟上。在乾淨的擷取中，`.failed` 為空。

```python
result = arr.capture("output/", processing="all")
for f in result.failed:
    print(f"SN:{f['serial']} {f['level']} not written: {f['error']}")
if any(f.get("disk_full") for f in result.failed):
    raise SystemExit("drive full -- free space and capture again")
```

##### 讀取擷取資料的 DAQ 會儲存 — `.daq_files` 和 `.daq_matches`

每當 DAQ 與相機或陣列綁定，或被納入資源池時，每次擷取都會儲存其使用的下行讀數： 位於擷取資料夾的 `daq/` 子資料夾中，其 `.match.json` 及 `.cameras.json` （即該讀數所屬的感測器）。此規則同樣適用於原始或去拜耳化擷取，且無論擷取來源是 GUI、CLI 或 SDK，檔案名稱皆相同。若擷取的電平未使用 DAQ （但 `radiance`、`reflectance` 或 `display` 級別仍會寫入該值）；`daq_tolerance_ms` 會調整該 擷取的容差線，絕不會移動實際使用的讀數。所指的 DAQ 為：綁定至相機或陣列的那個（GUI 中的光感測器選項、專案的綁定設定，或 `chloros-cli daq pool-bind`），否則即為後端所擁有的最新 DAQ。 CLI 將這些控制項標示為 `--no-daq` 和 `--daq-tolerance-ms`（[CLI 參考](cli-reference.md#the-daq-reading-a-capture-saves)）。

```python
r = cam.capture("out", levels=["raw"])      # a CameraSession; ArraySession results match
for m in r.daq_matches:                      # one per .daq in r.daq_files
    print(m["daq_file"], m["gap_ms"], m["in_tolerance"])
```

每次擷取都會將所使用的光感測器讀數儲存為 `.daq`，並附帶一個 `<file>.daq.match.json` 值，用以說明該讀數如何與曝光相關聯：`method`、`gap_ms`（從曝光到最接近的 DAQ 讀數）以及 `in_tolerance`（在 500 毫秒內，或該擷取的 `daq_tolerance_ms`）。反射率檔案的 XMP 所載的判定結果與 `Chloros:DlsMatchMethod`、`Chloros:DlsMatchGapS` 及 `Chloros:DlsInTolerance` 相同。

擷取會等待 DAQ 在曝光後的第一個讀數，以便能將其兩側的讀數進行混合（`method` `interpolated`）。 DAQ-E 在日光下每 0.4 秒傳送一次讀數，在室內則每 1 至 2 秒傳送一次，因此 `capture()` 會在約一個半讀數間隔後返回； 曝光本身並不會延遲。若 DAQ 已停止傳送讀數，系統不會等待其恢復。DAQ-E 的讀數代表自上次讀數以來該時間區間內的平均光照值，因此會被置於該區間的中間位置： `gap_ms` 會以該中點為基準進行測量，而 `.match.json`'的 `bracket_stamps_ns` 則提供讀數本身的時間戳記。`ArraySession.continuous()` 會進行等待，但不會減慢擷取速度。`capture_repeated()` 的回往返傳輸過程不會等待，因此能維持其速率，並與曝光前最接近的讀數進行配對。

讀數永不失效，因此不會因讀數延遲而遺失任何幀——取而代之的是會標記為：

- **DAQ 資料流停滯** — 系統會採用其最新讀數，且 `gap_ms` 會標示該讀數的滯後時間（`in_tolerance: false` 超過 500 毫秒）。
- **無 DAQ 可讀取**— 已斷開連接（`disconnect()`、`chloros-cli daq pool-disconnect`），或無法連通已綁定的 DAQ — 此時會重複使用相機的最後一個讀數，並針對**本次** 曝光：`method`、`retained`、`gap_ms` 來自此曝光， `in_tolerance` 經重新計算，而 `retained_from_epoch_ns` 則標示其最初配對的曝光。從未取得讀數的相機仍會顯示無反射率，且結果會說明原因 （`reflectance-skipped-no-fresh-dls` / `reflectance-skipped-bound-daq-unavailable (…)`，或擷取失敗）。

擷取畫面所儲存的檔案若位於 Chloros 中開啟之專案的資料夾內，將加入該專案的檔案清單中，如同在應用程式中進行擷取時一樣（儲存於其他位置的檔案則不受影響）： `CameraSession.capture()`、`ArraySession.capture()`（`smart=True` 亦然） 以及 `capture_cameras_transient()`。結果如 `.project_add` 所示（`capture_cameras_transient()`：字典中的 `project_add`） -- `held`、`files` 以及後端的 `message`；`held` 為 `True`，當處理執行中時，檔案會在處理結束後併入。若無檔案匯入專案，則為 `None`。

`CameraSession.capture()` 和 `ArraySession.capture()` 會將其作為結果的 `.daq_files` 和 `.daq_matches` 返回， 作為HTTP擷取回應（`/api/camera/<serial>/cli-capture`、`/api/camera/capture`、`/api/camera/capture-all-synced`、`/api/camera/array/capture`） 則包含 `daq_files` 和 `daq_matches`，每個 `.daq` 對應一個：`daq_file`、 `cameras`、`method`、`gap_ms`、 `in_tolerance`、`tolerance_ms` 以及 `stream_fault` — 此為 DAQ 自身的資料流錯誤，僅在讀數超出容差範圍時才會被查閱。在依賴擷取資料中的反射率之前，請先檢查其 `.match.json` 或 XMP 中的 `in_tolerance`，或 DAQ 的 [`status()`](#daqsensorsession-methods)。

`processing` 級別：

| 級別 | 輸出 |
| --- | --- |
| `"raw"` | 直接來自感測器的單通道拜耳（單色相機：單波段）訊號。 |
| `"debayered"` *(SDK預設)* | 3通道彩色：從原始影像中扣除校準套件的暗偏移量（DSNU），接著進行 OpenCV 的邊緣感知去馬賽克處理 — 不包含線性化 LUT、平場校正或解混 （單色相機：1 通道灰階）。 儲存的 TIFF 檔案採用 **RGB**通道順序（光度 RGB，因此任何影像檢視器皆可顯示真實色彩）；而 `grab()` / `frame_stream()` 所回傳的記憶體內陣列則採用 OpenCV 的**BGR** 順序 — 請將通道順序反轉（`frame[..., ::-1]`）以與儲存的檔案進行比對。`chloros-cli process` 寫入的結果與 `Debayered_Images` 相同。 |
| `"radiance"` | 透過完整的輻測鏈計算出的 float32 W/m²/sr/nm。僅限多光譜模式 — RGB 相機將被跳過。 |
| `"reflectance"` | uint16 0..32768（相容 Pix4D）；需搭配即時資料擷取（DAQ）配對以取得絕對參考值。僅限多光譜模式。 |
| `"display"` | 完整校正鏈，與 GUI 預覽相符（依據相機設定檔進行色彩校正、白平衡及伽瑪校正）。 |
| `"all"` | **每台相機針對每個適用層級** 產生一個檔案（與 GUI「擷取全部」/ CLI 預設值一致）。返回的 `CaptureResult` 隨後會針對每個 `(cam, level)` 包含一個幀字典，每個字典中皆包含該層級； 不適用的層級則會出現在 `.skipped` 中。用於讀取該擷取資料的 DAQ 會以 `.daq` 附檔的形式儲存。 |

> **注意 — 預設值與CLI不同。** `ArraySession.capture()` 的預設值為 `processing="debayered"`；`chloros-cli lattice array-capture` 指令的預設值為 `processing="all"`。 請從 SDK 明確傳入 `processing="all"`，以模擬 CLI/GUI 的多層級儲存功能。

##### 陣列擷取檔案名稱

每個層級皆儲存於 `output_dir` 的獨立子資料夾中（`raw/`、`debayered/`、 `radiance/`、`reflectance/`、 `preview/` 對應於 `display`、`index/`、`composite/`）。每個檔案的相機名稱格式為 `SN<serial>-<filter>`：先是序號，接著是相機型號中的濾鏡部分——例如 `SN214000533-FOCN` 對應於 `M3C-L87-FOCN`， `SN261902142-F450` 則對應於 `M3M-L87-F450` — 並與 `-` 結合， 因此檔案名稱與先前一樣，由 `_` 分隔的字段組成。 無法讀取型號的相機僅顯示為 `SN<serial>`，而早期版本的擷取檔名稱中則不包含篩選器。檔案名稱取決於系統要求會員提供多少層級的資訊：

| 要求成員提供 | 檔案名稱 | 範例 (`reflectance`，預設為 `format="tiff"`) |
| --- | --- | --- |
| **一級**（`processing="reflectance"`，或僅含一項的 `levels`） | `sync_<timestamp>_SN<serial>-<filter>.<format>` — 如同早期版本，無層級後綴，因此現有腳本仍可正常運作 | `reflectance/sync_<ts>_SN214000533-FOCN.tiff` |
| **多個層級**（`processing="all"`，或包含多於一項的 `levels`） | `sync_<timestamp>_SN<serial>-<filter>_<level>.<format>` | `reflectance/sync_<ts>_SN214000533-FOCN_reflectance.tiff` |

[鉗位遮罩](#unmix-clamp-mask--which-pixels-a-band-was-clamped-on) 緊接在產品名稱之後，`clamp_masks/<product folder>/<product name>_clampmask.tif`：單一級別的反射率擷取會寫入 `clamp_masks/<product folder>/<product name>_clampmask.tif`，而 `processing="all"` 則會寫入 `clamp_masks/reflectance/sync_<ts>_SN<serial>-<filter>_reflectance_clampmask.tif`。 後綴規則是根據各成員所請求的層級來決定的，因此跳過某個層級（例如 RGB 相機上的反射率）並不會改變它。請直接從回傳的幀中讀取路徑，而非自行建構。單一相機擷取 （`CameraSession.capture()`）總是會追加該層級：`capture_<ts>_SN<serial>-<filter>_<level>.tiff`。

##### 僅接受 PNG / JPEG 格式的顯示與索引產品

PNG 或 JPEG 每通道可儲存 8 位元。若要求其儲存超出其容量的色階，系統會在擷取任何內容前即拒絕該請求，並顯示訊息說明該色階名稱，同時指出 TIFF 為可儲存該色階的格式：

- **`radiance`**(float32) 以及**`reflectance`**(uint16, `32768` = 1.0) —— 因此 `processing="all"` 包含上述所有層級。 SDK 會針對參數進行此檢查：若未經請求便傳入 `capture()`，則會觸發**`ValueError`** 會觸發 *`CameraSession`、`ArraySession`、`CameraHandle`、 `ArrayHandle`、`capture_smart`）。
- **來自位元深度超過 8 位的攝影機所產生的 `raw` / `debayered`**（12 位元 M3C、Mono12 M3M）— 後端在知悉各攝影機的位元深度後，會拒絕這些資料： HTTP 400 狀態碼（包含 `error_code` 和 `FORMAT_CANNOT_HOLD_LEVEL`），並以 `ChlorosConnectError` 形式觸發 (`CameraSession` / `ArraySession`) 或 `RuntimeError` (`CameraHandle` / `ArrayHandle`)。單一層級的 `debayered` 擷取功能，當單一攝影機擷取 8 位元幀並儲存為 PNG／JPEG 格式時，系統不會拒絕此操作。

將輻測資料儲存為 `format="tiff"` (`ext=".tiff"`)；針對 `display` 及索引渲染，請使用 PNG/JPEG 格式。

#### 自動曝光的運作方式

- **先曝光，後增益。** 當增益設為「自動」時，增益會維持在 0 dB，直到曝光值達到允許的最大值為止。
- **手動曝光後，增益仍維持在「自動」模式**，直到您變更增益或將其設為手動為止。切回自動模式後，增益的上限將恢復至先前設定的值。
- **微小的亮部不會使場景變暗。** 佔畫面不超過 1% 的鏡面高光將被忽略；而在已知校準標靶的面板中，則可忽略佔單一面板面積不超過 0.5% 的區域。
- **數據擷取（DAQ）的曝光上限僅在校準標靶處於視野內時生效。** 此設定可防止白色面板發生剪切；若未偵測到面板，相機將根據場景進行曝光。
- **光線恆定時，曝光值與其上限保持不變；光線變化後，畫面將在數秒內穩定下來。** 在光線恆定的情況下，曝光值及其上限維持不變；當光線發生大幅變化後，畫面將在數秒內穩定下來。 過曝不再會永久限制曝光，且當感測範圍（ROI）改變時，系統會重新調整曝光值。
- **亮度目標值始終在可控範圍內。**當攝影機處於曝光極限且增益無法再提高時，亮度目標值不會上升，因此光線恢復時也不會導致畫面過曝。
- **若相機的自動曝光停止響應，系統會將其維持在當前光線條件下的曝光水準。** 在烈日下，相機可能會降至最短曝光時間並維持該狀態，畫面呈現暗黑，但所有設定數值皆顯示正常。Chloros 會監控畫面，將該相機維持在該光線條件下的曝光水準，並在光線變化時放開控制。 即使光感測器被遮蔽或處於黑暗中，也不會干擾此運作。

#### 陣列成員的曝光

LATTICE 相機將 `ExposureTime` 的上限設定為其幀率的一個週期。當陣列進行串流時，Chloros 會以略高於觸發率的速率運行每個啟用 `AcquisitionFrameRateEnable` 的成員， 且運行速率略高於觸發速率，因此單一元件的曝光上限會略低於觸發週期，**在任何 ROI 區域內**皆然：

- 單一元件的幀週期等於觸發週期減去週期 15% 的緩衝時間（至少 3 毫秒，至多為週期的一半），其曝光上限則等於該幀週期減去約 0.1 毫秒。 範例：10 fps 的觸發率（100 ms）會產生約 11.8 fps 的成員速率，以及約 85 ms 的曝光上限。
- **若手動設定的 `ExposureTime` 時間超過該上限，觸發器會放慢速度以配合該設定。**Chloros 會先將該成員的上限值設定為該數值，並讓較慢的速率運行幾個觸發週期（寫入過程可能多花幾秒鐘），然後才進行寫入， 且觸發速度絕不會超過曝光允許的上限。在自動速率陣列中，當每個因手動曝光而減慢速率的成員恢復為自動曝光後，該陣列便會每隔一個成員恢復先前運行速度。 相機本身無法接受的數值仍會被拒絕，此時錯誤訊息中會顯示其目前可接受的範圍，且若該成員的自動曝光僅因該次寫入而被關閉，則會恢復其自動曝光。在 1.3.0 版本之前，一個小型 ROI 會將每個成員的上限限制在感測器自身的自由運行週期內， 無論觸發率為何：在 768 × 304 解析度下為 4.9 毫秒，因此無法提升任何成員的亮度。
- Chloros 絕不會為了配合更快的觸發率而縮短手動曝光時間。自動觸發率會維持在手動曝光允許的範圍內；若固定觸發率快於曝光允許的範圍，該成員將保留足夠長的時間上限以配合該觸發率，且其觸發速度不會超過曝光允許的上限。當每個成員都必須響應每個觸發時，請降低曝光時間或固定觸發率。
- **讀回：** 讀取元件的節點會顯示 `AcquisitionFrameRateEnable` True 及 `AcquisitionFrameRate`（以該元件的速率顯示）；當陣列停止時，啟用狀態會重新關閉。
- **該陣列擁有這兩個節點。請勿在陣列成員上寫入 `AcquisitionFrameRateEnable` 或 `AcquisitionFrameRate`** — 請改用 `target_fps` / `set_target_fps()` 來設定陣列的速率時，請改用 `target_fps` / `set_target_fps()`。

使用 `arr.set_member_settings(serial, exposure_time=20000)` 寫入成員的曝光量（此為 `CameraSession.set_settings()` 的關鍵字；`chloros-cli lattice exposure --serial` 功能相同）。 該成員將保留在陣列中。請勿使用 `connect_camera()` 開啟陣列成員： connect 會拒絕已位於陣列中的攝影機，而它所返回的 `CameraSession` 會在斷開連接時將該攝影機從其陣列中移除。`ArraySession.status()["member_ae"]` 會回報每個成員的的曝光值與增益。

### 單一專案記錄，適用所有模式

專案資料夾儲存所有裝置的設定：`cameras.json` ——包含每台相機的設定（`cameras[].settings`，適用於單一相機及每個陣列成員），以及每個陣列的全陣列設定（`arrays[].capture`： 格式、ROI、像素合併、觸發率、線路預算）——以及 `sensors.json`，即每個光感測器的設定（積分時間、幀平均值、上限）。應用程式、CLI、SDK 和 MCP 伺服器皆以相同方式讀寫相同的檔案：

- **連接至某個專案下的裝置會被儲存至該專案中，並將每項變更記錄至其中。** 「隸屬於某個專案」意指在後端（應用程式端）開啟了該專案，或——透過 `ChlorosProject`（`open_project()`），該檔案會將其專屬資料夾名稱傳送至後端 (`project_path`)。當專案尚未包含的相機、陣列或光感測器連線時，系統會將其連同設定一併新增至專案檔案中（`connect_camera()`、`connect_array()`、光感測器連線、CLI、應用程式、 `chloros-mcp`），並附帶其設定。無論從何種模式變更（`set_settings()`、`ArraySession.set_member_settings()`、 `set_target_fps()`、陣列設定、`set_cap()`、CLI、 應用程式控制項）所做的任何變更，都會在片刻後儲存至該專案的檔案中。在相機上設定的曝光、增益或自動曝光目標會以「保留」狀態儲存，因此重新開啟時會再次保留該設定。
- **開啟專案時，系統會根據該專案設定裝置**，無論處於何種模式：若相機、陣列或光感測器已連接但未設定專屬參數，則會採用專案的儲存的設定（在開啟專案時為 `lattice cam-connect` / `array-connect`；重新開啟應用程式時為 `chloros-cli project connect`、`open_project().connect_all()`）。 若以新 ID 連接的陣列，會更新專案中已儲存的陣列並包含相同的相機；若陣列包含其他相機，則會新增至原有陣列旁。
- **應用程式自身的儲存操作絕不會以較舊的副本** 覆寫裝置的即時設定，並會保留透過 CLI 或 SDK 呼叫者所儲存的各成員覆寫設定。
- **您所設定的像素格式即為專案所保留的格式。** 無論是在「陣列設定」中、透過 `--pixel-format` 或 `connect_array(pixel_format=)` 設定的格式（包含 8 位元格式），都會被精確儲存並在重新連線時完全保留。 若未設定，每台相機將以 12 位元預設值連線（單色感測器上為 `BayerRG12`、`Mono12`）。格式名稱代表其位元深度； 而「系列」則指感光元件的系列，因此彩色攝影機絕不會被設定為單色格式，單色攝影機也不會被設定為拜耳格式， 且當格式被映射時，行 `[PixelFormat <serial>] ... written` 會明確標示此情況。8 位元擷取的輸出為 8 位元，包含反射率：經重新縮放並標記為其 `PixelScale`。
- 未隸屬於任何專案的裝置 則無任何記錄。
- **錄製檔案會跟隨其後建立的專案；若專案已開啟，則該錄製檔案將保留其原有專案。**當建立專案時 — `create_project()`、`chloros-mcp` 的 `create_project`，或應用程式中的**新專案**— 或複製（應用程式的**複製**，此操作會開啟副本）時，若錄製仍持續寫入先前開啟的專案中，錄製將繼續進行，且所寫入的內容最終會存入新專案中，位於該專案內的相同位置 （如下所示）。開啟現有專案 — `ChlorosProject.process()` （這會在後端開啟該專案），`chloros-cli process`，即應用程式的**開啟專案** — 此時錄製仍會繼續進行，並寫入原本正在錄製的專案中：錄製完成後，其檔案會加入該專案的檔案清單（應用程式會在開啟專案前停止所有錄製）。
  - `ArraySession.record()` 和 `ArraySession.burst()`：錄製停止時，影片或包含所有幀的連拍資料夾會整批移至該處。停止結果會報告新路徑 (`out_dir`、`video_path`)，並可據此生成連拍檔。 若磁碟機拒絕移動，錄製檔案將保留於原始寫入位置，結果會標示此狀況 (`relocation_error`)。已停止的連拍原始幀與 `.daq` 會加入該連拍最終歸屬之專案的檔案清單；影片則不會新增至 專案的檔案清單中。
  - `ArraySession.continuous()`：變更後的所有擷取內容皆儲存至新專案；變更前儲存的內容則維持原位。
  - `DAQSensorSession.record_start()`（以及應用程式自身的光感測器錄製內容）：當錄製停止時，`.daq` 會移至新專案的 `light_sensor` 資料夾中；`record_stop()` 會返回新路徑，且該檔案會加入該專案的檔案清單中。

  若錄製至開啟中的專案以外的資料夾，該檔案將保留原位置，而 `start_camera_continuous()` / 多串流工作則會保留其指定的資料夾。
- **在應用程式中，建立新專案後所進行的擷取作業會存入該專案。** 建立或複製專案時，所有正在執行的擷取程序與錄製作業都會持續進行，而之後進行的每項擷取（包括「全數擷取」、「開始擷取」）都會儲存至新專案並加入其檔案清單。 在應用程式中開啟現有專案時，會先停止所有正在進行的擷取作業。
- **專案在錄製期間不會被移動。**將開啟的專案移動至另一個工作目錄（應用程式中的**設定 → 工作目錄**；沒有任何 SDK 呼叫會移動專案， 但透過 HTTP 客戶端向 `/api/move-project` 發送請求則會移動）將因 HTTP 409 `PROJECT_RECORDING` 錯誤而被拒絕，只要陣列錄製或連拍、陣列連續擷取、相機連續／多串流工作或光感測器錄製正在寫入該專案時： `Project '<name>' is recording (<what>), so moving the project now is refused. Stop the recording, then try again.`

這與相機在斷開連接時留下的狀態無關（參見 [相機記憶體](#camera-memory--keep_settings--clear_settings)）：專案在開啟時會自行設定其相機。

### 攝影機記憶體 — `keep_settings` / `clear_settings`

當陣列連線時，其攝影機會採用陣列的設定，之後您可對其進行變更。 當 Chloros 釋放相機時 — `ArraySession.disconnect()` / `force_disconnect()`，即暫態函數結束時，或 `start_camera_continuous()` 停止時 ——系統會將每台攝影機恢復至連接前的原始狀態：其觸發器與同步線接線、PTP 及攝影機設定——`ExposureAuto` / `GainAuto` （以及手動模式 `ExposureTime` / `Gain`）、`TargetBrightness` 以及 AE 限制與阻尼、白平衡、黑電平、`PixelFormat`、 像素合併與感測區（ROI）、色彩處理流程（`GammaEnable` / `Gamma`、`LUTEnable`、`ColorTransformationEnable`）、`ReverseX` / `ReverseY`、`UserSetSelector`、PTP 的 `PtpSlaveOnly` / `PtpDomainNumber`、 封包大小（`GevSCPSPacketSize` / `DeviceStreamChannelPacketSize`）、調速 （`DeviceLinkThroughputLimitMode` / `DeviceLinkThroughputLimit`、`GevSCPD`、`GevSCFTD`）、區塊資料 （`ChunkModeActive`、其下每個 `ChunkEnable` 以及 `ChunkSelector`）、觸發輸入去毛刺，以及一個陣列提供從機的同步線 （`LineFilterSelector` / `LineFilterWidth`）以及擷取幀率 （僅限 `AcquisitionFrameRate` 且其啟用狀態為開時：若關閉，讀取的幀率為節拍器允許的最高值，而非設定值）。僅寫入差異部分：相機首先停止運作，以及曝光前所需的節拍器與幀率（這些參數需對應其幀週期）。

若要在專案外使用相機，有兩個參數會寫入相機的 **自有記憶體** —— 即相機開機時載入的內容，無論在何種軟體中（適用於 `disconnect()` / `force_disconnect()`、所有暫態函式、`start_camera_continuous()` 及 `ChlorosProject.disconnect_all()`；MCP：與 `disconnect_device` 及暫態工具相同的參數； CLI：`--keep-settings` / `--clear-settings`)：

- **`keep_settings=True`** 將每台攝影機的設定儲存至其記憶體中，維持會話結束時的狀態（UserSet2，即開機預設組）。 攝影機仍維持自由運行狀態（`TriggerMode` `Off`），同步線接線維持原狀。 若某台攝影機沒有 UserSet2，則會將設定儲存至 UserSet1；這台電腦會記住此狀態，因此 Chloros 的陣列自我修復機制——該機制會重置偏離 Chloros 基準值的 UserSet1——將不會干預該攝影機。
- **`clear_settings=True`** 將每台攝影機的記憶體重置為出廠設定：即 Chloros 的基準值，也就是其陣列自我修復機制所採用的設定 ——工廠預設值，包含 `TriggerMode`、`Off`、`TriggerSource`、`Line2`、自動曝光，以及RGB攝影機的校準白平衡 ——在 UserSet1 中保留，並設為開機預設組。
- 不可同時設定 （`ValueError`；後端回應 400）。回覆中包含 `camera_memory`，每台相機對應 `{"saved_to": "UserSet2" | "UserSet1" | None, "errors": [...]}` 或 `{"cleared": bool, "errors": [...]}` ——`force_disconnect()` 會將其傳回；針對某項工作，則會附帶 `stop()`。

該應用程式不提供這些選項。

**每種退出方式都會將攝影機還原。**還原至原始狀態的處理會在每個退出路徑上執行：`CameraSession.disconnect()`（應用程式的攝影機**X**）、應用程式中的**X** 若位於陣列中，或任何陣列成員（整個陣列將被釋放），並關閉應用程式（後端有最多 20 秒的時間將所有攝影機、陣列及正在運行的連續錄影還原；若暫存函式或單次呼叫仍在執行中，則會停在下一安全點，優先將其攝影機還原， 最長 8 秒，並回應 HTTP 503 `SHUTTING_DOWN`；仍在執行的 `connect_array()` 會在處理下一台攝影機前停止，並先將已開啟的攝影機歸還，最長 10 秒，並拋出 `ChlorosConnectError`（HTTP 503 `SHUTTING_DOWN`）；而在關機期間抵達的指令則不會開啟任何內容：HTTP 503 `SHUTTING_DOWN`），以及被終止或其終端關閉的 `chloros-cli` 指令（其工作會先被停止，並將攝影機還原）。發生當機或被強制終止的後端無法將任何內容還原；重新啟動會將每台攝影機還原至其記憶體中。**專案會自行設定其攝影機。** 當陣列透過專案連線時——無論是應用程式開啟的專案，或是 `ChlorosProject`， 該專案會向後端（`project_path`）宣告其名稱——每個成員將取得專案為其儲存的設定：曝光值、增益或自動曝光目標（被視為您的設定，因此自動曝光功能不會干預），以及其白平衡模式（Chloros 的校正； 相機本身的白平衡將保持關閉），無論相機記憶體中儲存了什麼內容。

### 拍攝模式與錄影器

陣列介面反映了 GUI 拍攝面板：單張／連拍／定時／最快快門模式，外加兩個錄影器（即時合成視訊與原始檔連拍 → 離線重新處理）。

```python
import time, chloros_sdk

with chloros_sdk.connect_array(serials) as arr:
    # Single (default) — one synced group
    arr.capture("out/", processing="reflectance")

    # Fastest — raw + .daq + combined index now, calibrate later
    arr.capture_fastest("flightline/")

    # Interval — one reflectance pass every 2 s, 5 passes (bounded so it ends)
    arr.capture_repeated("timelapse/", count=5, interval_s=2.0,
                         processing="reflectance",
                         on_capture=lambda i, r: print(f"pass {i}: {len(r)} frames"))

    # Continuous — every synced group for 60 s, saved by the backend itself
    with arr.continuous("run1/", processing="raw", duration_s=60) as run:
        run.wait()
    print(run.result["captures"], run.result["captures_per_s"],
          run.result["stop_reason"])

    # Combined-index video/GIF recorder: the GUI's combined view, or with no
    # GUI open the array's NDVI (pass expr="..." for another index)
    with arr.record("monitoring/", fps=10, gif=True) as rec:
        time.sleep(30)
    if rec.result and rec.result.get("error"):   # leaving the block does not raise
        print("recording failed:", rec.result["error"])
    else:
        print(rec.result["video_path"])

    # Raw-Bayer burst → offline reprocess into calibrated video(s)
    with arr.burst("capture/", duration_s=5) as b:
        pass
    out = arr.build_video(b.result["out_dir"], products=[
        {"kind": "per_cam", "level": "reflectance"},
        {"kind": "combined", "level": "index"}])
    print(out["outputs"])
```

- **`capture_repeated`**是 SDK 的連續／定時循環。由於沒有 `Ctrl+C` 可用於從腳本中中斷該循環， 您**必須** 傳入 `count` 和／或 `duration_s`（當任一條件達成時即停止）。`interval_s` 是從每次循環的起始點開始計時 （與 GUI 一致）。連續循環（`interval_s=0`）會串聯 `group_seq` → `after_seq`，因此它們是陣列中。其餘的關鍵字參數會直接傳遞至 `capture()`。
- **`continuous`**是後端自身的連續擷取功能——即 GUI 中的「連續模式」。**當您希望以高幀率將每個觸發事件記錄至磁碟時，請使用此選項而非 `capture_repeated()`：** `capture_repeated()` 會從您的腳本針對每個群組發出一次 HTTP 請求，而 `continuous()` 則會在後端內部維持兩個正在進行的擷取，群組之間無需往返傳輸。 在 9 台攝影機陣列上以 ROI 512 測量：每秒 73–77 次擷取，相較於該陣列約 78 次的發佈速率。請保留 `capture_repeated()` 用於間隔擷取（`interval_s`） ，並用於逐一檢查的掃描。 在 Windows 系統上，若需高幀率，請將後端從 Microsoft Defender 中排除（參見 [步驟 1](#step-1--install-the-chloros-platform-package)）。
- **`record`** 屬於 *監控級*： 它會記錄渲染後的即時綜合索引複合畫面。 當 GUI 的合併檢視視窗開啟時，它會記錄該視窗的顯示內容；若未開啟任何視窗，後端會自行渲染視圖：若您傳入陣列名稱，則為 `expr`；否則，則為該陣列的 NDVI（最接近 660 nm 和 850 nm 的紅光與 NIR 波段， 正如即時視圖所選取的），待陣列對齊後即會記錄。請在 `expr` 中以陣列所命名的名稱標示波段： M3C 相機為 `<band>_<filter>`（`NIR_850_RGN`、`Red_660_RGN`、`Orange_615_OCN`）， `Mono_<filter>` 適用於 M3M（`Mono_850`）；簡短的顏色名稱（`NIR`、`Red`） 僅在陣列中有一條頻帶具備該名稱時才有效。錄製期間開啟的 GUI 視窗，只要保持開啟狀態，其畫面便會被錄製下來。若錄製過程未寫入任何幀，則會因 `error` 及 `"no-frames: …"` 而失敗（此時陣列可能仍在對齊或穩定曝光）， 且會觸發 `stop()`（參見 [`RecorderHandle`](#recorderhandle)）。每個陣列僅能運行一個複合錄影器（若已有錄影器正在運行，則會觸發錯誤）。
- **`burst` → `build_video`** 屬於 *分析級*： `burst` 會以擷取迴圈的完整速率寫入原始影像幀 + 每幀清單 + 每個獨立 DLS 讀取對應一個 `.daq`，並在 `<output>/bursts/<base>/` 下運作（無鏈結， 無 exiftool、無即時預覽）。`build_video` 會將每個幀的時間與最接近的 `.daq` 進行比對，並重新執行匯入管線的輻照度／反射率／指數鏈。`products` 是一份 `{"kind": "per_cam"|"combined", "level": "radiance"|"reflectance"|"index"}` 的清單（預設：合併指數）。`burst().stop()` 還會自動觸發一次盡力而為的合併-指數建構，並在停止結果中以 `build_job` 形式返回。

##### `build_video` 的結果

| 鍵 | 含義 |
| --- | --- |
| `success` | 僅當每個請求的產品均按要求建置完成時，才會出現 `True`：至少有一個輸出，且 `skipped` 中無任何內容， 且不包含 `degraded`。 |
| `outputs` | 已寫入的影片（及 GIF）路徑。 |
| `errors` | 問題清單（文字形式）。 |
| `skipped` | 針對每個未產生影片的請求產品，各有一筆 `{"serial", "level", "reason"}` — 原因可能是相機的 （該相機無校準套件、無反射率的 DLS 讀數、處理鏈失敗）或複合索引的（該索引包含 `"kind": "combined"`、`serial` 及 `None` （各相機中無紅光／NIR波段組合，且無同時具備兩波段輻度值的群組）。任何一項條目都會觸發 `success` `False`；其餘影片仍會寫入並列出。 |
| `substitutions` | 相機渲染層級與請求不符，以及原因 — 當RGB相機要求輻度/反射率時，若該相機的校準套件可用，則會取得其顯示影片並進行白平衡。這並非錯誤。 |
| `notes` | 資訊性文字，非錯誤 — 例如：未設定單一相機索引表達式的相機將沒有索引影片，或RGB相機的顯示影片未進行白平衡（無校準套件，或校準套件中未包含工廠預設白平衡），並標明該相機名稱。 這並非錯誤。 |
| `degraded` | 當已寫入產品但並非所請求的產品時，會產生 `True`：離線組合索引為內建的紅光／NIR歸一化差異， 因此未對已配置的索引表達式進行評估。導致 `success` `False`。 |
| `combined` | 組合索引摘要： 所使用的波段、LUT 範圍、建構的幀。 |
| `clamp_masks` | 每個每幀校準 TIFF 檔案（`save_tiffs=True`）的 [解混鉗位遮罩](#unmix-clamp-mask--which-pixels-a-band-was-clamped-on) 路徑，位於 TIFF 檔案所在的 `<level>/` 資料夾旁的 `clamp_masks/<level>/` 內，每個曾發生像素鉗位的 TIFF 檔案對應一個遮罩。影片則不包含此類遮罩。 |
| `clamp_fractions` | 影片所包含的資訊（而非遮罩）：每個影片產品對應一個 `{"kind", "serial", "level", "frames"}` （`kind` `"combined"` 代表合併索引中的各成員相機），`frames` 為一組 `[frame, R, G, B]` 清單——該幀中，解混過程在 R / G / B 通道所屬頻帶內進行裁剪的像素比例 (`Chloros:Band{R,G,B}UnmixClampFraction`)。 |

#### `RecorderHandle`

由 `ArraySession.record()` 和 `ArraySession.burst()` 傳回。可將其用作上下文管理器，以便在範圍退出時自動停止，或手動驅動。

| 成員 | 說明 |
| --- | --- |
| `job_id` | 後端工作 ID（字串）。 |
| `kind` | `"composite"` （來自 `record`）或 `"raw"`（來自 `burst`）。 |
| `start_stats` | `start` 呼叫所回傳的字典。 |
| `result` | 執行期間的 `None`；停止後返回的最終停止結果字典。 |
| `stats(timeout=10.0)` | 即時工作統計資料（寫入的幀數、實際幀率、已用時間）。 |
| `stop(timeout=60.0)` | 停止錄影器；返回並快取最終結果（若停止請求本身失敗，則 `.result` 將保持為 `None`）。 對於失敗的 `record()` 錄製 — 即未寫入任何幀的情況（`error` `"no-frames: …"`） ——系統會將結果快取至 `.result`，隨後觸發 `ChlorosConnectError`。此操作具有幂等性（第二次呼叫將返回快取結果，且不會觸發異常）。 若未處理 `with` 區塊，錄製程式將停止運作但不會觸發異常，因此請在該區塊之後檢查 `rec.result.get("error")`。 |

```python
rec = arr.burst("capture/")
# ... drive manually ...
print(rec.stats()["frames"])
result = rec.stop()
print(result["out_dir"], result.get("build_job"))
```

#### `ContinuousHandle`

由 `ArraySession.continuous()` 傳回。請將其用作上下文管理器 — 若此時離開 `with` 區塊，而擷取仍在執行中，則會停止擷取 — 或者透過 `status()` / `wait()` / `stop()` 來驅動它。每次僅會針對每個陣列執行一次連續擷取。

| 成員 | 說明 |
| --- | --- |
| `start_status` | 由 start 呼叫所回傳的狀態字典。 |
| `result` | 執行期間為 `None`；擷取結束後的最終狀態字典（由 `stop()` 設定， 或由察覺其結束的 `status()` / `wait()` 設定）。 |
| `status(timeout=10.0)` | 即時狀態字典（下方欄位）。 |
| `wait(timeout=None, poll_s=0.5)` | 阻塞直至擷取自行結束 — 達到 `max_captures` 或 `duration_s`、陣列消失、磁碟空間已滿或發生錯誤 — 並回傳最終狀態。 若使用 `timeout`，則即使擷取仍在執行中，亦會回傳該秒數過後最新的狀態：請參閱 `running`。 |
| `stop(timeout=30.0)` | 停止擷取並回傳其最終狀態。冪等（第二次呼叫將回傳快取結果）。 |

狀態欄位：

| 欄位 | 含義 |
| --- | --- |
| `running` | 擷取執行期間為真。 |
| `captures` | 已儲存同步群組。 |
| `files` | 已寫入的檔案。 |
| `failed_files` | 無法寫入的檔案（如 [`CaptureResult.failed`](#files-that-could-not-be-written--captureresultfailed) 所示）. |
| `failed_captures` | 整體失敗的擷取嘗試，包括擷取程序曾重新嘗試的案例（陣列未能及時產生新群組，導致出現 HTTP 503 錯誤）。 |
| `skipped_groups` | 陣列已發佈但未被儲存的群組（擷取進度落後）。 |
| `captures_per_s` | 自開始以來的平均儲存率。 |
| `elapsed_s` / `started_at` | 迄今運行秒數；開始時間（Unix 紀元秒）。 |
| `last_group_seq` | 最新已同步並儲存的群組。 |
| `last_error` | 最新錯誤，或 `None`。 |
| `stop_reason` | 已結束一次：`count`（達到 `max_captures`），`duration` (已觸及 `duration_s`)，`stopped` (已呼叫 `stop()`)，`array_gone` (陣列已斷開連接)，`error`（連續發生 20 次失敗，或後端拒絕的請求），`disk_full`（磁碟無足夠空間存放的檔案）。`None` 執行中。 |
| `max_captures` / `duration_s` | 啟動時的範圍（`0` = 無）。 |
| `defender_cores` | 過去 10 秒時間窗內 Microsoft Defender 所使用的 CPU 核心數；`None` 直到測量到一個為止，且在未安裝 Defender 的主機上始終為 `None`。 |
| `defender_warned` | 當擷取記錄顯示 Defender 正在拖慢其運作（佔用 1 個或更多核心）時，此值即為 True。將後端排除在 Defender 掃描範圍外 — 請參閱 [步驟 1](#step-1--install-the-chloros-platform-package)。 |

```python
run = arr.continuous("captures/run1", processing="raw", max_captures=5000)
try:
    while True:
        st = run.wait(timeout=5)              # returns every 5 s while running
        print(f"{st['captures']} captures, {st['captures_per_s']:.1f}/s, "
              f"{st['skipped_groups']} skipped")
        if not st["running"]:
            break
finally:
    final = run.stop()                        # no-op once it has ended
if final["failed_files"] or final["stop_reason"] in ("disk_full", "error"):
    raise SystemExit(f"continuous capture: {final['stop_reason']}: "
                     f"{final['last_error']}")
```

### 將即時幀導入您自己的處理流程

若要將同步幀傳送至您自己的程式碼（例如機器學習、自訂檢視器或自建錄影器），請使用 `ArraySession.frame_stream()`。每個時標都會產生一個 `GrabResult`： `{serial: numpy array}`，**皆源自單一觸發器**，且該觸發器的 ID 會顯示於 `.group_id` 中。其實作方式與專案 API 中的 `ArrayHandle.frame_stream()` 相同， 因此 [每次讀取一個觸發器](#one-trigger-per-read--group_id)、[`band`](#exporting-a-single-band--band)、[`align`](#aligning-the-streams--align) 以及 [`bit_depth`](#bit_depth--trading-precision-for-bandwidth) 中的所有內容，均適用於這兩者。

```python
import chloros_sdk
from chloros_sdk import FrameFetchError

serials = ["213800234", "214000533", "214701288", "214701292"]   # master first

with chloros_sdk.connect_array(serials) as arr:
    try:
        for frames in arr.frame_stream(processing="raw", fps=10, duration=300,
                                       with_metadata=True):
            if not frames:              # no camera answered this tick
                continue
            if frames.errors:           # cameras missing from this trigger
                print("left out:", {s: str(e) for s, e in frames.errors.items()})
            for serial, (img, hdr) in frames.items():
                # img: numpy array (raw: one Bayer or mono plane)
                # hdr: group_id, frame_id, timestamp_ns, exposure_us, gain_db, ...
                my_pipeline(frames.group_id, serial, img, hdr["timestamp_ns"])
    except FrameFetchError as exc:      # the stream cannot continue
        print("stream ended:", exc.status, exc)
```

- **每個產出的集合都是一個新的觸發器。** `frames.group_id` 在同一個串流中絕不會重複；當您請求的 `fps` 數量超過陣列的執行速度時，串流會等待下一個觸發器，因此會以兩者中較慢的速度執行。
- **缺失的攝影機會被標示名稱，而非以其他內容填補。** 若某個攝影機在該觸發時無法被處理，該攝影機將不會出現在字典中，並會列於 `frames.errors` 中。
- **結束時會發出明顯提示。** 若發生拒絕情況，`FrameFetchError` 將在下一時標重複執行；若 60 秒內未收到幀，或 60 秒內未收到新觸發訊號，亦會重複執行。**其效能表現。**`frame_stream()` 透過本地 HTTP 連線傳輸完整的 NumPy 幀，最高傳輸速率可達**170 MB/s**左右。 這足以進行原始幀、分箱幀或 ROI 陣列的即時處理，但無法在全解析度下處理每個觸發訊號——若要保留**每個** 觸發訊號，請使用 [`continuous()`](#continuoushandle)。此數據於四鏡頭陣列（FOCN、FRGN、FRGB、F450）上以緊湊傳輸模式測量；每個時標皆為一組完整的觸發幀：

| | 全感測器（陣列 14.2–14.4 fps） | 2 × 2 像素合併（陣列 ~33 fps） |
| --- | --- | --- |
| `frame_stream(processing="raw")` | 6.7 組/秒 | 22.2 組/秒 |
| `frame_stream(processing="debayered")` | 2.3 組/秒 | 9.3 組/秒 |
| `frame_stream(processing="display")` | 0.8 組/秒 | 3.0 組/秒 |
| 每台攝影機 MJPEG， 720p 4 路串流 | 每路 13.5 fps；每幀皆與觸發訊號匹配 | 每路約 25 fps；4 路皆存在約 45% 的觸發訊號 |

請使用 `processing="raw"` 進行吞吐量處理及端側去馬賽克；`debayered` 層級還會移除校準套件中的暗電平偏移， 而 `raw` 的去馬賽克處理則不會。

#### 每台相機的 MJPEG 串流（不支援 RTSP）

**不支援 RTSP。** 串流介面為透過 HTTP 傳輸的 MJPEG（`multipart/x-mixed-replace`，每部分一個 JPEG，每部分皆帶有 `Content-Length`），每台攝影機一個 URL：

| URL | 提供內容 |
| --- | --- |
| `GET /api/camera/stream/<serial>` | 8 位元預覽，最高約 30 fps。對於陣列成員，`max_h` （預設 720，範圍 64–2160）與 `jpeg_q`（預設 80）分別設定尺寸與畫質。 |
| `GET /api/camera/<serial>/stream-annotated` | 相同預覽畫面，但疊加了 GUI 輔助線（斑馬線、網格、十字線、直方圖、峰值顯示、點測光）。`ChlorosLocal.camera_stream()` 會自動為您讀取相關資訊。 |
| `GET /api/camera/<serial>/stream-reflectance` | 每波段 8 位元反射率 — 請參閱 [即時反射率串流](#live-reflectance-stream-mjpeg)。 |

- **觸發標頭。** 對於陣列成員，所有三個串流的每個部分都包含 `X-Chloros-Group-Id`、 `X-Chloros-Frame-Id` 以及 `X-Chloros-Timestamp-Ns`。來自不同相機但具有相同 `X-Chloros-Group-Id` 的片段屬於同一個觸發器： 收集各部分，並根據該標頭進行分組以重新組裝成一組。獨立相機沒有分組：其預覽部分不攜帶任何這些標頭 （其反射率零件則攜帶一個空的 `X-Chloros-Group-Id`）。
- **預覽串流在負載下會丟失幀。** 它會傳送最新的一幀，並跳過任何無法及時傳送的幀，因此當陣列速率較高時，並非每個觸發訊號都會傳達至每台攝影機的串流中（參見上表）。對於每個觸發訊號，請使用 `frame_stream()` 或 `continuous()`。
- **當陣列的合併即時串流開啟時，`/api/camera/stream/<serial>` 會暫停**（`/api/camera/array/stream/<array_id>` — GUI 的陣列檢視會開啟該串流）。隨後，每個成員的串流會傳送一個「暫停（陣列串流活躍中）」的佔位幀，其中不含 `X-Chloros-*` 標頭， 且直到合併串流關閉前不再傳送任何內容；之後會自動恢復。`stream-annotated` 和 `stream-reflectance` 不會暫停。當腳本讀取各攝影機的預覽串流時，請關閉 GUI 的陣列檢視。
- **若需數值而非圖片，請使用 `frame_stream()`。** JPEG 為有損壓縮且為 8 位元；若路徑中未包含編解碼器，`frame_stream()` 將返回相同結果。

### 無人機測量之間隔擷取

測量飛行需要每飛行幾公尺就擷取一組同步影像，而非陣列所能產生的每一幀影像。有兩項設定可控制此功能：**`target_fps`**設定相機拍攝幀的頻率，而**`capture_repeated(interval_s=…)`** 則設定儲存幀的頻率。**在兩次儲存之間，相機會持續進行串流傳輸。** 每次間隔會儲存最新的同步影像組；其間的畫面則不會寫入。自動曝光會針對每幀畫面進行調整，因此當儲存發生時，曝光值已趨於穩定。並不存在相機在拍攝間歇停止運作、並為每次拍攝而喚醒的模式：若如此，每次拍攝都必須等待串流啟動且曝光穩定。**將幀率固定為間隔所需的速度。**若保持自動模式，陣列將以主機和網路所能承載的最高速率運行。將幀率固定在較低值，可在不改變儲存內容的前提下，降低相機發熱量、網路負載及主機 CPU 負載。請選擇能達到**每個間隔 2–4 幀** 的速率：

- 每次儲存都會在請求後等待下一幀，因此儲存的幀組最終會在預定時間後最多一個幀週期內完成。較高的幀率可將此時間差縮短。
- 自動曝光每幀調整一次，因此過低的幀率會減緩曝光跟隨飄過雲層的速度。

**根據飛行計畫計算間隔：**

```
spacing    = along-track footprint × (1 − forward overlap)
interval_s = spacing / ground speed
```

例如，60 公尺的拍攝範圍搭配 80% 向前重疊率，間距為 12 公尺；以 10 公尺/秒的速度飛行時，參數為 `interval_s=1.2`， 而固定為 `target_fps=3` 則會產生每間隔 3 至 4 幀的頻率。拍攝是基於時間的：SDK 沒有距離或 GPS 觸發模式，因此在測量線路上請保持穩定的地面速度。

**請在飛行器上儲存原始資料，並於事後進行校正。** `interval_s` 是從每次儲存的起始時間開始計測的，因此若某次儲存時間超過間隔時間，將導致下次儲存延遲開始，進而造成間距漂移。 若同時儲存原始資料與光感測器讀數（`processing="raw"`；該讀數預設會被儲存），可縮短每次儲存時間，並確保日後每幀影像皆可轉換為反射率。

```python
import chloros_sdk

INTERVAL_S = 1.2      # from the flight plan
FLIGHT_S = 20 * 60    # bounds the loop; it also stops at count= if given

with chloros_sdk.connect_array(serials, target_fps=3.0) as arr:
    groups = arr.capture_repeated(
        "survey/", interval_s=INTERVAL_S, duration_s=FLIGHT_S,
        processing="raw", render_index=False,
        on_capture=lambda i, r: print(f"group {i}: {len(r)} frames"))
```

- **`connect_array(target_fps=…)`**從開始時就固定此速率；**`arr.set_target_fps(fps)`**可在飛行途中變更速率而無需重新連線，而**`arr.set_target_fps(None)`** 則將其恢復為自動模式。若傳輸速率高於主機與網路所能承載的上限，系統會將其調降至該上限 — 請查閱結果中的 `clamped` — 而最低傳輸速率為 0.1 fps。
- **請勿使用 `burst()` 來稀疏幀。** 它會寫入陣列產生的每一幀；雖然它會跟在固定傳輸速率的 `target_fps` 之後，因此僅在需要每一幀時，這才是正確的工具。
- **`target_fps` 無法修復損壞的幀。** 降低此參數會減少產生的幀數，而非降低每幀的傳送速度；若遇到 GVSP 損壞的幀，請改為降低 `wire_ceiling_mbps`（參見 [陣列健康狀態](#array-health--which-subsystem-is-losing-frames)）。

```python
r = arr.set_target_fps(5.0)
if r["clamped"]:
    print(f"asked for {r['requested']} fps, running at {r['target_fps']}")
```

### 連接到已連線的陣列 — `attach_array`

若陣列已處於運作狀態 （由 GUI 開啟，或先前SDK會話已呼叫 `connect_array`），請使用 `attach_array` 取得其句柄，而非重新連線。在此情況下，`connect_array` 會始終拋出`Camera <sn> is already in array <id>`的錯誤，因為針對「池中成員」發送 POST 請求 `/array/connect` 並不具備幂等性； `attach_array` 會讀取 `/api/camera/array/list`，並透過 array_id 或 serials 進行比對。

```python
import chloros_sdk

# By serials (matches if every serial is a member of one existing array)
arr = chloros_sdk.attach_array(
    ["213800234", "214000533", "214701288", "214701292"])

# By array_id (when you've already noted it down)
arr = chloros_sdk.attach_array("array-1779862544497")

# attach_array returns the same ArraySession as connect_array
arr.capture("output/", processing="reflectance")
```

模式：與桌面 GUI 共用租戶的 SDK 腳本應先嘗試 `attach_array`，若池中尚無陣列，則回退至 `connect_array`。

```python
import chloros_sdk

try:
    arr = chloros_sdk.attach_array(serials)
except chloros_sdk.ChlorosConnectError:
    arr = chloros_sdk.connect_array(serials)
```

> **已連接的陣列是借用的，並非擁有的。** 來自 `attach_array` 的句柄絕不會釋放該陣列：無論是 `with` 區塊的退出，還是顯式的 `disconnect()` 呼叫，均不會釋放它（顯式呼叫會向 stderr 輸出一行訊息，表示 讓陣列持續運行）。此規則與 `CameraSession` / `DAQSensorSession` 相同。因此，無論是否使用 `with`，與 GUI 共存都是安全的：
>
> ```python
> with chloros_sdk.attach_array(serials) as arr:
>     arr.capture("output/", processing="reflectance")
> # … the array stays up for the GUI
> ```
>
> 當您確實打算終止連線時——例如要透過不同的 `wire_ceiling_mbps` 重新連線——請呼叫 `arr.force_disconnect()`。若在已連接的處理程序上執行普通的 `disconnect()` 後嘗試重新連線，將會因 `Camera <sn> is already in array <id>`，因為並未釋放任何資源。

### 網路分析輔助工具

在開啟陣列前相當實用——可預測您擬定的設定是否適用：

```python
result = chloros_sdk.analyze_array_network(
    master_serial="214701288",
    slave_serials=["213800234", "214000533", "214701162"],
    width=2048, height=1536,
    pixel_format="BayerRG12",
    binning=1,
)

if result["status"] == "ok":
    print("Use the requested settings.")
elif result["status"] == "auto_capped_fps":
    r = result["recommended"]
    print(f"Keep the resolution; cap the trigger rate at {r['recommended_target_fps']} fps")
elif result["status"] == "auto_shrunk":
    r = result["recommended"]
    # width / height are sensor pixels BEFORE binning (what connect_array
    # takes with binning=); out_width / out_height is what is streamed.
    print(f"Connect with width={r['width']} height={r['height']} "
          f"binning={r['binning']} ({r['out_width']}x{r['out_height']} streamed)")
elif result["status"] == "needs_force_slip":
    print("Sim-sync impossible on this wire; force_tier='slip-emit-and-capture' required")
```

```python
analyze_array_network(
    master_serial,
    slave_serials=None,
    width=2048, height=1536,              # sensor pixels before binning
    pixel_format=None,                    # None = what connect applies (Mono12 / BayerRG12)
    binning=1,                            # 1 | 2 | 4
    force_tier=None,
    packed="auto",                        # "auto" | "on" | "off"
    backend_url="http://127.0.0.1:5000",
    timeout=30.0,
) -> dict
```

它從不啟動後端：若後端未回應則會觸發 `ChlorosConnectError`，而被拒絕的請求則會作為結果返回。錯誤的 `packed` 會觸發 `ValueError`。

`status` 是 `ok` / `auto_capped_fps` / `auto_shrunk` / `needs_force_slip` （否則為 `error` — 包括被拒絕的請求：未登入、`error_code: AUTH_REQUIRED`，或方案低於 SDK 最低門檻、`PLAN_UPGRADE_REQUIRED`）。`auto_capped_fps` 表示所請求的解析度僅能在受限的觸發率下符合 RX 環——保留該解析度並將 `target_fps=result["recommended"]["recommended_target_fps"]` 傳遞至 `connect_array`（參見 [範例 6](#6-capability-probe-before-connecting-a-4-cam-array））。`auto_shrunk` 僅在無法以任何幀率傳送所請求的幀時出現；connect 絕不會自行縮減幀，因此請自行套用 `recommended`。

**如何解讀投影**（與 GUI「陣列設定」面板採用相同模型）：

- **連拍 (`frame_bytes_total`) 會以每台攝影機的實際像素格式進行加總。**單色**M3M**攝影機傳輸 Mono12（2 B/px），或透過 `pixel_format` 採用其深度通道的單色格式 （`"BayerRG8"` 即 `Mono8`， 1 B/px)，因此若未啟用深度功能，由三台單色相機組成的 4 台相機全解析度畫面大小為**~25 MB**，而非基於全 8 位元假設所得的 ~12.6 MB。後端會根據相機型號解析各相機的格式。
- **通量 (`burst_fits_nic_ring`) 會考量匯流排負載**， 而非「整批傳輸 vs 環形傳輸」：當主機從接收環中讀取資料的速度快於攝像模組填充環的速度時，模擬發射機制便能正常運作。即使傳輸批次超過環形容量，10G 主機搭配 1 GbE 攝像模組仍能**傳輸**全解析度影像； 而 1 GbE 主機則會阻塞（`needs_force_slip` / `auto_shrunk`）。
- **`achievable_fps_max` 是一個保守的串列讀取上限** — `max(readout+emit, N×emit)` 將每台相機的發射量限制在 1 GbE 相機鏈路之內，且與曝光值無關。 例如：4 台相機的全解析度 12 位元陣列約為 2.8 fps（與執行時測得的 ~2.7–3.0 相符）。 完整模型：[CLI 參考 → 陣列 fps 與連拍模型](cli-reference.md#array-fps--burst-model)。
- **連續擷取結束後，投影結果會顯示其維持的幀率。** 若以自動幀率進行 30 秒或更長時間的擷取，則記錄的是該陣列大小在此電腦上最終穩定的幀率；幀率控制器短暫嘗試的較高幀率將被忽略。從此之後，根據 `array-connect`'起，`connect_array()` 的分析結果以及「陣列設定」面板，當幀率低於模型預設值時，會引用該幀率，並在 `cycle_limiter` 中說明其限制因素：`"clock"` （陣列以最終穩定的速率運行，擷取迴路等待其觸發時鐘）、`"loop"`（擷取迴路時間已耗盡）、 `"capture"`（擷取落後並跳過幀組）或 `"measured"`（未明確說明是哪種情況）。模型的數值見於 `wire_fps_max`。 該陣列的數值仍會攀升至傳輸線所能承載的範圍。
- **超額訂閱（`oversubscribed: true`）意味著無論攝影機達到何種幀率，傳輸線都無法承載該陣列** — 幀率（fps）欄位（`achievable_fps_max` / `fps_bright` / `fps_dark`）讀取值為 0，且無法透過像素合併來解決此問題 （此舉僅會降低每幀的位元組數，而非每秒的步調位元組數）。單純因網路線速率過低並不會導致此狀況：此時幀率反而會下降，且若無法將攝影機的步調調得足夠慢，層級將變為 `slip-emit-and-capture` （串列模式）。解決方案包括減少攝影機數量、使用巨型幀，或更換更快的網路介面卡；`max_cams_collision_safe` 會回報您的線路與攝影機的上限值。 該回應中亦包含 `aggregate_demand_bps`、`collision_safe_ceiling_bps` 及 `per_cam_floor_bps`。請參閱 [超額訂閱](#over-subscription-the-per-cam-floor)。

### 發現與清單

```python
chloros_sdk.discover_lattice_cameras()   # list all cams visible to the backend
chloros_sdk.list_cameras()               # cams currently in the pool
chloros_sdk.list_arrays()                # active arrays in the pool
```

每個函式皆接受僅含關鍵字的 `backend_url=` 和 `timeout=30.0`，並回傳一組字典清單。 `discover_lattice_cameras()`（`serial`、`ip`、 `mac`、`model`、`lattice_model`、`vendor`、`connected`、 `array_id`, …) 亦會呼叫 `auto_start_backend=True`；而 `list_*` 的呼叫則永遠不會啟動後端。

---

## 智慧自動曝光 (Smart-AE) / 智慧擷取 (Smart-Capture)

LATTICE 陣列一經連接，便會在背景中持續執行自動曝光 (AE)，但新設定的場景需要片刻時間才能收斂。**Smart-capture** 是一項整合式的便利功能：它會輪詢每台攝影機的曝光值，待陣列在指定時間窗內穩定後，再觸發擷取。其功能等同於圖形使用者介面（GUI）：桌面應用程式的「智慧」擷取按鈕會呼叫相同的後端端點。

```python
import chloros_sdk

with chloros_sdk.connect_array([
        "213800234", "214000533", "214701288", "214701292"]) as arr:
    # Initial pose
    arr.capture("pose_a/", processing="reflectance", smart=True)
    input("Move the rig, then press Enter...")
    # New pose — smart-capture waits for AE to re-settle automatically
    arr.capture("pose_b/", processing="reflectance", smart=True)
```

若透過 `ChlorosProject`（下一節）進行控制，您將獲得更多調整選項：

```python
proj.arrays["main_rig"].capture_smart(
    output_dir="out/",
    processing="reflectance",
    settle_timeout_s=5.0,           # max wait
    stability_window_s=1.5,         # exposure must hold steady this long
    exposure_tolerance_pct=5.0,     # %-spread allowed within the window
)
```

「智慧自動曝光」策略預設為保守模式。若進行嚴謹的輻測工作，請收緊 `exposure_tolerance_pct` 參數；若面對快速變化的場景且僅需「大致準確」，則可放寬該參數。

---

## DAQ 感測器工作階段

用於光譜感測器的持久後端池（透過 USB 的 DAQ-U、透過 BLE 的 DAQ-M、透過乙太網路的 DAQ-E）。與相機介面相呼應：智慧偵測、池資源重複利用、幂等連接。

### 智慧偵測（零配置）

```python
import chloros_sdk

with chloros_sdk.connect_daq_sensor() as daq:
    print(daq.model, daq.transport, daq.address)
    for frame in daq.latest(n=10):
        spectrum = frame["spectrum"]   # list[float] (W/m²/nm if calibrated)
        is_sat = frame["is_saturated"]
        x, y, z = frame["x"], frame["y"], frame["z"]
        print(len(spectrum), is_sat)
```

優先順序：乙太網路 → BLE → USB。傳遞任何一個明確提示即可鎖定傳輸協定。

### 鎖定傳輸協定

```python
# DAQ-U on a specific serial port
daq = chloros_sdk.connect_daq_sensor(transport="usb", port="COM3")

# DAQ-M over BLE by MAC (implies transport="ble")
daq = chloros_sdk.connect_daq_sensor(mac="AA:BB:CC:DD:EE:FF")

# DAQ-E over Ethernet by hostname (implies transport="eth")
daq = chloros_sdk.connect_daq_sensor(eth_host="daq-e-xxx.local")

# A specific sensor by id (as list_daq_sensors() shows it; ':' or '-' alike)
daq = chloros_sdk.connect_daq_sensor(sensor_id="CB-7C-A7-5A-73")

# Tuning knobs
daq = chloros_sdk.connect_daq_sensor(
    port="COM3",
    integration_time=64,      # ms
    frame_avg=20,
    enable_ae=True,
    start_streaming=True,
)
```

`sensor_id=` 指定一個感測器，且僅會開啟該感測器。系統會將其與已開啟的感測器進行比對，接著向 DAQ-E 查詢其主機名稱或 IP（針對網路掃描中僅能透過 IP 識別的 DAQ-E，會以唯讀方式查詢其名稱、MAC 位址及序列號， 因此透過上述任一方式皆可找到該裝置），以及 DAQ-M 的 BLE MAC，並比對 DAQ-U 或 DAQ-M 在開啟後所回報的 ID（為進行檢查而開啟的任何其他裝置都會被重新關閉）。若未找到該裝置， `connect_daq_sensor` 會觸發 `ChlorosConnectError`，列出所找到的項目；並附上地址（`port=` / `mac=` / `eth_host=`）所對應的位址中存有不同感測器時，也會觸發錯誤。它絕不會退而採用任何可見的感測器——這正是零標誌呼叫的用途。

### `connect_daq_sensor()` 簽名

```python
connect_daq_sensor(
    *,                                    # every argument is keyword-only
    sensor_id=None,                       # one named sensor (see above)
    transport=None,                       # "usb" | "ble" | "eth"; None = smart-detect
    port=None,                            # DAQ-U serial port ("COM3", "/dev/ttyUSB0")
    mac=None,                             # DAQ-M BLE MAC (implies transport="ble")
    eth_host=None,                        # DAQ-E hostname or IP (implies transport="eth")
    integration_time=None,                # ms; None = the sensor's own
    frame_avg=None,
    enable_ae=None,
    cap_id=None,                          # cap correction: "sunshine_cosine", "fov_30", ..., "none" (bare)
    start_streaming=True,
    backend_url="http://127.0.0.1:5000",
    timeout=60.0,
    auto_start_backend=True,              # spawn a local backend if none is running
) -> DAQSensorSession
```

已位於感測器池中的感測器將以原樣歸還（即「借用」狀態）：`integration_time`、`frame_avg` 及 `enable_ae` 不會套用至該感測器。 請先將其斷開連接，再使用您的設定開啟它。`cap_id` 為例外情況：在已借用的感測器上，此設定會透過 `set_cap()` 進行設定。若未指定，則採用後端預設值（請參閱方法表下方的註解）。

### `DAQSensorSession` 方法

| 方法 | 說明 |
| --- | --- |
| `status(timeout=10.0)` | 彙總資料（串流／錄製狀態、波長範圍、校準 SHA、積分時間、frame_avg、AE 狀態）。針對串流已靜默的 DAQ-E：`reconnect` （當後端處於自動重新連線狀態時：`state`、`attempts`、`next_attempt_in_s`、`last_error`；`None` 表示串流中），`stream_lost_s`、`reconnects` 及 `last_reconnect`，而 `record_gaps` 則表示 正在進行的錄製；`rx_stream_fault` 表示重新連線狀態，直至收到第一個頻譜為止。 |
| `latest(*, n=1, timeout=10.0)` | 返回最多 N 個最近的頻譜幀（每個 `ts_ns`、`spectrum`、 `is_saturated`、`integration_time`、`x`、`y`、`z`). |
| `stream_start()` / `stream_stop()` | 恢復 / 暫停串流 （處理程序保持開啟）。 |
| `record_start(*, output_dir=None, device_name=None, timeout=15.0)` | 開始錄製 .daq 檔案。回傳檔案路徑。若 DAQ-U/M 未配備 AWS 校準套件，則此操作將被拒絕（DAQ-E 除外）。 |
| `record_stop()` | 停止錄製。回傳 `{path, rows, warning, gaps}`；若為乾淨錄製，則 `warning` 等同於 `None`， 否則則為後端診斷訊息（例如檔案為空或串流中存在空缺），該訊息亦會輸出至 stderr。 `gaps` 會列出串流未傳送任何資料的每個區段：`start_utc`、`end_utc`（`None`：在停止時仍會遺失）， `duration_s`, `reconnected`、`marked_in_file`。若為連續記錄，此處則為空。 |
| `set_cap(cap_id, timeout=15.0)` | 設定自下一個讀數起所套用的上限校正配置檔： `"sunshine_cosine"`、`"fov_15"` ... `"fov_90"`，或針對裸感測器的 `"none"`（或 `None`）。回傳感測器的摘要，可透過 `cap_id` 讀取該摘要。DAQ-E-S 無論被要求什麼，都會將 `as_recorded` 維持為當前狀態（其擴散器為內建式）。若校正曲線 無法載入時觸發 `ChlorosConnectError`。CLI：`chloros-cli daq pool-set-cap`。 |
| `enable_ptp(timeout=15.0)` | 僅限 DAQ-E：針對 PTP 已關閉的裝置，重新啟用 PTP （`status()["rx_ptp_enabled"]` 即為 `False`；此設定儲存於裝置中，因此將保持關閉狀態，直至重新啟用為止）。 回傳更新後的狀態。若為 DAQ-U / DAQ-M 裝置，或當裝置未確認時，將觸發 `ChlorosConnectError`。Chloros 絕不會自行將裝置重新啟用，因為關閉狀態可能是刻意設定的。 CLI：`chloros-cli daq pool-enable-ptp`。 |
| `device_info(timeout=15.0)` | 僅限 DAQ-E：裝置自身的設定。`firmware` (`running`、`bundled`、`update_available`)、 `ip` 及 `ip_mode` (`dhcp` / `linklocal` / `static`： 目前執行的設定），`network`（儲存的設定：`mode`，加上 `ip` / `mask` / `gateway`，靜態時），`network_change_pending` （在裝置執行儲存的設定之前為真，該設定包含 `set_network(..., reboot=False)`），`auto_stream`、`ptp`。 當裝置的版本無法被讀取時，`firmware["update_available"]` 將轉為 `None`。當裝置的資料流尚未傳送至這台電腦時，系統會在建立新連線時讀取該資料流，並將 `stream_silent` (`True`)、`stream_fault` 及 `reconnect`；若裝置未回應，則觸發CLI：`pool-info`。 |
| `set_network(mode="auto", *, ip=None, mask=None, gateway=None, reboot=True, force=False, comeback_timeout=45.0)` | DAQ-E 專用：DHCP (`"auto"`) 或靜態位址。重新啟動裝置以套用設定並重新連線；此工作階段將持續運作，且 `address` 會跟隨該裝置。`reboot=False` 僅會儲存設定。 請參閱 [DAQ-E 裝置設定](#daq-e-device-settings)。CLI：`pool-set-network`。|
| `set_auto_stream(enabled, timeout=15.0)` | 僅限 DAQ-E：在開機時且未連接任何軟體的情況下進行串流（設定儲存於裝置中）。回傳已確認的設定。CLI：`pool-set-auto-stream`。 |
| `reboot(*, comeback_timeout=45.0)` | 僅限 DAQ-E：重新啟動裝置，並以其原有設定重新連接。CLI：`pool-reboot`。 |
| `update_firmware(*, comeback_timeout=45.0)` | 僅限 DAQ-E：將此 Chloros 版本隨附的韌體刷寫至運行舊版本的裝置上，然後重新連接該裝置（約需一分鐘；請勿關機）。 若裝置恢復速度緩慢或以其他版本重新啟動，則會回傳 `updated`、`fw_before`、`fw_after`、`bundled` 以及 `warning`；若裝置恢復速度緩慢或恢復至其他版本時，則會回傳 `updated`；對於已更新至最新版本的裝置，則回傳 `False`。 若無法讀取裝置版本，則觸發 `ChlorosConnectError`。CLI：`pool-update-firmware`。 |
| `disconnect()` | 從資源池釋放。不-op 操作。若為本會話開啟的感測器，則為 `force_disconnect()`；當釋放未獲確認時（`with` 區塊的退出 從不拋出）。 |
| `force_disconnect(timeout=30.0)` | 無論由誰開啟，皆釋放該感測器 ——例如，一個因腳本當機而殘留在池中的感測器，新腳本僅能透過已附加的處理程序取得該感測器，其 `disconnect()` 不會執行任何操作。終止其他客戶端的感測器將結束該感測器正在進行的任何記錄。當後端確認釋放，或無需釋放時 （感測器不在池中，或無後端正在執行）；若釋放超時或遭後端拒絕，則拋出 `ChlorosConnectError` 錯誤。CLI：`chloros-cli daq pool-disconnect --sensor-id ID`。 |

屬性：`sensor_id`、`model`、`transport`、 `address`（緊接在 `set_network()` 重新定位的 DAQ-E 之後）、`info` 及 `backend_url`。

> **請說明該電容的名稱。** DAQ-U、DAQ-M 和 DAQ-E 出廠時未附電容，且 Chloros 無法偵測是否已安裝：若未連接 `cap_id` 便接上感測器，系統會取得 `sunshine_cosine` 的數值，其記錄資料顯示這是基於 (`cap_id_source=auto_default`)。在該修正下，未安裝電容的 DAQ-U 讀數約高出 30 倍，因此對於未安裝電容的感測器 （或 `set_cap("none")`），若安裝了蓋子，則讀數曲線會有所不同。

### DAQ-E-S — 固定式擴散器與傾斜感測器

**DAQ-E-S** 是一款 DAQ-E，但有兩項對軟體而言至關重要的差異。其餘所有功能——包括裝置偵測、連線、PTP、記錄以及下方的裝置設定——皆與 DAQ-E 完全相同。**其漫射器為固定式，因此無需進行遮光罩校正。** 內建的 Sunshine 型散射板在進行工廠校準測量時已安裝於裝置上，因此該校準值已能準確讀取向下輻射照度。 Chloros 會將其上限值設定為 `as_recorded`（無配置檔），無論系統要求為何：以該裝置命名的上限值（`sunshine_cosine`、 `fov_*`、`none`）均會被忽略並顯示錯誤訊息，且其記錄資料會儲存為 `cap_id_source=model`，因此「假設存在校正檔」的警告從不會對其觸發。 請勿手動對其資料套用上限：若在頂端放置 `sunshine_cosine`，讀數將高出約 11 倍。

**此感測器配備三軸加速度計。** 陽光漫射器的餘弦響應僅在水平狀態下有效，因此傾斜的感測器會回報錯誤的輻照度——且反射率會繼承此誤差——且無其他症狀。因此，每個 DAQ-E-S 讀數皆包含其姿態資訊：

| 欄位 | 含義 |
| --- | --- |
| `tilt_deg`, `roll_deg`, `pitch_deg` | 角度單位為度。當韌體標記角度為未定義時（自由落體、軸失效），則顯示為 `None`。 |
| `x_mg`、`y_mg`、`z_mg`、`mag_mg` | 以毫 g 為單位的加速度。 `mag_mg` 若遠離 ~1000，表示裝置正在移動，且該樣本的角度數據無實質意義。 |
| `cal_applied` | 設定：座標軸與角度以擴散器座標系為基準（已套用裝置的安裝校正）。清除：原始電路板座標系，此模式可能顯示高達 ~2.3° 的傾斜角度，但該傾斜並非真實存在。 |
| `sample_age_ms` | 傳送頻譜時加速度計樣本的採樣時間。 |
| `temp_raw` | 未經校準的晶片溫度 **趨勢** —— 僅顯示差異值，絕非溫度計讀數。 |

`DAQSensorSession.latest()` 會將這些資料回傳至各幀的 `imu` 鍵下（在 DAQ-U、DAQ-M 或標準 DAQ-E 上則為 `None`）。 由 DAQ-E-S 記錄的 `.daq` 會將相同數值儲存於其 `imu_*` 欄位中（在 DAQ-U、DAQ-M 或標準 DAQ-E 上則為 `imu_tilt_deg`、 `imu_roll_deg`、`imu_pitch_deg`、 `imu_mag_mg`、`imu_cal_applied`、……）中儲存相同的數值，因此讀者可藉此區分雲層與三腳架被撞倒的情況。在其他所有型號中，這些欄位皆為空。

姿態資料流會針對其自身的欄位進行報告，位於 `status()` （`rx_imu_stream_fault`、`rx_imu_stream_faults`、`rx_last_imu_stream_fault`）中報告其自身欄位，並與光譜欄位分開存放，以確保加速度計故障時，絕不會被誤判為記錄中的缺口。 `discover_daq_sensors()` 以及該會話的 `model` 顯示為 `DAQ-E-S`。

### DAQ-E 裝置設定

```python
with chloros_sdk.connect_daq_sensor(eth_host="daq-e-def330.local") as daq:
    info = daq.device_info()
    print(info["ip"], info["ip_mode"], info["firmware"])

    # A fixed address on the network this computer is on
    daq.set_network("static", ip="192.168.1.50", gateway="192.168.1.1")
    print(daq.address)              # 192.168.1.50: the session follows it

    if info["firmware"]["update_available"]:
        daq.update_firmware()
```

- **此電腦的網路中必須存在一個靜態位址。** 若某台電腦的子網未包含此電腦的：重新啟動後，將無法連線至感測器以將其恢復原狀。僅當此電腦將被重新指派位址，或路由至新子網時，才應通過 `force=True`。
- **`set_network`、`reboot` 及 `update_firmware` 會重新啟動感測器**，並在記錄期間觸發 `ChlorosConnectError`， 且若其未在 `comeback_timeout` 秒內恢復運作時。感測器 ID 及 `daq-e-<id>.local` 主機名稱絕不會變更。
- **重啟前剛開始的記錄會由重啟程序關閉並命名。** 當感測器恢復連線時，返回的字典會包含 `recording_closed`（檔案路徑） 以及一個 `warning` 訊息說明此情況。若感測器未能恢復連線，或韌體傳輸失敗，則由 `ChlorosConnectError` 訊息代替說明：其訊息結尾會顯示警告，且 `e.recording_closed` / `e.warning` 會被設定（若無任何錄製任務已關閉，則 `None` 也會被設定）。待感測器恢復連線後，請重新啟動錄製。
- **僅感測器本身會重新連線。** 重啟後，裝置必須回應其所在位置 ——且在 `set_network("auto")` 之後，亦須以其 `.local` 名稱——透過其原先擁有的網路介面卡（MAC）回報其名稱、主機名稱及型號； `address` 將成為回應的 IP 位址。以該名稱發出的回應僅在對應的 MAC 位址下有效，因此，若透過路由器連線至感測器的電腦，或透過 WSL2 的預設 NAT 網路連線，當感測器因 DHCP 重新分配而變更位址時，將不會被重新連線 （`ChlorosConnectError`；請以其新位址重新連線）。
- **韌體更新需要感測器重新連線。** `update_firmware()` 會向裝置發出邀請，而該裝置會透過作業系統選定的埠，建立回傳至此電腦的 TCP 連線以取得映像檔。後端伺服器會在可連通該裝置的網卡上進行監聽（若無法辨識具體網卡，則在所有網卡上監聽），並僅為該裝置提供服務： 在持續等待期間，任何其他連線均會被拒絕（後端輸出中顯示 `[DAQ-FW] … refused a firmware connection from <ip>`）；若防火牆阻擋此連線，更新將因 `device never connected back on TCP <port>` 而失敗。 Windows 安裝程式會為其新增規則（參見 [步驟 1](#windows-exe)）；若使用其他防火牆，請允許來自感測器子網的 TCP 傳入連線至後端。
- **在 WSL2 環境下，請使用鏡像網路**（`networkingMode=mirrored` 及 `.wslconfig`）：在預設的 NAT 模式下，所有靜態位址都會被拒絕（WSL 自身的位址不在感測器的網路中），資料流無法傳遞至 WSL，導致韌體更新無法完成。
- 圖形化介面 (GUI) 會維持與感測器的獨立連線：若透過腳本重新啟動，將終止 GUI 與感測器的連線。GUI 具有相同的設定（「光感測器」分頁、感測器設定、網路）。

### 發現 — 尋找可用於連線的位址

`discover_daq_sensors()` 會掃描 USB / BLE / ETH 介面，尋找您 *可能* 能開啟的感測器。它是 `discover_lattice_cameras()` 的 DAQ 對應版本，也是取得 **DAQ-M 的 BLE MAC 位址** 的唯一方法 的唯一途徑——DAQ-E 具有主機名稱，DAQ-U 則有 COM 埠，但 MAC 位址既不會印在裝置上，也不會由作業系統列出。

```python
for s in chloros_sdk.discover_daq_sensors():
    print(s["transport"], s["address"], s["model"], s["extra"])
# ble  C3:D8:85:E0:0A:19  DAQ-M  {'name': 'NSP32_SPECTRUM'}
# usb  COM3               DAQ-U  {'vid': '0403', 'pid': '6001', 'manufacturer': 'FTDI'}

# `address` is exactly what connect_daq_sensor wants:
for s in chloros_sdk.discover_daq_sensors(transports=["ble"]):
    if s["model"] == "DAQ-M":
        daq = chloros_sdk.connect_daq_sensor(mac=s["address"])
```

| 欄位 | 說明 |
| --- | --- |
| `transport` | `usb` \| `ble` \| `eth`. |
| `address` | COM 埠 / BLE MAC / 主機名稱 — 傳遞至 `connect_daq_sensor` 作為 `port=` / `mac=` / `eth_host=`。 |
| `display` | 人類可讀標籤。 |
| `model` | `DAQ-U` \| `DAQ-M` \| `DAQ-E` \| `DAQ-E-S`， 或 `None`。乙太網路感測器為 `DAQ-E-S` 或 `DAQ-E`，因其會在掃描期間被詢問時回報自身狀態（唯讀）， （若未能及時回應，則顯示為 `DAQ-E`）。當 USB 序列埠的橋接晶片為 DAQ-U 所使用的類型（FTDI、CH340、CP210x、Prolific）時，其讀取值為 `DAQ-U` —— 這是從埠描述中獲得的提示， 並非探針——而其他任何 USB 裝置則讀取為 `None`，該裝置仍會列出，因此您可以將其傳遞為 `port=`。 若串列埠既無 USB 廠商識別碼，亦未採用上述橋接晶片（例如主機板的 UART，如 Linux 的 `/dev/ttyS*`、藍牙串列連結、Intel AMT），則無法視為 DAQ-U，因此不會列於清單中。 |
| `extra` | 各傳輸層詳情 (BLE 廣播名稱；USB `vid`、`pid`、`manufacturer`、`serial_number`；DAQ-E IP/韌體/…）。空值將被省略。 |

| 參數 | 預設值 | 說明 |
| --- | --- | --- |
| `transports` | 全部三項 | 限制掃描的序列（或 CSV 字串）。當您清楚知道所需內容時，建議傳入此參數——BLE 是速度較慢的一環。 |
| `scan_timeout` | 5 | 每種傳輸方式的掃描時間窗（以秒為單位）；後端會將其限制在 1–20 之間。 |
| `timeout` | 60.0 | 整個呼叫的 HTTP 上限（與 SDK 其他處相同）。 |
| `auto_start_backend` | `True` | 若未運行任何後端，則啟動本地後端。對於遠端 `backend_url` 絕不會啟動後端。 |

> **池中已開啟的感測器不會顯示。** 已連線的 BLE 周邊裝置會停止廣播，且無法偵測已開啟的 COM 埠，因此發現功能僅列出 *可供連線* 的裝置。 剛連接完設備後出現空結果是預期的 — 若要取得已連接的設備，請使用 `list_daq_sensors()`。無法執行掃描的傳輸層（未安裝 bleak / zeroconf）會被跳過而非觸發錯誤，因此未安裝藍牙的機器仍會收到其 USB 和 ETH 的回應。

### 程式碼片段

```python
for s in chloros_sdk.list_daq_sensors():
    print(s["sensor_id"], s["model"], s["transport"], s["wavelength_range"])
```

### 哪個 DAQ 饋送哪台攝影機 — `daq_bindings()`

擷取影像的反射率，以及其儲存的 `.daq` 檔案，會使用綁定至該相機或陣列的 DAQ；否則則使用後端系統中最新的 DAQ。GUI 中的光感測器選取功能與 `chloros-cli daq pool-bind` 會設定此參數； 以下亦同：

```python
chloros_sdk.set_daq_bindings(cameras={"214701281": "COM7"},          # a sensor id or an address
                             arrays={"array-1779862544497": "daq-e-def330"})
chloros_sdk.set_daq_bindings(cameras={"214701281": None})            # None clears it
for serial, b in chloros_sdk.daq_bindings()["cameras"].items():
    print(serial, b["binding"], b["source"], b["resolved"]["in_pool"])
```

`daq_bindings()` 會傳回 `{"cameras": {serial: b}, "arrays": {array_id: b}}`：每個 `b` 皆包含 `binding`（設定的值）、`source`（`project`、 來自該專案的 `sensors.json`，或 `runtime`）、`known` / `known_as` （無論是相機或陣列金鑰，Chloros是否知曉其名稱，以及如何知曉：`connected`、`project` — 已開啟專案的 `cameras.json` ——或者，若為攝影機，則為 `seen`：先前曾在此電腦上開啟過；`known: False` 則代表未命名任何項目的金鑰）以及 `resolved` （其目前映射到的池感測器：`in_pool`、`sensor_id`、 `address`、`model`、`streaming`）。執行時綁定優先於專案的；專案檔案並不會被修改。斷開相機、陣列或 DAQ 的連接時，該硬體的所有綁定都會被解除，無論是由誰設定的（陣列成員的綁定會隨陣列一同解除）。

`set_daq_bindings()` 僅會綁定已連接、位於當前開啟的專案中或先前曾在此電腦上開啟過的攝影機，以及已連接或位於當前開啟的專案中的陣列（或已綁定的鍵）。任何其他鍵都會觸發 `ChlorosConnectError` 並以該鍵命名 （`400`、`code`、`UNKNOWN_BINDING_TARGET`，以及 `unknown` 鍵），且該呼叫中**沒有任何**內容會被綁定。 `None` 會清除任何鍵值。

### 環境 + 目標反射率 — `light_sensor_reflectance()`

兩個匯總的數據擷取模組（DAQ），一個朝上觀測天空（環境），另一個朝下觀測目標（物體），可根據其最新讀數提供目標的反射率與植被指數：即 GUI 中的「環境 + 物體」配對，以及 `chloros-cli daq reflectance`，無需再次開啟感測器。 兩者必須均已連接並處於資料串流狀態。

```python
r = chloros_sdk.light_sensor_reflectance("daq-e-def330", "daq-e-54b774")
print(r["reflectance_type"])                       # "Apparent Reflectance" when both are calibrated
for name, band in r["bands"].items():
    print(name, band["rho"] if band["valid"] else band["reason"])
for idx in r["indices"]:
    print(idx["name"], idx["value"])
```

```python
light_sensor_reflectance(
    ambient_sensor_id, object_sensor_id, *,
    bands=None,          # [{"name", "center_nm", "fwhm_nm", "role"}]; role blue / green / red / nir
    preset=None,         # a preset id from daq_band_presets(); default: the suggested one, else "GENERIC"
    expressions=None,    # custom indices: [{"name": "MyNDVI", "expr": "(NIR_850 - Red_660) / (NIR_850 + Red_660)"}]
    builtins=True,       # include the built-in indices
    backend_url="http://127.0.0.1:5000", timeout=30.0,
) -> dict
```

- **波段。** 每個波段的反射率即為該波段- 積分輻照度。`daq_band_presets(filter_code=None)` 列出了波段集：每個 LATTICE M3C 濾鏡對應一組（`RGN`、`OCN`、 `NGB`，即相機本身的波段模型，因此 DAQ NDVI 與相機 NDVI 代表相同的數值），以及 `GENERIC` 寬頻波段。 其 `suggested` 為預設值，對應 `filter_code`；否則即為首台連接相機的濾光片 （由 `suggested_from` 指定）。
- **回傳** `bands` （按頻段分別為 `rho`、`valid`、`reason`、`e_ambient`、 `e_object`)、`indices`（內建-ins），`custom`（各配有 `value`，或 `valid: False` 及 `error`），`invalid_bands`、`reflectance_type`、`preset` 以及這兩組讀數的 `timestamps` （`ts_ns`）。環境感測器無法測量的區間（超出其量測範圍或處於噪聲底限）將被視為無效，並標註原因；所有依賴該數值的指標均予以省略，絕不會以零為基準進行計算。
- **表觀值或相對值。** 依比例尺而定的指標（SAVI、OSAVI、EVI、MSAVI2、RDVI、TDVI、LAI、NLI、MNLI、FCI、GEMI）需要絕對比值，因此僅在兩組感測器均經過功率校準時才會產生 （每個感測器皆具備 `calibration_bundle_sha`）：`"Apparent Reflectance"`。否則，結果將為 `"Relative Reflectance"`，僅包含尺度不變的指標。
- **兩個不同的波長網格** 會根據其重疊部分進行比較：環境感測器的數據點位於物體的測量範圍內，物體的光譜則被插值到這些點上。
- **必須是兩個感測器，絕不能只有一個。** 若將單一感測器同時命名為兩個（無論採用感測器池接受的任何拼寫方式，只要 ID 相同），將引發錯誤 `ChlorosConnectError`：感測器對自身的讀數在每個波段皆為 1.0，這表示無法測量任何數值。 後端也會拒絕此操作（`POST /api/daq/indices` 回應 `400`， `code` `SAME_SENSOR`，而SDK傳送的 `ambient_sensor_id` 和 `object_sensor_id` 實為一個彙總感測器，例如其 ID 及其連接埠）。

### 與 CLI 及其他腳本的共存機制

後端的 DAQ 感測器池由該機器上的每個 CLI 和 SDK 客戶端共用。若感測器已存在於池中 — 無論是由 `chloros-cli daq pool-connect` 或其他腳本開啟 — 從 Python 呼叫 `connect_daq_sensor(port="COM3")` 將傳回一個借用的處理程序：其 `disconnect()` 為無操作（no-op），因此當範圍（scope）退出時，您的腳本不會將感測器從最初開啟它的客戶端手中奪走。 同樣的原則也適用於 `sensor_id=`（可透過 `:` 或 `-` 指定，兩者皆可），針對由主機名稱開啟時以 IP 命名（或反之）的 DAQ-E 感測器，該感測器在開啟時是以 IP 位址命名但由主機名稱呼叫（或反之），以及當後端回應時，若其已持有該感測器而觸發的零標誌智慧偵測——`with` 區塊僅會釋放其自行開啟的感測器。

Chloros GUI **不**屬於此資源池：其 DAQ 面板會自行建立與感測器的連線，因此 `connect_daq_sensor` 絕不會借用 GUI 已開啟的感測器。 請勿同時從 GUI 和腳本使用同一個感測器。GUI 所佔用的 DAQ-U 或 DAQ-M 通常會無法開啟， 但 DAQ-E 會接受第二個連線：腳本的 connect 指令會變更 GUI 下的裝置擷取設定，而其 `disconnect()` 會停止裝置的串流，這將終止 GUI 的即時預覽及其正在進行的任何錄製。請先在 GUI 中關閉該感測器。

### PTP 主時鐘 — `time_sync_status()`

後端執行 PTP 主時鐘，LATTICE 攝影機和 DAQ-E 裝置皆由此取得時間戳記。它會依需求啟動：`connect_camera()`、 若 PTP 主時鐘尚未運行，則由 `connect_array()`、`connect_daq_sensor()` 及 `connect_all()` 啟動它 ——包括隨後失敗的連線嘗試，例如 `connect_all()` 嘗試連線至不存在的 DAQ-E 裝置——此外，`test_array_sync()` 以及所有為單一指令開啟攝影機的呼叫 （`capture_cameras_transient()`、`start_multi_camera_stream()`、對齊呼叫、`capture_live_index()`）。

- **通報位置。** 在連接相機的有線網路介面卡上（包含一個 `169.254.x.x` 鏈路本地位址），否則若預設路由所用的介面卡是有線的，則在該介面卡上。絕不會在 Wi-Fi 或虛擬介面卡（Tailscale、WSL）上：因為其後方沒有相機或 DAQ-E。 若無此類網卡處於運作狀態，它將在無特定網卡上進行廣播（`health` `NO_SOCKETS`），並在下一台相機或 DAQ 連線時，自動綁定該網卡。
- **連線中斷後。** 當攝影機介面卡失去連線並恢復連線時（例如拔除纜線、更換交換器、USB 介面卡重新枚舉），主控器會自行重建套接字，無需重新啟動， 並重新檢查所有已連接的 DAQ-E，若發現有裝置因連線中斷而卡住，便會重新啟用該裝置。此外，當其所服務的所有攝影機和 DAQ-E 持續靜默 15 秒時，主控器也會重新建立連線。
- **是否能連線到實體線路？** 請參閱 `time_sync_status()`（`GET /api/time-sync/status`）。`state` 僅顯示主控器執行緒是否正在運行；`health` 則顯示其封包是否成功傳出： `OK`（未偵測到故障）、`STOPPED`、`NO_SOCKETS`、`SENDS_FAILING`、 `DEGRADED`（部分適配器故障）或 `PEERS_SILENT`（其所服務的攝影機與 DAQ-E 裝置停止回應），並在 `health_detail` 中顯示一句說明。 `announce_sent` / `sync_sent` / `delay_resps_sent` 僅統計已發送的訊息； `announce_send_errors`、`sync_send_errors`、`follow_up_send_errors`、`delay_resp_send_errors`、`last_send_error`、 `failing_interfaces` 和 `last_delay_req_age_s` 描述未成功傳送的訊息，而 `rebinds` / `last_rebind` 則描述重建的訊息。 `sync_sends_slow` 統計發送時間超過 200 微秒的同步操作，而 `sync_send_max_us` 則代表耗時最長者；每個 Follow_Up 的發送時間最多計入 200 微秒。 主節點在獨立的進程中運行（`own_process`、`process_pid`），在Windows系統上具有高優先級（`process_priority`）， 因此，繁忙的後端無法延遲時間戳記，也無法一次性調整所有攝影機的時鐘；若該程序無法啟動，則會如以往般在後端內部運行（`own_process` false）。

```python
st = chloros_sdk.time_sync_status()
if st.get("health") not in (None, "OK", "STOPPED"):
    print("PTP:", st["health"], "-", st.get("health_detail"))
    chloros_sdk.restart_time_sync()
```

| 函式 | 功能說明 |
| --- | --- |
| `time_sync_status()` | 主時鐘的狀態、健康狀態、時鐘識別碼、優先級及訊息計數器（如上所述）。`chloros-cli time-sync status`。 |
| `time_sync_peers()` | 其偵測到的 PTP 從屬時鐘 （攝影機、DAQ-E）：`peers`、各 `port_identity`、`last_seen_ip`、`last_delay_req_age_ms`、 `delay_req_count`。`time-sync peers`。 |
| `time_sync_cameras()` | 每台已連接的陣列攝影機的 `PtpStatus`、 `PtpOffsetFromMaster` 及 `PtpMeanPathDelay`（若相機韌體中無 PTP 節點，則僅顯示其序列號）。`time-sync cameras`。 |
| `restart_time_sync()` | 停止並重新啟動主節點；在 (`daq_ptp_rechecked`) 之後，會重新檢查每個彙總的 DAQ-E 的 PTP。當其 無法恢復時觸發 `ChlorosConnectError`。`time-sync restart`。 |
| `set_time_sync_priority(priority1=None, priority2=None)` | 其 PTP 優先級（0-255，數值越低越容易在最佳主節點選舉中勝出），適用於存在其他 PTP 主節點的網路。需確保該主節點正在運行。`time-sync set-priority`。 |

各自接受 `backend_url=` 和 `timeout=` 參數，並啟動尚未運行的本地後端。

---

## 攝影機實用工具

`chloros-cli lattice` 除了連接與擷取之外，還具備 `chloros_sdk`（模組 `chloros_sdk.camera_utils`）中普通函式所沒有的功能。每個函式都會呼叫 CLI 指令所使用的後端路徑，並遵循 CLI 的規則， 因此兩者的運作方式相同。它們接受相機的 **序列號**（而非連線），並使用與智慧連線呼叫相同的 `backend_url=` / `timeout=` 關鍵字； 若後端尚未執行，則會像 `connect_camera()` 一樣啟動該後端。若呼叫遭拒，則會觸發 `ChlorosConnectError` 並回傳後端的拒絕原因。

```python
import chloros_sdk as c

c.set_camera_color_profile("214701281", profile="natural")
c.set_camera_highlights("214701281", -40)
print(c.camera_firmware_status()["cameras"])
st = c.network_adapters(estimate=True, cameras=4, pixel_format="BayerRG12")
```

| 函式 | 功能說明 |
| --- | --- |
| `camera_color_profile(serial)` / `set_camera_color_profile(serial, profile=, custom_cct_k=)` | 顯示器色彩設定檔： `raw`、`linear`、`natural`、 `enhanced` 或 `custom_temp`，以及其色溫（限制在 2000-10000 K 之間；當配置檔為 `custom_temp` 時讀取）。 |
| `camera_color_adjust(serial)` / `set_camera_color_adjust(serial, saturation=, contrast=, reset=False)` | 顯示飽和度與對比度，各為 0-200 %（100 表示維持原狀）；`reset=True` 會將兩者均重設為 100。 |
| `camera_highlights(serial)` / `set_camera_highlights(serial, value)` | 降低顯示器渲染中最亮、最白的像素：-100..0（向零截斷，然後進行限制； 0 = 關閉)。 |
| `camera_white_balance_once(serial, settle_s=None)` | 單次白平衡：相機短暫串流影像，待數值收斂後，返回 `red_ratio` / `blue_ratio`（若無法回報，則 `ratios_read` 為 False）。它會保留已求得的白平衡；其開放式介面寫入的其他所有設定，均會以原始狀態回傳。 |
| `set_camera_white_balance(serial, mode=None, red=None, blue=None)` | `mode="auto"` （連續）或 `"off"`；或固定白平衡比例 `red` / `blue`，這些設定會優先將自動模式關閉。 相機拒絕的比率會以 `errors` 為名稱。|
| `camera_presets()` / `apply_camera_preset(serial, preset)` | 相機預設值，套用其中一項時：相機將維持該預設名稱所對應的設定 （回傳值為 `settings`，因相機隨後會讀取這些設定，例如其運作的幀率；若預設值未命名某個場，則回傳 `None`），而其開放寫入的其他所有設定則恢復為原始狀態。 `"default"` 會將攝影機重設為出廠預設值（即出廠設定，包含自由運行的自動曝光、關閉鏈路吞吐量限制與缺陷校正，並保留RGB攝影機的校準白平衡），且不會儲存至其記憶體中： 回應中包含 `factory_defaults: True` 以及 `settings` 的所有欄位。 在 `camera_presets()` 中，`default` 即為 `{factory_defaults: True, description}`。 |
| `camera_filter_models()` / `camera_filter_model_info(name)` | 濾光片模型，以及其中一個模型的詳細資訊（波段、光譜響應條件數）。相機的濾光片模型是從其型號字串中讀取的；並非手動設定。 |
| `camera_firmware_status(serial=None)` | 每台相機的的韌體：`current_version`、`available_version`、`update_offered`、`needs_minimum_update` 以及 `minimum_firmware`。 |
| `update_camera_firmware(serial, force=False)` | 將此 Chloros 版本的韌體燒錄至一台攝影機：恢復出廠設定、燒錄、重新開機，Chloros 基準設定將被覆寫； 相機的校準設定將被保留。需時數分鐘；相機必須保持通電。`force=True` 用於重新燒錄現有相機的韌體。 |
| `camera_power_probe(serial)` | 相機的電源與熱感測節點及其溫度。 |
| `sleep_camera(serial)` / `wake_camera(serial)` | 相機的低功耗閒置模式（相機將保持休眠狀態直至被喚醒： 其鏈路吞吐量限制已啟用（`DeviceLinkThroughputLimitMode` `On`）且設定為最低值，節能模式已啟用（若相機具備此模式），並顯示啟用前後的溫度； 喚醒後將鏈路恢復至休眠時的狀態（包含傳輸速率上限、節能模式及其封包延遲 `GevSCPD`——當傳輸速率上限啟用時，攝影機會將此值維持為 0；相關記錄會保留在此電腦上，因此後續後端進行喚醒時也能取得此資訊）， 或者，針對這台未由本電腦置入睡眠狀態但看似已進入睡眠狀態的攝影機（其限制值設定在底限），關閉限制功能，並將 `GevSCPD` 維持在 0（出廠設定）； 若限制值設定在閾值以上，則維持原狀。僅保留電源狀態；其他所有設定項的寫入值均恢復為原始狀態。 |
| `calibrate_camera_with_target(serial, filter=None, attempts=30, settings=None)` | 持續擷取幀，直到MAPIR校準目標進入視野，並從其面板中擬合各通道的係數：`target_id`、`panel_dns`、 `coefficients`。 |
| `network_adapters(estimate=False, cameras=1, pixel_format="BayerRG8")` | 此電腦用於 GigE Vision 的乙太網路介面卡設定（MTU、鏈路速度、巨型幀）以及任何 `issues`； `estimate=True` 會新增 `cameras` 相機所能獲得的幀率與頻寬。 |
| `fix_network_settings()` | 將 GigE Vision 適配器設定套用至這台電腦的的相機轉接器。此操作會變更轉接器的設定，且僅在以系統管理員身分執行 Chloros 時方能運作；否則將顯示偵測結果的日誌並終止執行。 |
| `apply_vignette_correction(input_dir, output_dir, lens_model=None)` | 對資料夾中的影像進行暗角校正，每張影像皆使用其對應鏡頭的平場圖：`processed`、 `lens_models`（用於校正該檔案的鏡頭）、`skipped`（附帶校正原因）。若未指定 `lens_model`，則各檔案的鏡頭資訊將取自其 EXIF 型號欄位； `lens_model` 適用於「型號」欄位未指定相機的檔案，若檔案的「型號」與此設定相牴觸，則該檔案將被跳過。僅限檔案。 |
| `read_dls(mac, filter=None, camera_serial=None)` | 來自 DLS（藍牙下行光感測器）的一組讀數： 光譜、各波段輻照度，以及透過相機自身響應獲得的波段幅度（當提供 `camera_serial` 時）。`discover_daq_sensors(transports=["ble"])` 用於查找 MAC 位址。 |
| `probe_system(pixel_format="BayerRG12", discover=True)` / `system_info()` | 主機的相機硬體及其建議的配置（`report_text`），以及系統的 CPU / GPU / PyTorch 執行環境。 |

- **部分主機需要獨佔相機。** `camera_white_balance_once()`, `apply_camera_preset()`、`sleep_camera()` / `wake_camera()`、`calibrate_camera_with_target()` 及 `update_camera_firmware()` 會開啟 開啟相機。當Chloros正在控制該相機時（無論是透過應用程式連線、作為陣列成員與`connect_camera()`連結，或是目前正被其他指令使用， 例如陣列測試或擷取指令正在開啟其相機），後端會以 HTTP 409 `CAMERA_HELD` 狀態碼拒絕請求（在 `code` 和 `error_code` 兩者皆然），並說明如何釋放該資源。色彩設定、高光、`camera_power_probe()` 以及讀取功能在已連接的攝影機上均可正常運作。
- **色彩、高光和白平衡僅適用於 RGB 攝影機。** 在單波段單色 (M3M) 攝影機上，這些設定函式不會進行任何變更，並會返回 `{"skipped": True, "reason": ...}`，與 CLI 的行為相同。這些屬於顯示設定：僅適用於即時預覽與顯示層級的擷取畫面，絕不涉及輻射度或反射率。

---

## 暫存陣列、對齊配置檔與索引

在沒有持久連線的情況下，`chloros-cli lattice` 的運作方式與 `chloros_sdk`（模組 `chloros_sdk.array_utils`）中的函式相同：`continuous`、`multi-capture`、`multi-stream`、`multi-test`、`multi-detect`, `analyze-array`、`align-calibrate`、`align-apply`、`align-stream`、`align-info`、 `align-reorder` 以及 `index`。每個指令都會傳送CLI命令所發出的請求，並採用CLI的預設值。

```python
import chloros_sdk as c

prof = c.calibrate_alignment_profile(["214000535", "214000551", "214000556"],
                                     "D:/rig/alignment_profile.json", method="orb", model="rigid")
print({sn: t["reprojection_error"] for sn, t in prof["transforms"].items()})

c.capture_aligned_multiband("D:/rig/alignment_profile.json", "D:/rig/out")
idx = c.capture_live_index("D:/rig/alignment_profile.json", preset="NDVI",
                           output_dir="D:/rig/ndvi", capture_level="reflectance")

with c.start_multi_camera_stream(["214000535", "214000551"], "D:/rig/stream", fps=5, count=50) as job:
    print(job.wait()["final_stats"])          # waits for the 50 groups, then closes the cameras
```

- **它們會自行開啟攝影機。** 每個指令都會開啟其對應的攝影機、執行相關作業，然後關閉攝影機，因此需要使用Chloros未持有的攝影機（在應用程式中已連線， 透過 `connect_camera()`，或作為陣列成員）的攝影機；後端會以 HTTP 409 狀態碼 `CAMERA_HELD` 拒絕並說明如何釋放一個。暫時性開啟需要一段時間（幾台攝影機約需 40 秒）。對於重複擷取， `connect_array()` 執行一次即為最快速的路徑，且適用於所有匯出類型；而暫存擷取則會儲存去拜爾化後的影像。`calibrate_alignment_profile()` 為例外情況：在已連線的該類相機陣列上，它會從陣列自身的幀中進行解析 （`used_live_array`）。
- **`serials`** 為待開啟的攝影機，首個會觸發扳機；`None` 則會開啟網路上的所有攝影機。
- **`settings`** 是一個用於覆寫攝影機設定的字典，其中各欄位由CLI的旗標設定：`exposure_time` (微秒)， `gain` (dB)、`pixel_format`、 `width`、`height`、`target_brightness`、`ae_damping`、`ae_upper_limit`、 `frame_rate`、`gev_packet_size` 以及 `preset`（一個以名稱命名的相機預設）。設定曝光值或增益值會關閉自動控制功能，其作用與標誌參數相同。
- **`gpu`**：`True` 強制使用 GPU 變形，`False` 則強制使用 CPU （`--gpu` / `--no-gpu`）；`None` 則讓後端自行選擇。
- **`keep_settings`**/**`clear_settings`**（所有開啟攝影機的函式，預設為 `False`）：執行結束時，每台攝影機都會恢復至初始狀態；`keep_settings=True` 則會將執行過程中的設定儲存至攝影機自身的記憶體中（自由運行模式）， `clear_settings=True` 則將該記憶體重置為出廠設定 — 請參閱 [攝影機記憶體](#camera-memory--keep_settings--clear_settings)。 回覆（針對某項工作，即 `stop()` 的）會包含 `camera_memory`。
- **路徑屬於後端**：當後端位於本機時，相對路徑會以您的工作目錄為基準轉換為絕對路徑。對齊配置檔功能（`alignment_profile_info()`、`reorder_alignment_profile()`）會像CLI一樣，從本電腦讀取該檔案。

| 函式 | 功能說明 |
| --- | --- |
| `start_camera_continuous(serial, output_dir="output", *, format="tiff", jpeg_quality=95, queue_depth=64, settings=None)` | 持續錄製一臺攝影機的畫面，直到手動停止為止。 返回一個 `CliJob`。一次 `tiff` 執行所產生的幀，即為 `raw` 檔案，該檔案由陣列擷取功能針對該攝影機寫入（以感測器的原始值，未經壓縮）； `png` 和 `jpg` 各佔 8 位元，因此這些執行結果為可供檢視的去馬賽克幀。綁定或匯集式 DAQ 會將執行中的每幀寫入一個 `.daq` 檔案，存於該執行結果的的 `daq/` 資料夾中寫入一個 `.daq` 檔案（其統計資料中顯示為 `daq_files`）。每個幀都會連同完整的元資料（型號、擷取時間、鏡頭、波段、原始幀校正）寫入一次； 當所有已擷取的幀皆寫入後（最長 25 秒），`stop()` 會返回一次，此時才會關閉相機。若相機仍處於硬體觸發待命狀態 （`TriggerMode` `On`）的相機，在本次運行期間將設定為自由運行模式（起始回應中的 `trigger` 會說明實際偵測到的狀態）。 在停止時，攝影機將恢復至被發現時的狀態——包括其觸發設定以及運行期間變更的所有設定——除非傳回 `keep_settings=True`（將運行設定儲存至攝影機自身的記憶體中，即自由運行模式）， 或使用 `clear_settings=True` 將該記憶體重置。若某次運行未錄製任何幀，系統將在 10 秒後透過 `stop_reason` `no_frames` 自動終止運行，並將原因記錄於 `last_error` 中記錄原因，而其 `stop()` 會將該原因回傳為 `error`。因寫入器無法跟上而遺漏的幀會被計數，並透過 `drop_reason` 說明原因。對於 Chloros 持有的攝影機，請使用 `CameraSession.capture_repeated()`。`lattice continuous`。 |
| `start_multi_camera_stream(serials=None, output_dir="output", *, fps=3.0, count=None, format="tiff", jpeg_quality=95, settings=None)` | 於 `fps`，針對 `count` 組別，或直至停止為止。回傳一個 `CliJob` （其 `start_reply` 包含 `n_cams` 及線路 `pacing`）； 其最終統計數據包含已擷取、已寫入及已捨棄的群組數量、寫入器無法寫入的檔案，以及相機未傳送任何資料的群組。 `lattice multi-stream`。 |
| `capture_cameras_transient(serials=None, output_dir="output", *, format="tiff", jpeg_quality=95, settings=None)` | 一次同步擷取，每台相機一張去拜爾化影像：`results` (`filepath`、`serial`、`frame_id`) 以及 `pacing`。 若某組缺少一台攝影機，系統會先顯示已儲存的檔案數量。未連接陣列的 `lattice multi-capture`。 |
| `test_array_sync(serials=None, count=10, ptp_settle_s=None, timeout=None)` | `count` 以固定曝光值進行同步擷取： `sync_groups`、`master`，每觸發一次對應一行 `group_timestamps`（若某台相機漏拍一次，該行即為空），以及 `sync`：每台相機相對於主相機的偏移量，以每次觸發為單位（`offsets_us`：每台相機分別為 `mean_us`、`min_us`、 `max_us`、`jitter_us`、 `groups`)，以單一時鐘為基準（`clock_domain`：`ptp`，或 `device` — PTP 關閉，每台相機使用自身的運行時間時鐘，因此其偏移量取決於開機時間，僅透過抖動來測量同步）。若觸發標記同時包含裝置時鐘與 PTP（PTP 關閉或仍在鎖定中），則該觸發將被排除並計入（`groups_mixed_domain`）；同樣地，若 PTP 時鐘與主時鐘的偏差仍大於 `agree_within_us` (`groups_clocks_disagree`)；`notes` 即為此說明。`clock_offset_mean_us` / `clock_offset_jitter_us` 分別為每台相機的最大平均值與抖動值。 當觸發訊號遺漏、單一時鐘下所有攝影機的觸發次數少於兩次，或時鐘從未同步時，`success` 會為 false，而 `error` 則會為 true (`sync.problems`)；無論何種情況，這些資料列都會被回傳。 若 PTP 主時鐘尚未運行，則會啟動它（`time_sync` 表示 `grandmaster_running` 和 `grandmaster_started_by_this_test`），並開啟每台攝影機的 PTP（如同 `connect_array()` 所做），且在首次觸發前，會等待每台攝影機的 PTP 時鐘鎖定至主時鐘並穩定下來： `PtpStatus` 作為某個主時鐘的從時鐘，與其時間差在 100 微秒內且穩定維持 3 秒，且每台攝影機均以這種方式鎖定， 且無中斷，持續 120 微秒——約自上電起 2 分鐘。剛鎖定時，時鐘在讀取「從屬」狀態後的約 2 分鐘內，誤差可能高達 ~200 微秒，而相機自身的 `PtpOffsetFromMaster` 並不會顯示此狀況； 經過 120 秒後，相機的時鐘誤差將收斂至約 10 微秒。若某台相機自先前測試結束後仍保持鎖定狀態（且 PTP 功能保持開啟，此為後端所觀察到的狀況），則計時將從該次鎖定時刻開始計算， 因此連續進行的測試無需再次等待；若測試中才開啟的相機，或是被偵測到 PTP 已開啟但此後端尚未見其鎖定（例如在其他處開啟，或於後端重新啟動前），則需等待 120 秒；在包含這兩類相機的陣列中，剛鎖定的相機仍需等待其指定時間。 若某台攝影機離開從屬狀態、超過 100 微秒，或在等待期間連續跟隨另一台主時鐘進行 3 次讀取 （1.5 秒）期間，將重新開始計時；此類讀取會記錄一至兩次（`[array-sync] PTP SN:<serial> off lock at <s> s (<n> of 3): <status> <±x> us gm <id> -- locked <s> s, counting on`），計時隨後繼續，而無法讀取的主導節點 ID 將被視為未知，而非狀態變更。 若所有攝影機的計數在同一時間（一秒內）同時重置，表示主節點的時間在移動，而非攝影機的時間：後端日誌顯示為 `[array-sync] PTP grandmaster step: all <n> cameras restarted at <s> s …`，`ptp_lock` 將其列於 `grandmaster_steps` 中，而隨後超時的等待會以 `every camera's count restarted together at <s> s -- the PTP grandmaster's time moved under them …` 開始記錄錯誤。其時間限制並未因此延長。`ptp_settle_s`（0-600；若無穩定等待則為 `0`）設定此執行程序的穩定時間；若設定其他數值將遭拒絕（HTTP 400）。 該等待的上限為結算時間加上 90 秒（預設為 210 秒，至少 90 秒），且呼叫會等待後端 `timeout` 秒：預設為 240 秒加上結算時間（360 秒）， 這與 `chloros-mcp` 的 `test_array_sync` 行為一致。`ptp_lock` 包含 `converged`、 `waited_s`、`timeout_s`、`settle_s`、`locked_s` （每台攝影機已鎖定並保持穩定的時間長度）、`found_settled`（被偵測時，攝影機已鎖定達穩定時間）以及，針對每台攝影機，`status`、`offsets_ns`、 `slave_s` （已鎖定時間；「無」表示未鎖定）以及 `remaining_s`（若無任何攝影機的 PTP 會開啟，則顯示「無」）。後端日誌每 5 秒記錄一行，包含每台攝影機的狀態、已鎖定時間以及剩餘時間。 每台攝影機開機時 PTP 皆處於關閉狀態；隨後時鐘約在 10 秒內完成鎖定。之後，每台攝影機將恢復至測試發現時的狀態——其 PTP、自動曝光及增益均維持測試期間設定的「關閉」狀態 （`ExposureAuto` / `GainAuto`，以及手動設定的 `ExposureTime` / `Gain`）， 其餘情況則不作調整，除非為 `keep_settings=True` —— 因此，若某台相機的 PTP 仍處於開啟狀態（`PtpEnable=True`，或執行 `keep_settings=True` 的測試） 將保持鎖定狀態，且一旦此後端偵測到該相機已鎖定並維持穩定時間，後續測試便不會等待它。若時鐘未能穩定，在任何觸發發生前 `success` 即為假，而 `error` 則說明應採取的行動： 從未鎖定的相機（`not_slave`）表示主節點無法與其建立連線（`time_sync_status()`）；反覆斷線的相機則以其鎖定時間命名 (`remaining_s`)；時鐘仍在收斂中 (`far`)：請檢查 `time_sync_status()`，若要在測試前讓它們鎖定，請開啟 PTP 並保持開啟 (`PtpEnable=True`，每台攝影機皆然，或 `keep_settings=True`；測試會將設定恢復為原始狀態，因此會保持開啟)，然後執行測試。`lattice multi-test`。 |
| `detect_array_master(line=None)` | 同步線路上的攝影機與預設主機：`master_serial`， `slave_serials`，以及被偵測的 `line`。`lattice multi-detect`。 |
| `analyze_array_models(camera_models=None, *, width=2048, height=1536, n_active=None, force_tier=None, binning=1, packed="auto", pixel_format=None)` | 這些攝影機型號（產品名稱、 `"LATT-M3C-L87-FOCN"`）會出現在這台電腦的網路中：`tier`、`achievable_fps_max`、線路預算（`max_cams_collision_safe`、 `oversubscribed`）、`cam_caps` 以及 `fix_suggestions`。預設情況下，系統會列出已偵測到的攝影機型號； 目前未開啟任何攝影機。關於您擁有的攝影機及其連線所需的設定，請參閱 [`analyze_array_network()`](#network-analysis-helper)。`lattice analyze-array`。 |
| `calibrate_alignment_profile(serials=None, save_path="alignment_profile.json", *, frames=3, method="orb", model="rigid", reference_serial=None, checkerboard=None, control_points=None, preview=False, vignette=False, roi=None, mask_path=None, rms_threshold_px=None, per_cam_overrides=None, settings=None, ...)` | 根據即時影像幀建立對齊配置檔並儲存。 `method`：`orb`、`akaze`、`phase`、`checkerboard` （搭配 `checkerboard="9x6"`，即內角）或 `manual`（搭配 `control_points={serial: [[x, y], ...]}`，參考攝影機亦同）。`model`： `translation`、`rigid`、`affine` 或 `homography`。 更細緻的選項則是CLI的旗標：`max_features`、`ratio_threshold`、`matcher`（`bf` / `flann`）、`knn_k`、`prefilter`（`none` / `gradient` / `clahe` / `blur` / `hist_match`)，`ransac_threshold_px`， `ransac_max_iters`、 `ransac_confidence`、`averaging`、`lock_rotation`、`lock_scale`、`lock_axis`； `per_cam_overrides` 即為 `{serial: {option: value}}`。`rms_threshold_px` 拒絕儲存較差的解。 回傳 `save_path`、`output_width` / `output_height`、`crop_roi`、 `reference_serial`、`transforms`（每台相機分別為 `reprojection_error`、 `num_matches`）以及 `preview_path`。`lattice align-calibrate`。 |
| `capture_aligned_multiband(profile_path, output_dir="output", *, serials=None, format="tiff", bit_depth=None, gpu=None, no_crop=False, bands=None, order=None, vignette=False, per_band=False, per_camera=False, interpolation="linear", border_mode="constant", border_value=0, settings=None)` | 透過該配置檔產出的一張對齊多波段影像：`files`（該多波段TIFF檔及其`bands`與尺寸， 加上每波段各一份（`per_band`）及每台相機各一份（`per_camera`））以及 `vignette_corrected`。 `bands` 篩選頻段，而 `order` 則依名稱（清單或逗號分隔字串）排序頻段。`lattice align-apply`。 |
| `start_aligned_stream(profile_path, output_dir="output", *, serials=None, fps=3.0, count=None, ...)` | 流對齊多頻段影像（與 `capture_aligned_multiband()` 相同的變形選項），適用於 `count` 或直至停止為止。 返回一個 `CliJob`；其最終統計數據包含 `frames_written` 及任何 `frame_groups_incomplete`。`lattice align-stream`。 |
| `alignment_profile_info(path)` | 摘要為 `align-info` 的設定檔會輸出以下內容：`name`、`created`、`method`、`reference_serial`、 `sensor_resolution`、`output_width` / `output_height`、`crop_roi`、`camera_serials`、 `transforms` （每台攝影機 `model`、`reprojection_error`、`num_matches`、`reference`）, `camera_models`, `layer_order`（每台攝影機分別為 `band_names`、`wavelengths_nm`、`enabled`）；`profile` 即為完整檔案。 |
| `reorder_alignment_profile(path, order=None, enable=None, disable=None)` | 就地修改設定檔：`order` 為頻段名稱（每個名稱會將其對應攝影機的圖層向上移動，依此順序）， `enable` / `disable` 為序列號。返回新的 `layer_order`。`lattice align-reorder`。 |
| `index_catalog()` | 索引預設值（`presets`：分別為 `name`、`channels`、 `formula`）及顏色 `gradients`. `lattice index --list-presets` / `--list-gradients`。 |
| `compute_index(input_path, *, preset=None, formula=None, channel_map=None, output=None, capture_level="unknown", output_format="all", colorize=False, gradient="RdYlGn", vmin=None, vmax=None, percentile=None, bg_mode="clip")` | 對齊的多波段 TIFF 檔案的索引：包含 `preset` (`NDVI`、`NDRE`、 ……）或 `formula` 對應於符號 `channel_map` 所映射的頻帶（`{"NIR": "NIR_850", "Red": 0}`，依名稱或索引）。預設的符號 `channel_map` 會省略並選取該檔案中名稱以此開頭的單一頻段（`red` → `Red_660`， `nir` → `NIR_850_RGN`），其名稱由檔案本身決定（其波段清單，或Chloros擷取資料／產品的 `Chloros:Band{R,G,B}Name`）， 因此，Chloros反射率擷取影像中的NDVI無需使用`channel_map`； 若兩個波段對應同一個符號，系統會直接拒絕該組合，而非進行推測。`capture_level` 說明波段所包含的內容：`reflectance` 或 `radiance` 則使數值具量化意義； 未經校準的影像堆疊（`align-apply` 所指）會產生定性值； `"unknown"` 從檔案中讀取資料，包含Chloros檔案中的 `Chloros:ProcessingLevel`。回傳 `stats` （形狀、1–99 百分位數範圍、LUT 範圍、所使用的 `capture_level`）以及寫入輸入旁 （`<input stem>_<preset>.tif`、`..._LUT.png`）旁，或寫入 `output` 處，即路徑：若檔案名稱帶有圖像副檔名，則以其主檔名作為命名基礎 （`output="D:/out/ndvi_FRGN.png"` 會寫入 `ndvi_FRGN.tif` 和 `ndvi_FRGN_LUT.png`），若無圖像副檔名，則採用前綴命名法 （`D:/out/ndvi` → `ndvi.tif`、 `ndvi_LUT.png`）；從 `files` 讀取路徑。這些檔案包含輸入的相機記錄（製造商／型號／序號、GPS、鏡頭、UTC 拍攝時間、校準識別碼，但絕不包含其 raw-DN 校準）、`Chloros:ProcessingLevel=index`，以及索引作為其單一波段（`XMP-Camera:BandName`，TIFF 與 LUT PNG 皆同）。僅限檔案；無相機資料。`lattice index --input`。 |
| `capture_live_index(profile_path, *, preset=None, formula=None, channel_map=None, serials=None, output_dir="output", output_prefix="index", capture_level="reflectance", save_multiband=False, ...)` | 擷取、校正（`reflectance` 或 `radiance`； `raw` 會跳過此步驟），並在單一步驟中執行對齊與索引計算；`save_multiband=True` 會保留經校正的圖層堆疊，供 `compute_index()` 稍後取用。 返回 `stats`（附帶其所使用的頻段的 `serials`）以及 `files`。 它會開啟 `serials`（第一個會觸發扳機），否則則開啟該配置檔中 `layer_order` 條目已啟用的相機，且參考相機優先 （即使其頻段已停用仍會開啟）；頻段已停用的攝影機絕不會進行校準。`channel_map` 僅在堆疊無法回應某個符號時才需要， 例如在 `compute_index()` 中：預設組的符號會選用該名稱以之開頭且已啟用的單一頻段，此檢查會在任何相機開啟前進行（若無相機或有多台回應，則予以拒絕，並列出其名稱）；自訂的 `formula` 則需針對每個符號進行設定。 `index.tif` 與 LUT 攜帶堆疊中的相機、擷取時間（參考相機的曝光時間，UTC）及鏡頭，兩者皆將名稱作為其唯一頻段（例如 `XMP-Camera:BandName`， `NDVI`：彩色 PNG 檔案以索引作為名稱，而非其色彩通道)。`lattice index --live`。 |

### `CliJob`

這三個串流所返回的處理程序識別碼。該工作在後端執行，並在停止之前持續掌控其相機。

| 成員 | 說明 |
| --- | --- |
| `stats(timeout=10.0)` | `running`、`state`、`stats`（已儲存／ 寫入、丟棄、`fps`、`elapsed`、寫入失敗、`stop_reason`；若為持續性工作，則還包含 `frames_grabbed` 及 `frames_queued` —— 每個擷取的幀皆會被寫入、排入佇列、丟棄或失敗 —— 以及 `drop_reason`、 `writers` 以及 `write_ms`（單一幀的寫入平均耗時），還有 `output_dir`。`state` 為 `running`、`ended`（其已自行停止運作且尚未被停止）、`stopping`（某個停止中的實例正在寫出其佇列： 即時統計資料，以及 `stopped_by`）或 `finished`（其最終統計資料，停止任務的 `error` 和 `stopped_by`）。每個客戶端都會將已停止工作的記錄保留 10 分鐘（最多保留最後 32 筆），另一個腳本或 `chloros-mcp` 會讀取相同的狀態與最終統計資料。一個工作 `chloros-cli` 已啟動 (`lattice continuous`、`multi-stream`、`align-stream`) 當 CLI 停止回報長達 30 秒時（被終止，或其機器進入休眠狀態），後端會停止該工作 — 並透過 `stop_reason`、`client_gone` 及 `stopped_by` 向後端通報； 從此處讀取其統計資料並不能使其保持運行狀態。由 SDK 啟動的任務會持續執行直至被停止。每次啟動都會將其 `timeout` 作為後端的截止時間（`client_deadline_s`）：若在後者（仍處於開機狀態）之後開啟的攝影機被重新置回並關閉，且未啟動任何工作，則該呼叫會觸發 `ChlorosConnectError`（`CLIENT_DEADLINE_PASSED`）。由工作持有的攝影機，其所有其他開啟嘗試 （`connect_camera()`， 其他啟動、暫時性擷取）時，會以 HTTP 409 狀態碼 `CAMERA_HELD` 拒絕，並標明該工作名稱及其啟動者。 |
| `stop(timeout=30.0)` | 停止該工作並關閉其攝影機： `final_stats`、`output_dir`，以及當停止操作未順利完成時發出的 `error`。第二次呼叫會返回相同的回應， 從另一台客戶端發出的停止指令亦然：當第一台客戶端仍在寫出其佇列時發送的指令會等待其完成（最長 25 秒；超過此時間便會觸發異常，最終統計資料將顯示於 `stats()`）。 |
| `wait(timeout=None, poll_s=0.5)` | 等待其自行結束（達到 `count`、磁碟空間已滿或發生錯誤），然後透過 `stop()` 收集最終計數。 `timeout` 會在程序仍在執行時觸發 `TimeoutError`；一個沒有 `count` 的工作，只有在被停止時才會結束。 |
| `job_id`、`kind`、`output_dir`、`start_reply`、 `stopped` | 其識別碼與類型（`continuous`、 `multi-stream`, `align-stream`），其寫入位置、起始路徑的回覆，以及 `stop()` 是否已執行。 |

請將其用作上下文管理器，以便在作用域退出時停止執行。

---

## 專案資料夾與配方

**專案資料夾** 是建立新專案的資料夾，由 Chloros 應用程式、CLI 及 SDK（模組 `chloros_sdk.system_utils`；`chloros-cli project-folder`）共用：

```python
print(chloros_sdk.project_folder())                # the configured folder, else the default
chloros_sdk.set_project_folder("D:/Flights")       # created if missing
chloros_sdk.reset_project_folder()                 # -> {"path": <default>, "was_set": True}
```

若已設定的資料夾不再存在，系統將回退至預設值（`Documents/MAPIR/Chloros Projects`），此行為與應用程式相同。 當後端位於本機時，提供給 `set_project_folder()` 的相對路徑將以您的工作目錄為基準。

**配方** 是指 `chloros-cli project run PATH RECIPE` 針對已儲存專案執行的 YAML / JSON 動作清單。`run_project_recipe()` 則透過 CLI'專屬執行器（隨 SDK 安裝）來執行，因此同一份配方在以下兩種情況下的行為完全一致：

```python
r = chloros_sdk.run_project_recipe("D:/campaign/morning.yaml", "D:/Projects/Field_A")
print(r["ok"], r["actions"])
print(r["log"])                                    # what the CLI would have printed

chloros_sdk.run_project_recipe({"actions": [{"apply": {"name": "FrontLeft", "settings": {"exposure_us": 5000}}},
                                            {"wait": "2s"},
                                            {"capture": {"name": "FrontLeft", "output": "pose_a/", "format": "tiff"}}]},
                               "D:/Projects/Field_A")
```

```python
run_project_recipe(recipe, project_path, *, dry_run=False, echo=False,
                   backend_url="http://127.0.0.1:5000") -> dict
```

- **`recipe`** 是一個 `.yaml` / `.yml` / `.json` 檔案，或是配方本身：一個包含 `actions`（`apply`、 `capture`、`burst`、`stream`、 `sensor`、`wait`) 所組成的字典，並可選地包含 `overrides` 和 `stop_on_error`（預設值為 true： 執行程序將在第一個失敗的操作處停止）。這些操作參照CLI參考文件中的 [Recipe DSL](cli-reference.md#recipe-dsl)；相對路徑 `output` 將根據您的腳本工作目錄進行解析。
- **回傳值** `ok`（所有動作皆成功）、`actions`（成功數量）、`log` （執行器的輸出；`echo=True` 也會在執行時輸出此內容）以及 `validation`。若食譜檔案遺失或無法解析，則會觸發 `ChlorosConnectError`。
- **在執行前已檢查。**`validation` 即為 `{"ok", "errors", "warnings", "steps"}`：每個動作的鍵值必須精確對應一個已知的動詞（`{"capture": {"name": ...}}`， 而非 `{"action": "capture"}` 或 `{"type": "capture"}`），該動詞須指明專案中正確類型的相機、陣列或感測器。若配方出現錯誤，則**不會**執行任何動作（`ok` false， `log` 中的錯誤），無論 `stop_on_error` 如何指定。未使用的鍵、 以及該配方所驅動但目前未連接的任何陣列或光感測器，均會觸發 `warnings`。配方不會連接裝置：請先透過專案的首個（`ChlorosProject.connect_all()`、`chloros-cli project connect`、 或 MCP 工具 `connect_project_devices`）進行連接。`dry_run=True` 不會執行任何操作，並返回 `{"recipe": ..., "validation": ...}`。
- 一個流程中每次僅能執行一個配方。YAML 配方需要 PyYAML。

---

## 專案自動化 — `ChlorosProject`

已儲存的 Chloros 專案是一個資料夾，其中包含 `cameras.json` + `sensors.json` + `project.json`。`open_project(path, relink=None)` 會載入清單檔（`relink`：此為已移動專案中遺失的原始檔目前所在的資料夾，後端載入專案時會使用此資料夾——`process()`——詳見 **已移動的專案** （位於 `ChlorosLocal` 之下）），而 `connect_all` 則會將所有已儲存的攝影機、陣列及光感測器連線至網路，並套用其儲存的設定——此硬體狀態與 GUI 所產生的完全相同。光感測器是透過後端的 [DAQ 感測器池](#daq-sensor-sessions) 進行連接，因此需要後端正在運行（參見 [`SensorHandle`](#sensorhandle)）。

### 最簡範例

```python
import chloros_sdk

proj = chloros_sdk.open_project("/home/user/Chloros Projects/Field_A")
report = proj.connect_all(verbose=True)
print(report)  # {'cameras': {...}, 'arrays': {...}, 'sensors': {...}}

# Cameras and arrays are addressable by name OR serial / array_id
cam = proj.cameras["FrontLeft"]
cam.capture("./out", format="tiff", processing="reflectance")

arr = proj.arrays["main_rig"]
arr.capture("./out", format="tiff", processing="reflectance")

# Light sensors are addressable by name OR slot_id
reading = proj.sensors["Sky"].read()   # (spectrum, is_saturated, ...) or None

# One frame from every camera and array, one spectrum from every sensor
proj.capture_all("./out")

proj.disconnect_all()
```

或作為上下文管理器：

```python
with chloros_sdk.open_project("/path/to/proj") as proj:
    proj.connect_all()
    proj.arrays["main_rig"].capture("./out", processing="reflectance")
```

### `ChlorosProject` 方法

| 方法 | 說明 |
| --- | --- |
| `connect_all(*, cameras=True, arrays=True, sensors=True, verbose=False, align=None, packed=None)` | 偵測並連線所有已儲存的攝影機、陣列及光感測器。回傳按類別區分的連線報告 （`{"cameras": {...}, "arrays": {...}, "sensors": {...}}`，分別包含 `connected`、`missing` 及 `errors`；`arrays` 亦包含 `alignment`)。當後端在 `127.0.0.1:5000` 上處於監聽狀態時，系統將透過該端點搜尋並連線至已儲存的攝影機與陣列。若無後端，此流程將自行透過 `lattice_sdk` 自行尋找並驅動它們，此過程需 Arena SDK 執行環境及 OpenCV （`cv2`）——SDK 輪子並未安裝這兩項；若無法安裝，所有已儲存的相機與陣列將連同原因一併列於 `errors` 下，而 `missing` 則保持為空。 一個正在運行但拒絕SDK的後端（登出狀態：HTTP 401；Copper以下方案：403）並不屬於「無後端」情況：每個已儲存的相機、陣列和光感測器都會列在 `errors` 下，並附上後端的理由， 且此進程中未開啟任何項目。每次呼叫都會重新檢查後端，因此登入後或啟動 Chloros 後，`connect_all()` 會連接到同一個專案物件。 若後端已存在包含相同攝影機（來自 GUI 或 `chloros-cli project connect`）的儲存陣列，則會直接附加該陣列，不會重新套用其儲存的設定， 且 `disconnect_all()` 會讓其持續運行。光感測器始終透過後端處理（參見 [`SensorHandle`](#sensorhandle)）。它絕不會啟動新的後端。 |
| `disconnect_all()` | 拆除所有內容。首先釋放每個裝置；接著，若後端未確認已釋放某台攝影機、陣列或光感測器（拆除操作超時或遭拒絕）， 則會分別為其建立一個 `ChlorosConnectError` 物件。若裝置已不存在， 或無後端執行中，則保持靜默。其他客戶端已開啟的陣列或光感測器將維持運行。專案的 `with` 退出程序絕不會觸發。 |
| `capture_all(output_dir=".")` | 從每個已連線的攝影機 + 陣列各擷取一幀畫面， 以及每個已連線的光感測器各一個光譜（`SensorHandle.read()`），依序擷取（先是攝影機，接著是陣列，最後是感測器），而非在同一瞬間擷取。 |
| `stream(camera, overlays=False, fps=10.0)` | 產生來自指定攝影機（或陣列）的 BGR `numpy` 幀的產生器。`overlays=False` 會讀取具有此句柄的幀`grab()` 於 `debayered` 層級： 若專案透過後端連接，則透過後端讀取幀；否則採用直接的 `lattice_sdk` 擷取迴圈（陣列會產生 `{serial: frame}` 字典）。`overlays=True` 會透過 `ChlorosLocal.camera_stream()` → 後端之 `/api/camera/<serial>/stream-annotated` MJPEG 串流， 並將攝影機儲存的 `ui.overlay` 區塊作為查詢參數傳遞。需啟用後端模式、OpenCV (`cv2`) 以及 **獨立運作的攝影機**： 直接模式的攝影機會觸發 `RuntimeError`（後端無法擷取此進程所擁有的攝影機），而陣列會觸發 `NotImplementedError`（針對每台攝影機進行複合疊加 — 依名稱串流某個成員）。單次執行的等效寫法： `CameraHandle.capture(annotated=True)`。 |
| `align_arrays(align=True, verbose=False)` | 對每個目前已連接的陣列執行對齊。 |
| `process(mode="parallel", wait=True, progress_callback=None, poll_interval=2.0)` | 對專案的影像執行校準／索引處理流程（包含 `ChlorosLocal.process`；這四項是**唯一**被接受的關鍵字參數 — `indices=` 等 會引發 `TypeError`；透過 `ChlorosLocal.configure()` 設定索引）。懶惰地建構一個 `ChlorosLocal()`，該物件會自動啟動後端。 |

屬性：
- `proj.cameras` — 以名稱和序列號為鍵的 `Dict[str, CameraHandle]`。
- `proj.arrays` — `Dict[str, ArrayHandle]`，以名稱和 array_id 為鍵。
- `proj.sensors` — `Dict[str, SensorHandle]`，以名稱和 slot_id 為鍵。
- `proj.config` — `project.json["config"]` 字典。

### `CameraHandle`

```python
cam = proj.cameras["FrontLeft"]

# Save a frame to disk (processing-aware)
filepath = cam.capture(
    output_dir="./out",
    format="tiff",
    processing="radiance",           # see the level table below
    apply_calibration=True,          # DSNU + flat + 3x3 unmix + NIST
    apply_white_balance=True,        # DLS-aware WB
    apply_index=False,
    index_expression=None,
)

# In-memory grab (numpy array)
frame = cam.grab(processing="debayered")
frame, header = cam.grab(processing="radiance", with_metadata=True)

# Frame iterator (generator)
for arr in cam.frame_stream(processing="debayered", fps=5, count=100):
    my_analysis(arr)
```

**處理層級。** `capture()`、`grab()` 以及 `frame_stream()` 皆採用相同的 `processing`
標記，且此鏈為累積式 — 每個層級都會執行其上方的所有內容：

| 層級 | 輸出 | 備註 |
| --- | --- | --- |
| `raw` | 1 通道拜耳，感測器原生 | 無去馬賽克處理。此層級不提供疊加功能。 |
| `debayered` | 3 通道色彩（**預設** 適用於 `CameraHandle` 及所有 `grab()` / `frame_stream()`；`ArrayHandle.capture()`、`capture_smart()` 及 `stream()` 預設為 `raw`)：記憶體中為 BGR 格式（`grab()` / `frame_stream()`），由 `capture()` 儲存為 RGB 格式 | 在後端模式下，作為擷取： 先從原始影像中扣除校準套件的暗場偏移量（DSNU），再進行 OpenCV 的邊緣感知去馬賽克處理——不使用線性 LUT、平場校正或解混色。唯一能在非後端模式下運作的層級，其採用的是相機函式庫的純雙線性去馬賽克 （無暗電平偏移）。儲存的 TIFF 為光度RGB，因此需將記憶體中幀的通道 (`frame[..., ::-1]`) 以便與檔案進行比較。 |
| `radiance` | float32, W/m²/sr/nm, **每波段一個平面** | 完整的輻測鏈：去馬賽克 + 3×3 解混（多光譜） + DSNU + 平場校正 + NIST 標準化，並已將曝光 × 增益除去，因此數值為絕對值。 |
| `reflectance` | uint16，32768 = 1.0（或 uint8，255 = 1.0 — 參見 `bit_depth`），**每波段一個平面** | 輻度除以下行輻照度 (ρ = π·L/E)。需 DLS/DAQ 讀數 — 請參閱下方註解。 |
| `display` | 8 位元 sRGB 風格 | 相當於 GUI 的渲染：透過相機的活躍色彩配置檔進行 CCM + 白平衡 + 伽瑪校正。 |

僅在直接（無後端）模式下，`debayered` 才可用：`grab()` 在另一層級上會觸發
`NotImplementedError`，`capture()` 會忽略 `processing`，而 `frame_stream()` 則會在每個層級觸發
`RuntimeError`。 `reflectance` 需要可用的下行讀數 — 幀終點會自動將
匯總的 DAQ 拉入相機的 DLS 插槽，但若未綁定任何 DAQ，鏈路將拒絕
反射率輸出，並在回傳的元資料中如實標記降級狀態，而非默默地
回傳品質較差的結果。

> **反射率 DN 比例尺 — 請勿硬編碼。** LATTICE 反射率採用 `32768` = ρ 1.0 並標記
> XMP `Chloros:PixelScale=32768`；Survey3 反射率採用 `65535` = ρ 1.0，且不攜帶
> `Chloros:*` 標籤。請讀取該標籤並以此數值進行除法運算。該數值定義於 uint16 域中，因此對於所有進行重新縮放的格式（16 位元 TIFF、8 位元 TIFF、
> 8 位元 PNG/JPG、32 位元百分比），其值皆維持為 `32768`
> — 請先將儲存的資料類型歸一化回 uint16（從 8 位元轉換為 ×257，從浮點數轉換為 ×65535）。唯一
> 的例外是 1.3.0 之前版本所產生的檔案：當時 8 位元 TIFF 會被 *裁切* 而非重新縮放，因此
> 無法用任何比例來描述，且不包含 `PixelScale` 或 MicaSense 元組。若 LATTICE 反射率檔案中缺少標籤，
> 應視為「無有效比例」，而非預設值。

**校準標籤：僅限於 `raw`。** 擷取資料的 `raw` 檔案包含相機的校準封包： `CalibrationJSON` 有效載荷以及與 MicaSense 相容的標籤（`BlackLevel`、`RadiometricCalibration`、`Band{R,G,B}RadiometricCalibration`、 `MicaSenseVignetting*`），這些標籤描述感測器的原始數值。其他各層級的檔案（`debayered`、`display`、`radiance`、`reflectance`、`index/` 渲染檔）以及每個連續錄製／多鏡頭串流的幀，均包含該鏡頭的識別碼 （`XMP-mapir:CameraSerial`、`CameraModel`、`FilterType`、`LensModel`、 `CalBundleSha256`, ...），以及該幀本身的記錄資料 `XMP-mapir:FrameJSON`（曝光、增益、 溫度、像素格式、擷取時間、DAQ 匹配），因為每個 `process()` 產品都攜帶此識別碼。處理程序會根據該檔案的序號及其 `CalBundleSha256` 值，在 Chloros 的校準快取中查找此檔案的校準資料；若該電腦未儲存此資料，則會下載該套件一次。

#### 由多台相機產生的檔案

由多台相機的波段組合而成的檔案——多波段疊加檔 (`capture_aligned_multiband()`、`start_aligned_stream()`、`capture_live_index(save_multiband=True)`、`Registration.save_multiband()`)、由多台相機生成的即時索引檔（`index.tif`、`index_LUT.png`），以及陣列擷取的 `composite/` 影像——會列出所有參與生成的相機名稱：

- **每- 按攝影機列出，依波段順序排列**（合成圖：背景層，接著是前景層；索引層， 其波段來源的相機）：`XMP-mapir:SourceCameraSerials`、`SourceCameraModels`、`SourceSensorProfiles`、`SourceProductLines`、`SourceFilterTypes`、`SourceLensModels`、`SourceCalDateUTCs`、`SourceCalBundleSha256s` （即每台相機的波段所經過的校準套件；否則——則為 `capture_aligned_multiband()` / `start_aligned_stream()` 堆疊，其波段未經過任何校準套件——即相機套件Chloros所快取的， 即其擷取檔案的名稱)，`SourceLightboxCertIds`、`SourceHalogenLampCertIds`、`SourcePackTiers` ——並列清單，每個清單的第 *n* 項描述同一台相機；`-` 則表示該相機無相關值（無校準包，鏡頭型號未知）。每台相機僅佔 **一**項：若其資料包中某欄位有多個值——例如資料包中的鹵素燈認證 — 這些項目會以 `;`（`F-1479-1;F-1480-1;F-1481-1`）連接，絕不使用逗號，因此無論列印方式為何，清單中每台相機皆對應**一個** 項目。
- **EXIF `SerialNumber`** 包含相同的序列號，以逗號分隔（`214000535,214000551`）。
- 該欄位**不**包含單一攝影機的識別碼（`XMP-mapir:CameraSerial`、`CalBundleSha256`、……）：這些識別碼僅標示單一攝影機及其套件，系統會根據這些資訊解析檔案的校準參數。由多台攝影機的波段 （`index.tif`、`index_LUT.png`）所計算出的索引，其 EXIF 資料為 `Model` `LATTICE`（該攝影裝置）， 而非單一相機的 `LATT-…` 型號。**單一**相機的波段檔案——無論是各波段的 TIFF 檔案、`capture_aligned_multiband(per_camera=True)` 幀、單一相機的堆疊或即時索引——都會攜帶該相機自身的識別標籤， 這點與其他所有擷取檔案相同。`capture_aligned_multiband()` 的堆疊檔、各波段檔及各相機檔，以及每個 `start_aligned_stream()` 堆疊檔及其各波段檔，皆包含其群組的擷取時間 （參考相機的曝光時間，UTC），而單色相機的 `per_camera` 幀僅為單一平面，其格式與 `bit_depth` 堆疊相同。**相機方向（翻轉／旋轉）。** 來自 `grab()` / `frame_stream()` 的幀在每個層級上始終保持
相機的原生方向。儲存的擷取檔遵循相機的方向
設定（GUI 中的「水平鏡像」、「垂直鏡像」或「旋轉」）：

- **`raw` 和 `debayered` 檔案完全呈現感光元件讀取的原始資料**，無論設定為何——若將拜耳
  馬賽克旋轉或翻轉，將形成不同的色彩濾鏡模式，而校準圖是
  基於感光元件本身的網格。
- **`display`、`radiance`、`reflectance` 及索引檔案均經過定向處理** — 在所有
  校正運算後旋轉一次。其鉗位遮罩位於同一網格上。
- **每個擷取檔案皆在 XMP 中記錄設定**：`XMP-Camera:ChlorosFlipHorizontal` /
  `ChlorosFlipVertical` (`true`/`false`)、`ChlorosRotation`（0 / 90 / 180 / 270，順時針，翻轉後
  翻轉後）以及 `ChlorosOrientationApplied` — `false` 適用於原始與去拜耳化影像，`true` 則適用於已定向的
  產出結果。 `ChlorosProcessing` 列出對該檔案像素所施加的操作（若為
  原生格式則為 `none`）。 這包括連續錄製的每一幀（相機本身的設定，
  未套用）以及多相機串流的每一幀（身份：其成員幀為原生格式，如同
  陣列擷取那樣）。
- **處理專案時會將錄製的取向**套用至其從原始檔產生的成果
  （首先在感測器座標系中進行校準，接著進行陣列對齊變形，最後是方向校正），
  因此這些產出與即時擷取的產出相符——包含未校準的感測器響應／經暗角校正的
  備用產出；其去拜耳化產出則保持原生狀態。
- **Chloros 影像檢視器會顯示根據其記錄進行旋轉的原始或去拜耳化檔案** —— 僅限顯示，
  檔案本身不會被修改；沒有記錄的檔案則會以儲存時的狀態顯示。
- **沒有記錄的擷取會被視為原生格式進行處理。** 在 1.3.0 版本之前，單鏡頭擷取
  會自行轉換其原始檔與去拜耳化檔案；Chloros 無法區分此類原始檔與原生檔，
  也不會進行推測。
- **陣列擷取預設會將每個成員檔案保留為原生格式**，並記錄其標識；
  `ArraySession.capture(orient=True)` 會處理其產出（請參閱陣列擷取控制項）。

#### 每波段輸出與混合陣列

在 `radiance` 和 `reflectance` 層級，相機會回傳 **該相機每個波段對應的一張平面** —— M3C 為三張 則有三個平面），M3M 則為一個（其
單一窄頻濾光片）。單色相機的重複平面會被合併，因此 **混合
M3C/M3M 陣列會因序列而產生不同的陣列形狀**。請從幀
元數據中讀取 `bands` 關鍵字以了解各平面的內容；切勿根據通道數推斷：

```python
for serial, (arr, hdr) in arr_handle.grab(processing="reflectance",
                                          with_metadata=True).items():
    print(serial, arr.shape, hdr["bands"])   # ('…', (1536, 2048, 3), ['nir','green','red'])
                                             # ('…', (1536, 2048, 1), ['F685'])
    rho = arr.astype("float32") / hdr["pixel_scale"]
```

使用這些數值前，請務必先檢查 `hdr["processing_level"] == "reflectance"`。 降級
（無 DAQ 讀數、無校準包、頻段佈局未知）會透過 `processing_level` 和
`radiometric_refused` 進行回報，而非提升等級——您所取得的陣列仍為有效的幀， 只是
並非您所要求的產品。經降級處理的儲存擷取檔，其 XMP 中會包含與
`Chloros:RadiometricRefused` （例如 `no-band-layout`、`no-dls-snapshot`）。

#### 每次讀取一個觸發訊號 — `group_id`

**陣列成員的影像取自陣列的同步幀群組，而非各攝影機自身的
擷取緩衝區。** 硬體同步陣列中的攝影機會同步曝光，但每台攝影機的 N 次讀取
並不會一同 *採樣* —— 讀取每台攝影機最新緩衝幀的過程會跨越觸發
，且只要有一幀遺漏，整組影像便會失同步，而系統卻毫無察覺。

從群組中提供影像雖是必要的，但僅靠這一點尚不充分：每個請求仍會
在自身當下時刻擷取 *最新* 的群組快照，而擷取迴圈可能在兩次讀取之間發布新畫面。
在實際運作的 4 台攝影機陣列上測量發現，**20 次平行讀取中有 12 次橫跨兩個群組** —— 且總是
精確相鄰。因此讀取會被 **固定**：

1. 每個幀都攜帶 `group_id`，這是陣列的單調發布計數器。
2. `grab()` 會讀取主節點（若主節點無法回應，則讀取下一個可回應的攝影機），
   接著將其餘攝影機「錨定」至其落腳的群組 ID（HTTP
   端點上的 `?group_id=N`）。
3. 後端會將最近的群組保留 **至少半秒鐘**，因此即使某個同級節點
   已被後續發布所取代，仍會繼續處理觸發該節點的請求——這段時間足以讓每台攝影機的
   反射率匯出資料的往返傳輸時間。若後端不再
   持有該固定組別，系統會以 `409` 和 `group_aged_out: true` 拒絕該請求，而非默默回傳
   並以不同的觸發訊號回應——`grab()` 會重新讀取該設定。

啟用固定功能後重新測量：**20 次讀取中有 20 次皆返回單一觸發訊號。** `grab()` 會自動為您處理此事；
僅當您直接驅動 HTTP 端點時，才需要使用 `group_id=`。

- 若某台攝影機無法透過固定群組進行服務——無論是該群組中缺席，或是拒絕請求
  （例如其不具備的 `band`，或無法產生的電平）——則該攝影機將被 **省略，而非由**
  ，而其他攝影機仍會被回傳。
- `grab()` 回傳的字典是一個 `GrabResult` （一個普通的 `dict` 子類別；
  `from chloros_sdk import GrabResult`）： `.errors` 會將每個
  被省略的序列映射至其讀取時觸發的例外，而 `.group_id` 則是共用群組。一個空的
  `.errors` 表示所有攝影機都屬於此群組。
- 獨立運作的攝影機沒有群組，會回報 `group_id: None`，並且會**拒絕** `group_id=`
  引腳，而非忽略它。

```python
frames = arr_handle.grab(processing="reflectance")
for serial, err in frames.errors.items():
    print(f"SN:{serial} omitted: {err}")       # e.g. status=400 ... band 'F685' not in [...]

frames = arr_handle.grab(processing="reflectance", with_metadata=True)
ids = {hdr["group_id"] for _a, hdr in frames.values()}
assert len(ids) == 1, f"straddled triggers: {ids}"     # belt and braces
```

陣列成員的**每個相機專用 MJPEG 串流**上都會發佈相同的計數器 — 包括
預覽 (`/api/camera/stream/<serial>`)、標註預覽 (`/stream-annotated`) 以及
反射率串流（`/stream-reflectance`）——在每個部分中均以 `X-Chloros-Group-Id` 的形式發布， ，
此外還有 `X-Chloros-Frame-Id` 和 `X-Chloros-Timestamp-Ns`。串流是連續的而非
特定時間點的，因此可透過收集各部分並根據該標頭進行分組來進行配對——無需
設定任何引腳。獨立相機沒有群組。 請參閱
[每台攝影機的 MJPEG 串流](#per-camera-mjpeg-streams-no-rtsp)。

**`frame_stream()` 絕不會陷入靜默狀態。** 若某個時標因可自行消除的原因而失敗，
則會被跳過——若為攝影機串流，該時標將無回傳結果；若為陣列串流，則會回傳
已回應的攝影機 （可能為零）。這些原因包括：尚未收到幀；攝影機的擷取執行緒暫停了一
兩秒（`404 not streaming`，後端會在另一台攝影機連線
或斷線時，或設定變更導致池重新平衡時回報此狀態）；後端重新啟動或 `5xx`；對齊
仍在穩定中 （`409 … retry shortly`）；以及因 DAQ 尚未有讀數而導致的反射率拒絕
（`no-dls-snapshot`，過期或空的讀數）。 若在下一幀仍發生拒絕，情況只會重演——例如
未知錯誤 `band`、`bit_depth=8`（涉及輻照度）、缺乏輻照度量表的校準包，或是
因**完全沒有 DAQ*而導致的反射率拒絕*（無任何數據綁定至相機或其陣列，亦無任何數據連接至
後端： 該拒絕的 `dls_source` 即為 `"none"`），任何其他 `4xx` ——
將觸發 `chloros_sdk.FrameFetchError` （一個 `RuntimeError`，攜帶 `.status` 和 `.body`）
。同樣地，若連續 **60 秒** 跳過時標且期間無任何幀， 無論 `fps` 為何——時間
長到足以讓對齊過程穩定下來並完成後端重啟，卻又短到足以讓一台
確實已損毀的攝影機能終止串流。該放棄錯誤同樣屬於 `FrameFetchError`，無論是針對單一攝影機或
陣列，且最後一次讀取所產生的例外錯誤代碼為 `.errors` （而對於攝影機，則為 `__cause__`）。對於
陣列而言，「拒絕」表示每台攝影機皆遭拒絕；該例外狀況的 `.errors` 包含每台攝影機的
具體原因。

**陣列 `frame_stream()` 會將每個觸發器產出一次**（`ArrayHandle` 與 `ArraySession` 皆然）。若
某個時標的最新資料集正是其已產出的觸發條件，則該時標會等待下一個觸發條件，而非
再次產出相同的資料集；因此，資料流的運行速度將取自 `fps` 與該陣列自身幀
率，且連續的資料集絕不會共用同一個 `group_id`。停止發佈的陣列會觸發
`FrameFetchError`（「60 秒內無新觸發」），而非重複其最後一組資料。若無此機制，當
串流請求的資料量超過陣列所發布的數量——或在連線後立即開始運作——便會
產生重複資料： 在一項測量運行中，20 組資料包含 11 個不同的群組 ID。

```python
from chloros_sdk import FrameFetchError

try:
    for arr in cam.frame_stream(processing="radiance", bit_depth=8):
        my_analysis(arr)
except FrameFetchError as exc:
    print(exc.status, exc.body)        # the HTTP status and the backend's reason
```

#### 匯出單一頻段 — `band`

`grab()` / `frame_stream()` 配合 `band="<name>"`，可將相機的其中一個波段匯出為
單通道 `(H, W, 1)` 陣列 ——其形狀與單色相機產出的相同，因此無論單波段輸出是
來自 M3M 還是透過分割 M3C 所產生，處理方式皆完全相同。FRGN 的紅波段：

```python
red = cam.grab(processing="reflectance", band="red")        # (1536, 2048, 1)
```

可接受的拼寫形式包括完整的佈局名稱（`Red_660`、`NIR_850`、`Green_550`）或其簡稱
頭部 （`red`、`nir`、`green`），不區分大小寫。若名稱未知，系統將拒絕該名稱並列出可用的
名稱清單。

> **選取是根據名稱，而非通道號 —— 這是刻意為之。** 「通道 1」在
> 陣列的 BGR 通道順序中是 `Green_550`，但在濾波器程式碼的字母順序中則為 *紅色*（`RGN`），而這兩者
> 並不一致。若選錯了，將產生一個完全合理但屬於錯誤頻段的幀，而
> 後續任何環節都無法偵測到。 正因如此，才沒有數值索引。

#### 串流對齊 — `align`

`align=True` 會將幀變形至其陣列的共通參考幀。每台攝影機皆透過
**相同的** 輪廓解析度與裁切 ROI 進行變形， 因此，獨立開啟的各相機資料流
輸出時會是 **相互配準** 的 — 您無需同時請求它們：

```python
frames = arr_handle.grab(processing="reflectance", align=True)   # {serial: array}
# or one MJPEG stream per camera, each co-registered on its own:
#   GET /api/camera/<serial>/stream-reflectance?align=1
```

需求與行為：

- 攝影機必須位於具備對齊
  配置檔的 **硬體同步** 陣列（M8 同步纜線）中 — 共註冊假設陣列成員同時曝光，而這點是軟體或
  自由運行陣列無法提供。否則將因 `409` 而被拒絕。
- **請求將被拒絕，絕不會在未對齊的情況下靜默處理。** 擷取路徑可能會在發出警告後以未對齊狀態儲存
  ，因為檔案會保留其來源標籤，日後可重新註冊；但即時串流則沒有
  這樣的第二次機會，因此缺少配置檔屬於錯誤，而非備用方案。
- **請求對齊會觸發該過程。** 對沒有配置檔的陣列請求 `align` 會啟動
  校準，並返回 `409` 以及其正在等待的內容（例如 *「等待曝光
  在 214000551 (1/3) 處穩定」*) — 重試後通常會在數十秒內成功。
  這在無頭模式下至關重要：連線時的自動對齊會在 30 秒後放棄，且在
  都會被跳過，因此若某個陣列的曝光
  收斂速度較慢，過去往往會導致整個會話都無法取得曝光曲線，且若不開啟
  桌面綜合檢視畫面便無從取得。`alignment_status` 也會回報
  `ae_align_holdouts`，標示仍在穩定中的相機及其進度。
- 請為同一陣列的每個串流賦予**相同的 `max_h`**，否則其裁切區域會落在不同的像素
  網格上。
- 對齊操作是在位元深度編碼 *之前* 進行的，因此變形運算會對全精度
  數值進行插值。

#### `bit_depth` — 以精度換取頻寬

`grab()` 和 `frame_stream()` 接受 `bit_depth=16`（預設）或 `bit_depth=8`：

| `bit_depth` | 反射率編碼 | 成本 |
| --- | --- | --- |
| `16` | uint16，32768 = ρ 1.0， 留有至 ρ 2.0 的餘裕 | 全精度。 |
| `8` | uint8 涵蓋 `0..refl_max` （預設 ρ 1.0） | 字節數減半；在預設上限時，量化精度約為 0.004 ρ。超過上限的值將遭截斷 — 詳見下文。 |

##### 8 位元上限 — `refl_max`

`refl_max` 設定 ρ 所涵蓋的 8 位元範圍， 因此 `pixel_scale = 255 / refl_max` 中的 ρ 仍為
`pixel / pixel_scale`。`grab()` 和 `frame_stream()` 將其視為相機與
陣列句柄上皆視為關鍵字（HTTP 端點上的 `?refl_max=`）。提高該值可為鏡面
高光預留空間，但步長會相應變粗：

```python
frame = cam.grab(processing="reflectance", bit_depth=8, refl_max=1.5)   # 0..1.5 in 0..255
```

| `refl_max` | `pixel_scale` | 步長 |
| --- | --- | --- |
| `1.0`（預設） | 255 | 0.0039 ρ |
| `1.5` | 170 | 0.0059 ρ |
| `2.0` | 127.5 | 0.0078 ρ |

> **自動曝光無法修正 8 位元剪裁問題，且嘗試使用自動曝光反而會使資料品質更差。**
> ρ = π·L/E（已將曝光與增益除出），因此 ρ 是場景的特性，而非
> 相機設定的特性——在活陣列上測量時，將曝光減半會使平均 ρ 變動 1.3%。反之，提高
> 曝光以「適應」動態範圍，反而會使像素進入 **感測器** 飽和狀態，這
> 會低估 L 值，並使 ρ 產生 **偏低** 的偏差，且不會觸發錯誤提示（測量結果：最大 ρ 值從 1.203 降至
> 0.985，曝光倍增時，平均值下降 −23%）。 請選用 `refl_max` 的動態範圍；避免感光元件
> 飽和本就是自動曝光（AE）的功能，這能保障 ρ 的 *有效性*，而非其動態範圍。

**請參閱 `reflectance_clipped_fraction`， 而非該標記。** `reflectance_clipped_at_1_0` 會因
單一像素而觸發——在真實場景中，它報告的值為 `True`，而實際比例卻是 `0.000001`，即一像素
。該分數值正是決定是否應提高上限的依據。

**`bit_depth=8` 僅限反射率。** 若針對 `radiance` 請求此參數，系統將予以拒絕而非
處理： 反射率具有固定的 0..1 範圍，可維持在 8 位元編碼下；而輻照度則沒有
固定範圍，因此要將其塞入 8 位元中，需要進行每幀百分位數拉伸，其映射關係
會隨著每幀而變化——這只是一張圖片，而非實際測量值。

#### 即時反射率串流 (MJPEG)

`GET /api/camera/<serial>/stream-reflectance` 以
`multipart/x-mixed-replace` 的形式提供每波段 8 位元反射率——M3C 格式為 3 通道JPEG，M3M 格式為灰階。查詢參數：
`fps`（預設值 10）、`jpeg_q`（預設值 95）、 `max_h`（0 = 原生），`band`（一個命名波段，如
上所述），`align` （`1` 需共同註冊 — 規則與 SDK 的 `align` 相同），`refl_max`（8 位元
編碼上限，如上所述）。每個部分包含
`X-Chloros-Group-Id`、`X-Chloros-Frame-Id`、`X-Chloros-Timestamp-Ns`、`X-Chloros-Bands`、
`X-Chloros-Pixel-Scale`、`X-Chloros-Aligned` 以及 `X-Chloros-Reflectance-Clipped`。
**來自不同攝影機但攜帶相同 `X-Chloros-Group-Id` 的部分即為同一觸發器** —
這就是如何將同一陣列中每台相機的多個串流重新組裝起來，且此機制在
伺服器端得到保證，因為陣列成員是從同步的幀群中傳送的（參見
[每次讀取一個觸發器](#one-trigger-per-read--group_id)）。若在連線時無法取得反射率，則會傳回 `409`；若在運行中途無法取得反射率，則會終止該串流，因此串流停滯
代表「不再有反射率」，而非「仍正常，只是靜態」。

> **JPEG 是有損壓縮的 — 這些像素代表測量值。** 壓縮錯誤會直接影響
> 反射率數值，且在邊緣處最為嚴重，而邊緣正是材料邊界的所在。正因如此，
> 預設品質值設為 95。若需完整保留數值，請使用
> `grab(processing="reflectance")` / `frame_stream(...)`，這兩者會產生相同結果，且路徑中不含
> 編解碼器（若限制因素是線路成本而非編解碼器，則使用 `bit_depth=8`）。

> **EXIF 資料會一併傳遞至匯出檔案。** `process()` 會將來源擷取檔的 GPS 區塊
> **及其 ExifIFD** 複製到每個產品中，因此匯出檔會包含 `FocalLength`、`FNumber`、
> `ExposureTime`、`ISO`、`DateTimeOriginal` 及 `CameraSerialNumber`，以及
> 地理參照。PIX4D 正是透過 `FocalLength` 來計算地面樣本距離——若缺少此資料，
> 重建結果將退回到極度錯誤的比例尺（曾有實測案例將 411 公尺的場地
> 誤判為 47.8 公里）。此副本刻意未採用 `-all:all`：IFD0 的結構標籤會破壞
> LATTICE 輸出，而 `ExifImageWidth`/`Height` 則因描述的是原始
> 擷取資料而非匯出的柵格，故被排除在外。

擷取階段的子標記（適用於輻射計量層級 — `radiance`、`reflectance`、 `display`）：

| 標誌 | 預設值 | 含義 |
| --- | --- | --- |
| `apply_calibration` | `True` | DSNU + 平場校正 + 3x3 解混 + NIST 輻測量級。 |
| `apply_white_balance` | `True` | 白平衡 LUT。當 DAQ 與相機綁定時，會考量 DLS。 |
| `apply_index` | `False` | 植被指數評估。 |
| `index_expression` | `None` | 覆寫公式。若非為空 → 自動啟用指數。 |
| `annotated` | `False` | 疊加 GUI 裝飾（斑馬線／網格／峰值）。不適用於 `raw`。 |

### `ArrayHandle`

```python
arr = proj.arrays["main_rig"]

# Single synced capture group
files = arr.capture("./out", format="tiff", processing="reflectance")
# → {"213800234": "/path/to/x.tif", "214000533": "/path/to/y.tif", ...}

# Multi-level: each serial's value becomes an ordered LIST, not a str
files = arr.capture("./out", processing="all")
# → {"213800234": ["/raw.tif", "/debayered.tif", ...], "combined": "/idx.tif"}

# Smart capture (wait for AE to settle)
result = arr.capture_smart(
    "./out", processing="reflectance",
    settle_timeout_s=5.0,
    stability_window_s=1.5,
    exposure_tolerance_pct=5.0,
)
print(result["frames"], result["settle"])

# In-memory grab: {serial: numpy array}
frames = arr.grab(processing="debayered")
frames = arr.grab(processing="radiance", with_metadata=True)

# Stream-to-disk loop
arr.stream(count=60, output_dir="./stream", fps=5, processing="raw")

# Frame-iterator (tolerates per-cam drops; great for downstream analysis pipelines)
for frames in arr.frame_stream(processing="radiance", fps=5, count=100):
    if "213800234" in frames:
        my_analysis_pipeline(frames["213800234"])

# Preview iterator (live MJPEG-equivalent; tolerates partial cycles)
counts = arr.preview_stream("./preview", fps=3.0, duration=30.0)
print(counts)  # frames written per serial
```

> **回傳值為 `chloros_sdk.CapturePaths`， 類型為 `CapturePathMap`，而非 `Dict[str, str]`。**
> `chloros_sdk.CapturePathMap` 即為 `Dict[str, Union[str, List[str]]]`：單層
> `processing` 會為每個序列號提供一條路徑，而 `processing="all"` 則提供
> **有序清單**，列出該相機所儲存的所有產品。`ArrayHandle.capture()` 接受
> 不接受 `levels=`，並會忽略其不認識的關鍵字參數（`levels`、 `daq`、
> `render_index`、`force_daq`），因此僅傳入此處列出的參數 （`aligned=False`
> 會儲存原生檔案，如同 `ArraySession.capture()` 所示）；針對這些控制項，請使用
> `ArraySession.capture()`。若存在即時串流的複合視窗，其資料將以額外的
> `"combined"` 鍵下，而非序列鍵下。假設使用 `str` 的程式碼在
> 清單形式下會發生錯誤，且不會有任何類型檢查器提出異議——註解指出 `Dict[str, str]`
> 一段時間，這也是為何會存在這個別名。若要使用扁平形式，請進行
> 正規化：
>
> ```python
> paths = arr.capture(processing="all")
> flat = [p for v in paths.values()
>         for p in (v if isinstance(v, list) else [v])]
> ```

> **無法寫成：`.failed`。** 映射 `capture()` 所返回的是
> `chloros_sdk.CapturePaths` —— 一個普通的 `dict` 子類別，因此上述所有內容
> 無需修改即可正常運作 —— 該類別同時包含 `.failed`：即後端無法寫入的檔案，
> 其條目與 [`CaptureResult.failed`](#files-that-could-not-be-written--captureresultfailed)
> （`serial`、`level`、 `error` 以及 `disk_full: True`，此情況發生於磁碟已滿時）。 部分
> 擷取仍會回傳所有曾被寫入的路徑；在磁碟空間已滿的情況下，對應表為
> 空，且 `.failed` 說明了原因。 `.skipped` 列出各攝影機的跳過項目，
> `.group_seq` 則列出同步群組， 而 `.alignment` / `.oriented` 則是
> 後端的，如同 `CaptureResult` 所示。`capture_smart()` 將其回傳為
> `result["frames"].failed` 和 `result["failed"]`。
>
> ```python
> paths = arr.capture("./out", processing="all")
> if any(f.get("disk_full") for f in paths.failed):
>     raise SystemExit("drive full -- free space and capture again")
> ```

### 陣列對齊

`ArrayHandle` 會公開完整的對齊表面。預設情況下，分析報告僅限於當前工作階段 — 若要永久儲存，請明確呼叫 `export_alignment()`。

```python
from chloros_sdk import AlignmentSpec

arr = proj.arrays["main_rig"]

# Defaults: ORB / affine / one synced snapshot — same as the GUI's auto-cal
result = arr.calibrate_alignment()
print(result["profile"]["rms_residual_px"])

# Custom spec for tough scenes (low-contrast canopy)
spec = AlignmentSpec(
    method="feature_orb",         # feature_orb / feature_akaze / phase_correlation / checkerboard / manual
    model="rigid",                # translation / rigid / affine / homography
    num_frames=5,
    max_features=8000,
    ratio_threshold=0.7,
    ransac_threshold_px=2.0,
    min_matches=30,
    max_reproj_err_px=2.0,
)
arr.calibrate_alignment(spec)

# Or tweak one knob at a time
arr.calibrate_alignment(num_frames=3, model="affine")

# Manual control points: each camera's pixel coordinates of the same
# features, the reference camera's included (>= 2 per camera). Every one
# of num_frames is solved from these points.
arr.calibrate_alignment(method="manual", model="rigid", control_points={
    "214701173": [[412, 300], [1630, 310], [1020, 1180]],   # reference (master)
    "214701292": [[418, 297], [1637, 306], [1027, 1177]],
})

# Inspect / manipulate
status = arr.alignment_status()
arr.tweak_alignment("214701292", dx=2.5, dy=-1.0, rotation_deg=0.0, scale=1.0)
arr.export_alignment("/tmp/main_rig_alignment.json")
arr.import_alignment("/tmp/main_rig_alignment.json", validate=True)
arr.clear_alignment()
```

#### 連線時對齊

`connect_all(align=...)` 可在連線時自動對齊每個陣列：

```python
# Align every array with defaults
proj.connect_all(align=True)

# Per-array control
proj.connect_all(align={
    "main_rig": AlignmentSpec(num_frames=5, model="affine"),
    "side_rig": True,             # use defaults
    "verify_rig": False,          # skip
})
```

若未指定，則回退至 `project.json["config"]["auto_align_on_connect"]`。

### `SensorHandle`

一個已儲存的光感測器（DAQ-U、DAQ-M 或 DAQ-E），由 `connect_all()` 在後端的 [DAQ 感測器池](#daq-sensor-sessions) 中透過 `connect_all()` 開啟的已儲存光感測器 — 此感測器池亦為 `connect_daq_sensor()` 和 `chloros-cli daq pool-connect` 所使用。

```python
reading = proj.sensors["Sky"].read()
if reading is not None:
    spectrum, is_saturated, integration_time, x, y, z = reading
```

- `read()` 等待感測器的 **下一個** 光譜 — 呼叫時已緩衝在記憶體中的光譜不計入 ——並傳回 `(spectrum, is_saturated, integration_time, x, y, z)`；若 5 秒內未收到任何資料，則傳回 `None`。 已暫停的資料流已恢復。當感測器已校準時，`spectrum` 的單位為 W/m²/nm。
- `.sensor` 為感測器的 [`DAQSensorSession`](#數據擷取感測器會話)，其餘情況則為：`latest(n=...)`、`status()`、`record_start()` / `record_stop()`。
- `.slot_id`、`.model`、 `.serial`、`.name` 以及 `.settings` 均為從 `sensors.json` 所儲存的項目。

**`connect_all` 所還原的內容。** 每個感測器都會在其儲存的位址（即其 COM 埠、藍牙位址或 DAQ-E 主機）上被開啟；若專案曾記錄該感測器的 ID，則必須是該感測器：若該位址連接的是其他裝置，系統將拒絕連接並視為未連接。其儲存的積分時間、幀平均值及自動曝光設定將被套用，並套用其蓋帽校正：

| 儲存的 cap | 透過 |
| --- | --- |
| 一個 cap 設定檔（例如 `fov_30`） | 該設定檔。 |
| 未選擇 | `sunshine_cosine`（連接預設值），記錄為假設值（`cap_id_source=auto_default`），因此處理過程會發出警告。感測器出廠時為裸感測器：請選擇已安裝的蓋板。 |
| `none` — 在 GUI 中選為 **無 (裸感測器)** | 無蓋校正。 |

DAQ-E-S 無論先前儲存了什麼設定，均不採用電容配置檔：其擴散片是校準的一部分（參見 [DAQ-E-S](#daq-e-s--fixed-diffuser-and-tilt-sensor)）。由早期版本Chloros儲存的專案中，裸感測器被記錄為無選項，因此重新連線時會顯示 `sunshine_cosine`；請在 GUI 中再次選擇 **無 (裸感測器)** 並儲存專案。

若感測器已在感測器池中由其他客戶端開啟（例如透過CLI或另一腳本），則會直接連接該感測器，無需重新套用設定， 且 `disconnect_all()` 會讓其繼續運行；而由 `connect_all()` 開啟的感測器則會被釋放。與 `connect_daq_sensor()` 相同，Chloros 圖形使用者介面（GUI）**並非**感測器池的一部分： 請在透過腳本連接專案之前，先於 GUI 中關閉該感測器。**需求與報告。** 光感測器需要一個正在運行的 Chloros 後端，且須登入付費的 Chloros+ 層級（Copper 或更高），而 `connect_all()` 從不啟動後端。若無後端運行，所有已儲存的感測器都會列在 `report["sensors"]["errors"]` 下，並標示此狀況。 後端無法開啟的感測器也會列於此處，並附上後端所顯示的理由；對於光感測器，`report["sensors"]["missing"]` 將保持為空。

---

## 直接硬體（無後端）

若您希望 LATTICE 攝影機（CI、無頭機器人、嵌入式系統）完全不依賴後端，請直接導入 `lattice_sdk` — 該模組會由 `chloros_sdk` 重新匯出。 關於 `CAMERA_AVAILABLE` 的注意事項： `lattice_sdk` 雖位於 SDK 輪子中，但需同時具備 Arena SDK 執行環境與 OpenCV（`cv2`），而該輪子並未安裝這兩者。光感測器需透過後端運作—— [`connect_daq_sensor()`](#daq-sensor-sessions) 或專案中的 `proj.sensors` — 唯有一項例外，即 DAQ-M 頻帶幅度輔助函式 [`DLS`](#daq-m-band-magnitudes--dlsband_magnitudes)，該輔助程式還需 `bleak`。

```python
from chloros_sdk import (
    # cameras
    LatticeCamera, CameraSettings, PRESETS, CameraPool,
    Calibration, CalibrationCoefficients, FilterModel, list_filters,
    DLS, NetworkDiagnostics, gpu_info, gpu_available,
    # discovery
    discover_cameras, discover_cameras_via_backend,
    # exceptions
    LatticeError, CameraNotFoundError, StreamError, CaptureError,
    CalibrationError, NetworkError, DLSError,
)

# Find a camera and capture in one go
cams = discover_cameras(timeout_ms=3000)
print(cams)

settings = PRESETS["high_quality"]
with LatticeCamera(serial="213800234", settings=settings) as cam:
    result = cam.capture(output_dir="./out", format="tiff")
    print(result.filepath, result.width, result.height)
```

##### 預設值與觸發機制

四個預設值中有三個採用 **自由運行** 模式：相機持續曝光，且
`capture()` 會傳回下一幀畫面。`triggered` 則是例外 ——它會讓
相機處於待機狀態，等待第 2 線的硬體邊沿訊號，因此在邊沿訊號出現前不會進行任何擷取。

| 預設模式 | 觸發方式 | 適用情境 |
| --- | --- | --- |
| `default` | 自由運行 | 一般用途 |
| `high_speed` | 自由運行 | 8 位元，60 fps 上限，短曝光 |
| `high_quality` | 自由運行 | 12 位元，無幀率上限 — 靜態影像的常用選擇 |
| `triggered` | **就緒，第 2 線** | 相機透過 M8 同步線連接，並由其他裝置觸發 |

若您選擇 `triggered`（或自行設定為 `trigger_mode="On"`），且第 2 線路
未接任何驅動裝置，每個 `capture()` 都會超時 ——這是正確的，因為您要求
相機等待。SDK 會在此情況發生時說明原因；請參閱
[擷取期間發生 SC_ERR_TIMEOUT](#direct-hardware-backend-free)。

> **注意 — 連線時的「GVSP 探測」/ `SC_ERR_TIMEOUT -1011` 訊息並非錯誤。**
> 連線時，SDK 會嘗試協商**巨型幀**（9000 位元組的 GVSP 封包），以提升傳輸量。 在直接點對點的 NIC 連結上（例如連結本地的 `169.254.x.x` 位址），網路通常無法傳輸巨型幀，因此此探測會超時並記錄如下內容：
>
> ```
> [Network] GVSP probe: unexpected error (TimeoutError: ... SC_ERR_TIMEOUT -1011)
> [Network] GVSP probe at 9000 did not deliver a complete buffer; reverting to ICMP-chosen size
> [Network] GVSP packet size: 1500 bytes (standard)
> ```
>
> 這是**設計上的備用機制**：SDK 會自動切換回標準的 1500 位元組封包，而攝影機仍會正常連線（隨後的 `[chunk-enable …]` 記錄行屬於正常的連線流程）。擷取功能仍可正常運作。
>
> 您可以跳過此探測，但 **它不僅僅是消除日誌記錄的工具——它會直接關閉巨型幀功能。** 無論您的網路品質多好，攝影機對「Don't-Fragment」ping 請求的回應大小上限始終為 1500 位元組，因此僅靠 ping 測試永遠無法偵測巨型幀；唯有此探測功能能做到。若停用此功能，攝影機將永遠以標準 1500 位元組封包運作， 無論在何種網路環境下：
>
> ```bash
> CHLOROS_GVSP_PROBE_FALLBACK=0   # gives up jumbo — see the warning it prints
> ```
>
> 僅在您 *確定* 網路無法傳輸巨型幀的情況下才值得啟用，如此一來每台攝影機可節省約一秒的連線時間。 由於這是一項實質性的權衡而非表面上的調整，SDK 現在會在您使用時明確標示：
>
> ```
> [Network] ⚠️ GVSP probe disabled (CHLOROS_GVSP_PROBE_FALLBACK=0) — staying at
> 1500 bytes, jumbo NOT tested. … if this network does carry it, you are giving
> up ~1.45x wire ceiling. Unset the variable to test for jumbo.
> ```
>
> **除非有正當理由，否則請勿更改此設定。**若保持啟用狀態，每台攝影機會在每個後端會話中測量一次，並在連線狀態變更時（例如更換網卡、連線速度改變，或因切換交換機而導致連線中斷）再次測量；重新連線時會重複使用先前測量結果。**若後端交換機保持連線狀態但內部配置變更，請重新啟動後端以進行重新測量。** 首次連線時，此檢查偶爾可能耗時數分鐘，若發生此情況，日誌中會明確標示。
>
> 若您 *希望* 獲得巨包吞吐量，請啟用端到端巨包 （網卡 MTU 設為 9000 且交換機能傳輸巨型封包），或當您確定鏈路支援時，透過 `CHLOROS_GVSP_PACKET_SIZE_FORCE=9000` 將其固定 —— 不過建議採用針對每條指令的 `CHLOROS_GVSP_PACKET_SIZE_FORCE=9000 python …`，而非永久設定， 因為固定大小的設定會跳過探測步驟，並停止適應前端網路。路徑中的**每個**裝置都必須能傳輸巨型封包——包括任何 PoE 分流器或注入器，這通常是原本具備巨型封包傳輸能力卻無法傳輸的原因。

> **在執行 `capture()` / `grab*()` 期間出現的 `SC_ERR_TIMEOUT -1011` 屬於不同的問題 ——那才是真正的錯誤。**
> 上述說明僅針對由**連線時間探測**記錄的 `-1011`。若同樣的錯誤由**擷取** 引發，則表示攝影機已成功連線，但未傳送任何影像：
>
> ```
> File ".../lattice_sdk/camera.py", line ..., in grab_frame_with_metadata
>   buffer = self._get_buffer(timeout)
> lattice_sdk.exceptions.CaptureError: Capture failed: ... SC_ERR_TIMEOUT -1011
> ```
>
> 關鍵跡象在於：該攝影機的 *控制* 通道狀態正常——發現功能運作正常，設定及 `[chunk-enable …]` 寫入皆成功——但 *每* 一幀影像皆超時。
>
> **通常的原因是攝影機已設定為硬體觸發模式。** 當出現 `trigger_mode="On"` 和 `trigger_source="Line2"` 錯誤時，攝影機在 M8 同步線纜接收到電訊號邊沿之前，完全不會傳送任何訊號。 若該線路未由任何纜線驅動，每次擷取都會無限期等待。攝影機並未故障，網路也運作正常 ——它只是完全按照指令運作。
>
> `CameraSettings()` 以及 `default` / `high_speed` / `high_quality` 預設為自由運行模式，而在啟用狀態下因超時而失敗的擷取操作會顯示具體原因，而非僅輸出單純的 `-1011`。`PRESETS["triggered"]` 會啟用 Line2，這是設計上的行為。
>
> 若要強制任何攝影機進入自由運行模式：
>
> ```python
> import copy
> settings = copy.deepcopy(PRESETS["high_quality"])   # PRESETS entries are shared
> settings.trigger_mode = "Off"        # free-run; don't wait for an M8 edge
> ```
>
> 若使用 `trigger_mode="Off"` 時仍發生超時，則表示該攝影機確實未傳輸資料 — 請將日誌及 `ip link show` 寄送給我們。

#### 色彩配置檔（RGB 即時預覽）— `set_color_profile`

`LatticeCamera.set_color_profile(profile, custom_cct_k=None)` 用於選擇 RGB 攝影機上 **即時預覽** 的顯示色彩配置檔（多光譜攝影機會忽略此設定）：

| 配置檔 | 含義 |
| --- | --- |
| `raw` | 完全繞過輻測鏈。 |
| `linear` | DSNU + 平滑校正 + 白平衡，無 CCM，無伽瑪校正。 |
| `natural` | 線性 + 測量所得的 CCM + sRGB 伽瑪，僅採用簡易處理（色度平滑 + 高光去飽和） — 這是寫實的預設模式。 |
| `enhanced` | `natural` 加上完整的 Hub-Parity 後製處理（去色邊、鮮活度、CLAHE 局部對比）。畫面更豐富，但**每幀後製成本約為兩倍**，因此即時幀率較低。 |
| `custom_temp` | `natural`，但白平衡固定為 `custom_cct_k` 開爾文 （忽略 DLS；後端側限制在 2000–10000 K）。 |

此設定檔僅為**即時預覽專用**的速度／畫質調節鈕：儲存的擷取畫面無論選用何種設定檔，皆會獲得完整的豐富後製效果，因此選擇 `natural` 以換取幀時間，並不會降低寫入磁碟的畫質。 未知配置檔會觸發 `ValueError`；當可連線至 chloros 後端時，變更也會透過 POST 傳送至該後端，以便下一幀預覽畫面反映此變更（未使用後端的直接 SDK 使用者仍會收到設定變更）。

```python
with LatticeCamera(serial="214701292") as cam:   # RGB cam
    cam.set_color_profile("enhanced")            # richer look, lower LIVE fps
    cam.set_color_profile("custom_temp", custom_cct_k=5600)
```

#### 亮點（RGB顯示） — `set_highlights`

`LatticeCamera.set_highlights(value)` 會從 **顯示器** 在RGB攝影機上進行渲染——此為解決明亮白色表面渲染成扁平白色塊狀物的修正方案。`value` 即為 `-100`..`0`（經裁剪）；`0` 為關閉狀態，此時渲染結果將與未啟用此設定時完全相同。較強的設定會降低顯示器的白色亮度（在 `-100` 時約降至 60%）並擴寬色調曲線， 因此，預覽的自動亮度曾將其推至純白的表面，將能恢復其細節。

這雖是顯示控制，但與飽和度和對比度（僅改變即時預覽）不同，它也會改變您儲存的內容：即時預覽和**顯示層級**的擷取檔都會遵循此設定，因此這些檔案也會顯得更暗（它們承襲 `Chloros:DisplayHighlights`）； 輻照度與反射率匯出檔則永遠不會受此影響，且此設定無法恢復已剪裁的感光元件資料——若要避免此情況，請降低該場景的曝光值。多光譜與單色相機將忽略此設定。該數值儲存於 `settings.highlights` 並透過 POST 傳送至可連線的 chloros 後端， 與 `set_color_profile` 完全相同；其 CLI 對應指令為 `lattice highlights --set N`。

```python
with LatticeCamera(serial="214701292") as cam:   # RGB cam
    cam.set_highlights(-60)                      # 0 = off, -100 = strongest
```

#### 單色 (M3M) 相機與 `Calibration`

單色 **M3M** 相機 (`M3M-<lens>-F<wavelength>`) 為單波段： 僅有一層灰階平面，無拜耳馬賽克，亦無 3×3 光譜串擾矩陣。`Calibration` 能識別此特性，並揭露一個 `is_mono` 標誌。 反射率仍適用於各波段的輻測圖（解混矩陣為單位矩陣），但針對單一相機進行的多波段運算會產生結果，而非返回無意義的數值：

```python
from chloros_sdk import Calibration, CalibrationError

calib = Calibration("M3M-L87-F685")
print(calib.is_mono)        # True  (False for any M3C / RGN Bayer cam)
print(calib.filter_type)    # 'mono'  (sentinel; not a real crosstalk key)

# NDVI needs two bands (Red + NIR); one mono band can't supply both.
try:
    calib.compute_ndvi(reflectance_frame)
except CalibrationError as e:
    print(e)   # "...single-band mono (M3M) camera. Combine multiple..."
```

若要利用單色硬體建立植被指數，請將數台不同波長的 M3M 相機組合成對齊的多波段堆疊（參見 [陣列對齊](#array-alignment)），並針對該堆疊計算指數，而非僅針對單一相機。

#### DAQ-M 波段亮度值 — `DLS.band_magnitudes`

`DLS(mac_address, filter_model="RGN-IMX265")` 透過其專屬的 BLE MAC 位址，經由藍牙連線至 DAQ-M。 這是唯一一類能直接與感測器通訊（而非透過後端）的光感測器類別，因此除了需要直接相機類別所需的元件（`CAMERA_AVAILABLE`：Arena SDK 執行環境與 OpenCV）之外，還需 `bleak` 來支援藍牙 （`pip install "chloros-sdk[camera]"` 或 `pip install bleak`；若缺少此組件，`DLS` 將觸發 `DLSError`）。若連接至其他位置的 DAQ-M （後端資源池、專案、GUI）會停止廣播，因此請先在該處釋放資源。 **必須指定 `mac_address`**：由於每個裝置都有其專屬的預設值，因此此處無預設值；若未指定，則在嘗試任何連線前，`DLS()` 會觸發 `DLSError`。請使用 `discover_daq_sensors(transports=["ble"])`（參見 [發現](#discovery--finding-an-address-to-connect-with)）或 `chloros-cli daq pool-discover --only ble` 來查找 MAC 位址。

`DLS.band_magnitudes(spectrum=None, *, spectral_response=None)` 會整合一個 DAQ-M 頻譜（當 `spectrum` 為 `None` 時的最新讀數； 若尚無此值，則為 `DLSError`) 整合至相機的三個通道響應中，並返回 `{"red": …, "green": …, "nir": …}` — 這些鍵名依此順序對應各通道；若使用RGB相機的曲線，第三個（`nir`）即為其藍色通道。

- 若未指定 `spectral_response`，則使用濾鏡模型的預設曲線（即 `filter_model` 所建構的 `DLS`），該曲線是根據單一裝置測量所得。
- `spectral_response` 則採用相機**自身**的測量曲線： 一個字典 `{"wavelengths": [...], "R": [...], "G": [...], "B": [...]}`，該字典儲存於該相機的校準包中（`color_correction.spectral_response_smoothed` — 適用於 RGB 相機；多光譜校準包不包含此類曲線）。若無法使用該字典，則會回退至機隊曲線。

```python
from chloros_sdk import DLS

# camera_curves: {"wavelengths": [...], "R": [...], "G": [...], "B": [...]}
with DLS(mac_address="AA:BB:CC:DD:EE:FF") as dls:
    reading = dls.acquire()
    fleet = dls.band_magnitudes(reading)                         # fleet curve
    own = dls.band_magnitudes(reading,
                              spectral_response=camera_curves)   # the camera's own
```

CLI 的 `chloros-cli lattice dls --camera SERIAL` 透過後端執行相同操作，從已載入或快取的校準包中查找相機的曲線，並標示所使用的曲線來源。 透過 HTTP 傳輸時，即為 `POST /api/dls/cli-read` 搭配 `mac` （此參數為必填：若未提供，回覆將為 HTTP 400 錯誤，格式為 `code` `MAC_REQUIRED`），且正文中包含 `camera_serial`； 當使用該相機自身的曲線時，回應中的 `band_magnitudes_source` 會變成 `camera <serial>`；若使用車隊曲線（即未載入曲線的相機：非 RGB，或未載入套件 或未存入快取）。

---

## 例外情況

擷取基類以處理「Chloros 發生任何錯誤」的情況：

```python
import chloros_sdk

try:
    chloros_sdk.process_folder("/path/to/folder")
except chloros_sdk.ChlorosAuthenticationError:
    print("Run `chloros-cli login` first.")
except chloros_sdk.ChlorosLicenseError:
    print("Chloros+ subscription required.")
except chloros_sdk.ChlorosError as e:
    print(f"Chloros error: {e}")
```

> `ChlorosAuthenticationError` 和 `ChlorosConfigurationError` 是 `chloros_sdk` （以及 `chloros_sdk.exceptions`）的屬性，但未包含在 `__all__` 中，因此 `from chloros_sdk import *` 不會將其導入：請按名稱導入。`ChlorosError` 家族由 `ChlorosLocal` 及其基於該類建立的函式（`process_folder()`、`process_lattice_capture()`）所衍生； 智慧連接介面會派生 `ChlorosConnectError`。

層級結構：

```

ChlorosError
├── ChlorosBackendError           (backend binary not found / failed to start)
├── ChlorosConnectionError        (HTTP transport failure; no backend running
│                                  with auto_start_backend=False)
├── ChlorosLicenseError           (403 PLAN_UPGRADE_REQUIRED: plan below Copper)
├── ChlorosAuthenticationError    (401 AUTH_REQUIRED: not logged in)
├── ChlorosConfigurationError     (bad configure() inputs)
└── ChlorosProcessingError        (pipeline failed; create/import refused)
    └── ChlorosProjectFilesMissingError  (open_project(): none of the
                                  project's source files found, even after
                                  the relink; .missing_roots, .audit)

ChlorosConnectError                (the smart-connect surface: connect_*, attach_array,
                                    list_*, discover_*, every CameraSession /
                                    ArraySession / DAQSensorSession method,
                                    RecorderHandle, ContinuousHandle,
                                    disconnect_all(), and analyze_array_network()
                                    when no backend answers; 401 and 403 included.
                                    Derives from plain Exception, NOT from
                                    ChlorosError, so `except ChlorosError` will
                                    not catch it. .warning: a warning the
                                    backend sent beside its error (also at the
                                    end of the message); .recording_closed: the
                                    recording a failed DAQ-E restart closed;
                                    both None otherwise)

FrameFetchError (RuntimeError)     (grab() / frame_stream() refusals; .status,
                                    .body, .errors)
ValueError                         (capture() arguments: PNG/JPEG with a level they
                                    cannot hold; a bad packed=)
FileNotFoundError / ImportError    (open_project(): no such folder / no project
                                    loaders)
RuntimeError                       (CameraHandle / ArrayHandle: a backend refusal)

lattice_sdk exceptions:
LatticeError
├── CameraNotFoundError
├── CameraConnectionError
├── StreamError
├── CaptureError
├── CalibrationError
├── NetworkError
└── DLSError
```

---

## 端對端範例

### 1. 處理資料夾並顯示自訂進度條

```python
from chloros_sdk import ChlorosLocal

def progress(percent, message):
    bar = "#" * (percent // 5)
    print(f"\r[{bar:<20s}] {percent:3d}% {message}", end="", flush=True)

with ChlorosLocal() as cl:
    cl.create_project("FieldA_2026-05-26")
    cl.import_images("C:/DroneImages/Flight001", recursive=True)
    cl.configure(
        debayer="High Quality (Faster)",
        vignette_correction=True,
        reflectance_calibration=True,
        indices=["NDVI", "GNDVI", "OSAVI", "SAVI"],
        export_format="TIFF (16-bit)",
    )
    cl.process(progress_callback=progress)
print()
```

### 2. 即時 LATTICE 陣列 → 反射率 + DAQ 參考

```python
import chloros_sdk

# Open a paired sensor first so the array's reflectance step has an
# absolute reference. Smart-detect picks USB / BLE / ETH automatically.
with chloros_sdk.connect_daq_sensor() as daq:
    with chloros_sdk.connect_array([
            "213800234", "214000533", "214701288", "214701292"
    ]) as arr:
        # Smart capture: wait for AE to settle, then snap
        arr.capture("./out", processing="reflectance", smart=True)

        # Record the corresponding DAQ frames as ground truth
        daq.record_start(output_dir="./out", device_name="sky-reference")
        # ... do whatever capture campaign ...
        info = daq.record_stop()
        print(info["path"], info["rows"])
```

### 3. 專案驅動的擷取專案

```python
import time, chloros_sdk

with chloros_sdk.open_project("/home/user/Chloros Projects/Field_A") as proj:
    report = proj.connect_all(verbose=True, align=True)
    if report["arrays"]["errors"]:
        raise SystemExit(f"Array(s) failed to connect: {report['arrays']['errors']}")

    rig = proj.arrays["main_rig"]

    # Re-align right before the campaign
    rig.calibrate_alignment(num_frames=5)
    rig.export_alignment("./alignments/main_rig.json")

    # 50 sequential single-frame captures at 2 fps
    for i in range(50):
        frames = rig.capture(
            output_dir=f"./out/frame_{i:04d}",
            processing="reflectance",
            apply_calibration=True,
            apply_white_balance=True,
        )
        time.sleep(0.5)

    # End-of-day: process the captured folder. process() accepts only
    # mode/wait/progress_callback/poll_interval — indices come from the
    # project's saved config (or set them via ChlorosLocal.configure()).
    proj.process()
```

### 4. 多鏡頭幀流 → NumPy 處理管線

```python
import chloros_sdk
import numpy as np

with chloros_sdk.open_project("/path/to/proj") as proj:
    proj.connect_all()
    rig = proj.arrays["main_rig"]

    for frames in rig.frame_stream(
            processing="radiance",
            fps=5.0, count=300,
            apply_calibration=True,
            apply_white_balance=True):
        # frames is {serial: numpy_array}; cams not delivering this tick are omitted
        for serial, frame in frames.items():
            print(serial, frame.shape, frame.dtype, frame.mean())
```

### 5. 無頭直接硬體（無後端）擷取腳本

```python
from chloros_sdk import LatticeCamera, PRESETS, discover_cameras

cams = discover_cameras(timeout_ms=3000)
print(f"Found {len(cams)} cams")

settings = PRESETS["high_quality"]
for c in cams:
    with LatticeCamera(serial=c.serial, settings=settings) as cam:
        result = cam.capture(output_dir="./out", format="tiff")
        print(c.serial, result.filepath)
```

### 6. 連接 4 鏡頭陣列前的功能檢測

```python
import chloros_sdk

serials = ["214701288", "213800234", "214000533", "214701162"]

probe = chloros_sdk.analyze_array_network(
    master_serial=serials[0],
    slave_serials=serials[1:],
    width=2048, height=1536,
    pixel_format="BayerRG12",
)

if probe["status"] == "ok":
    arr = chloros_sdk.connect_array(
        serials, width=2048, height=1536, pixel_format="BayerRG12")
elif probe["status"] == "auto_capped_fps":
    r = probe["recommended"]
    print(f"Keeping resolution; capping trigger rate at "
          f"{r['recommended_target_fps']} fps")
    arr = chloros_sdk.connect_array(
        serials, width=2048, height=1536, pixel_format="BayerRG12",
        target_fps=r["recommended_target_fps"])
elif probe["status"] == "auto_shrunk":
    r = probe["recommended"]
    print(f"Auto-shrinking to {r['out_width']}x{r['out_height']} "
          f"binning={r['binning']} for sim-sync")
    arr = chloros_sdk.connect_array(
        serials,
        width=r["width"], height=r["height"],     # before binning
        pixel_format=r["pixel_format"], binning=r["binning"])
elif probe["status"] == "needs_force_slip":
    print("Wire can't sustain sim-sync; falling back to slip mode")
    arr = chloros_sdk.connect_array(
        serials, force_tier="slip-emit-and-capture")
else:
    raise RuntimeError(f"Probe error: {probe.get('error')}")
```

### 7. 相當於擷取配方（純 Python）

`run_project_recipe()` 會直接執行 CLI 配方（參見 [專案資料夾與配方](#project-folder--recipes)）。若改以 Python 編寫，步驟如下：

```python
import time, chloros_sdk

with chloros_sdk.open_project("/path/to/proj") as proj:
    proj.connect_all()
    cam = proj.cameras["FrontLeft"]
    rig = proj.arrays["main_rig"]
    sky = proj.sensors["Sky"]

    # apply: connect_all() applied each camera's saved settings. CameraHandle
    # has no settings method; a CameraSession on the same pooled camera does
    # (it is borrowed, so it never releases the camera)
    chloros_sdk.connect_camera(cam.serial).set_settings(
        exposure_time=5000, gain=6.0)

    # wait
    time.sleep(2)

    # capture
    cam.capture("pose_a/", format="tiff", processing="radiance")

    # stream
    rig.stream(count=60, fps=5, output_dir="stream/", processing="raw")

    # sensor read
    print(sky.read())
```

---

## MCP 伺服器（AI 助理）

`chloros-mcp` 允許支援 [模型上下文協定](https://modelcontextprotocol.io)（MCP）的 AI 助理（例如 Claude Desktop、Claude Code 或 Cursor）驅動 Chloros：處理影像（建立專案、匯入擷取檔案夾、 選擇設定、進行處理、回報執行結果）並操控 LATTICE 相機、陣列及 DAQ 光感測器（連接、預覽、設定曝光與色彩、擷取、記錄、更新韌體）。其工具涵蓋此 SDK 的所有功能。您只需以自然語言下達指令，助理便會呼叫伺服器的工具：

```text
You:  Process the flight in D:\Flights\2026-10-17 to reflectance and NDVI.

The assistant calls:
  create_project          {"project_name": "Flight_2026-10-17"}
  import_images           {"folder_path": "D:\\Flights\\2026-10-17"}
  configure_processing    {"indices": ["NDVI"]}
  start_processing        {}
  get_processing_status   {}      (every 30-60 s until its state is "finished")

You:  Show me what the array sees, then capture reflectance every 10 seconds
      for 5 minutes.

The assistant calls:
  list_devices            {}
  get_camera_preview      {"array_id": "array-1779862544497"}
  start_capture_run       {"array_id": "array-1779862544497", "mode": "interval",
                           "interval_s": 10, "duration_s": 300,
                           "processing": "reflectance"}
  get_job_status          {"job_id": "..."}   (until its state is "finished")
```

伺服器是基於此 SDK 建構的：處理工具會呼叫 [`ChlorosLocal`](#image-processing--chloroslocal)，而硬體工具則由 [smart-connect 呼叫](#smart-connect-for-lattice-cameras)、[DAQ 會話](#daq-sensor-sessions)、`ChlorosProject` 以及 [相機實用程式](#camera-utilities)，所有這些都與 Chloros 應用程式所使用的同一後端進行通訊。其請求攜帶 `X-Chloros-Client: sdk`， 因此後端會套用 SDK 的登入與方案檢查機制，並遵循 SDK 的相關規則（例如，LATTICE 擷取畫面將無法使用「Texture Aware」功能）。

### 安裝

伺服器已包含在 `chloros-sdk` 套件中；無需安裝其他內容。SDK 會安裝一個 `chloros-mcp` 指令，其功能與 `python -m chloros_sdk.mcp_server` 相同：

```bash
pip install chloros-sdk        # not needed where the Chloros installer set up the SDK (Step 2)
chloros-mcp --help
```

- **Windows：** Chloros 安裝程式會透過 pip 將 SDK 安裝至您的 Python 環境中，並將 `chloros-mcp.exe` 放置於該 Python 的 `Scripts` 資料夾內。
- **Linux（.deb）：** 此套件會將SDK安裝至系統的 `python3` 目錄，並將 `chloros-mcp` 指令放置於 `/usr/bin` 目錄中，因此該指令會出現在每位使用者的 `PATH` 目錄中。在 virtualenv 環境中，`pip install /usr/lib/chloros/sdk/chloros_sdk-*.whl` 會將 `chloros-mcp` 加入 virtualenv 的 `bin` 目錄中。

伺服器僅使用 SDK（Python 3.7+）之上的標準函式庫。MCP 客戶端會啟動它，並透過 stdin/stdout 與其通訊；您無需自行執行它，除非使用 `--help`。

### 客戶端設定

**Claude Desktop：**將伺服器新增至 `claude_desktop_config.json`（在 Windows 系統中為 `%APPDATA%\Claude\claude_desktop_config.json`；透過**設定 > 開發者 > 編輯設定** 可開啟該視窗），然後重新啟動 Claude Desktop：

```json
{
  "mcpServers": {
    "chloros": {"command": "chloros-mcp"}
  }
}
```

**Claude 程式碼：**

```bash
claude mcp add chloros -- chloros-mcp
```

**游標：**與 Claude Desktop 相同的 `mcpServers` 設定，位於 `~/.cursor/mcp.json`（所有專案；Windows 系統為 `%USERPROFILE%\.cursor\mcp.json`）或 `.cursor/mcp.json` 中。

選項應置於 `args` 中，或位於 Claude Code 指令（`claude mcp add chloros -- chloros-mcp --no-auto-start`）之後。 若要使用特定的 Python SDK 取代 `chloros-mcp` 中的 `PATH`， 請使用該Python模組執行（若非`PATH`上的第一個`python3`，則其完整路徑應為`command`）：

```json
{
  "mcpServers": {
    "chloros": {"command": "python3", "args": ["-m", "chloros_sdk.mcp_server"]}
  }
}
```

由於客戶端是在未使用您的 shell 的情況下啟動伺服器，因此若客戶端回報伺服器無法啟動，可能是因為 `chloros-mcp` 並未出現在客戶端所見的 `PATH` 上。 請改為指定完整路徑（Windows 系統為 `where chloros-mcp`，Linux 系統為 `which chloros-mcp`，其中 .deb 檔案位於 `/usr/bin/chloros-mcp`），或使用包含 SDK 的 Python 執行該模組。

### 選項

| 選項 | 預設值 | 功能說明 |
| --- | --- | --- |
| `--backend-url URL` | `http://127.0.0.1:5000` | 要使用的 Chloros 後端。 |
| `--no-auto-start` | off | 絕不啟動後端：使用已運行的後端，例如 Chloros 應用程式、Linux 後端服務，或透過隧道連接到另一台機器的 （參見 [遠端後端模式](#remote-backend-mode-pip-only-host-via-tunnel)；此時兩台機器上的資料夾路徑必須指向相同的資料夾）。 |
| `--backend-exe PATH` | Chloros 安裝的位置 | 當沒有後端正在運行時，針對 Chloros 安裝在 SDK 無法偵測到的位置時，應啟動的後端。 |
| `--log-level LEVEL` | `WARNING` | `DEBUG`、`INFO`、 `WARNING` 或 `ERROR`。日誌會輸出至 stderr，多數客戶端會將其顯示在 MCP 日誌中；stdout 僅傳輸協定訊息。 |

### 工具

每個工具的參數皆為其封裝之 SDK 呼叫的參數；客戶端會列出每個參數及其完整的結構。路徑皆為絕對路徑。

**處理**

| 工具 | SDK 呼叫 | 功能 |
| --- | --- | --- |
| `create_project` | `ChlorosLocal.create_project` | 建立專案並將其設為開啟的專案，後續處理工具將以此專案為基礎進行操作。若僅指定名稱，則會在預設專案資料夾中建立資料夾；若指定絕對路徑，則直接在該位置建立資料夾。 |
| `open_project` | `ChlorosLocal.open_project` | 開啟已儲存的專案，並將其設為當前開啟的專案。 已移動專案的路徑將重新指向其目前所在的資料夾；`relink_folder`（絕對路徑）則指定其他遺失檔案目前所在的資料夾。 傳回 `source_files` 和 `image_count`。未找到任何項目：列出其位置時發生錯誤，並命名為 `relink_folder`；目前未開啟任何項目。 |
| `import_images` | `import_images` | 匯入Survey3的 `.RAW`+`.JPG` 配對檔，LATTICE `.tif` 擷取資料及 DAQ `.daq` 記錄；回傳 `count`（影像）與 `scan_count`（記錄）。 |
| `configure_processing` | `configure` | 僅變更傳入的設定，不變更其他設定（值：[支援的值](#supported-values)）；`custom_settings` 傳送完整的設定檔，內容與傳入值完全一致。 |
| `get_project_settings` | `get_config` | 讀取已開啟專案中儲存的處理、匯出及索引設定。 |
| `start_processing` | `process(wait=False)` | 開始處理已開啟的專案，並立即返回。 |
| `get_processing_status` | `get_status` | 回報 `state`（`idle`、`starting`、`processing` 或 `finished`）及進度； 執行完成後，若無法驗證匯出檔案，則產生 `summary`、`export_check_failure`；若影像遺失，則產生 `image_failure`：其餘產出檔案均會寫入；將該執行任務報告為失敗。 |
| `process_folder` | `process_folder` / `process_lattice_capture` | 透過單一呼叫為資料夾建立專案、匯入專案、進行設定並啟動執行 （針對 LATTICE 擷取的 `lattice: true`）。當執行任務正在進行時，系統會在建立任何內容之前即拒絕該請求，且後端此時也會拒絕來自任何工具的專案變更。 |
| `export_light_sensor` | `export_light_sensor` | 針對專案的光感測器記錄，寫入經校準的 `.daq` 和 `.csv` 檔案 記錄，且不進行影像處理；若未緩存，則從 MAPIR Cloud 取得工廠校準資料（參見 [光感測器記錄](#light-sensor-recordings--calibrated-daq--csv)）。 |
| `read_image_audit_tags` | `read_image_audit_tags` | 讀取經處理影像的 Chloros 稽核元資料（來源、校準、已套用的處理階段）。 |

**相機與陣列**

| 工具 | SDK 呼叫 | 功能說明 |
| --- | --- | --- |
| `list_devices` | `list_cameras`, `list_arrays`、`list_daq_sensors` | 目前在任何客戶端中已連線的裝置（每個裝置皆對應 `connected_here`）；以及伺服器上正在執行的任務。 |
| `discover_devices` | `discover_lattice_cameras`、`discover_daq_sensors` | 可連接的攝影機與光線感測器。 |
| `get_device_status` | `CameraSession.status`、`ArraySession.status` / `capability`, `DAQSensorSession.status` | 單一攝影機、陣列（含其功能與運作狀態）或光感測器的即時狀態。 |
| `analyze_array_network` | `analyze_array_network` | 此主機與網路可為擬議陣列承載的內容。 |
| `connect_camera` / `connect_array` | `connect_camera` / `connect_array` | 連接一台攝影機，或一個同步陣列（主機優先）. |
| `connect_project_devices` | `open_project(...).connect_all` | 連接儲存專案中記錄的所有裝置，並套用其設定。 |
| `disconnect_device` | `disconnect` / `force_disconnect` | 釋放攝影機、陣列或光感測器（參見「行為」）。陣列上的 `keep_settings: true` 會將每台攝影機的設定儲存至其專屬記憶體中，`clear_settings: true` 則會重置該記憶體 （其結果包含 `camera_memory`）。 |
| `disconnect_project_devices`、`align_project_arrays`、`capture_project_devices` | `disconnect_all`、`align_arrays`、`capture_all` | 專案中的所有裝置，一併列出。 |
| `read_camera_settings` / `set_camera_settings` | `CameraSession.read_nodes` / `set_settings` | 透過友善名稱或 GenIC節點名稱讀取或寫入攝影機設定。 |
| `set_array_frame_rate` | `ArraySession.set_target_fps` | 固定陣列的幀率，或將其還原為自動幀率。 |
| `capture_camera` / `capture_array` | `CameraSession.capture` / `ArraySession.capture` | 單次擷取（`fastest` 與 `wait_for_exposure` 為 `capture_fastest` 及曝光已確定的擷取檔）。若無 `output_dir`，則會儲存至開啟的專案的 `captures` 資料夾中，且檔案會加入該專案（暫不處理，直至一次處理執行結束）；回覆中的 `project_add` 即表明此點。 |
| `start_capture_run` | `ArraySession.continuous`、`capture_repeated` | 連續或間隔擷取，作為一項工作。 |
| `start_array_recording` | `ArraySession.record` / `burst` | 作為工作處理的組合索引影片，或原始連拍。 |
| `build_burst_video` | `ArraySession.build_video` | 將連拍資料重新處理為經校準的影片，作為一項工作。 |
| `get_job_status` / `stop_job` | | 追蹤或停止一個工作。 |
| `get_camera_preview` | | 每台攝影機的即時預覽，以影像形式回傳。 |
| `grab_frames` | `ArraySession.grab`, `CameraHandle.grab` | 處理層級中的一幀（或一組同步影像），不儲存：包含各平面統計資料及每幀的PNG檔案。 |
| `calibrate_array_alignment`、`get_array_alignment`、`clear_array_alignment`、`nudge_array_alignment`、`export_array_alignment`, `import_array_alignment` | `ArrayHandle.calibrate_alignment`, `alignment_status`, `clear_alignment`、`tweak_alignment`、`export_alignment`、`import_alignment` | 陣列的對齊方式。 |

**光感測器**

| 工具 | SDK 呼叫 | 功能說明 |
| --- | --- | --- |
| `connect_light_sensor` | `connect_daq_sensor` | 連接 DAQ-U、DAQ-M 或 DAQ-E（`cap_id` 用於設定其上限校正）。 |
| `get_light_sensor_reading` | `DAQSensorSession.latest` | 最新讀數：時間戳記、飽和度、光譜範圍與峰值，以及按波長區間平均的光譜（`full_spectrum` 適用於每個數據點）。 |
| `set_light_sensor_streaming` | `stream_start` / `stream_stop` | 啟動或停止資料串流。 |
| `start_light_sensor_recording` / `stop_light_sensor_recording` | `record_start` / `record_stop` | 以工作為單位記錄 `.daq` 檔案； `duration_s` 會自動停止該作業。 |
| `set_light_sensor_cap` | `DAQSensorSession.set_cap` | 電容校正 （`none` 適用於裸感測器）。 |
| `get_light_sensor_bindings` / `set_light_sensor_bindings` | `daq_bindings` / `set_daq_bindings` | 各感測器分別為哪台相機及陣列提供反射率數據。 |
| `get_light_sensor_band_presets` / `get_light_sensor_reflectance` | `daq_band_presets` / `light_sensor_reflectance` | 波段集，以及兩個串流感測器的環境光 + 物體反射率與折射率。 |
| `get_light_sensor_device_info`、`set_light_sensor_network`、`set_light_sensor_auto_stream`、`enable_light_sensor_ptp`、`reboot_light_sensor`, `update_light_sensor_firmware` | `device_info`, `set_network`、`set_auto_stream`、`enable_ptp`、`reboot`、 `update_firmware` | DAQ-E 本身的設定，儲存於裝置中。 |

**相機公用程式** ([相機公用程式](#camera-utilities))

| 工具 | SDK 呼叫 | 功能說明 |
| --- | --- | --- |
| `get_camera_color_settings` / `set_camera_color_settings` | `camera_color_profile`、`camera_color_adjust`、`camera_highlights` 及其設定器 | RGB 攝影機的顯示色彩： 色階檔與色溫、飽和度／對比度、高光。 |
| `run_camera_white_balance` / `set_camera_white_balance` | `camera_white_balance_once` / `set_camera_white_balance` | 單次白平衡，或自動／關閉／固定比例。 |
| `list_camera_presets` / `apply_camera_preset` | `camera_presets` / `apply_camera_preset` | 相機預設模式。 |
| `list_filter_models` | `camera_filter_models`、`camera_filter_model_info` | 濾鏡型號，或依名稱逐一選擇。 |
| `get_camera_firmware_status` / `update_camera_firmware` | `camera_firmware_status` / `update_camera_firmware` | 韌體版本，以及一則快訊（預設為一項任務）。 |
| `get_camera_power_info` / `set_camera_power` | `camera_power_probe`, `sleep_camera`、`wake_camera` | 電源與熱狀態；睡眠或喚醒。 |
| `calibrate_camera_with_target` | `calibrate_camera_with_target` | 當前視圖中校準標靶的係數。 |
| `get_network_adapters` / `fix_network_settings` | `network_adapters` / `fix_network_settings` | 攝影機網路介面卡、估算值及修正 （僅限管理員）。 |
| `apply_vignette_correction` | `apply_vignette_correction` | 對一整個資料夾的影像進行暗角校正。 |
| `read_dls` | `read_dls` | 一次 DLS 讀取。 |
| `probe_system` / `get_system_info` | `probe_system` / `system_info` | 主機的硬體與執行環境。 |

**瞬態相機、對齊配置檔與索引** ([瞬態陣列、對齊配置檔與索引](#transient-arrays-alignment-profiles--indices))。此處所有開啟相機的工具均會使用 `keep_settings` / `clear_settings`（預設值為 false）：請參閱 [相機記憶體](#camera-memory--keep_settings--clear_settings)。

| 工具 | SDK 呼叫 | 功能說明 |
| --- | --- | --- |
| `start_camera_continuous` / `start_multi_camera_stream` | `start_camera_continuous` / `start_multi_camera_stream` | 將單一攝影機錄製，或同步串流多台攝影機，作為一項工作 （`duration_s` 會自動停止該工作）。 |
| `capture_cameras_transient` | `capture_cameras_transient` | 進行一次同步擷取，無需連接陣列。 |
| `test_array_sync` / `detect_array_master` | `test_array_sync` / `detect_array_master` | 觸發同步檢查，以及哪些攝影機位於同步線上。 |
| `analyze_array_models` | `analyze_array_models` | 若在該網路中部署此型號的相機陣列，將獲得何種效果。 |
| `calibrate_alignment_profile` | `calibrate_alignment_profile` | 求解並儲存對齊設定檔（`options` 包含更細緻的求解器選項）。 |
| `capture_aligned_multiband` / `start_aligned_stream` | `capture_aligned_multiband` / `start_aligned_stream` | 一張已對齊的多波段影像，或作為工作處理的一系列此類影像。 |
| `read_alignment_profile` / `reorder_alignment_profile` | `alignment_profile_info` / `reorder_alignment_profile` | 配置檔的內容；其波段順序及啟用的相機。 |
| `list_index_presets`、`compute_index`、`capture_live_index` | `index_catalog`、`compute_index`, `capture_live_index` | 索引預設值與漸層；多波段 TIFF 的索引；一步完成「擷取至索引」操作。 |

**時間同步、專案資料夾與配方** ([PTP 主時鐘](#ptp-grandmaster--time_sync_status), [專案資料夾與配方](#project-folder--recipes))

| 工具 | SDK 呼叫 | 功能說明 |
| --- | --- | --- |
| `get_time_sync_status` | `time_sync_status`、`time_sync_peers`、 `time_sync_cameras` | PTP 主時鐘的健康狀態、其對等節點以及每台陣列相機的 PTP 狀態。 |
| `restart_time_sync` / `set_time_sync_priority` | `restart_time_sync` / `set_time_sync_priority` | 重新啟動它；設定其優先級。 |
| `get_project_folder` / `set_project_folder` / `reset_project_folder` | `project_folder` / `set_project_folder` / `reset_project_folder` | 建立新專案的資料夾。 |
| `run_project_recipe` | `run_project_recipe` | 執行已儲存專案的製程配方，並以工作單形式執行並記錄其日誌。 首先進行檢查：若配方有錯誤則會被拒絕且不執行任何操作；針對未連接的陣列／感測器會發出警告（此時不會連接它們：需先執行 `connect_project_devices`）。`dry_run` 會返回檢查結果且不執行任何操作。 |

沒有工具可開啟已儲存的專案進行處理：每個處理工作皆以 `create_project`（或 `process_folder`）開始。

### 需求

- **同一台電腦上已安裝 Chloros。** 處理作業由 Chloros 後端執行。伺服器亦可改用透過隧道連線的後端（`--backend-url` 搭配 `--no-auto-start`，詳見 [選項](#options)）。
- **需在該電腦上登入**，可透過 Chloros 應用程式或使用 `chloros-cli login` 登入。伺服器無法代您登入：若在未登入狀態下呼叫此工具，系統會回報「未登入 Chloros」並附上後端的理由，助理會向您說明。
- **任何付費的 Chloros+ 方案（Copper 或更高）。** 伺服器需透過 SDK 存取，而免費的 Iron 方案則不提供此功能：若在免費方案中呼叫該工具，系統會回報「此 Chloros 方案不提供此功能」，並附上後端的原因。
- 需搭配 Python 3.7 或更新版本及 SDK，並需具備可啟動本地（stdio）伺服器的 MCP 客戶端。

### 運作行為

- **後端。** 啟動伺服器或列出其工具時不會執行任何操作；首次呼叫工具時才會建立連線。若 Chloros 應用程式（或 Linux 後端服務）正在運行，伺服器將使用其後端並讓其持續運行。 若無後端正在運行，伺服器會啟動已安裝的後端，且以無視窗模式運行。當伺服器退出時（您關閉客戶端，或客戶端重新啟動伺服器），它會停止其啟動的擷取工作與光感測器錄製，並釋放已連接的裝置；後端則持續運行， 因此未完成的處理流程將持續執行至結束，若 30 分鐘內無人使用，它會自動停止運作。在 `--no-auto-start` 情況下，伺服器絕不會啟動後端：若在無後端運行的狀態下呼叫工具，系統會提示您開啟 Chloros 應用程式，待您完成後，下一次呼叫即會建立連線。
- **僅有一個開啟的專案。** 後端僅有一個開啟的專案，當 Chloros 應用程式開啟時，此專案會與該應用程式共用；因此，在助理運作期間，請勿在應用程式中處理其他專案。
- **僅會變更傳入的設定。** `configure_processing` 會將所有未傳入的設定保留為其儲存的值（SDK 的 `configure()` 會在未傳入參數時，將去拜爾演算法、暗角校正、反射率校準、PPK 及匯出格式重設為預設值； 伺服器會從專案中填入這些設定）。若無法讀取專案的儲存設定，則不會進行任何變更並會顯示此訊息，除非這五項設定皆已提供。`indices` 會覆寫專案的索引清單。結果清單會列出以 `settings_sent` 列出傳送的內容，並在 `warnings` 下為相機波段無法生成的索引命名。
- **處理採用「開始後輪詢」模式。** `start_processing` 會在執行開始後立即返回，隨後處理過程需耗時數分鐘至數小時， 隨後輔助程式會接續執行 `get_processing_status`。`state` 可能為 `idle`（無處理任務）、`starting`（後端尚未回報運行狀態，最長可能延遲 20 秒）、`processing` 或 `finished`，並附上該次運行的 [摘要](#post-run-summary--hints)：包含其輸出內容，以及當輸出量少於預期時的提示（無影像、未找到校準目標）。產出檔案會寫入專案資料夾中，與應用程式中的做法相同。 當運行仍在進行中時，系統會拒絕 `start_processing`；若後端停止回應，則會被報告為錯誤，而非已完成的運行。
- **助理所見內容。** SDK 的警告會以 `warnings` 的形式顯示在工具結果中，而失敗狀況（已登出、計畫問題、路徑錯誤）則會以錯誤文字呈現，助理可讀取並將其轉達給您。 若清單項目超過 20 項，將截取前 20 項，並以 `<key>_total` 標示其完整長度。路徑必須為絕對路徑。
- **裝置為共用。** 攝影機、 陣列及光感測器皆位於後端，因此助理可驅動已透過 Chloros 應用程式連線的裝置，其變更內容將顯示於該應用程式中。`disconnect_device` 會釋出伺服器所連線的裝置；若另一位使用者（應用程式、`chloros-cli`、 腳本、另一個 `chloros-mcp`）所連接的裝置，僅會透過 `force: true`（即 SDK 的 `force_disconnect`）釋放，且當後端知曉連接來源時，會回報拒絕名稱； 後端完全未持有的裝置，則會被報告為未連線（`released: false`、`connected: false`，無需釋放任何內容）。後端執行的錄影或串流會帶有其 `get_job_status` 檢視，以及 `counts` — `frames_written`、`frames_dropped`、 `frames_failed`、`drop_reason`（多鏡頭串流的幀群） — 以及一行 `summary`（「寫入 220 幀，丟棄 55 幀，0 次失敗：...」），在執行期間顯示，並於完成後顯示其最終數值；另一位客戶的清單中 （`other_clients_jobs`）顯示相同內容（`list_cameras()`、`list_arrays()` 及 `list_daq_sensors()` 則分別顯示各裝置的的 `connected_by`：`client`、`label`、 `name`、`at`)。 每個 `chloros-mcp` 都會向後端將自身命名為 `chloros-mcp (pid N)`（`chloros_sdk.connect.CLIENT_LABEL`，傳送時稱為 `X-Chloros-Client-Label`；任何 SDK 腳本均可設定此值）。停止其他客戶端的光感測器錄製同樣會消耗 `force`。
- **長時間運作即為一項工作。** `start_capture_run`、 `start_array_recording`、`build_burst_video`、`start_light_sensor_recording`、`start_camera_continuous`、 `start_multi_camera_stream`、`start_aligned_stream`、`run_project_recipe`、 以及任何針對 `background` 所要求的擷取、求解或索引操作，都會立即返回一個 `job_id`；隨後是 `get_job_status`，而 `stop_job` 則結束 。除非您傳入 `max_captures`、`duration_s`（或針對突發處理時傳入 `max_frames`）；當磁碟機剩餘空間不足時，`min_free_gb` 將拒絕啟動。除非您設定限制，否則不會施加任何限制。後端執行的錄製或串流（`start_camera_continuous`、 `start_multi_camera_stream`、`start_aligned_stream`）會保留後端的工作 ID，因此另一個 `chloros-mcp` —— 即稍後啟動的 —— 會透過該 ID 找到它： `get_job_status` 會跟隨其後（並列出諸如 `other_clients_jobs`、源自 `GET /api/camera/cli-jobs` 的工作，每個工作皆包含 `started_by`）， 而 `stop_job` 僅能透過 `force: true` 來終止它。任何其他工作 （無論是擷取執行、陣列錄製或突發處理、背景擷取）皆在啟動它的 `chloros-mcp` 中執行，且僅有該伺服器知曉此狀況： 當另一個工作請求該任務時，系統會如此顯示。已被停止的工作在寫出其佇列時讀取 `stopping`，接著讀取包含最終計數的 `finished`， 持續 10 分鐘，並從任何 `chloros-mcp` 讀取： 啟動該任務的伺服器會以相同方式偵測到其他伺服器的強制停止。`get_job_status` 絕不會等待停止程序：無論是在自身停止期間（其自身的 `stop_job`、其他客戶端的，或是因工作自行結束而釋放攝影機的停止指令），它會立即以 `stopping` 及即時計數值回應；而 `stop_job` 若處於工作停止狀態，則會明確標示此狀態。已結束的工作會透過 `stop_reason` 中說明結束原因：其自身原因（`no_frames`、`disk_full`、`member_missing`、 或針對陣列連續執行時為 `count`、`duration`、`array_gone`)，而 `limit reached` 僅在計數顯示其 `count`（或某個突發事件的 `max_frames`、某個記錄的的 `duration_s`）達到時，`error`、`stopped` （搭配 `stopped_by`）當客戶端停止它時，否則為 `finished`。
- **擷取檔的儲存位置。** 存至開啟專案的 `captures` 資料夾中，應用程式會將擷取內容儲存於此處，除非指定了 `output_dir`。
- **影像。** `get_camera_preview` 和 `grab_frames` 除了其 JSON 資料外，還會回傳助理可檢視的影像。`grab_frames` 影像會進行對比度拉伸以利檢視；其統計數據則是實際數值。
- **當伺服器退出時**，其啟動的任務將停止，且其啟動的光感測器記錄也會關閉。若伺服器曾啟動後端，則會在停止後端之前釋放先前連接的裝置；而在應用程式的後端中，這些裝置仍保持連接狀態。
- **通訊協定。** 僅支援 stdio（無 HTTP 端點）、僅限工具（無資源或提示）、文字與圖像結果，以及 MCP 通訊協定版本 2024-11-05 至 2025-11-25。**未包含的功能。** 登入或登出（帳戶屬於您；伺服器無法登入），以及停止伺服器未啟動的後端。此 SDK 在處理與硬體方面的其他所有功能，皆有對應的工具。（DAQ-E 的配置——其名稱、認證憑證、 OTA 密碼、簽名要求、恢復出廠設定——以及直接 DAQ 伺服器的配置，均由 `chloros-cli` 專屬處理，此功能亦包含在 SDK 中：若名稱或憑證錯誤，將導致所有 Chloros 安裝版本無法存取該裝置。）

---

## 後端自動啟動

智慧連線（smart-connect）的入口點——`connect_camera`、`connect_array`、 `connect_daq_sensor`、`discover_lattice_cameras` 以及 `discover_daq_sensors` —— 是一組輕量級的 HTTP 客戶端，它們假設後端正在 `127.0.0.1:5000` （智慧連線介面的預設 URL）上進行監聽。當 GUI 或 CLI 已處於運行狀態時，便會存在一個後端。若從裸腳本執行， 則可能沒有——因此這些函式會在首次呼叫前 **自動啟動內建的後端二進位檔**（無視窗模式，與 `ChlorosLocal` 的運作方式相同），然後等待最多 60 秒直至其就緒。 `list_cameras()`、`list_arrays()`、`list_daq_sensors()`、 `attach_array()` 和 `analyze_array_network()` 則永遠不會啟動後端。

規則：

- **僅會啟動指向本機URL的連線。** 指向 `localhost` / `127.0.0.1` / `[::1]` 才符合資格；任何其他主機均被視為他人的機器，絕不會被啟動。
- **若後端在回應前即退出，則等待程序將終止。** 此時呼叫會立即拋出異常，並傳回退出代碼、所執行的命令以及後端輸出最後幾行內容，而非等待 60 秒。退出代碼 98 屬例外情況： 另一個後端已佔用該埠或鎖定檔案（可能是 Chloros 應用程式、後端服務，或是仍在啟動中的其他腳本），因此該呼叫會持續等待該後端回應。`ChlorosLocal` 的行為亦同。
- **後端將保持運行以供重複使用**（與CLI相同）— 當您的腳本退出時，不會進行隱含的關機。重新執行腳本時，會重複使用該正在運行的後端。
- **若需停用此功能，請在上述任何呼叫中使用 `auto_start_backend=False`**（例如：當您指向遠端後端，或自行管理後端生命週期時）。

```python
import chloros_sdk

# Fresh shell, no backend running, no GUI open — this still works:
with chloros_sdk.connect_camera("213800234") as cam:   # spawns the backend
    cam.capture("output/")

# Remote backend (via tunnel — see Remote-Backend Mode): don't spawn one locally
arr = chloros_sdk.connect_array(serials,
                                backend_url="http://127.0.0.1:5000",
                                auto_start_backend=False)
```

若無法定位或啟動內建二進位檔，後續的 HTTP 呼叫會觸發可處理的、**具平台意識** 的 `ChlorosConnectError` 錯誤，而非單純的連線遭拒追蹤訊息 — 在 Windows 上，它會引導您至桌面應用程式或 `chloros-cli` 指令；在 Linux（無 GUI）上，則會引導您至 `chloros-cli` 指令或 `.deb`。

---

## 環境與標頭

SDK 會為每個後端 HTTP 呼叫標記 `X-Chloros-Client: sdk`。後端會套用 SDK/CLI 授權規則（需登入 **且** 擁有付費的 Chloros+ 方案），而非 GUI 的免費層級路徑。 此設定會在匯入時自動完成 — 您無需進行任何操作。

此標頭會新增至每個 `requests` / `urllib` 呼叫中，無論使用哪個埠（包括您自己的本地服務）——只要其 URL 包含 `localhost` 或 `127.0.0.1` ——並會覆寫您自行設定的值。對其他主機的呼叫則不受影響。

若要覆寫後端URL，請傳入 `backend_url=`（`ChlorosLocal` 接受 `backend_url=` 或 `api_url=`；而 `process_folder()` 和 `process_lattice_capture()` 僅接受 `api_url=`):

```python
chloros_sdk.connect_camera("213800234", backend_url="http://127.0.0.1:5000")
chloros_sdk.connect_array(serials, backend_url="http://127.0.0.1:5000")
chloros_sdk.connect_daq_sensor(eth_host="daq-e-1.local",
                                backend_url="http://127.0.0.1:5000")
chloros_sdk.ChlorosLocal(backend_url="http://127.0.0.1:5000")
```

（已安裝的後端僅監聽迴路介面，因此局域網中的 `backend_url` 無法連線至該後端；請參閱 [遠端後端模式](#remote-backend-mode-pip-only-host-via-tunnel) 以了解隧道模式。）

---

## 版本控制與相容性

- SDK 版本以 `chloros_sdk.__version__` 形式呈現。
- SDK 的行為會與捆綁的後端版本綁定。將舊版 SDK 與新版後端混合使用通常可行（向前相容的端點），但若將新版 SDK 與舊版後端混合使用，可能會在新端點上引發 `404` 錯誤 — 請將桌面應用程式升級至相容版本。
- 智慧連線介面（`connect_camera` / `connect_array` / `connect_daq_sensor`）與網路分析端點會返回穩定的 JSON 架構；新欄位會以追加方式加入。

---

## 疑難排解要點

- **`ChlorosAuthenticationError: Login required`**（源自 `ChlorosLocal`；智慧連線呼叫因相同原因會觸發 `ChlorosConnectError: Login required. …`） → 在此機器上執行一次 `chloros-cli login EMAIL PASSWORD`，或透過 Chloros 桌面應用程式登入。
- **`ChlorosProcessingError: Failed to start processing: … The project folder is too deep for this computer …`**（或源自 `create_project()` 的 `Failed to create project: …`）→ 此機器已停用 Windows 長路徑功能（此為 Windows 預設設定）， 因此路徑長度不得超過 260 個字元，且產品路徑位於專案資料夾下方約 130 個字元之處。系統未進行任何處理。此訊息會顯示符合條件的最長專案資料夾名稱：請改用較短的專案資料夾名稱，或啟用 Windows 長路徑功能並重新啟動 Chloros。 Windows 安裝程式會提供「啟用 Windows 長路徑」選項，預設為勾選狀態；否則請使用群組原則：*電腦設定 > 管理範本 > 系統 > 檔案系統 > 啟用 Win32 長路徑*。
- **`ChlorosConnectError: No Chloros backend is running …`** → 智慧連線會呼叫自動啟動本機後端，因此此錯誤僅在無法找到或啟動內建二進位檔時才會出現（例如：僅有 pip專用主機未安裝桌面套件）。此訊息會根據平台調整：在 Windows 上請開啟桌面應用程式或執行任何 `chloros-cli` 指令； 在 Linux 上執行 `chloros-cli` 指令（無 GUI 介面）或安裝 `.deb`。若為遠端後端，請傳入 `backend_url=` （以及 `auto_start_backend=False`）。
- **`CAMERA_AVAILABLE == False`** 在匯入時 → `lattice_sdk` 載入失敗：Arena SDK 執行階段，或 OpenCV (`cv2`)，無法在此Python環境中匯入。後端驅動的表面 (`connect_*`，透過後端進行投影，`ChlorosLocal`) 仍可正常運作。
- **陣列在全感測器模式下連接，但幀率偏低**→ 設計使然：自動模式會維持全感測器，並讓幀率降至線路所能承載的水平。傳遞 `binning=2` / `4`，或使用較小的 `width` / `height` 以獲得更高每秒幀數，亦可啟用巨型幀 / 升級鏈路。 像素合併與 ROI**無法**解決彙總超訂閱問題（`oversubscribed: true`，fps 欄位 0）： 線路承載的攝影機數量過多無法透過像素合併/ROI 解決 — 請減少攝影機數量、啟用巨型幀，或更換為更快的網路介面卡（參見 [超訂閱](#over-subscription-the-per-cam-floor)）。
- **`analyze_array_network()` 報告網卡接收環（RX ring）過小（約 0.26 MB）／連接閘道顯示「FRAMES WILL DROP」** → 主機網卡的接收環設定為預設值（網卡驅動程式更新後通常會重設為 32）。若使用 Realtek USB 10GbE 適配器，請設定 `ReceiveBufferLen=256` 和 `PendingReceives=64`（需提升權限），然後重新啟動後端服務，使其重新讀取環。完整操作步驟： [CLI 參考 → 主機網路介面卡設定與調校](cli-reference.md#host-nic-setup--tuning-lattice-arrays)。
- **主機在重新啟動／關機時當機，隨後出現 WMI `Invalid class` 錯誤／網路介面卡無法啟用** → 過時的 USB 10GbE 驅動程式導致 `DRIVER_POWER_STATE_FAILURE`（藍屏 `0x9F`）。請將網卡驅動程式更新至最新版本（≥ 2026），並重新套用接收環設定。 請參閱 [CLI 參考 → 主機 NIC 設定與調校](cli-reference.md#host-nic-setup--tuning-lattice-arrays)。
- **陣列擷取失敗，出現 HTTP 503「無新同步幀組」** → 陣列在等待期間未發佈任何新的幀組。檢查 `arr.status()`：`stalled: True` (或 `streaming: False`) 與 `actual_fps` 對照，若 `actual_fps` 為 0，表示擷取迴圈已停止發佈 — 請斷開並重新連線至陣列。若 `stalled` 為 False 且 `actual_fps` 大於 0，請重試。 緊接在 `connect_array()` 之後，已有擷取作業等待第一組資料長達 20 秒。
- **`status()` 顯示 `link_speed_mbps: 4294`** → Windows 無法讀取適配器的傳輸速率；此為 USB 以太網適配器的常見現象，並非故障。請參閱 [預期幀率](#expected-frame-rates)。
- **`update_firmware()` 因 `device never connected back on TCP <port>` 而失敗** → 防火牆阻擋了 DAQ-E 回傳至此電腦的連線。Windows 安裝程式會新增此規則（請參閱 [DAQ-E 裝置設定](#daq-e-device-settings)）；若使用其他防火牆，請允許來自感測器子網的 TCP 傳入連線至後端。
- **反射率被拒絕** → 若要取得絕對刻度反射率，必須將活躍的 DAQ 綁定至攝影機（或陣列）。可透過 GUI 進行綁定，或使用 `processing="radiance"`（W/m²/sr/nm），此模式無需配對感測器。
- **`smart=True` 擷取時間比預期更長** → AE 收斂取決於場景動態。若使用 `ArrayHandle.capture_smart()`，請擴大 `exposure_tolerance_pct`， 縮短 `stability_window_s` 或降低 `settle_timeout_s` 值，以獲得更快且較不穩定的觸發效果。

---

## 參見

- [CLI 參考手冊](cli-reference.md) — 每個 CLI 子指令都對應一個 SDK 呼叫。
- Chloros手冊中的 [DAQ 光感測器](https://mapir.gitbook.io/chloros/daq-light-sensors) —— 感測器、其上限與校準範圍、記錄功能、DAQ-E 網路連線。
- [LATTICE 攝影機](https://mapir.gitbook.io/chloros/lattice-cameras/lattice)（見於 Chloros 手冊）— 連接方式、攝影機設定、擷取模式、陣列。
- 線上文件：`https://mapir.gitbook.io/chloros/api-python-sdk`
