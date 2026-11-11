# CLI：命令列

> **完整參考手冊：**[CLI 參考手冊](reference/cli-reference.md) 詳列了**每個子指令的所有參數**，並針對 AI 助理進行了優化 — 將其 URL 貼入您的助理中，並詢問可執行的指令：`https://mapir.gitbook.io/chloros/reference/cli-reference`
>
> **AI 工具小技巧：** 只需在手冊任一頁面的 URL 後方附加 `.md`（例如 `https://mapir.gitbook.io/chloros/reference/cli-reference.md`）， 而 `https://mapir.gitbook.io/chloros/llms.txt` 則會為大型語言模型 (LLM) 提供整本手冊的索引。

> **想讓您的助理直接執行 Chloros 嗎？** `chloros-mcp` 是一個隨 Python SDK 安裝的 MCP 伺服器，可與 Claude Desktop、Claude Code、Cursor 及其他 MCP 客戶端配合使用：請參閱 [如何在 AI 助理中使用 Chloros](ai-assistants.md)。

<figure><img src=".gitbook/assets/cli.JPG" alt=""><figcaption></figcaption></figure>
<!-- SCREENSHOT-UPDATE: banner shows CLI 1.1.0; reshoot the CLI welcome/banner output on the 1.3.0 build so the version line reads "Chloros CLI 1.3.0" -->


## 什麼是 CLI


`chloros-cli` 是 Chloros 桌面應用程式所使用之相同處理引擎的命令列前端。 它是一個透過 Chloros 後端（位於 `127.0.0.1:5000` 上的本地伺服器）運作的輕量級 HTTP 客戶端——大多數指令會自動啟動後端，因此腳本只需單一 `chloros-cli process …` 呼叫即可。 後端隨後會持續運行以等待下一條指令： 它會在執行 `chloros-cli logout` 時停止（除非應用程式已開啟，或正在執行或建置中），或在執行 `chloros-cli backend stop` 時停止，亦或在 30 分鐘內無客戶端、執行程序或已連線裝置時自動停止。 `chloros-cli backend status` 會顯示此狀態。

它可在 **Windows 10/11 (x64)**及**Linux (x86_64，以及 JetPack 6 上的 NVIDIA Jetson arm64)** 環境中運行，適用於任何終端機，無需圖形介面。 請使用以下指令驗證安裝：

```bash
chloros-cli --version    # prints "Chloros CLI 1.3.0"
```

指令集一覽：

* **處理與帳戶** — `process`、`login`、`logout`、`status`、 `export-status`、`language`（38 種語言 — 請參閱 [支援的語言](supported-languages.md)）， `set-project-folder` / `get-project-folder` / `reset-project-folder`, `selftest`, `update` （僅限 Linux／Jetson）
* **實機硬體** — `lattice`（LATTICE 攝影機控制，45 項以上子指令）、`daq pool-*`（DAQ 光感測器）、`time-sync` (PTP)
* **自動化** — `project`（無頭模式執行已儲存的 Chloros 專案，包含 YAML 擷取配方）

值得了解的全域選項：`--port N`（後端連接埠，預設為 `5000`）、`-v/--verbose`、 `--restart`（強制重新啟動後端）、`--backend-exe PATH`。完整清單請參閱 [CLI 參考手冊](reference/cli-reference.md)。

***

## 安裝

CLI **已內建於各平台的 Chloros 安裝程式中** — 無需另行下載 CLI。請從 [下載](download.md) 頁面取得安裝程式。

### Windows


安裝程式會將 CLI 放置於：

```

C:\Program Files\Chloros\cli\chloros-cli.exe
```

並將該資料夾新增至您的系統 `PATH` 一次（升級時也會移除先前安裝程式留下的重複項目）——安裝完成後請 **開啟新的終端機**，以便系統能偵測到更新後的 `PATH`。 安裝程式還會將啟動腳本（`Chloros_CLI.bat` / `Chloros_CLI.ps1`）放置於安裝根目錄中，並新增一個**Chloros CLI** 開始功能表捷徑，每個項目都會開啟一個已預先載入 `chloros-cli` 並可立即使用的終端機。

