# Linux 概覽

Chloros 1.3.0 為 **CLI**和**Python SDK** 提供原生 Linux 支援——包含無顯示器的多光譜影像處理，以及對 LATTICE 攝影機和 DAQ 光感測器的即時控制——適用於 Linux 工作站、伺服器及 NVIDIA Jetson 邊緣裝置。

{% hint style="info" %}
**Linux 系統上不提供桌面 GUI。**Chloros 桌面 GUI 僅限 Windows 系統使用。Linux 使用者需透過 [CLI](../CLI.md) 及 [Python SDK](../api-python-sdk.md) 與 Chloros 進行互動。 `.deb` 確實會在您的應用程式選單中新增一個**Chloros CLI** 項目 —— 它僅會開啟一個執行 `chloros-cli` 的終端機模擬器。
{% endhint %}

***

## 平台支援對照表

| 功能 | Windows (GUI) | Windows (CLI/SDK) | Linux amd64 (CLI/SDK) | Linux arm64 / Jetson (CLI/SDK) |
| --- | --- | --- | --- | --- |
| **桌面 GUI** | 是 | 不適用 | 否 | 否 |
| **命令列介面 CLI** (`chloros-cli`) | 是 | 是 | 是 | 是 |
| **Python SDK** (`chloros-sdk`) | 是 | 是 | 是 | 是 |
| **影像處理管線** | 是 | 是 | 是 | 是 |
| **LATTICE 相機控制（即時）** | 是（「相機」分頁） | 是 (`chloros-cli lattice`、SDK) | 是 | 是 |
| **DAQ 光感測器（即時）** | 是（「光感測器」分頁） | 是 (`chloros-cli daq pool-*`、SDK) | 是 | 是 |
| **PTP 時間同步（主機為主時鐘）** | 是 | 是（`chloros-cli time-sync`） | 是 | 是 |
| **GPU 加速（CUDA）** | 是 | 是 | 是 | 是（JetPack 6） |
| **紋理感知去拜爾演算法**（需具備 CUDA GPU） | 是（Chloros+） | 是（Chloros+） | 是（Chloros+） | 是（Chloros+） |
| **動態運算適應** | 是 | 是 | 是 | 是 |
| **後端作為系統服務** (`chloros-backend@<user>.service`) | 否 | 否 | 是（需手動啟用） | 是（需手動啟用） |
| **就地更新程式** (`chloros-cli update`) | 否（需執行安裝程式） | 否（需執行安裝程式） | 是 | 是 |

***

## 支援的架構

| 架構 | 說明 | 套件 |
| --- | --- | --- |
| **amd64 (x86_64)** | 標準桌面／伺服器處理器（Intel、AMD） | `chloros_<version>_amd64.deb` |
| **arm64 (aarch64)** | ARM 處理器 — NVIDIA Jetson Orin 系列 (JetPack 6) | `chloros_<version>_arm64_jp6.deb` |

## 支援的 Linux 發行版

* **Ubuntu 22.04 LTS 或更新版本** (amd64)
* **Debian 12 或更新版本** (amd64)
* **NVIDIA JetPack 6** (arm64 — Jetson Orin 平台)

***

## Linux 使用者可獲得的內容

