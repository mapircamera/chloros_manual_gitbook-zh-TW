# API：Python SDK

{% hint style="info" %}
**正在尋找完整的 API 嗎？** 本頁面為實作教學。所有公開類別、方法、精確的簽名以及可直接複製貼上的範例，皆收錄於 [SDK 參考手冊](reference/sdk-reference.md)，該手冊已針對 AI 助理進行優化。**正在使用 AI 助理嗎？** 將此 URL 貼入聊天視窗，即可取得完整的最新 Chloros 1.3.0 API：

`https://mapir.gitbook.io/chloros/reference/sdk-reference.md`

本手冊的每頁內容皆可透過其小寫 slug + `.md` 以原始 Markdown 格式取得，而整本手冊的索引則位於 `https://mapir.gitbook.io/chloros/llms.txt`。
{% endhint %}

**Chloros Python SDK**（PyPI 上的 `chloros-sdk`）可透過 Python 驅動桌面應用程式所能執行的一切功能：批次影像處理、LATTICE 即時攝影機與陣列控制、DAQ 光感測器工作階段，以及儲存專案的自動化執行。 它是在 GUI 和 CLI 所使用的相同本地後端（HTTP，位於 `127.0.0.1:5000`）之上的一層輕量級介面，因此這三種介面上的行為完全一致。

## 安裝

此 SDK 隨 Chloros 桌面套件一併提供，該套件同時包含處理後端與硬體執行環境，且版本始終與之相符。

**步驟 1 — 安裝 Chloros。**Windows：從 [下載](download.md) 頁面執行桌面安裝程式（預設路徑為 `C:\Program Files\Chloros\`）。 Linux：使用 `sudo apt install ./chloros_1.3.0_<arch>.deb` 安裝 `.deb` 套件（[Linux 安裝](linux/linux-installation.md)）。**步驟 2 — 檢查 SDK**（Python 3.7 以上）。通常無需安裝任何內容：

* **Windows：** 安裝程式會將內建的 SDK wheel 安裝至系統 Python 中（若未安裝 Python，則跳過此步驟）。
* **Linux：** `.deb` 會將系統 SDK `python3` 安裝至 `/usr/lib/python3/dist-packages`。 其所需套件 `python3-numpy` 和 `python3-requests` 由該套件建議安裝：`apt install` 會一併安裝，而 `dpkg -i` 則不會。
* **虛擬環境** 無法偵測到系統中的套件；請將隨附的 wheel 檔案安裝至其中： 在 Linux 上安裝 `pip install /usr/lib/chloros/sdk/chloros_sdk-*.whl`，或在 Windows 安裝資料夾的 `sdk` 資料夾中安裝 `chloros_sdk-*.whl`（預設為 `C:\Program Files\Chloros\sdk\`）。

```bash
python3 -c "import chloros_sdk; print(chloros_sdk.__version__)"   # Windows: py -c "…"
```

在未安裝 Chloros 套件的機器上，若其 SDK 驅動著另一台機器的後端（參見 [僅透過 pip 安裝會獲得什麼](#what-a-pip-only-install-gets)），請從 PyPI 安裝：`pip install chloros-sdk`。 PyPI 會隨著發行版本更新，因此內容與最新穩定版保持一致。

**步驟 3 — 每台機器登入一次：**

```bash
chloros-cli login user@example.com 'YourPassword'
```

憑證會快取於 `~/.chloros/` 中（兩大平台皆然）。在 Windows 上，您亦可透過桌面應用程式的「使用者」<img src=".gitbook/assets/icon_user.JPG" alt="" data-size="line"> 標籤頁進行登入。此 SDK 需訂閱付費的 Chloros+ 方案 — 請參閱下方的 [授權要求](#license-requirement)。

| 需求 | 詳細資訊 |
| --- | --- |
| **已安裝 Chloros** | Windows：桌面安裝程式；Linux：`.deb` 套件（提供後端二進位檔及 SDK） |
| **Python** | 3.7 或更高版本（於 3.10 版本上開發/測試） |
| **作業系統** | Windows 10/11 64 位元、Ubuntu 22.04 LTS 或更新版本，或 NVIDIA Jetson（JetPack 6） |
| **授權** | 有效的 Chloros+ 登入帳號，任何付費方案（Copper 或更高） |

## 60 秒搞定

只需一則呼叫，即可建立專案、匯入資料夾、設定處理流程並執行管線——若後端尚未運行，系統將自動啟動：

```python
import chloros_sdk

