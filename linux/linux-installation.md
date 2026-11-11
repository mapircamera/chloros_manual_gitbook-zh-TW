# Linux 安裝

Chloros 針對 Linux 提供的發行版本為 `.deb` 套件，該套件會安裝 CLI、後端伺服器以及 Python SDK（針對 `python3` 系統，另附有與版本相符的 wheel 檔案供虛擬環境使用）。

套件檔案名稱包含版本與架構：`chloros_1.3.0_amd64.deb` 適用於 x86_64， 而 `chloros_1.3.0_arm64_jp6.deb` 則適用於 arm64（NVIDIA Jetson、JetPack 6）。請在以下指令中替換為您實際下載的檔案。

{% hint style="info" %}
**

1.3.0 版新功能：程式以未解壓縮形式提供。** 後端與CLI會以資料夾形式安裝，分別為 `/usr/lib/chloros/backend/` 和 `/usr/lib/chloros/cli/`， 而非單一的自解壓縮檔案。它們不再於每次啟動時自動解壓縮至臨時資料夾，因此啟動速度大幅提升（在 x86_64 平台上，後端過去每次啟動時需耗費約 100 秒進行解壓縮） ，且不再需要在 `/tmp` 目錄中預留數 GB 的可用空間。amd64 版本的 `.deb` 檔案大小也僅為原先的一半左右。
{% endhint %}

***

## Linux amd64 (x86_64)

### 系統需求

| 需求項目 | 最低要求 | 建議要求 |
| --- | --- | --- |
| **發行版** | Ubuntu 22.04 LTS+ / Debian 12+ | Ubuntu 24.04 LTS |
| **處理器** | x86_64 (Intel/AMD) | Intel Core i7 或更高規格 |
| **記憶體 (RAM)** | 8GB | 16GB 或以上 |
| **顯示卡** | 無（由 CPU 處理） | 配備 4GB+ VRAM 的 NVIDIA GPU（12GB+ 可解鎖 `GPU_PARALLEL`，7GB+ 可避免「Texture Aware」功能在單一影像路徑中啟用） |
| **儲存空間** | 2GB 可用空間 | 具備 10GB+ 可用空間的 SSD |
| **Python** | Python 3.7+（適用於 SDK） | Python 3.10+ |

> **不支援 Ubuntu 20.04 和 Debian 11。** `.deb` 的依賴清單
> 源自 Chloros 後端實際連結的組件，其中包含一個
> `libc6` 版本，其新於 focal 和 bullseye 發行版所附帶的 glibc 2.31，因此 `apt` 會直接拒絕
> 安裝，而非讓其日後在執行時發生錯誤。

### 安裝

```bash
sudo apt install ./chloros_1.3.0_amd64.deb
```

{% hint style="info" %}
**請使用 `apt install ./…`，而非 `dpkg -i`。** `apt` 會解決該套件的依賴關係，並同時安裝其建議的項目：Python SDK 的需求， `python3-numpy` 以及 `python3-requests`。`dpkg -i` 則不會安裝這兩項。 CLI 和後端是自包含的，即使沒有這些組件也能運行，但 `import chloros_sdk` 需要它們；安裝輸出會列出任何缺失的組件。請保留 `./`， 以便 `apt` 能直接安裝該檔案，而非尋找同名的套件。此套件亦宣告了 `libgl1`，此檔案會由內建的 OpenCV 載入，而精簡版伺服器映像檔可能未包含此檔案。
{% endhint %}

驗證安裝：

```bash
chloros-cli --version    # prints "Chloros CLI 1.3.0"
```

<!-- SCREENSHOT-NEEDED: Terminal on Ubuntu 22.04 immediately after `sudo apt install ./chloros_1.3.0_amd64.deb`, showing the full postinst output: the "Chloros installed successfully!" banner, the Usage lines, the "Python SDK:" block ("chloros_sdk 1.3.0 for python3" and the "In a virtualenv: pip install /usr/lib/chloros/sdk/chloros_sdk-*.whl" line), any "GPU Acceleration:" detection line, and the closing "Systemd Service (optional): sudo systemctl enable --now chloros-backend@<user>.service" hint -->

***