* **Chloros CLI** — 用於批次處理、自動化及腳本編寫的完整命令列介面
* **Chloros Python SDK** — 用於研究流程與自訂工具的 Python 程式化介面，由 `.deb` 安裝至系統 `python3` （`/usr/lib/chloros/sdk/` 中包含適用於 virtualenvs 的版本匹配 wheel 檔案；若機器未安裝 Chloros 套件，請從 PyPI 取得）
* **LATTICE 相機控制** — 透過 `chloros-cli lattice` 及 SDK 偵測、連接、配置並擷取 LATTICE 相機及同步多相機陣列的影像；`.deb` 內含相機所需的 Arena SDK 執行環境
* **DAQ 光感測器控制** — 透過 `chloros-cli daq pool-*` 和 SDK 連接 DAQ-U/M/E 感測器、串流經校準的光譜，並記錄 `.daq` 檔案
* **PTP 時間同步** — Chloros 後端執行 PTP 主時鐘，LATTICE 相機與 DAQ-E 感測器皆以此為主時鐘進行同步； 可透過 `chloros-cli time-sync` 進行監控，並使用 `chloros-backend@<user>.service` systemd 單元以無頭模式維持其運作（參見 [Linux 安裝](linux-installation.md#always-on-ptp-for-headless-hosts))
* **專案自動化** — 透過 `chloros-cli project` 及 SDK 的 `open_project` 無互動模式執行已儲存的專案
* **GPU 加速** — 在 NVIDIA GPU 上進行 CUDA 加速處理（amd64 架構的桌面 GPU，以及搭配 JetPack 6 的 Jetson GPU）
* **動態運算適應** — 自動硬體偵測與處理策略選取，並提供 `CHLOROS_STRATEGY` 覆寫功能作為專家級的應急方案
* **所有處理功能** — 與 Windows 相同的處理流程：校正、暗角校正、植被指數，以及所有匯出格式
* **Chloros+ 功能** — 多執行緒（管線式）處理、Texture Aware 去拜爾演算法（於 CUDA GPU 上執行）以及自訂指數，需訂閱付費版 Chloros+ 方案

## Linux 使用者無法獲得的功能

* **桌面 GUI** — 無圖形介面；所有互動皆透過 CLI 或 Python SDK 進行
* **影像檢視器** — 無互動式影像檢視器、網格檢視或地圖標記
* **視覺化專案管理** — 專案需透過 CLI 指令與 SDK 呼叫來建立與執行（硬體本身 — 相機、感測器、擷取 — 仍可透過終端機完全控制）***

## 授權要求

CLI 和 SDK 存取需具備 **付費 Chloros+ 方案 — Copper 或更高階**（Copper、Bronze、Silver、Gold）。免費的**Iron** 方案不提供 CLI/SDK 存取權限。此限制由後端強制執行，不僅限於 CLI：

| 情況 | 後端回應 |
| --- | --- |
| 未登入 | `401` 搭配 `error_code: AUTH_REQUIRED` |
| 以免費的「鐵」級別登入 | `403` 搭配 `error_code: PLAN_UPGRADE_REQUIRED` |

`chloros-cli status` 適用於任何層級 — 這是唯一不受閘門限制的路由 — 因此拒絕的原因始終可見。

***

## 在 Linux

1. **安裝 Chloros** — 請參閱 [Linux 安裝指南](linux-installation.md) 以進行 `.deb` 的安裝
2. **驗證** — 執行 `chloros-cli --version` 會輸出 `Chloros CLI 1.3.0`；執行 `chloros-cli selftest` 則會執行 7 步驟診斷
3. **檢查 Python SDK** —— `.deb` 會安裝它：`python3 -c "import chloros_sdk; print(chloros_sdk.__version__)"`（若在 virtualenv 中，則為 `pip install /usr/lib/chloros/sdk/chloros_sdk-*.whl`）
4. **登入** — `chloros-cli login your@email.com 'your-password'`（每台機器僅需一次；套件升級會保留登入狀態，移除套件則會清除登入資訊）
5. **處理您的第一個資料集** — `chloros-cli process ~/datasets/flight001`

關於 NVIDIA Jetson，請參閱專屬的 [NVIDIA Jetson 指南](nvidia-jetson-guide.md)，以了解平台專屬的設定、熱管理行為及現場部署相關資訊。

***

## 下一步

* [Linux 安裝](linux-installation.md) — 針對 amd64 和 arm64 的詳細安裝步驟、檔案位置及疑難排解
* [NVIDIA Jetson 指南](nvidia-jetson-guide.md) — 針對 Jetson 的設定、記憶體與熱管理行為，以及現場部署
* [CLI：命令列](../CLI.md) — CLI 指南
* [API：Python SDK](../api-python-sdk.md) — SDK 指南
* [CLI 參考手冊](../reference/cli-reference.md) 與 [SDK 參考手冊](../reference/sdk-reference.md) — 1.3.0 版本的完整命令/API 清單
* [動態運算適應](../processing-architecture/dynamic-compute-adaptation.md) —— 說明 Chloros 如何適應您的硬體

{% hint style="info" %}
**透過程式化方式讀取本手冊。** 每頁內容亦以原始 Markdown 格式提供，其 URL 為該頁 URL 加上 `.md`（例如 `https://mapir.gitbook.io/chloros/linux/linux-installation.md`），且整本手冊的目錄已發佈於 [`https://mapir.gitbook.io/chloros/llms.txt`](https://mapir.gitbook.io/chloros/llms.txt)。
{% endhint %}