results = chloros_sdk.process_folder(
    "C:/DroneImages/Flight001",
    indices=["NDVI", "GNDVI", "OSAVI"],
)
```

（在 Linux 上，請使用 Linux 路徑：`/home/user/drone_images/flight001`。此 SDK 在兩個平台上的運作方式完全相同。）

正在處理 LATTICE 擷取資料夾嗎？ 請使用 LATTICE 專用封裝程式 — 它會套用正確的預設值（不進行面板目標偵測，採用標準去拜爾演算法）：

```python
results = chloros_sdk.process_lattice_capture(
    folder_path="C:/Captures/2026-05-13_Field",
    indices=["NDVI"],
)
```

## `ChlorosLocal` — 完整管線控制

若需執行超過單行指令的操作，請使用 `ChlorosLocal`。它會使用已運行的後端（即桌面應用程式的後端），或在首次使用時啟動已安裝的後端（`auto_start_backend=True`； 在 Windows 系統上，預設會從 Chloros 安裝資料夾中的 `backend` 資料夾啟動，預設為 `C:\Program Files\Chloros\backend\`），用以建立並配置專案、監控執行進度，並回傳執行後的摘要。

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

{% hint style="info" %}
請保留預設的 `http://127.0.0.1:5000`，而非替換為 `localhost` — 在 Windows 系統上，`localhost` 會先解析為 `::1`，且針對僅支援 IPv4 的後端，每次請求約需耗時 2 秒。
{% endhint %}

請將其用作上下文管理器。離開該區塊會終止此實例對後端的存取，但不會終止後端本身：後端會繼續運行以供下一個腳本使用，並在閒置 30 分鐘後自動停止。`shutdown_backend()` 會終止此實例所啟動的後端。

```python
import chloros_sdk

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

`configure()` 接受以下關鍵字： `debayer`、`vignette_correction`、`reflectance_calibration`、`indices`、`export_format`、 `ppk`、`daq_log_path`、`input_level`、`radiometric_output`、`array_alignment`、 `array_alignment_crop`、`array_alignment_interpolation` 以及 `custom_settings`。主要數值：

```python
# export_format
"TIFF (16-bit)"           # default, recommended
"TIFF (8-bit)"            # 8-bit TIFF; what all-8-bit images get for a deeper format
"TIFF (32-bit, Percent)"  # reflectance percentage as float32
"PNG (8-bit)"
"JPG (8-bit)"

# debayer
"High Quality (Faster)"                  # standard, default
"Texture Aware (Slow, Highest Quality)"  # neural debayer, Chloros+ only
```

LATTICE 專屬的控制參數（`input_level`、`radiometric_output` 以及 `array_alignment*` 系列）及其完整的數值表，詳見 [SDK 參考手冊](reference/sdk-reference.md#supported-values)中已完整記載其值表。

### 監控進度

```python
def show_progress(percent, message):
    print(f"[{percent:3d}%] {message}")

with chloros_sdk.ChlorosLocal() as cl:
    cl.create_project("FieldA")
    cl.import_images("C:/DroneImages/Flight001")
    cl.configure(indices=["NDVI"])
    cl.process(progress_callback=show_progress, poll_interval=1.0)
```

### 讀取執行後摘要 — 並偵測空執行

執行完成後，`process()` 會將後端的處理摘要附加為 `result["summary"]`。`summary["hints"]` 中的每個條目都是一句完整的說明，闡述任何值得注意的事項——例如， 為何某次執行未產生任何輸出——且每條提示都會以 Python 格式重新發送為 `UserWarning`，因此即使您從未檢查該字典，空執行也能自行診斷：

```python
result = cl.process()
for hint in result.get("summary", {}).get("hints", []):
    print("HINT:", hint)
# hints also arrive on the warnings channel:
#   python -W always::UserWarning your_script.py
```

{% hint style="warning" %}
**當執行結果未產生任何影像時，`process()` 不會拋出異常。**這是 SDK 與 CLI 刻意設計不同的唯一之處： `chloros-cli process` 會將「已請求產出，但未寫入任何產出」視為失敗並以非零狀態退出，而 SDK 則會正常返回，並透過 `summary` / 提示訊息回報此狀況。 若您的處理流程應在執行結果為空時停止，請自行檢查此狀況——檢視 `summary`（或統計專案資料夾下的檔案數量），而非依賴例外狀況。
{% endhint %}

## Smart Connect — 即時硬體

有三個輔助程式會在後端硬體池中開啟持久性連線 — 該硬體池與 CLI、其他腳本以及 GUI 的相機分頁所使用的池相同，因此 SDK 腳本可與它們共存，無需爭奪串列埠或網路頻寬。若未運行任何後端，這三個輔助程式都會自動啟動一個本地後端。 GUI 的 DAQ 面板不屬於此資源池：請先在 GUI 中關閉光感測器，再透過腳本開啟它。

### 單一 LATTICE 攝影機 — `connect_camera`

```python
import chloros_sdk