## Linux arm64 (NVIDIA Jetson)

### 系統需求

| 需求 | 最低 | 建議 |
| --- | --- | --- |
| **平台** | 搭載 JetPack 6 的 NVIDIA Jetson | Jetson Orin NX 16GB 或 AGX Orin |
| **JetPack** | JetPack 6.x | 最新版 JetPack 6 |
| **記憶體 (RAM)** | 8GB（GPU/CPU 共用） | 16GB+ 共用（12GB+ 為並行處理的門檻） |
| **儲存空間** | 2GB 可用空間 | NVMe SSD 且具備 10GB+ 可用空間 |
| **Python** | Python 3.7+（適用於 SDK） | Python 3.10+ |

### 安裝

```bash
sudo apt install ./chloros_1.3.0_arm64_jp6.deb
chloros-cli --version
```

與 amd64 版的 `.deb` 佈局相同（包含 Python SDK、接收緩衝區及後端服務），適用於運行 **JetPack 6**（L4T R36）的 Jetson Orin / Orin NX / Orin Nano。 其 PyTorch 採用 JetPack 6 針對 Orin 打造的 CUDA 版本，因此 GPU 處理（包含 Texture Aware 去拜爾演算法）將在 Jetson 的 GPU 上執行。 此套件本身並不包含 CUDA：它使用 JetPack 6 所安裝的 CUDA 和 cuDNN 函式庫（若映像檔中缺少這些函式庫，則使用 `sudo apt install nvidia-jetpack`）。 若無法使用 CUDA GPU，處理將在 CPU 上執行，且 `--debayer texture-aware` 會使用標準去拜爾演算法（後端日誌中會有一行 `[DEBAYER] … needs a CUDA GPU …` 說明原因）。有關 Jetson 的記憶體、散熱及現場部署行為，請參閱 [NVIDIA Jetson 指南](nvidia-jetson-guide.md)。***

## Python SDK 安裝（所有 Linux 系統）**`.deb` 會自動為您安裝 SDK。** 兩個套件（amd64 和 Jetson）皆會為系統安裝 SDK `python3`，因此 `import chloros_sdk` 在安裝完成後即可直接運作，無需 pip 亦無需網路連線。 其套件（`chloros_sdk`、`lattice_sdk`、`lattice_cam`、 `arena_api` 以及專案載入器）皆位於 `/usr/lib/python3/dist-packages` 中，並由該套件所管理：升級時會替換這些元件，移除套件時則會一併移除。 版本號始終與您安裝的CLI及後端相符。

SDK 本身的依賴項 `python3-numpy` 和 `python3-requests` 僅為 **建議安裝，並非必需**：`sudo apt install ./chloros_<version>_<arch>.deb` 會安裝這些套件，而 `dpkg -i` 則不會。安裝輸出會報告 SDK 所導入的版本 `python3`、任何缺失的依賴項， 以及 `python3` 路徑上其他位置的任何 `chloros_sdk` 檔案（例如先前透過 `pip install --user` 安裝的版本），若其覆蓋了套件的副本，都會在安裝輸出中列出。**在 virtualenv 環境中**（該環境無法偵測到系統套件），請將隨附的 wheel 檔案安裝至其中（完全離線）：

```bash
pip install /usr/lib/chloros/sdk/chloros_sdk-*.whl
```