### Linux


請使用 `apt` 安裝適用於您系統架構的 `.deb`（請勿使用 `dpkg -i`，該版本會跳過 Python SDK 的建議需求）：

```bash
# Linux x86_64
sudo apt install ./chloros_1.3.0_amd64.deb

# NVIDIA Jetson (arm64, JetPack 6)
sudo apt install ./chloros_1.3.0_arm64_jp6.deb
```

此安裝程式會將 `chloros-cli` 安裝至 `/usr/bin/chloros-cli` （已包含於 `PATH` 中），後端及 CLI 程式則解壓縮至 `/usr/lib/chloros/backend/` 與 `/usr/lib/chloros/cli/` （`/usr/lib/chloros/chloros-backend` 是一個用於啟動後端的啟動程式），後端運作 LATTICE 攝影機所需的 Arena SDK 執行環境，以及系統專用的 Python SDK（`python3`）。 詳情請參閱 [Linux 安裝指南](linux/linux-installation.md)。

### 驗證

```bash
chloros-cli --version    # "Chloros CLI 1.3.0"
chloros-cli selftest     # 7-step diagnostic: backend, API, GPU/CUDA, denoiser models
chloros-cli status       # license tier + logged-in user
```

***

## 登入與授權

CLI（以及 Python SDK）的存取需具備 **付費的 Chloros+ 方案**—— 任何付費方案皆包含此權限；免費方案則不包含。此限制由後端在**伺服器端** 強制執行，而非由 CLI 二進位檔執行：未登入的呼叫將因 `401 AUTH_REQUIRED` 而遭拒絕， 而在免費方案下，已登入的呼叫則會返回 `403 PLAN_UPGRADE_REQUIRED`，無論呼叫來源是 `chloros-cli`、SDK 還是手動編寫的 HTTP 客戶端。 請至 [https://cloud.mapir.camera/pricing](https://cloud.mapir.camera/pricing) 進行升級。

**每台機器僅需登入一次**：

```bash
chloros-cli login user@example.com 'YourPassword'
chloros-cli status
```

<figure><img src=".gitbook/assets/cli login_w.JPG" alt=""><figcaption></figcaption></figure>
<!-- SCREENSHOT-UPDATE: login success output predates 1.2.0; reshoot `chloros-cli login` followed by `chloros-cli status` on the 1.3.0 build showing the license tier line -->


{% hint style="warning" %}
**含特殊字元的密碼**（`$`, `!`, spaces): wrap the password in**single quotes**, as shown above. In PowerShell double quotes, `$$` 會被殼層篡改（CLI 會在遇到 401 錯誤時偵測到此問題並自動重試，但使用單引號可完全避免此問題）。
{% endhint %}

該會話已快取於 `~/.chloros/user_session.json`，並會在方案的寬限期內持續離線運作（月費方案為 30 天，年費方案則直至到期為止）。 即使未訂閱付費方案，`chloros-cli status` 仍可運作，因此拒絕的原因始終可見。

{% hint style="danger" %}
**要排程無介面工作嗎？請先登入。**若在**無快取會話**的情況下執行後端啟動指令（`process`、`status`、 `export-status`, …）若在**無快取會話**的情況下執行，不會立即失敗 — 而是會切換至互動式 `Email:` / `Password:` 提示符，並從標準輸入（stdin）接收指令。 因此，無人值守的 cron 工作或 CI 步驟將會**因等待輸入而卡住**。在排程任何任務之前，請先在該機器上執行一次 `chloros-cli login EMAIL 'PASSWORD'`。
{% endhint %}

***

## 首次處理執行

將 `process` 指向擷取檔案的資料夾 — 它會自動偵測 Survey3（`.raw` + `.jpg`）、 LATTICE（`.tif`/`.tiff`）、`.dng`，或上述格式的混合：

```bash
chloros-cli process "C:\Images\flight_001"          # Windows
chloros-cli process ~/images/flight_001              # Linux
```

