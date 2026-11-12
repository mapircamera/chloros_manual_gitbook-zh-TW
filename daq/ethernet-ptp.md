# DAQ-E 網路連線與時間同步

> 感測器的實體網路設定——包括線路佈置、PoE、IP 位址指派以及裝置本身的網路設定——詳見 **[DAQ 使用手冊](https://mapir.gitbook.io/daq/daq-e/network-setup)**。 本頁說明 Chloros 端相關內容：連線、時間同步，以及當發現程序未找到裝置時應如何處理。

DAQ-E 是 DAQ 產品系列中的乙太網路成員：透過 PoE 供電、藉由 mDNS（服務 `_daq-e._tcp`）進行發現，並可透過由其感測器 ID 衍生的主機名稱進行存取 — `daq-e-<6 hex>.local`，例如 `daq-e-def330.local`。本頁說明其如何在網路中傳輸資料，以及如何參與 PTP 時間同步。

## 傳輸模式

| 模式 | 端點 | 使用者 | 備註 |
| --- | --- | --- | --- |
| **多播**（預設） | UDP `239.10.10.10:5002` | 同一局域網內任意數量節點皆可接收相同資料流 | 每個資料包均經過 CRC-16/CCITT 驗證 |
| **原始模式** | TCP 埠 `5000` | 僅限一位客戶端（專用） | 與 DAQ-U 具備字節相容性，可直接套用 |

Chloros 預設使用多播模式，這使得 GUI、CLI 和 SDK 都能同時監控同一感測器。

## 網路需求

* **同一廣播域。** 執行 Chloros 的機器必須與感測器位於同一 L2 網路區段 — mDNS 發現機制無法穿越路由器。
* **防火牆。** 在 Windows 系統上，安裝程式會為後端在每個網路設定檔中新增 **Chloros 後端 (UDP-In)**防火牆規則，該規則涵蓋 DAQ-E 資料（UDP 5002）、 mDNS（UDP 5353）及 PTP（UDP 319/320）；此外，還會新增一條僅限於本機子網的**「Chloros 後端（TCP-In）」** 規則，用於 DAQ-E 在韌體更新期間與電腦建立回連線。 在 Linux 系統上，若安裝時該防火牆處於啟用狀態，`.deb` 會為 `ufw` 或 `firewalld` 開啟 UDP 埠。 若使用第三方防火牆，或於後續啟用防火牆，則需對相同埠開放存取權限。
* **PoE 供電，無狀態 LED。** DAQ-E 本身沒有 LED 指示燈 — 請透過交換機或注入器埠上的連結/PoE 指示燈確認供電狀態，並在通電後等待數秒，讓裝置完成開機並加入網路。

## 連線

**GUI：** 「光感測器」分頁 → 「連線感測器」 → 裝置類型「DAQ-E (乙太網路)」。裝置偵測僅在連線對話方塊顯示於螢幕時執行（Windows 系統會進行 mDNS 瀏覽及 ARP 掃描），每 15 秒重複一次；按下「重新整理」按鈕可立即重新掃描。 偵測到的感測器會顯示在下拉式選單中；系統會自動選取第一個偵測到的感測器。

<!-- SCREENSHOT-NEEDED: DAQ connect dialog with Device Type set to "DAQ-E (Ethernet)" and at least one discovered sensor listed in the Hostname/IP dropdown (e.g. daq-e-xxxxxx.local), Connect button enabled. -->

**CLI**（後端正在執行）：

```bash
chloros-cli daq pool-connect --eth                              # auto-discover on the LAN
chloros-cli daq pool-connect --eth-host daq-e-def330.local      # explicit host — the reliable form
chloros-cli daq pool-connect --eth-host 192.168.1.57            # a plain IP works too
```

池中已存在的 DAQ-E 可透過其主機名稱或 IP 位址識別：若以另一組位址再次連線，系統將返回現有的池項目，而非再次建立該裝置的連線。

### 具備多張網路介面卡的主機與開機後的首次連線

在具備多個活躍網路介面主機上，即使感測器狀態正常，開機後的**首次** `pool-connect --eth` 連線仍可能返回空結果——當 ARP 快取尚未預熱時，偵測掃描可能會遺漏感測器所在的介面。 可靠的解決方法是跳過自動發現，並明確傳遞位址：

```bash
chloros-cli daq pool-connect --eth-host daq-e-def330.local
```

`--eth-host` 可接受 mDNS 主機名稱或 IP 位址，始終能鎖定正確的感測器，且是腳本及無頭安裝的建議格式。在 GUI 中，請使用連線對話方塊的「重新整理」按鈕，並等待重新掃描週期完成。

## 裝置設定與韌體

感測器本身儲存其網路設定：DHCP（出廠預設值，若無 DHCP 伺服器回應則退而採用 169.254.x.x 的鏈路本地位址）或靜態位址。它亦儲存開機時是否開始串流的設定。 請透過圖形化介面（感測器的齒輪圖示設定，**網路**：**變更…**、**開機時串流**、**重新啟動感測器**；請參閱 [DAQ 標籤頁](gui.md))，或透過 CLI 和 SDK 使用 `pool-set-network`、 `pool-set-auto-stream` 及 `pool-reboot` (`DAQSensorSession.set_network()`、`set_auto_stream()`、`reboot()`；請參閱 [CLI 參考手冊](../reference/cli-reference.md#daq-e-device-settings))。 感測器會重新啟動以套用網路變更，而 Chloros 會重新連線至該感測器。靜態 IP 位址必須位於執行 Chloros 的電腦所屬網路中：若重新啟動後無法連線至該位址，Chloros 將拒絕接受該位址。感測器的 ID 及其 `daq-e-<id>.local` 主機名稱永遠不會改變。 1.3.0 版本新增功能。

**韌體更新已整合至 GUI 中。** 當已連線的 DAQ-E 所執行之韌體版本早於您 Chloros 建置版本所附帶的映像檔時，其感測器列會顯示一個琥珀色的 **有更新可用** 圓點，且齒輪設定模態視窗中會提供一個「更新至 \<version\>」按鈕。 更新過程透過網路進行，約需 30 秒；感測器會自動重新啟動並重新連線，若傳輸中途中斷，當前韌體將保持不變。 若透過 CLI 或 SDK 操作，或在未提供 GUI 的 Linux 系統上，請執行 `chloros-cli daq pool-update-firmware --sensor-id <id>`（`DAQSensorSession.update_firmware()`）。

<!-- SCREENSHOT-NEEDED: DAQ-E per-sensor settings modal showing the DAQ-E-only rows: Hostname/IP, Firmware row with the "Update to <ver>" button (or "Up to date"), and the PTP Sync row with a live state value. -->

## PTP 時間同步

DAQ-E 韌體 v1.2.0+ 作為普通（僅從屬）時鐘參與 IEEE 1588 PTPv2 協定。 **Chloros 主機的後端即為 PTP 總時鐘** — 區域網路 (LAN) 上的每台 DAQ-E 及每台 LATTICE 攝影機均在網域 0 中對其進行從屬同步，確保所有裝置的時間戳記誤差控制在約 1 毫秒以內。 正是這個共用時鐘，使得 DAQ 讀數的時間戳記能與相機的曝光時間相匹配（參見 [錄製與 .daq 格式](recording.md)）。

透過CLI檢查同步狀態：

| 指令 | 顯示內容 |
| --- | --- |
| `chloros-cli time-sync status` | 主機主時鐘狀態、BMCA 優先級、時鐘識別碼，以及 `health`：`OK`，或 `SENDS_FAILING` / `DEGRADED` / `NO_SOCKETS` / `PEERS_SILENT`，並附有 `health_detail` 說明其封包為何無法傳送至網路 |
| `chloros-cli time-sync peers` | 所有偵測到的從屬裝置（DAQ-E 感測器 + LATTICE 攝影機） |
| `chloros-cli time-sync cameras` | 各攝影機的 PTP 狀態 (`PtpStatus`、`PtpOffsetFromMaster`、`PtpMeanPathDelay`) |
| `chloros-cli time-sync restart` | 重新啟動主控程序 |

在圖形使用者介面（GUI）中，DAQ-E 設定模態視窗會顯示一個即時更新的 **PTP 同步** 列，其中包含感測器當前的 PTP 狀態。**若 DAQ-E 的 PTP 已關閉，則會保持關閉狀態。** 該裝置會將 PTP 的開啟/關閉設定儲存於其自身記憶體中，因此，若裝置的 PTP 先前已關閉，則在重新連線或重新開機後仍會保持關閉狀態，且其時間戳記不會與 PTP 同步。 `chloros-cli daq pool-list` 會標記此類裝置（`!! PTP is switched OFF on this unit …`），並輸出可解決此問題的指令：

```bash
chloros-cli daq pool-enable-ptp --sensor-id daq-e-def330
```

Chloros 絕不會自行將裝置重新啟用，因為關閉狀態可能是刻意設定的。

針對嚴格對齊使用者之詳細說明：

* 每個串流資料包皆包含一個標誌欄位；**當幀的時間戳記已透過 PTP 同步時，第 2 位元會被設定**。需要嚴格相機／數據擷取（DAQ）對齊的管線應以此位元作為觸發條件。
* 在進行同步擷取之前，請確認感測器出現在 `chloros-cli time-sync peers` 中，且 `chloros-cli daq pool-list` 未標記其 PTP 處於關閉狀態。 (MAPIR 的內部直接硬體工具亦可透過 `--wait-ptp` 標誌，在 PTP 鎖定時觸發錄製，該標誌會等待最多 15 秒，直至感測器進入 SLAVE 狀態；此工具不屬於隨附的 CLI 範疇。)
* 當 PTP 處於主動從屬狀態時，感測器會拒絕手動時鐘推送（「PTP 正在提供時鐘」）。這是設計上的行為——請信任 PTP。

## Linux 注意事項

* **無 root 權限的 PTP 連接埠。** PTP 綁定 UDP 319/320 埠。 `.deb` 會寫入 `/etc/sysctl.d/60-chloros-ptp.conf`（`net.ipv4.ip_unprivileged_port_start = 319`），因此後端會以您的使用者身分綁定這些埠。 自 1.3.0 版本起，此套件不會在後端設定任何檔案權限（`setcap`），亦無需 `libcap2-bin`；請勿自行新增權限，否則解壓縮後的後端將無法啟動。 若 PTP 無法綁定 319 號埠，請參閱 [Linux 安裝說明](../linux/linux-installation.md#ptp-will-not-start--cannot-bind-port-319)。

* **無顯示器的 Jetson / Linux 主機：** 此套件會安裝後端服務，但不會啟用任何服務。若要實現無需手動執行命令即可持續運作的 PTP（以及 DAQ 可用性），請以您的使用者身分啟用該服務：

  ```bash
  sudo systemctl enable --now chloros-backend@$USER.service    # $USER expands to your user name
  ```

  若未啟用，PTP 僅會在有人啟動後端時運行——例如執行 `chloros-cli` 指令或 SDK 腳本。 請參閱 [Linux 安裝指南](../linux/linux-installation.md#always-on-ptp-for-headless-hosts)，了解 root 權限 (`chloros-backend.service`) 及其切換方法。

## 疑難排解：「未找到 DAQ-E 裝置」

| 檢查項目 | 詳細說明 |
| --- | --- |
| 電源 | 感測器上無 LED 指示燈 — 檢查交換機／注入器的 PoE 及連線指示燈；通電後請等待數秒 |
| 廣播域 | 主機與感測器位於同一 L2 區段；mDNS 無法路由 |
| 防火牆 | Windows：安裝程式中的 **Chloros Backend (UDP-In)** 規則會允許 DAQ-E 串流（UDP 5002）、mDNS（5353）及 PTP（319/320）進入；第三方防火牆亦需設定相同規則。 Linux：`.deb` 僅在安裝時防火牆處於啟用狀態的情況下，才會於 `ufw` / `firewalld` 中開啟這些埠 — 若後續啟用防火牆，請自行允許這些連線 |
| 多網卡主機 | 探索功能會透過 mDNS 進行瀏覽，並以 ARP 作為備用方案；`--eth-host <ip-or-hostname>` 會跳過探索步驟並直接連線 |
| GUI 重新掃描 | 探索功能僅在連線對話方塊開啟時執行；請使用「重新整理」 |