**從 PyPI 安裝** — 僅適用於未安裝 Chloros 套件的機器，且其 SDK 驅動著另一台機器上的後端（參見 [遠端後端模式](../reference/sdk-reference.md#remote-backend-mode-pip-only-host-via-tunnel)）：

```bash
pip install chloros-sdk
```

可選附加項目，適用於 pip 安裝（虛擬環境或 PyPI）：

| 附加項目 | 指令 | 新增內容 |
| --- | --- | --- |
| `progress` | `pip install chloros-sdk[progress]` | `sseclient-py`（用於即時進度串流） |
| `camera` | `pip install chloros-sdk[camera]` | 適用於 BLE (DAQ-M) 傳輸的 `bleak` |

驗證 SDK：

```bash
python3 -c "import chloros_sdk; print(chloros_sdk.__version__)"
```

{% hint style="info" %}
Python SDK 透過 `.deb` 安裝的本地 HTTP API（`http://127.0.0.1:5000`）與後端進行通訊。 `ChlorosLocal` 以及連線輔助程式會在需要時自動啟動它；已儲存專案中的 `connect_all()` 則永遠不會自動啟動， 因此，在無顯示器的機器上，請將後端以 [服務](#always-on-ptp-for-headless-hosts) 模式執行。請務必使用字面上的 IPv4 位址，而非 `localhost` —— `localhost` 可能會解析為 `::1`，且每個請求約需耗時兩秒。
{% endhint %}

***

## 首次設定

### 1. 登入

存取 CLI 和 SDK 需具備付費的 Chloros+ 方案（**Copper** 或更高），此限制由伺服器端強制執行：未登入的呼叫者會獲得 `401 AUTH_REQUIRED`，而免費方案（Iron）的呼叫者則會獲得 `403 PLAN_UPGRADE_REQUIRED`。

```bash
chloros-cli login your@email.com 'your-password'
```

憑證會快取於 `~/.chloros/user_session.json` 中。

{% hint style="info" %}
**升級後會保留您的登入狀態；移除則會清除該狀態。** 若在現有安裝上安裝較新版本的 `.deb`（包括透過 `chloros-cli update` 進行安裝），將保留 `~/.chloros/user_session.json` 及快取的授權，因此無介面模式的 CLI/SDK 工作將持續運作。 若移除套件（`sudo apt remove chloros`），將清除該機器上所有使用者的儲存登入資訊及授權快取，因此重新安裝時需再次安裝 `chloros-cli login`。
{% endhint %}

### 2. 檢查您的授權狀態

```bash
chloros-cli status
```

`chloros-cli status` 適用於任何方案（包括免費方案），因此您隨時都能了解為何能或無法存取服務。

### 3. 執行系統診斷

```bash
chloros-cli selftest
```

系統將依序執行七項檢查，若其中任何一項失敗，指令將以非零狀態碼結束：

| # | 檢查項目 | 驗證內容 |
| --- | --- | --- |
| 1 | **版本** | CLI 回報其版本（`v1.3.0`）。 |
| 2 | **埠可用** | 埠 5000 為空閒狀態，*或* 已由運作正常的 Chloros 後端回應（此情況亦視為通過）。 |
| 3 | **後端啟動** | 後端二進位檔成功啟動。 |
| 4 | **API 測試 (`/api/test`)** | 後端回應 `status: ok`。 |
| 5 | **系統資訊** | 從 `/api/system-info` 輸出 `GPU: <name>, CUDA: <bool>, PyTorch: <version>`。 |
| 6 | **去噪器模型** | 找到 `*.pth.enc` 模型 （在 Linux 上：`/usr/lib/chloros/models`）。 |
| 7 | **CUDA + 去噪器**| 「Texture Aware」功能實際上可使用 — 需要 CUDA**以及** 至少一個模型檔案。 |

執行程序以 `N/7 checks passed` 結束，並按名稱列出所有失敗項目。

### 4. 處理您的第一個資料集

```bash
chloros-cli process ~/datasets/flight001
```

***

## 檔案與目錄

### 每位使用者

Chloros 會將每位使用者的憑證和 CLI 設定儲存在單一跨平台目錄中，即 **`~/.chloros/`**（在 Windows 上為 `%USERPROFILE%\.chloros\`）。另外兩個 Linux 專屬快取則遵循 XDG 規範 — 若已設定，這些快取會使用 `XDG_CONFIG_HOME` / `XDG_CACHE_HOME`。

| 路徑 | 用途 |
| --- | --- |
| `~/.chloros/user_session.json` | 由 `chloros-cli login` 寫入的登入會話快取（在套件升級時保留；移除套件時清除） |
| `~/.chloros/working_directory.txt` | 預設專案資料夾覆寫設定（`chloros-cli set-project-folder` / `get-project-folder` / `reset-project-folder`） |
| `~/.chloros/cli_language.json` | 命令列介面 (CLI) 語言偏好設定 (`chloros-cli language <code>`) |
| `~/.chloros/user.json` | 與 Windows 圖形介面共用的語言設定 — 此處的 `language` 優先於 `cli_language.json` |
| `~/.chloros/update_cache.json` | Linux/Jetson 啟動時檢查更新的 1 小時快取 |
| `~/.chloros/backend.log` | 後端由 CLI 啟動時的後端日誌 |
| `~/.chloros/camera_cal/<serial>/<bundle_sha>/` | 按相機儲存的 LATTICE 校準包快取，以序號和套件雜湊值作為索引 |
| `~/.chloros/daq_cap_profiles/<u\|m\|e>/<cap_id>.json` | DAQ 容差校正設定檔的可選使用者覆寫設定 |
| `~/.config/chloros/system_config.json` | 來自「動態運算適應」（Dynamic Compute Adaptation）的快取硬體設定檔 — 刪除此檔案可強制執行新的硬體偵測 |
| `~/.cache/chloros/logs/backend_<YYYYMMDD_HHMMSS>.log` | 後端伺服器日誌，每次啟動產生一個檔案 |
| `~/Documents/MAPIR/Chloros Projects/` | 未設定覆寫時的預設專案資料夾 |

### 系統範圍

| 路徑 | 用途 |
| --- | --- |
| `/usr/bin/chloros-cli` | 封裝腳本 — 將 `LD_LIBRARY_PATH` 設定為捆綁的原生函式庫 （若該資料夾存在，則將 `TMPDIR` 設定為 `/mnt/ssd/tmp`），隨後執行實際二進位檔 |
| `/usr/bin/chloros-backend` | 封裝腳本 — 針對後端執行相同操作，並將其標記為已安裝的生產版本 |
| `/usr/lib/chloros/cli/`、`/usr/lib/chloros/backend/` | 已編譯的CLI與後端，以解壓縮形式安裝（1.3.0版新增）； 程式分別為 `cli/chloros-cli` 和 `backend/chloros-backend` |
| `/usr/lib/chloros/chloros-cli`、`/usr/lib/chloros/chloros-backend` | 兩行啟動腳本，用於啟動上述實際程式 — 保留在這些路徑中，因為 SDK、CLI 以及舊版腳本會在此處搜尋它們 |
| `/usr/lib/chloros/arena_runtime/` | LATTICE 相機所需的 Arena SDK 執行階段環境 |
| `/usr/lib/chloros/models/*.pth.enc` | 由「Texture Aware」去拜爾演算法所使用的加密去噪模型 |
| `/usr/lib/python3/dist-packages/chloros_sdk/`（以及 `lattice_sdk`、`lattice_cam`、`arena_api`，即專案載入器） | Python SDK，已為系統 `python3` 安裝且由該套件所擁有 |
| `/usr/lib/chloros/sdk/chloros_sdk-*.whl` | 與此精確建置版本完全匹配的 Python SDK wheel，供 virtualenv 使用 |
| `/usr/lib/chloros/exiftool` | 內建的 ExifTool 啟動程式（僅在系統中不存在 exiftool 時，才會建立符號連結至 `/usr/local/bin/exiftool`）。自 1.3.0 版本起，ExifTool 腳本隨附其專屬的 Perl 函式庫（位於 `/usr/lib/chloros/exiftool-dist/`），因此無需系統 `libimage-exiftool-perl` 套件即可執行 |
| `/etc/chloros/update.conf` | 由 `chloros-cli update` 讀取的更新通道設定 |
| `/etc/sysctl.d/60-chloros-ptp.conf` | 設定 `net.ipv4.ip_unprivileged_port_start = 319`，使後端能在無需 root 權限的情況下綁定 PTP 埠 |
| `/etc/sysctl.d/60-chloros-gige.conf` | 將核心的套接字接收緩衝區 (`net.core.rmem_max`、`net.core.rmem_default`) 至 32 MB，並於安裝時套用這些設定 （若已設定較高的值則維持不變）。在內核預設情況下，攝影機封包會在內核中被丟棄，導致陣列遺失幀數並降低幀率；當緩衝區餘量不足時，`chloros-cli lattice network` 會回報緩衝區狀態。 請參閱 [Linux 主機 — 套接字接收緩衝區](../reference/cli-reference.md#linux-hosts--socket-receive-buffers) |
| `/etc/ld.so.conf.d/Arena_SDK.conf` | 將動態載入器指向 `/usr/lib/chloros/arena_runtime` |
| `/lib/udev/rules.d/70-chloros-daq.rules` | 授予已登入使用者存取 DAQ-U USB 序列橋接器的權限 (CP2102N, `10c4:ea60`) |
| `/lib/systemd/system/chloros-backend@.service` | 選擇性啟用的「始終運行」後端服務，該服務會以您指定的使用者身分執行後端（已安裝，**尚未啟用**）——需啟用此服務，請參閱 [始終運行 PTP](#always-on-ptp-for-headless-hosts) |
| `/lib/systemd/system/chloros-backend.service` | 相同服務，但以 **root**身分執行後端（已安裝，**未啟用**），保留此設定可確保啟用該服務的機器持續運行 |
| `/var/log/chloros/` | root 服務的詳細日誌，可由 `adm` 群組讀取 （當套件被徹底移除時將被刪除） |
| `/usr/share/applications/chloros-cli.desktop` | 「Chloros CLI」應用程式選單項目，用於開啟終端機 |

**防火牆（1.3.0 版新增功能）。** 若在安裝時 `ufw` 或 `firewalld` 處於活躍狀態，此套件將開啟 UDP 埠 `5002` （DAQ-E 資料）、`5353`（mDNS 發現）、`319` 及 `320`（PTP），並在移除套件時刪除這些規則。 若已安裝防火牆但未啟用，則不會新增任何設定，並會列印出日後啟用時應允許的埠號。它不會開啟任何 TCP 埠： 當感測器透過 TCP 連回這台電腦以擷取影像時，若防火牆處於啟用狀態並阻擋該連線，DAQ-E 韌體更新將因 `… device never connected back on TCP <port> …` 錯誤而失敗。

## 後端可執行檔位置

CLI 和 SDK 會自動偵測後端：

| 元件 | 路徑 |
| --- | --- |
| CLI | `/usr/bin/chloros-cli` |
| 後端 | `/usr/lib/chloros/chloros-backend`（`/usr/lib/chloros/backend/chloros-backend` 的啟動程式） |

可透過 `--backend-exe` CLI 標誌或 `backend_exe` SDK 建構函式參數， 並使用 `--port` 設定埠號（預設為 `5000`）。

{% hint style="info" %}
`CHLOROS_BACKEND_URL` 會將 **`lattice`**、**`project`**以及**`daq pool-*`** 指令系列。 核心指令（`process`、`login`、`logout`、`status`、 `export-status`、`time-sync`、`selftest`）會刻意忽略該參數，並始終將目標鎖定為 `http://127.0.0.1:<port>`。
{% endhint %}

***

## Linux 上的 LATTICE 攝影機與 DAQ 光感測器

所有 live-hardware 指令集均可在 Linux（amd64 和 Jetson）上運作：

* **`chloros-cli lattice`** — 用於偵測、連線、設定以及從 LATTICE 攝影機和同步陣列擷取影像。`.deb` 整合了其所需的 Arena SDK 執行時環境，並將其註冊至動態載入器。
* **`chloros-cli daq pool-*`** — 透過後端池連接 DAQ-U/M/E 光感測器，串流經校準的光譜、記錄 `.daq` 檔案並交換 cap 設定檔；針對 DAQ-E，可變更其網路設定、重新啟動及更新韌體。 `chloros-cli daq discover` 會掃描可連接的感測器，而 `chloros-cli daq --help` 則列出 `pool-*` 的子指令。
* **`chloros-cli project`** — 無介面模式下執行已儲存的專案（包含其攝影機、感測器及處理設定）。
* **`chloros-cli time-sync`** — 檢查 Chloros 後端為 LATTICE 相機和 DAQ-E 感測器所執行的 PTP 主時鐘。

```bash
# DAQ-E at a known address — the reliable path on multi-homed hosts
chloros-cli daq pool-connect --eth-host 192.168.2.50

# DAQ-U over USB serial
chloros-cli daq pool-connect --port /dev/ttyUSB0

# What is connected, then the latest calibrated spectrum as JSON
chloros-cli daq pool-list
chloros-cli daq pool-latest --sensor-id daq-e-a1b2c3 --json
```

`--sensor-id` 是 `pool-latest` 的必備條件， `pool-stream`、`pool-record` 及 `pool-set-cap` 所必需；`pool-list` 顯示目前池中存在的 ID。

{% hint style="info" %}
**`--eth-host` 跳過發現階段。** `pool-connect --eth` 透過 mDNS 偵測到 DAQ-E，其 ARP 備用機制可在 ARP 快取清空的狀態下正常運作；若將感測器的 IP 位址或主機名稱傳遞給 `--eth-host`，即可直接連線至該感測器。
{% endhint %}

**DAQ-U 序列埠權限** 由已安裝的 udev 規則處理（`uaccess` + 群組 `dialout`）。若已連接的感測器仍無法存取，請重新載入規則或重新插拔感測器：

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=tty
```

請參閱 [CLI 參考手冊](../CLI.md) 以了解完整的指令集。

### 無顯示器主機的常駐 PTP

此套件會為後端安裝兩個 systemd 服務，但兩者均未啟用。 若在無顯示器的 Jetson 或伺服器上，需為 DAQ-E 感測器和 LATTICE 攝影機持續運行 PTP 時間同步，或當 SDK 腳本連接已儲存的專案時（`connect_all()` 絕不會啟動後端），請以 **您的使用者身分** 啟用該服務：

```bash
sudo systemctl enable --now chloros-backend@$USER.service    # $USER expands to your user name
systemctl status chloros-backend@$USER.service
```

`chloros-backend@<user>.service` 會以該使用者身分執行後端：其寫入的擷取資料歸該使用者所有，詳細日誌記錄於該使用者的 `~/.cache/chloros/logs` 中，且會使用該使用者的 `chloros-cli login` （`~/.chloros`），因此請先以該使用者身分登入一次。安裝套件時會輸出此指令，並填入您的使用者名稱。

若未以服務形式運行，後端（以及其所包含的 PTP 主時鐘）僅會在被啟動時運行——例如透過 `chloros-cli` 指令或 SDK 腳本。

該單元會將後端綁定至 `127.0.0.1:5000` （單元內的 `CHLOROS_HOST` / `CHLOROS_PORT` 環境設定；可透過 `sudo systemctl edit chloros-backend@.service` 覆寫），並在發生錯誤後 5 秒重新啟動它。

**root 服務。**`chloros-backend.service` 以**root** 身分執行後端。該服務仍會安裝，以便已啟用它的機器能持續運作， 但 root 後端所寫入的一切皆歸 root 所有，因此使用者無法刪除自己的擷取資料；其日誌位於 `/var/log/chloros`，可由 `adm` 群組讀取。若要切換至您的使用者帳戶：

```bash
sudo systemctl disable --now chloros-backend.service
sudo systemctl enable --now chloros-backend@$USER.service
```

**服務的後端是共享的，絕不會被客戶端停止。** `chloros-cli` 和 SDK 腳本會直接使用現有的後端：`process` 在其上執行，並載入自身專案以取代上一個專案。 在其他情況下，CLI 會重新啟動後端（例如 `--restart`，或是進口時未找到任何項目的 `process`）， 它會標示該單元名稱、輸出 `sudo systemctl restart <unit>` 並退出 `1`，因為以這種方式停止的後端將保持停機狀態。請自行使用 `sudo systemctl restart chloros-backend@$USER.service` 重新啟動它。 服務的後端絕不會在閒置時自行停止，而 `chloros-cli backend stop` 會拒絕此操作，並標示出 `systemctl stop` 指令。**PTP 如何取得其埠號。** PTP 使用 UDP 319/320，這兩者均低於標準的 1024 特權埠下限。 該套件的 `postinst` 會透過 `/etc/sysctl.d/60-chloros-ptp.conf` 寫入 `net.ipv4.ip_unprivileged_port_start = 319`，這使得後端能在以您的使用者身分執行時綁定這些埠。 自 1.3.0 版本起，此套件不再為後端設定檔案權限（`setcap`），且 `libcap2-bin` 亦不再是依賴項：在未解壓縮的後端中，檔案權限會阻礙其啟動， 因此請勿自行新增此權限。***

## Bash 腳本範例

{% hint style="info" %}
**適合腳本使用的退出代碼。**`chloros-cli process` 在成功時退出 `0`，**失敗時則返回非零值 — 包括請求影像產品但未寫入任何產品的執行情況** （此時會輸出 `Processing finished but wrote no image products.`，並列出專案資料夾名稱及常見原因）。成功執行時會回報寫入的影像產出數量（`Image products written: N`，若同時寫入遮罩則會額外回報 `(plus M clamp masks)`）。退出代碼：`0` 表示成功，`1` 表示失敗，`2` 表示參數錯誤，`130` 表示執行中斷。
{% endhint %}

### 處理多個資料集

```bash
#!/bin/bash
for dataset in ~/datasets/2026/*/; do
    echo "Processing $(basename "$dataset")..."
    if chloros-cli process "$dataset" --format "TIFF (32-bit, Percent)"; then
        echo "Done: $(basename "$dataset")"
    else
        echo "FAILED: $(basename "$dataset")" >&2
    fi
done
```

### 採用自訂設定進行處理

```bash
#!/bin/bash
chloros-cli process ~/datasets/field_a \
    --output ~/output/field_a \
    --format "TIFF (32-bit, Percent)" \
    --indices NDVI GNDVI OSAVI \
    --debayer texture-aware \
    --no-vignette
```

有效的 `--format` 值共有四種，且其中包含空格 — 請務必以引號括起：

| `--format` 值 | 輸出資料夾 |
| --- | --- |
| `TIFF (16-bit)` *(預設)* | `tiff16` |
| `TIFF (8-bit)` | `tiff8` |
| `TIFF (32-bit, Percent)` | `tiff32` |
| `PNG (8-bit)` | `png8` |
| `JPG (8-bit)` | `jpg8` |

`--debayer` 接受 `standard`（預設）或 `texture-aware`（Chloros+）。

### 透過 Cron 進行自動化處理

```cron
# Process any new datasets at 2 AM daily
0 2 * ** /usr/bin/chloros-cli process /data/incoming --output /data/processed >> /var/log/chloros.log 2>&1
```

### Python SDK 範例

```python
from chloros_sdk import process_folder

# One-line processing
result = process_folder(
    "/home/user/datasets/flight001",
    indices=["NDVI", "OSAVI"],
    export_format="TIFF (32-bit, Percent)"
)
```

***

## 疑難排解

### 安裝後找不到 CLI

```bash
# Check if the binary exists
which chloros-cli
ls -la /usr/bin/chloros-cli

# List everything the package installed
dpkg -L chloros

# Reload your shell
source ~/.bashrc
```

### 權限遭拒

```bash
sudo chmod +x /usr/bin/chloros-cli
sudo chmod +x /usr/lib/chloros/chloros-backend /usr/lib/chloros/backend/chloros-backend
```

### 新增檔案權限後後端無法啟動

請勿在後端執行 `setcap`。 自 1.3.0 版本起，此套件不再設定任何檔案權限，而 PTP 的 319/320 埠號源自 [PTP 如何取得其埠號](#always-on-ptp-for-headless-hosts) 中所述的 `sysctl` 設定。 檔案權限會將未解壓縮的後端置於安全執行模式，在此模式下它無法找到自身的函式庫，因此無法啟動。若您曾新增此設定，請將其移除：

```bash
sudo setcap -r /usr/lib/chloros/backend/chloros-backend
```

### PTP 無法啟動／無法綁定 319 埠

請確認非特權埠的下限值是否已調低；若尚未調整，請針對本次開機重新套用該設定：

```bash
sysctl net.ipv4.ip_unprivileged_port_start     # expect 319
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=319
```

接著檢查主節點：

```bash
chloros-cli time-sync status
chloros-cli time-sync peers
```

### `lattice` 指令執行失敗，錯誤訊息為「找不到 LATTICE 攝影機驅動程式」

後端無法載入其隨附安裝的 LATTICE 攝影機執行時模組。所有 `lattice` 指令皆在後端執行；CLI 本身並不包含任何攝影機驅動程式。請重新安裝 Chloros（`sudo apt install --reinstall ./chloros_1.3.0_<arch>.deb`）：該安裝程式會將執行時元件安裝於後端旁（位於 `/usr/lib/chloros/arena_runtime` 目錄中），並向動態載入器進行註冊。若要檢查該註冊狀態：

```bash
cat /etc/ld.so.conf.d/Arena_SDK.conf     # expect /usr/lib/chloros/arena_runtime
ls /usr/lib/chloros/arena_runtime | head
```

### 後端啟動失敗

```bash
# Check if port 5000 is already in use
lsof -i :5000

# Kill any existing process on port 5000
kill $(lsof -t -i :5000)

# Try starting with a different port
chloros-cli --port 5001 process ~/datasets/flight001
```

啟動失敗的後端日誌位於 `~/.cache/chloros/logs/`。若 5000 埠屬於某個後端服務（`systemctl status chloros-backend@$USER.service`），請重新啟動該服務，而非終止其程序。

### 未偵測到 CUDA

```bash
# Check NVIDIA driver installation (x86_64)
nvidia-smi

# Check CUDA availability
nvcc --version

# On Jetson, check the L4T release (JetPack 6 is R36) and that JetPack's CUDA libraries are installed
cat /etc/nv_tegra_release
sudo apt install nvidia-jetpack
```

`chloros-cli selftest` 在同一行中報告了相同內容：`GPU: <name>, CUDA: <bool>, PyTorch: <version>`。

### 缺少共用函式庫

```bash
sudo apt-get update
sudo apt-get install -f

# Check for missing libraries (the program itself, not the launcher script)
ldd /usr/lib/chloros/backend/chloros-backend | grep "not found"
```

### SD 卡系統上的啟動速度緩慢

自 1.3.0 版本起，後端與 CLI 皆以未解壓縮形式安裝，因此不再於每次啟動時自動解壓縮至臨時目錄。 若該資料夾存在，臨時檔案會自動儲存至 `/mnt/ssd/tmp`；否則，您可以將其指向一個高速檔案系統：

```bash
export TMPDIR=/mnt/nvme/tmp
```

***

## 在 Linux 系統上更新 Chloros

`update` 指令僅適用於 Linux/Jetson 系統。該指令會檢查在 `/etc/chloros/update.conf` 所設定的更新通道中發布的版本，並提供下載及安裝相應的 `.deb`：

```bash
# Check for updates without installing
chloros-cli update --check

# Check for and install updates
chloros-cli update
```

在 Linux/Jetson 環境中，CLI 也會在每次啟動時執行非阻塞式更新檢查 （結果會於 `~/.chloros/update_cache.json` 中快取一小時），並在存在更新版本時輸出 `Update available: vX.Y.Z`。您的設定、專案及登入狀態在更新後仍會保留。

套件升級過程中會暫停所有後端服務，因此正在執行的腳本將失去其相機和感測器連線，隨後會重新啟動每個已啟用的服務（若某個服務是手動啟動但從未啟用，則會保持停止狀態，如同重新開機後的情況）。

## 解除安裝

```bash
sudo apt remove chloros
```

解除安裝會停止並停用所有後端服務，將非特權埠下限恢復為預設值（1024），移除接收緩衝區設定檔（當前運行中的數值將保留至重新開機為止）、內建的 exiftool 符號連結、 Arena 載入器的設定檔、套件新增的防火牆規則以及其安裝的 Python SDK，並清除快取的憑證。您的專案與 `~/.chloros/` 資料檔將不受影響。 `sudo apt purge chloros` 亦會刪除 `/var/log/chloros`。

***

## 後續步驟

* [NVIDIA Jetson 指南](nvidia-jetson-guide.md) — 針對 Jetson 的優化與部署
* [CLI：命令列](../CLI.md) — CLI 指南
* [API：Python SDK](../api-python-sdk.md) — SDK 指南
* [CLI 參考手冊](../reference/cli-reference.md) 及 [SDK 參考手冊](../reference/sdk-reference.md) — 1.3.0 版本的完整命令/API 清單
* [動態運算適應](../processing-architecture/dynamic-compute-adaptation.md) —— 說明 Chloros 如何適應您的硬體