進度串流會根據各管線執行緒（偵測、分析、處理、匯出）即時更新，而成功執行的任務會透過回報寫入的影像產出數量來結束： `Image products written: N`，若執行過程同時寫入 [鉗位遮罩](output-image-formats.md#clamp-masks)，則隨後會出現 `(plus M clamp masks)`（鉗位遮罩不計入產品數量）。

<!-- SCREENSHOT-NEEDED: terminal capture of a `chloros-cli process` run on a LATTICE captures folder completing successfully — per-thread progress lines visible and the final "Image products written: N (plus M clamp masks)" summary line -->


若已有Chloros 後端正在運行（例如桌面應用程式的後端），`process` 會使用該後端並在其內開啟該專案。當應用程式正開啟某個專案時，系統會拒絕此操作：請從應用程式中處理，或關閉該專案。 若前一次執行仍在進行中，系統會等待最多 15 秒，之後才會拒絕執行。`--restart` 會強制啟動一個新的後端。 在 Linux 上，由 systemd 服務執行的後端絕不會被重新啟動：`process` 會直接使用現有後端，而在原本應重新啟動後端的情況下（例如 `--restart`，或進口操作未找到任何項目時）， 它會輸出 `sudo systemctl restart <unit>` 並退出 `1`。

### 輸出檔案的存放位置

`process` 會寫入 **專案資料夾**，而非您的輸入資料夾：

* 若未指定 `-o`：專案將建立於您的預設專案資料夾下（與 GUI 共用；請透過 `get-project-folder` / `set-project-folder` 進行管理； 若未設定，則位於您的使用者資料夾中的 `Documents/MAPIR/Chloros Projects`），若未指定名稱，則以 `-n/--project-name` 或時間戳記（`YYYYMMDD_HHMMSS`）命名。
* 若使用 `-o PATH`：該資料夾 **即** 為專案資料夾。若該資料夾已包含 `project.json`，系統將建立以 `_1`/`_2`… 為後綴的同級檔案，而非覆寫原有檔案。若指定相對路徑 `PATH`，則會以您執行 `chloros-cli` 的資料夾為基準。

{% hint style="info" %}
**相對路徑（於 1.3.0 版本中變更）。** 後端開啟的每個路徑——無論是任何指令（包括擷取指令的預設 `output`）所對應的 `-o`/`--output`， `--input`、`--burst-dir`、`--profile`、`--save` 以及 `--mask` — 會根據您執行 `chloros-cli` 的資料夾進行解析，因此 `-o captures/` 會寫入 `<current folder>/captures/`。 在 1.2.0 版本中，後端會將這些路徑解析至其自身的安裝資料夾，但會因「存取被拒絕」而失敗。針對遠端後端（`CHLOROS_BACKEND_URL`），路徑會以輸入的原始形式傳送，並在該機器上進行解析。
{% endhint %}

{% hint style="warning" %}
**Windows：除非已啟用長路徑，否則請保持專案資料夾名稱簡短。** 當 Windows 關閉長路徑功能（此為 Windows 預設設定）時，任何檔案路徑長度不得超過 260 個字元，且產品路徑位於專案資料夾下方最多約 130 個字元的深度內。 `process` 會在啟動前檢查此項：若專案資料夾層級過深，系統會顯示 **「此專案資料夾對這台電腦而言層級過深」** 並拒絕處理，同時會提供最長可容納的專案資料夾路徑以及如何啟用長路徑的說明——此時不會進行任何處理。 請改用較短的 `-o`（或 `set-project-folder`），或啟用 Windows 長路徑（安裝程式的 **啟用 Windows 長路徑** 選項，預設為勾選狀態），並重新啟動 Chloros。
{% endhint %}

在專案內部，產品會 **依相機分類，再依檔案格式分類**：

```
<project>/
├── project.json
├── calibration_data.json
└── LATT-M3M-L41-F550/                  # one folder per camera model+lens+filter
    ├── tiff16/
    │   ├── Reflectance_Calibrated_Images/
    │   ├── Debayered_Images/
    │   ├── Preview_Images/
    │   └── NDVI_Index_Images/           # one folder per requested index
    └── tiff32/
        └── Radiance_Images/             # float32 radiance always lands here
```

LATTICE 的相機資料夾為 `LATT-<sensor>-<lens>-F<filter>`（與擷取檔的 EXIF 資訊 `Model` 相符），而 Survey3 的相機資料夾則為 `<model>_<filter>` （例如 `Survey3N_RGN`）則對應於 Survey3。格式資料夾的命名規則依循 `--format`：`tiff16`、`tiff8`、 `png8`、`jpg8` 或 `tiff32` 對應於 `TIFF (32-bit, Percent)`。

對於多光譜 M3C 相機，格式資料夾中除產品資料夾外，也可能出現一個 `clamp_masks/` 資料夾（例如 `tiff16/clamp_masks/Reflectance_Calibrated_Images/` 和 `tiff32/clamp_masks/Radiance_Images/`）。 該資料夾存放輻照度與反射率產品的每像素遮罩，其中波段解混將負值限制為零 — 請參閱 [限制遮罩](output-image-formats.md#clamp-masks)。任何產品資料夾都不會包含遮罩。

{% hint style="info" %}
**每個匯出的產品均保留原始檔案的名稱。**例如，`capture_..._raw.tif` 的輻照度匯出檔仍命名為 `capture_..._raw.tif` —— 它僅是存放於 `tiff32/Radiance_Images/` 目錄中。**資料夾用來識別產品，而非檔案名稱**，因此請使用通配符搜尋目錄，而非搜尋 `*radiance*` 這樣的副檔名。
{% endhint %}

### 您實際會使用的選項

| 標誌 | 預設值 | 功能說明 |
| --- | --- | --- |
| `-o, --output PATH` | 預設專案資料夾 | 專案資料夾位置（參見上文）。 |
| `-n, --project-name NAME` | 時間戳記 | 專案名稱。 |
| `--format FMT` | `TIFF (16-bit)` | 選自 `TIFF (16-bit)`、 `TIFF (8-bit)`、`TIFF (32-bit, Percent)`、`PNG (8-bit)`、`JPG (8-bit)` 之一。 Chloros 不會進行升頻：在所有 8 位元影像上，較深的格式會寫入為 `TIFF (8-bit)`，且僅會標示一次。 |
| `--indices NAME [NAME ...]` | 無 | 待匯出的植被指數（參見 [植被指數](#vegetation-indices)）。 |
| `--debayer {standard,texture-aware}` | `standard` | `texture-aware` = 神經網路去拜爾演算法，速度較慢， 最高品質（Chloros+，需 CUDA GPU：若無 CUDA GPU，則執行時使用 `standard`；LATTICE 擷取影像不適用此選項，該類影像始終採用 `standard`）。 |
| `--vignette / --no-vignette` | 啟用 | 暗角校正。 |
| `--reflectance / --no-reflectance` | 啟用 | 反射率校正；對於 LATTICE 而言，此選項同時也是反射率乘積功能的開關。 |
| `--input-level {auto,raw,debayered,processed}` | `auto` | 強制 LATTICE TIFF 檔案使用管線入口點。 |

至於其他所有項目——目標偵測調整、PPK、曝光定位點、陣列對齊標誌——請參閱 [CLI 參考手冊中的 `process` 章節](reference/cli-reference.md)。

***

## 選擇要匯出的內容（LATTICE 產品）

LATTICE 處理會在 **單次執行中**擴展至**所有適用產品**。每個產品的四個開關皆**預設為開啟**；請使用 `--no-` 格式來停用其中一個：

| 開關 | 產品 |
| --- | --- |
| `--debayered` | 線性去馬賽克 → `Debayered_Images/`： 移除校準套件的暗偏移，接著進行邊緣感知去馬賽克，不進行線性校正、平場校正或解混 — 此檔案與擷取功能在 `debayered` 層級所儲存的檔案相同。維持感光元件的原生方向（記錄於 XMP 中）。 |
| `--preview` | 顯示預覽（白平衡 + 伽瑪校正；多光譜影像的假色拉伸）→ `Preview_Images/` |
| `--radiance` | float32 輻照度，W/m²/sr/nm → `Radiance_Images/`（始終為 `tiff32/`） |
| `--reflectance` | uint16 反射率，Pix4D 相容 → `Reflectance_Calibrated_Images/` |

僅限 RGB 主相機會輸出去拜耳化 + 預覽資料 — 對於寬頻感測器而言，各波段的輻度/反射率並無實質意義，因此這些切換選項對其而言無效。Survey3 的 `.raw` 會忽略這些切換選項，並遵循標準的反射率/目標路徑。

```bash
# Radiance only — no DAQ downwelling needed
chloros-cli process ~/captures/lattice_flight --no-debayered --no-preview --no-reflectance
```

**`--reflectance-source {auto,target,daq}`**（預設為 `auto`）選擇反射率參考： `auto` 將通過品質保證（QA）的幀內 [校準目標](calibration-targets.md) 設為絕對參考，並在無目標存在時回退至 DAQ 光感測器的下行分界值 (ρ = π·L/E)；當無標靶存在時（若標靶通過品質保證且設有標記，則其測量分數將與 DAQ 結果相互競爭）；`target` 採用嚴格模式（不進行 DAQ 替代）； `daq` 採 DAQ 權威模式。可透過 `--target-reflectance-dir` 提供每單位的測量標靶掃描資料。

{% hint style="info" %}
**讀取反射率像素：**表示 ρ = 1.0 的 DN 為**單一光源** — LATTICE 檔案會在 XMP 中標記 `Chloros:PixelScale=32768`； Survey3 檔案使用65535，且不包含`Chloros:PixelScale`標籤（但會包含其他`Chloros:*`處理標籤）。 請讀取該標籤並以此為基準進行除法運算，而非假設為恆定值。詳細資訊及一個刻意設計的無比例邊緣案例，請參閱 [CLI 參考手冊](reference/cli-reference.md)。
{% endhint %}

**處理流程始終從 `raw` 開始。** 衍生產品（去拜耳化／輻射度／反射率匯出檔）絕不會被送回處理管道——若重新匯入並處理，將導致校準運算重複應用兩次，因此 Chloros 會跳過這些檔案並明確標示。 `--input-level` 是當您確實需要強制指定起始點時，刻意預留的「逃生通道」。

***

## 當執行失敗時

自 1.2.0 版本起，`process` 會明確顯示失敗訊息，而非在沒有任何輸出結果的情況下「成功」：

* 若執行程序 **請求了產出但未寫入任何內容**— 僅包含 `project.json` 和 `calibration_data.json` — 則會輸出 `Processing finished but wrote no image products.` 並**以非零狀態退出**， 因此腳本能夠偵測到此情況。常見原因包括：輸入資料夾未被識別為擷取資料（請檢查佈局及 `--input-level`），或是所有請求的產出皆不適用於該批相機（例如：從僅支援 RGB 的相機請求輻射度／反射率）。
* **刻意僅執行元資料處理**（所有產品選項皆關閉，無 `--indices`）仍視為成功 —— 此時輸出空影像即為正確結果。
* 請重新執行並啟用 `--verbose`，並檢查後端日誌中的 `[LATTICE-EXPORT]` / `[EXPORT-CHECK]` 記錄行，這些記錄會說明各相機被跳過的具體原因。
* 在Windows 系統中，若已關閉長路徑功能，當專案資料夾路徑過深而超出 260 個字元的限制時，系統會在處理開始 **之前** 拒絕該專案，並使用符合限制的最長專案資料夾路徑（詳見 [輸出檔案存放位置](#where-the-outputs-land)）。
* 擷取指令（`lattice capture`、`lattice array-capture` 及其他擷取指令系列）會列出所有無法寫入的檔案名稱，並以 `1` 狀態碼結束執行；當磁碟空間已滿時，重複擷取作業將停止。

退出代碼：`0` 成功 · `1` 一般性失敗 · `2` 參數錯誤 · `130` 遭 Ctrl+C 中斷。

***

## 植被指數

執行 `--indices` 時，請提供一個或多個預設名稱；每個指數都會存放在專屬的 `<INDEX>_Index_Images/` 資料夾中：

```bash
chloros-cli process ~/images/flight_001 --indices NDVI GNDVI OSAVI
```

`process --indices` 接受的 22 個預設名稱：

`NDVI` `GNDVI` `NDRE` `OSAVI` `SAVI` `MSAVI2` `EVI` `MSR` `TDVI` `LAI` `GCI` `GRVI` `GSAVI` `GOSAVI` `NLI` `MNLI` `RDVI` `WDRVI` `CVI` `ENDVI` `GLI` `VARI`

若專案中尚未列出的名稱被新增至專案中，系統會將其加入。若該名稱並非預設名稱，或攝影機的頻段無法生成該名稱，則會跳過該名稱並發出警告，同時標示該名稱。 若CLI 無法讀取專案的當前索引清單，則會保留清單原樣並發出警告，而非替換該清單。

{% hint style="warning" %}
**共有三份索引清單——請勿混淆。**GUI 的「專案設定」下拉選單中有 27 個公式（新增 `FCI1`、`FCI2`、`GARI`、 `GEMI`、`LCI` — 這五項僅限於 GUI 使用，且**不適用於** `--indices`）。 「即時／離線」`lattice index --preset` 指令使用其專屬的 22 項預設清單。相關公式與波段運算詳見 [多光譜指數公式](project-settings/multispectral-index-formulas.md)。
{% endhint %}

***

## DAQ 光感測器：快速入門

`daq pool-*` 系列透過後端持久化資源池驅動 MAPIR DAQ 光譜感測器（透過 USB 連接的 DAQ-U、透過 BLE 連接的 DAQ-M，以及透過乙太網路連接的 DAQ-E）——GUI、CLI 和 SDK 皆共用一個即時處理句柄。 **`pool-*` 是出廠版 CLI 中受支援的 DAQ 路徑**；您可能會看到其他提及的 `daq` 子命令，這些屬於 MAPIR 內部僅供來源使用的介面，執行時會以明確的錯誤訊息引導您至 `pool-*`。

```bash
# 1. Open a pooled session (pick the line matching your sensor)
chloros-cli daq pool-connect                              # smart-detect
chloros-cli daq pool-connect --port COM3                  # DAQ-U on a specific COM port
chloros-cli daq pool-connect --mac AA:BB:CC:DD:EE:FF      # DAQ-M by BLE MAC
chloros-cli daq pool-connect --eth-host daq-e-xxx.local   # DAQ-E by hostname (reliable)

# 2. List pooled sensors and their ids
#    (DAQ-U ids look like 'CB-7C-A8-2E-5F'; DAQ-E ids like 'daq-e-def330')
chloros-cli daq pool-list

# 3. Read the latest calibrated spectrum (W/m²/nm)
chloros-cli daq pool-latest --sensor-id CB-7C-A8-2E-5F

# 4. Record a calibrated .daq file for 60 s
chloros-cli daq pool-record --sensor-id CB-7C-A8-2E-5F --duration 60 \
  -o ~/Documents/spectra --device-name "field-A"

# 5. Release
chloros-cli daq pool-disconnect --sensor-id CB-7C-A8-2E-5F
```

若未指定 `--duration`，則 `pool-record` 會執行至 `pool-record --stop`； 預設輸出目錄為 `~/Documents/DAQ Live View/` **位於後端主機上**。 電容校正曲線在連線時選取（`--cap-id`，後端預設為 `sunshine_cosine`），並可透過 `pool-set-cap` — 電容配置檔及感測器的校準範圍，詳見本手冊的 DAQ 章節。

{% hint style="warning" %}
**多網卡主機上的 DAQ-E：** 即使感測器狀態正常，開機後首次執行 `pool-connect --eth` 自動偵測仍可能失敗。`--eth-host <ip-or-hostname>` 才是可靠的方案 —— 每當偵測結果為空時，請改用此選項。
{% endhint %}

***

## LATTICE 攝影機、PTP 與專案自動化

`lattice` 系列（含 45 多個子指令）涵蓋 LATTICE 攝影機的端到端操作：偵測、單次擷取、透過 GUI「智慧預備連線」流程建立的持久同步陣列、瀏覽器即時預覽、對齊、索引運算，以及主機網卡診斷。 舉例說明：

```bash
chloros-cli lattice info                                          # discover cameras
chloros-cli lattice capture -o output/                            # one frame, all export types
chloros-cli lattice array-connect --serials SN1,SN2,SN3,SN4       # persistent synced array
chloros-cli lattice array-capture --processing reflectance -o out/
```

此外：`chloros-cli time-sync` 會回報由 Chloros 主機執行的 PTP 主時鐘狀態（LATTICE 攝影機與 DAQ-E 感測器皆以此為主時鐘，以實現跨裝置時間戳記同步）， 而 `chloros-cli project` 則開啟一個儲存的 Chloros 專案，並以無視窗模式驅動其相機、陣列及感測器——包含透過腳本執行的 YAML 擷取配方。

這三個系列（`lattice`、 `project`、`daq pool-*`）也是唯一支援 `CHLOROS_BACKEND_URL` 來驅動 **遠端** 後端的系列；核心指令始終以本地主機為目標。

完整的操作指南收錄於本手冊的 LATTICE 章節中；所有參數皆列於 [CLI 參考手冊](reference/cli-reference.md)。

***

## 疑難排解：前五大常見問題

| 症狀 | 解決方法 |
| --- | --- |
| `Login required` 或排程工作在 `Email:` 提示符處卡住 | 在此機器上執行一次 `chloros-cli login EMAIL 'PASSWORD'` — 沒有快取會話的指令將以互動模式執行，而非立即失敗。 |
| `backend unreachable` | 啟動 Chloros 桌面應用程式，或手動啟動後端：Windows 上的 `"C:\Program Files\Chloros\backend\chloros-backend.exe"`（位於安裝資料夾內的 `backend` 資料夾； 該資料夾不位於 `PATH` 內），Linux 上的 `chloros-backend`。`chloros-cli backend status` 會顯示是否正在執行； `lattice`/`project`/`daq pool-*` 絕不會啟動後端。若您將其指向遠端後端，請檢查 `CHLOROS_BACKEND_URL`。 |
| 陣列連線受阻：`FRAMES WILL DROP` / `Reduce ROI to enable` | 主機網路介面卡 (NIC) 接收環路重設為預設值 — 這是先前運作正常的主機突然無法連線的首要原因，通常發生在更新 NIC 驅動程式之後。 請從 **具有管理權限的** 終端機執行 `chloros-cli lattice network --fix`（或設定 `ReceiveBufferLen=256`、`PendingReceives=64`）；請參閱參考手冊中的 *主機網路介面卡設定與調校*。 |
| `daq` 子指令退出："需要完整的 DAQ 套件…" | 請使用 `daq pool-*` 指令，這些指令會透過後端驅動同一感測器：連線、串流、記錄、封包擷取選項，以及針對 DAQ-E 的網路設定、重新啟動與韌體更新（或透過 Python 執行 `chloros_sdk.connect_daq_sensor()`）。 |
| Jetson 在處理大型資料夾前會顯示交換空間警告 | 新增基於檔案的交換空間 — CLI 會顯示需執行的確切 `fallocate`/`swapon` 指令。 |

***

## 取得協助

```bash
chloros-cli --help              # top-level help
chloros-cli process --help      # per-command help
chloros-cli lattice --help
chloros-cli daq --help          # lists the pool-* subcommands
```

* **所有參數、所有子指令：** [CLI 參考手冊](reference/cli-reference.md)
* **Python 對應指令：** [Python SDK](api-python-sdk.md) 及 [SDK Reference](reference/sdk-reference.md)
* **支援：** info@mapir.camera · [https://www.mapir.camera/community/contact](https://www.mapir.camera/community/contact)