# Open by serial; reuses existing pool entry if one exists
with chloros_sdk.connect_camera("213800234") as cam:
    cam.set_settings(exposure_time=10000, gain=0.0)   # microseconds, dB
    cam.capture("output/")
```

{% hint style="info" %}
**相對路徑會指向腳本執行所在的位置（此行為於 1.3.0 版本中變更）。** 若使用本機後端，相對路徑 `output_dir`（包含預設的 `"output"`）——以及後端開啟的其他路徑 （`burst_dir`、`profile_path`、`save_path`、 `mask_path`）——會以您的腳本工作目錄為基準進行解析，因此 `cam.capture("output/")` 會寫入 `<current folder>/output/`。 在 1.2.0 版本中，已安裝的後端會根據其自身的安裝資料夾來解析這些路徑，並因「存取被拒絕」而失敗。若對應至遠端後端，路徑則會以輸入的原始形式傳送。
{% endhint %}

### 同步陣列 — `connect_array`

`connect_array` 是多攝影機架設的建議起始點。它執行與 GUI 相同的智慧預備流程：網路分析、同步層自動選取、PTP 時間同步、各攝影機的像素格式選取、AE 初始化，以及 GPIO 觸發器就緒。 **第一個序列號為主裝置**（負責發射硬體觸發脈衝）；其餘則為從裝置。

```python
with chloros_sdk.connect_array(
        ["213800234", "214000533", "214701288", "214701292"]) as arr:
    arr.capture("output/", processing="reflectance")
```

在任何陣列擷取中加入 `smart=True`，即可在觸發前等待所有攝影機的自動曝光穩定下來。

1.3.0 版本新增的 `ArraySession` 功能，可直接提供同步幀，無需寫入檔案：

* `arr.grab()` 會針對每台相機返回一幀影像，所有影像均來自單次觸發，並以 numpy 陣列形式呈現（`{serial: array}`）； `.group_id` 為共用觸發器，而 `.errors` 則代表任何未被包含的相機。
* `arr.frame_stream(fps=5.0)` 每 tick 產出一組此類資料，供您在自己的處理管線中進行即時處理。
* `arr.continuous("output/")` 會執行後端自身的連續擷取（即 GUI 中的「連續模式」），並返回一個包含 `ContinuousHandle` 與 `status()` 的 `arr.continuous("output/")`， `wait()` 以及 `stop()` —— 當您需要以高頻率儲存每個觸發訊號時，請以此取代擷取迴圈。

有關擷取模式（單次／連續／間隔／最快）、記錄器、突發轉視訊以及陣列對齊，請參閱 [SDK 參考手冊](reference/sdk-reference.md#synchronized-array--arraysession-smart-prep)。

### DAQ 光感測器 — `connect_daq_sensor`

若未傳入參數，`connect_daq_sensor()` 會自動偵測傳輸協定（優先順序：乙太網路 → BLE → USB）：

```python
with chloros_sdk.connect_daq_sensor() as daq:    # smart-detect USB / BLE / ETH
    for frame in daq.latest(n=5):
        print(frame["spectrum"][:10])
