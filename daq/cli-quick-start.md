# CLI 快速入門（pool-*）

隨附的 `chloros-cli` 透過 **`daq pool-*`** 指令集驅動 DAQ 感測器——這是一組輕量級 HTTP 客戶端，透過 Chloros 後端的持久化感測器池來操作感測器。 由於後端負責管理傳輸通道，因此 GUI、CLI 及 SDK 腳本皆共用一個活躍的處理程序，無需爭奪同一端口。 客戶所需的一切功能皆可透過 `pool-*` 實現：連線、串流、記錄經校準的 `.daq` 檔案，以及切換 cap 設定檔。

`pool-*` 是已發布版本中受支援的 DAQ 介面。`chloros-cli daq --help` 列出 `pool-*` 的子指令，而 `daq discover` / `daq list` 會執行與 `pool-discover` 相同的掃描。 若無直接硬體 DAQ 子命令可用，系統會以明確的錯誤訊息結束執行，指出缺失的套件名稱，並引導您回歸至 `pool-*` —— 絕不會發生無聲失敗的情況。 （`pip install chloros-sdk` 同樣不提供直接硬體指令。）

***

## 先決條件

* **Chloros 後端必須正在運行** — `pool-*` 指令是 HTTP 客戶端，而非硬體驅動程式。在 Windows 上，請啟動 Chloros 桌面應用程式（它會啟動後端）。 在無顯示器的 Linux/Jetson 系統上，請以您的使用者身分啟用服務：`sudo systemctl enable --now chloros-backend@$USER.service`（請參閱 [Linux 安裝](../linux/linux-installation.md#always-on-ptp-for-headless-hosts)）。
* **Chloros+（付費方案）登入**：請先執行 `chloros-cli login`。 權限驗證在伺服器端進行 — 若未登入，指令執行時會因 `401 AUTH_REQUIRED` 而失敗；在免費（Iron）方案中，則會因 `403 PLAN_UPGRADE_REQUIRED` 而失敗。
* 這些指令預設以 `http://127.0.0.1:5000` 為目標；若您的後端在其他位置運行，`daq pool-*` 系列會遵循 `CHLOROS_BACKEND_URL` 環境變數的設定。

***

## 五分鐘的會話

```bash
# 1. Connect a sensor into the backend pool (pick the line matching your model)
chloros-cli daq pool-connect                                  # smart-detect any DAQ
chloros-cli daq pool-connect --port COM3                      # DAQ-U on a specific COM port
chloros-cli daq pool-connect --mac AA:BB:CC:DD:EE:FF          # DAQ-M by BLE MAC
chloros-cli daq pool-connect --eth-host daq-e-def330.local    # DAQ-E by hostname (reliable)
#    (a DAQ-M's BLE MAC is not printed anywhere: find it with `chloros-cli daq pool-discover --only ble`)

# 2. List the pool — this shows the sensor_id used by every command below
chloros-cli daq pool-list

# 3. Read the most recent calibrated spectrum frame (add --json for scripting)
chloros-cli daq pool-latest --sensor-id daq-e-def330 --json

# 4. Record a calibrated .daq file for 60 seconds
chloros-cli daq pool-record --sensor-id daq-e-def330 --duration 60 \
  --device-name "field-A"

# 5. Release the sensor when done
chloros-cli daq pool-disconnect --sensor-id daq-e-def330
```

***

## `pool-connect` — 在感測器池中開啟一個感測器

| 變體 | 含義 |
| --- | --- |
| `daq pool-connect` | 智慧偵測：尋找本機上的任何 DAQ。 |
| `daq pool-connect --port PORT` | 位於特定序列埠上的 DAQ-U（例如：`COM3`、`/dev/ttyUSB0`）。 |
| `daq pool-connect --ble` | 透過 BLE 連接的 DAQ-M，自動掃描 MAC 位址。 |
| `daq pool-connect --mac MAC` | 透過已知 BLE MAC 位址連線至 DAQ-M（意指 `--ble`）。 |
| `daq pool-connect --eth-host HOST` | 透過已知主機名稱或 IP 位址連線至 DAQ-E — **可靠的路徑**。 |
| `daq pool-connect --eth` | 透過自動發現功能連線至 DAQ-E（mDNS，並以 ARP 作為備用方案）。請參閱下方的注意事項。 |

調校旗標，皆為選填：

| 旗標 | 含義 |
| --- | --- |
| `--integration-time MS` / `-t MS` | 手動設定的積分時間（單位：毫秒）。 |
| `--frame-avg N` / `-f N` | 每個報告頻譜的幀數平均值。 |
| `--no-ae` | 停用自動曝光（預設為開啟）。 |
| `--no-stream` | 建立連線但不啟動串流（稍後可透過 `pool-stream --start` 繼續）。 |
| `--cap-id CAP` | 亮度上限校正設定檔；後端預設值為 `sunshine_cosine`。 請參閱 [`pool-set-cap`](#pool-set-cap-declare-the-fitted-cap)。 |

若將已存在於彙集中的 DAQ-E 透過其另一組位址（其 `daq-e-<id>.local` 主機名稱或 IP 位址）進行連線，系統將返回現有的彙集項目，而非再次開啟該裝置。

{% hint style="warning" %}
**`--eth` 自動偵測注意事項。** 在具備多網介面（多於一個活躍網路介面）的主機上，即使感測器狀態正常，開機後 *首次* 執行的 `pool-connect --eth` 仍可能返回空結果——這是因為在 ARP 快取尚未預熱時，自動偵測過程可能會遺漏該感測器的介面。 若 `--eth` 未偵測到任何結果，請重新嘗試，或直接跳過偵測並改用 `--eth-host <ip-or-hostname>`，這是在多網介面主機上較為可靠的處理方式。 DAQ-E 的主機名為 `daq-e-<id>.local`（例如 `daq-e-def330.local`）；其純 IP 位址同樣適用。
{% endhint %}

## `pool-discover` — 不建立連線的掃描

```bash
chloros-cli daq pool-discover                  # USB, BLE and Ethernet
chloros-cli daq pool-discover --only ble       # BLE only: this is how you find a DAQ-M's MAC
chloros-cli daq pool-discover --json           # machine-readable
```

掃描可連線的感測器，並顯示連線所需的位址，但不會實際建立連線。`--only` 接受以逗號分隔的 `usb,ble,eth` 子集； `--timeout SEC` 設定每種傳輸協定的掃描時間（預設 5 秒，限制範圍為 1–20 秒）。 池中已開啟的感測器不會列出（已連線的 DAQ-M 會停止廣播）——請使用 `pool-list` 來處理這些感測器。 在正式發佈的版本中，`chloros-cli daq discover` 和 `daq list` 會執行相同的掃描。

## `pool-list` — 查看已連接的裝置

顯示後端叢集中的所有感測器，包括其他所有指令所需的 `sensor_id`：

| 型號 | `sensor_id` 格式 | 範例 |
| --- | --- | --- |
| DAQ-U / DAQ-M | 5 八位元組連字型 | `CB-7C-A8-2E-5F` |
| DAQ-E | `daq-e-<6 hex digits>` | `daq-e-def330` |

已關閉 PTP 的 DAQ-E（設定已儲存於裝置中）會在 `pool-list` 中標記為 `!! PTP is switched OFF on this unit (saved in its flash): its timestamps are not PTP-synced.`，並附上重新啟用 PTP 的指令 — 請參閱 [`pool-enable-ptp`](#pool-enable-ptp-switch-a-daq-es-ptp-back-on)。

## `pool-latest` — 讀取光譜幀

```bash
chloros-cli daq pool-latest --sensor-id daq-e-def330 --recent 10 --json
```

返回最新的一幀，或最近的 `--recent N` 幀；`--json` 會輸出可供腳本使用的機器可讀格式。 幀資料為經輻測校正的譜線輻照度（W/m²/nm），基於 135 點、340–1010 nm 的網格，並已套用感測器的蓋板輪廓。 若需獲得定量輻照度數值，請對至少 15 秒的幀進行平均計算——此為儀器特性，並非缺陷。

## `pool-stream` — 暫停或恢復串流

```bash
chloros-cli daq pool-stream --sensor-id daq-e-def330 --stop    # pause
chloros-cli daq pool-stream --sensor-id daq-e-def330 --start   # resume
```

## `pool-record` — 記錄 `.daq` 檔案

```bash
chloros-cli daq pool-record --sensor-id daq-e-def330 --duration 150 \
  --output ~/Documents/spectra --device-name "rooftop-A"
chloros-cli daq pool-record --sensor-id daq-e-def330 --stop
```

| 標誌 | 預設值 | 含義 |
| --- | --- | --- |
| `--duration SEC` / `-d SEC` | `0` | 錄製長度（以秒為單位）； `0` 表示執行至您發出 `--stop` 為止。 |
| `--output DIR` / `-o DIR` | `~/Documents/DAQ Live View/` | 輸出目錄。 若後端位於本電腦上，相對路徑將以執行命令的資料夾為基準進行解析；若為遠端後端，則路徑將在 **執行後端的機器上** 進行解析。 |
| `--device-name NAME` | — | 與錄製檔案一併儲存的標籤。 |
| `--stop` | — | 停止正在進行的錄製。 |
| `--confirm-after SEC` | `20` | 啟動後，等待最長此時間長度以供寫入第一行資料；若未收到任何資料，則發出顯著警告（`!! WARNING: no rows written after …s. This recording is producing an EMPTY file.`）。 `0` 會停用此檢查。 |
| `--heartbeat SEC` | `30` | 搭配 `--duration` 使用時，每 SEC 秒輸出一行數，若行數停止增加則發出警告。 `0` 會停用此功能。 |

{% hint style="info" %}
記錄是在後端進行的，因此 `.daq` 檔案會存放在 **後端機器** 的檔案系統中 — 預設位置在 `~/Documents/DAQ Live View/` 該處，未必是您執行 CLI 的位置。 檔案名稱包含感測器 ID 及時間戳記。
{% endhint %}

## `pool-set-cap` — 宣告安裝的蓋帽

```bash
chloros-cli daq pool-set-cap --sensor-id daq-e-def330 --cap-id sunshine_cosine
```

蓋帽 ID 用於選取應用於每個光譜的工廠測量校正曲線，且 **必須與感測器上實際安裝的蓋帽相符** —— 無論是感測器還是軟體都無法自行偵測蓋帽，而此選項會被標記在每個 `.daq` 檔案中。 各處的預設值皆為 `sunshine_cosine`（每台 DAQ 出廠時均已安裝 Sunshine 餘弦校正蓋，設計上具有約 12 倍的衰減係數——若未聲明更換校正蓋，光譜校正結果將因此產生大致相當於該係數的誤差）。

| `--cap-id` | 適用於 |
| --- | --- |
| `sunshine_cosine`（預設） | DAQ-U、DAQ-M、DAQ-E |
| `fov_15`、`fov_45`、`fov_90` | DAQ-U、DAQ-E |
| `fov_30`、`fov_60` | 僅限 DAQ-U |
| `none` | 僅限 DAQ-E — 請參閱註解 |

若蓋帽識別碼超出感測器設定範圍，連接時將因明確錯誤而被拒絕。 `none` (DAQ-E) 表示蓋帽已實體移除 — 此設定仍會針對 DAQ-E 的內嵌式玻璃擴散片套用出廠幾何輪廓，因此並非無操作（no-op）；而未裝蓋帽的 DAQ-E 屬於實驗台配置，並非受支援的現場配置。 （裸露的 DAQ-U 屬於真正裸露狀態，完全不需要校正輪廓；DAQ-M 則需搭配其 Sunshine 蓋使用。）

## `pool-disconnect` — 釋放感測器

```bash
chloros-cli daq pool-disconnect --sensor-id daq-e-def330   # one sensor
chloros-cli daq pool-disconnect --all                      # everything in the pool
```

`pool-disconnect` 會釋放任何曾開啟該感測器的感測器，因此也會釋放因腳本當機而留在感測器池中的感測器（前提是該感測器上的記錄已先完成）。 若感測器不在感測器池中，則無感測器可釋放：該指令會輸出 `<sensor-id>: already released (not in the pool)` 並退出 `0`。若後端拒絕或未確認釋放操作，則會退出 `1`。

## `pool-enable-ptp` — 重新啟用 DAQ-E 的 PTP

```bash
chloros-cli daq pool-enable-ptp --sensor-id daq-e-def330
```

DAQ-E 會將其 PTP 的開啟/關閉設定儲存於自身記憶體中，因此一旦 PTP 被關閉，該裝置便會保持關閉狀態——且其時間戳記不會與 PTP 同步——直到重新開啟為止。 `pool-list` 會標記此類裝置，並為其輸出此指令。Chloros 絕不會自行將裝置重新啟用，因為關閉狀態可能是刻意設定的。新增於 1.3.0 版。

## DAQ-E 裝置設定

```bash
chloros-cli daq pool-info --sensor-id daq-e-def330
chloros-cli daq pool-set-network --sensor-id daq-e-def330 \
  --static 192.168.1.50 --gateway 192.168.1.1
chloros-cli daq pool-set-network --sensor-id daq-e-def330 --auto
chloros-cli daq pool-set-auto-stream --sensor-id daq-e-def330 --on
chloros-cli daq pool-reboot --sensor-id daq-e-def330
chloros-cli daq pool-update-firmware --sensor-id daq-e-def330
```

`pool-info` 會顯示感測器的韌體（以及此Chloros版本是否包含更新版本）、其位址及取得方式、與當前運行設定不同的儲存網路設定、開機串流及 PTP。 `pool-set-network` 會重新啟動感測器以套用變更，並以其原先的設定重新連線；`--no-reboot` 則僅將設定儲存下來。 靜態位址必須位於這台電腦的網路中，否則 Chloros 會拒絕該位址；`--force` 會覆寫此限制，但僅限於將被重新分配位址或路由至該位址的電腦。 當感測器正在記錄時，將拒絕重新啟動。`pool-update-firmware` 是 GUI 中 **「更新至 \<version\>** 功能在命令列介面的對應指令，也是在沒有 GUI 的 Linux 系統上更新 DAQ-E 的方式。新增於 1.3.0 版本。***

## 指令摘要

| 指令 | 用途 |
| --- | --- |
| `daq pool-connect [--port P \| --ble \| --mac M \| --eth \| --eth-host H] [-t MS] [-f N] [--no-ae] [--no-stream] [--cap-id CAP]` | 在後端池中開啟一個感測器。 |
| `daq pool-discover [--only usb,ble,eth] [--timeout SEC] [--json]` | 掃描感測器而不建立連線（尋找 DAQ-M 的 BLE MAC）。 |
| `daq pool-list` | 顯示後端池中所有感測器及其 `sensor_id` 值。 |
| `daq pool-latest --sensor-id ID [--recent N] [--json]` | 最近的 N 個已校準光譜幀。 |
| `daq pool-stream --sensor-id ID [--start \| --stop]` | 恢復／暫停串流。 |
| `daq pool-record --sensor-id ID [-d SEC] [-o DIR] [--device-name NAME] [--confirm-after SEC] [--heartbeat SEC] [--stop]` | 開始／停止 `.daq` 錄製（後端端）。 |
| `daq pool-set-cap --sensor-id ID --cap-id CAP` | 在執行期間切換電容校正配置檔。 |
| `daq pool-disconnect --sensor-id ID [--all]` | 釋放一個感測器，或釋放所有感測器。 |
| `daq pool-enable-ptp --sensor-id ID` | 為 PTP 已關閉的 DAQ-E 重新啟用 PTP。 |
| `daq pool-info --sensor-id ID [--json]` | DAQ-E 的韌體、位址、儲存的網路設定、開機串流及 PTP。 |
| `daq pool-set-network --sensor-id ID (--auto \| --static IP [--mask M] [--gateway G]) [--no-reboot] [--force]` | DHCP 或靜態 IP 位址；重新啟動並重新連線感測器。 |
| `daq pool-set-auto-stream --sensor-id ID (--on \| --off)` | 開機時即開始串流。 |
| `daq pool-reboot --sensor-id ID` | 重新啟動 DAQ-E 並重新連線。 |
| `daq pool-update-firmware --sensor-id ID` | 將此版本的 DAQ-E 韌體刷寫至舊版感測器。 |

***

## DAQ-E 首次連線的疑難排解

1. DAQ-E 沒有狀態指示燈 — 請透過交換機或注入器埠上的 PoE/連結指示燈確認供電狀態，並在通電後等待數秒，讓裝置完成開機並加入網路。
2. 後端主機必須與感測器位於**同一廣播域**內 — mDNS 無法穿越路由器。
3. 防火牆必須允許 mDNS（UDP 5353）、DAQ-E 資料（UDP 5002）及 PTP（UDP 319/320）通過。 Windows 安裝程式會為此新增一則規則，而 Linux `.deb` 則會在安裝時已啟用的 `ufw` / `firewalld` 中開啟這些連接； 若使用第三方防火牆，或日後啟用的防火牆，則需允許這些通訊。
4. 仍無法從 `--eth` 取得任何回應？ 請使用 `--eth-host` 並搭配裝置的主機名稱（`daq-e-<id>.local`）或 IP 位址，此方式將跳過自動偵測步驟。

***
{% hint style="info" %}
**給 AI 助理的小提示。** 本手冊的每頁皆以原始 Markdown 格式提供 — 請在頁面的小寫 URL 片段後方附加 `.md`（本頁為：`https://mapir.gitbook.io/chloros/daq/cli-quick-start.md`）； 機器可讀的索引為 `https://mapir.gitbook.io/chloros/llms.txt`。 如需 `chloros-cli daq` 及其他所有命令家族的完整參數級文件，請參閱 [CLI 參考手冊](../reference/cli-reference.md) (`https://mapir.gitbook.io/chloros/reference/cli-reference.md`)； 在 [SDK 參考手冊](../reference/sdk-reference.md) 中，Python 路徑為 `chloros_sdk.connect_daq_sensor()`。
{% endhint %}
