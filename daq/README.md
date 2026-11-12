# DAQ 光感測器

> **正在尋找相關硬體資訊嗎？**感測器本身——包括型號、安裝方式、保護蓋、連接埠、電源以及 SCANNER 應用程式——相關說明請參閱**[DAQ 使用手冊](https://mapir.gitbook.io/daq)**。 本章節將說明如何透過 Chloros 使用這些感測器。

MAPIR 的 **DAQ** 光感測器會將環境光測量為經輻射計校正的光譜。在 Chloros 中，它們扮演兩種角色：

* **獨立光譜儀器** — 即時光譜圖、色度數據以及 `.daq` 記錄，皆可透過 [光感測器分頁](gui.md)、[CLI](cli-quick-start.md) 或 Python SDK 取得。
* **用於反射率計算的向下輻照度來源** — 在處理過程中， Chloros 會將您的 `.daq` 讀數插值至每個擷取畫面的曝光時間戳記，並利用測得的向下輻照度，將相機輻度轉換為反射率 (`--reflectance-source daq`)；對於已校準的波段，無需場景內面板。

<!-- SCREENSHOT-NEEDED: product photo of the DAQ-U, DAQ-M, and DAQ-E units side by side, each with its Sunshine cosine-corrector cap fitted (request from hardware team — no repo asset exists) -->

***

## 三種型號，一種資料格式

| 型號 | 傳輸方式 | 發現機制 |
| --- | --- | --- |
| **DAQ-U** | USB（序列埠） | 序列埠掃描 |
| **DAQ-M** | 藍牙低功耗 | 依名稱進行 BLE 掃描 |
| **DAQ-E** | 以太網（IPv4，PoE 供電） | mDNS `_daq-e._tcp`（主機名稱 `daq-e-<id>.local`） |

**DAQ-E-S（1.3.0 版本新增）。** 一款配備固定式擴散片與傾斜感測器的 DAQ-E。Chloros 將其列為 `DAQ-E-S`，其運作方式與 DAQ-E 完全相同，惟無需進行電容校正，且每筆讀數皆包含裝置的傾斜、滾轉與俯仰角度。 請參閱 [DAQ-E-S：固定擴散板與傾斜](caps-and-range.md#daq-e-s-fixed-diffuser-and-tilt)。

這三款設備均採用相同的通訊協定，並傳輸完全相同的資料：

* 每幀皆包含 **135 點光譜（波長範圍 340 至 1010 nm，間隔 5 nm）**，以及 CIE XYZ 三刺激值。
* **以 W/m²/nm 為單位的輻量校正光譜輻照度** — 數據傳送至您手中之前，已套用各裝置的出廠校正套件（以及其有效的遮光罩校正配置檔）。
* 相同的 **`.daq` 記錄格式**（SQLite 檔案）。無論檔案由哪種傳輸方式產生，後續處理流程皆完全一致。

傳輸堆疊（USB 序列、BLE、mDNS/zeroconf）已整合於 Chloros 後端之中 — 無論是透過 GUI 還是 CLI 的 `pool-*` 指令與這三種型號通訊，皆無需額外安裝任何軟體。

***

## 校準範圍：報告值為 340–1010 nm，校準值約為 374–974 nm

感測器會報告完整的 340–1010 nm 網格，但可追溯至 NIST 的輻測增益範圍約為 **374–974 nm**。 Chloros 會拒絕對任何光譜權重有少於一半位於該校準範圍內的相機波段進行絕對反射率除法運算；被跳過的波段會以跳過原因 `dls-uncalibrated-band-<nm>` 進行回報。

在現有 LATTICE 濾光片型號中，僅 **F988** 受此影響：

F988 的反射率是透過現場反射率面板進行校準的：由於該波段超出 DAQ 光感測器的校準範圍，因此 Chloros 會採用您最近一次的面板擷取數據，並在兩次面板測量之間維持該值。

若僅有 DAQ 資料可用時處理 F988 擷取資料，Chloros 將拒絕該波段基於 DAQ 的反射率，並以跳過原因 `dls-uncalibrated-band-988` 進行報告 ——[反射率面板工作流程](../calibration-targets.md) 才是 F988 受支援的處理路徑。

***

## 感測器 ID

每個 DAQ 都會回報一個穩定的感測器 ID。其格式因型號而異：

| 型號 | ID 格式 | 範例 |
| --- | --- | --- |
| DAQ-U | 5 八位元組，以連字號分隔 | `CB-7C-A8-2E-5F` |
| DAQ-M | 5 八位元組，以連字號分隔 | `CB-74-02-30-6B` |
| DAQ-E | `daq-e-<6 hex digits>` | `daq-e-def330` |

感測器 ID 為：

* 鐫刻於其記錄的每個 `.daq` 檔案中，
* Chloros 用於擷取該裝置出廠校準套件的金鑰，
* 您在CLI的`pool-*`指令中傳遞給`--sensor-id`的值，以及
* 對於 DAQ-E 而言，同時也是其 mDNS 主機名稱（`daq-e-def330.local`）——此即 `--eth-host` 所接受的值。

***

## 工廠校準與雲端

每台 DAQ 裝置均透過可追溯至 NIST 的輻射測量鏈進行個別出廠校準，且 Chloros 會為每台裝置載入以感測器 ID 為關鍵的校準套件。 各單元的校準報告（PDF）可從感測器的 [光感測器標籤頁](gui.md) 中的設定中下載。

{% hint style="warning" %}
**DAQ-U 和 DAQ-M 需連線至雲端才能進行校準。**這兩款型號均不於裝置內儲存任何資料：其出廠校準套件存放於 MAPIR 的雲端，並透過感測器 ID 進行擷取（隨後儲存於本地快取）。 Chloros 需要網際網路連線，才能從 DAQ-U 或 DAQ-M 傳輸經校準的 W/m²/nm 數據。**DAQ-E 則屬例外** — 其校準資料儲存於裝置內。
{% endhint %}
<!-- PRE-PUBLISH-CHECK: LAUNCH item 3 (DAQ-M end-to-end connect smoke) was still unverified as of 2026-08-16 — re-confirm the DAQ-M cloud-calibration flow on the release build before publishing this page. -->

***

## 記錄資料的儲存位置

| 介面 | 預設 `.daq` 儲存位置 |
| --- | --- |
| 圖形化使用者介面 —「光感測器」分頁 | `<project folder>/light_sensor/`（完成的記錄會自動新增至專案中） |
| CLI — `daq pool-record` | `~/Documents/DAQ Live View/`（位於執行後端程式的機器上） （當後端位於同一台電腦時，相對路徑 `-o` 會根據您執行命令的資料夾進行解析） |

每個 `.daq` 檔案名稱均包含感測器 ID 及時間戳記。

***

## 本章內容

* [**Chloros 中的「DAQ」分頁**](gui.md) — 完整的 GUI 操作指南：各型號的連接方式、單一感測器設定、光譜圖表、即時色度數據、雙感測器反射率，以及記錄功能。
* [** CLI 快速入門 (pool-\*)**](cli-quick-start.md) — 透過 `chloros-cli daq pool-*` 驅動 DAQ 感測器，以及受支援的命令列路徑。
* [**上限曲線與校準範圍**](caps-and-range.md) — 各型號可用的上限曲線、宣告方式，以及校準光譜範圍的詳細說明。
* [**記錄與 .daq 格式**](recording.md) — `.daq` 的 SQLite 格式及記錄工作流程。
* [**DAQ-E 網路連線與時間同步**](ethernet-ptp.md) — DAQ-E 傳輸模式與 PTP 時間同步。
* [**反射率工作流程**](reflectance.md) — 利用 DAQ 下行輻射資料生成反射率。
* 如需完整的旗標層級文件，請參閱 [CLI 參考手冊](../reference/cli-reference.md)（第 `chloros-cli daq` 節）以及 [SDK 參考手冊](../reference/sdk-reference.md)（`chloros_sdk.connect_daq_sensor()`），這兩份文件均以人工智慧助理可直接運用為撰寫目標。