```

每個幀包含 135 點的 `spectrum`（校準後單位為 W/m²/nm）、一個 `is_saturated` 標誌，以及 CIE `x`、 `y`、`z`。若要指定特定感測器或傳輸協定，請傳入一個明確的提示：

```python
daq = chloros_sdk.connect_daq_sensor(transport="usb", port="COM3")
daq = chloros_sdk.connect_daq_sensor(mac="AA:BB:CC:DD:EE:FF")        # implies BLE
daq = chloros_sdk.connect_daq_sensor(eth_host="daq-e-xxx.local")     # implies Ethernet
```

請注意，電容校正配置檔（`cap_id`）**並非** SDK 的可調整參數 —— 應改透過 `chloros-cli daq pool-connect --cap-id …` / `pool-set-cap` 進行選取。

### 儲存的專案 — `open_project`

已儲存的 Chloros 專案會保留其已連接的硬體（`cameras.json` + `sensors.json` 以及 `project.json`）， 而 `chloros_sdk.open_project(path)` 可一次重新連線其攝影機、陣列及光感測器，並透過裝置名稱觸發擷取。 光感測器會透過正在運行的 Chloros 後端重新連線，並保留其儲存的設定與捕捉能力。請參閱參考文件中的 [專案自動化](reference/sdk-reference.md#project-automation--chlorosproject)。

## 僅安裝 pip 版本可獲得的功能

在使用硬體介面之前，請先檢查模組層級的可用性標誌：

```python
import chloros_sdk
print(chloros_sdk.__version__)
print("CAMERA_AVAILABLE =", chloros_sdk.CAMERA_AVAILABLE)    # True iff lattice_sdk imported cleanly
print("PROJECT_AVAILABLE =", chloros_sdk.PROJECT_AVAILABLE)  # True iff ChlorosProject deps available
```

在 **僅** 安裝 `pip install chloros-sdk` 且未安裝 Chloros 桌面套件的主機上：

* `ChlorosLocal`、`process_folder` 以及 `process_lattice_capture` **無法**運作 — 它們需要桌面安裝程式內附的後端二進位檔。
* 智慧連線輔助程式（`connect_camera`、`connect_array`、`connect_daq_sensor`）是純粹的 HTTP 客戶端，因此可與位於另一台機器上的後端配合運作 ——但隨附的後端僅綁定迴路連接，因此您必須自行進行端口轉發（例如 `ssh -N -L 5000:127.0.0.1:5000 user@chloros-host`），並將 `backend_url="http://127.0.0.1:5000"` 與 `auto_start_backend=False` 一併傳遞。 請參閱 [遠端後端模式](reference/sdk-reference.md#remote-backend-mode-pip-only-host-via-tunnel)。
* 直接硬體 LATTICE 類別（`LatticeCamera`、 `CameraPool`, …) 雖可導入，但需使用桌面套件中的 Arena SDK 執行時環境 — 若缺少此環境，`CAMERA_AVAILABLE` 將呈現為 `False`。
* DAQ 感測器僅能透過（隧道傳輸的）後端驅動 — `connect_daq_sensor()`，或專案的 `proj.sensors`；該後端會在自身機器上開啟感測器。

## AI 助理 — `chloros-mcp`

此 SDK 包含 `chloros-mcp`，這是一個 MCP 伺服器，可讓 Claude Desktop、Claude Code、Cursor 及其他 MCP 客戶端驅動 Chloros：處理影像，並運作 LATTICE 相機、陣列及 DAQ 光感測器。 其工具涵蓋了此 SDK 的功能。它會隨 SDK 一併安裝，因此對於 Claude Code 而言，設定只需一行指令：

```bash
claude mcp add chloros -- chloros-mcp
```

其他客戶端：[如何在 AI 助理中使用 Chloros](ai-assistants.md)。 選項與所有工具：[MCP 伺服器 (AI 助理)](reference/sdk-reference.md#mcp-server-ai-assistants)。

## 授權要求

存取 SDK 需具備任何付費方案（**Copper 或更高階**，包含 Copper / Bronze / Silver / Gold）的有效 Chloros+ 登入帳戶； 免費的 Iron 方案不提供 SDK/CLI 存取權限。此限制為**伺服器端** 強制執行：每個 SDK 請求必須同時包含有效會話及付費方案，否則後端將返回 `403` / `PLAN_UPGRADE_REQUIRED` （由 `ChlorosLocal` 觸發的錯誤代碼為 `ChlorosLicenseError`，由 `connect_*` 輔助函式觸發的錯誤代碼則為 `ChlorosConnectError`）。 已登出狀態的呼叫者會收到 `401` / `AUTH_REQUIRED` （`ChlorosAuthenticationError`）——重新執行 `chloros-cli login` 可解決第一種情況，但無法解決第二種情況。

離線使用在方案的寬限期內有效：服務等級會從伺服器驗證快取（5 分鐘）或已簽名且與機器綁定的授權快取（月費方案為 30 天；年費方案則至訂閱到期為止）中讀取。 當寬限期結束時，方案將轉為免費方案，且 SDK 存取權將中斷，直到該裝置至少連線至伺服器一次為止。`chloros-cli status` 在免費方案下仍可存取，因此原因始終可見。 請參閱 [Chloros+ 登入](chloros+-login.md)。

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

所有管線例外 (`ChlorosBackendError`、`ChlorosConnectionError`、`ChlorosLicenseError`、`ChlorosAuthenticationError`、 `ChlorosConfigurationError`、`ChlorosProcessingError`）皆源自 `ChlorosError`。 一個需注意的細節：`ChlorosConnectError` — 僅由 `connect_camera` / `connect_array` / `connect_daq_sensor` 觸發 —— 源自普通的 `Exception`，**並非**源自 `ChlorosError`，因此 `except ChlorosError` 無法擷取此錯誤。 完整的層級結構請參閱 [SDK 參考手冊](reference/sdk-reference.md#exceptions)。

## 參見

* [SDK 參考手冊](reference/sdk-reference.md) — 針對 AI 助理優化的完整 API 介面。
* [CLI 參考](reference/cli-reference.md) — 每個 CLI 子指令皆對應一個 SDK 呼叫。
* [在 AI 助理中使用 Chloros](ai-assistants.md) — `chloros-mcp` 的設定與提示範本。
* [下載](download.md) — 適用於 Windows 和 Linux 的安裝程式。
