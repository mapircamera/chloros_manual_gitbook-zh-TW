---
metaLinks:
  alternates:
    - https://app.gitbook.com/s/o044KN3Ws0uIDvOmSkcR/download
---

# 下載

下載最新版本的 Chloros，開始進行多光譜影像處理。

### 系統需求

#### Windows


| 需求          | 最低需求                                              | 建議需求                                          |
| -------------------- | ---------------------------------------------------- | ---------------------------------------------------- |
| **作業系統** | Windows 10（64 位元）                                  | Windows 11（64 位元）                                  |
| **處理器**        | Intel Core i5 或同等規格                          | Intel Core i7 或更高規格                              |
| **記憶體 (RAM)**     | 8GB                                                  | 16GB 或以上                                         |
| **顯示卡**    | 相容於 DirectX 11                                | 配備 4GB 以上 VRAM 的 NVIDIA 顯示卡                            |
| **儲存空間**          | 6GB 可用空間                                       | 配備 10GB 以上可用空間的 SSD                            |
| **顯示螢幕**          | 1920x1080                                            | 2560x1440 或更高解析度                                  |
| **網路連線**         | 需用於 \[可選] Chloros+ 授權啟用          | 需用於 \[可選] Chloros+ 授權啟用          |

#### Linux amd64 (x86_64)

| 需求                        | 最低需求                    | 建議需求               |
| ----------------- | -------------------------- | ------------------------- |
| **發行版**  | Ubuntu 22.04 LTS 以上 / Debian 12 以上 | Ubuntu 24.04 LTS      |
| **處理器**     | x86_64 (Intel/AMD)        | Intel Core i7 或更高階   |
| **記憶體 (RAM)**  | 8GB                        | 16GB 或以上              |
| **顯示卡**        | 無 (由 CPU 處理)      | 配備 4GB+ VRAM 的 NVIDIA GPU |
| **儲存空間**       | 2GB 可用空間             | 10GB+ 可用空間的 SSD       |
| **Python**        | Python 3.7+（適用於 SDK）      | Python 3.10+              |

#### Linux arm64 (NVIDIA Jetson)

| 需求        | 最低需求                      | 建議需求                     |
| ---------------- | ---------------------------- | ------------------------------- |
| **平台**     | 搭載 JetPack 6 的 NVIDIA Jetson | Jetson Orin NX 16GB 或 AGX Orin |
| **記憶體 (RAM)** | 8GB（GPU/CPU 共用）         | 16GB+ 共用                    |
| **儲存空間**      | 2GB 可用空間               | 具備 10GB+ 可用空間的 NVMe SSD        |
| **Python**       | Python 3.7+（適用於 SDK）        | Python 3.10+                    |

{% hint style="info" %}
**GPU 加速**：配備 NVIDIA GPU 的 Chloros+ 使用者可利用 CUDA 加速，大幅提升處理速度。此功能適用於 Windows 和 Linux amd64 系統上的 NVIDIA 桌上型 GPU，以及搭載 JetPack 6 的 NVIDIA Jetson。Chloros+ 使用者還可透過多執行緒處理獲得最高速度。
{% endhint %}

***

## 下載 Chloros


### 最新穩定版：版本 1.3.0（2026 年 11 月 11 日）

### <a href="https://drive.google.com/uc?export=download&#x26;id=1HjwrUY4M7HGxDbMybO7iPe_6JoHnUGr4" class="button primary">下載適用於 Windows 的 Chloros（.exe）</a>


### <a href="https://drive.google.com/uc?export=download&#x26;id=1dB8-ke3wxNXpw_e1qJ4BhwBpCoNd4kLS" class="button primary">下載適用於 Linux amd64 的 Chloros（.deb）</a>


### <a href="https://drive.google.com/uc?export=download&#x26;id=1d1OwdcYA4Rf4jkuPi2IBeWT2772_HnyO" class="button primary">下載適用於 Linux arm64 / Jetson 的 Chloros (.deb)</a>


#### Windows 安裝程式（GUI + CLI + 後端）

* **檔案類型**：.exe（Windows 安裝程式）

**安裝步驟：**

1. 下載上方的 .exe 檔案
2. 雙擊安裝程式以開始安裝
3. 依照安裝精靈的提示操作
4. 選擇安裝目錄（預設：`C:\Program Files\Chloros\`）
5. 在 **「選擇其他任務」**頁面中，請保持**「系統設定」** 兩項選項皆勾選（建議 — 詳見下方）
6. 完成安裝並啟動 Chloros 或 Chloros CLI
7. 使用您的 [MAPIR Cloud Chloros+ 帳戶](https://cloud.mapir.camera/pricing) 登入 （或繼續使用免費版本）

**系統設定選項**（預設均已勾選）：

| 安裝程式中的選項名稱 | 功能說明 | 是否取消靜默安裝 |
| --- | --- | --- |
| **啟用 Windows 長路徑**（允許專案資料夾的路徑長度超過 260 個字元；建議啟用） | 將 Windows 的系統級設定 `LongPathsEnabled` 設為 1。Windows 預設為關閉此設定，此時任何檔案路徑長度不得超過 260 個字元；Chloros 產品會將專案資料夾置於路徑深處（約 130 個字元以下），因此若關閉長路徑功能，系統將拒絕更深層的專案資料夾。 此設定僅對安裝後啟動的程式生效。即使解除安裝後，此設定仍會保留，因為其他程式可能依賴此設定。 | `/MERGETASKS="!longpaths"` |
| **將 Chloros 後端排除於 Microsoft Defender 掃描範圍外**（可大幅加快攝影機擷取速度；建議啟用） | 在安裝資料夾中為 `chloros-backend.exe` 新增 Microsoft Defender 程序排除項目，使即時防護不會掃描後端寫入的每個檔案，無論其寫入位置為何。 在高擷取速率下，掃描所耗費的時間約佔擷取速率的 30%（在 9 台攝影機連續擷取的情況下，每秒擷取 77.7 次與 52.7 次相比）。 不會排除其他程式，且此排除設定會在解除安裝時移除。若系統未安裝 Defender 或其遭政策鎖定，此步驟將靜默失敗，安裝程序仍會繼續進行。 | `/MERGETASKS="!defenderexclusion"` |

若採用靜默安裝（`/SILENT` 或 `/VERYSILENT`），則會同時套用這兩項選項，除非您透過顯示的 `/MERGETASKS` 值選擇不採用。

{% hint style="success" %}
安裝程式會將CLI 資料夾 (`C:\Program Files\Chloros\cli\`) 新增至您的系統 `PATH` 一次 — 重新安裝或升級時將不再追加重複項目，並會移除先前安裝程式所新增的重複項目。 安裝完成後，請開啟新的終端機以使用 `chloros-cli`。安裝程式還會針對後端，在每個網路設定檔中新增兩項 Windows 防火牆規則： **Chloros 後端 (UDP-In)**，用於攝影機串流、PTP、mDNS 及 DAQ-E 資料串流；以及**Chloros 後端 (TCP-In)**，僅限於本機子網，用於 DAQ-E 韌體更新，此時感測器會連回這台電腦以擷取影像。 這兩項規則將在解除安裝時一併移除。
{% endhint %}

#### Linux amd64 (.deb 套件 — CLI + 後端)

* **檔案類型**：.deb（Debian/Ubuntu 套件）
* **架構**：x86\_64 (amd64)

```bash
sudo apt install ./chloros_1.3.0_amd64.deb
chloros-cli --version     # Verify installation: "Chloros CLI 1.3.0"
```

#### Linux arm64 — NVIDIA Jetson (.deb 套件 — CLI + 後端)

* **檔案類型**：.deb（JetPack 6）
* **架構**：aarch64（arm64）

```bash
sudo apt install ./chloros_1.3.0_arm64_jp6.deb
chloros-cli --version     # Verify installation: "Chloros CLI 1.3.0"
```

請使用您實際下載的檔案名稱，並保留 `./`，以便 `apt` 能直接安裝該檔案，而非根據名稱搜尋套件。請使用 `apt install` 進行安裝， 而非 `dpkg -i`：apt 還會一併安裝 Python SDK 的依賴項，這些是套件「建議」而非「要求」的，而 `dpkg -i` 會跳過這些安裝。 自 1.3.0 起，後端和 CLI 會以未解壓縮的狀態安裝在 `/usr/lib/chloros/` 下，因此它們啟動時無需先自行解壓縮。

Jetson 套件使用 JetPack 6 的 CUDA 版本 PyTorch，因此包含在內的 Texture Aware 去拜爾演算法（processing）會在 Jetson 的 GPU 上執行。 它使用 JetPack 6 安裝的 CUDA 和 cuDNN 函式庫（若映像檔中缺少這些函式庫，則為 `sudo apt install nvidia-jetpack`）。

請參閱 [Linux 安裝指南](linux/linux-installation.md) 以獲取詳細的設定說明，並參閱 [NVIDIA Jetson 指南](linux/nvidia-jetson-guide.md) 以獲取針對 Jetson 的特定指引。

#### Python SDK（所有平台）

每個安裝程式都會捆綁一個相應的 `chloros_sdk` wheel 檔案，因此 SDK 版本始終與已安裝的 GUI／CLI／後端相符。在 Windows 系統上，安裝程式會自動將其安裝至系統 Python 環境中。 在 Linux 系統上，`.deb` 會將 SDK 安裝至系統的 `python3` 本身（位於 `/usr/lib/python3/dist-packages` 中）， 因此 `import chloros_sdk` 在安裝完成後即可正常運作，無需 pip 亦無需網路連線。僅有虛擬環境（其無法存取系統套件）需要將捆綁的 wheel 安裝至其中：

```bash
pip install /usr/lib/chloros/sdk/chloros_sdk-*.whl
```

對於僅支援 pip 的主機（未安裝 Chloros 套件；此時 SDK 會驅動另一台機器上的後端），SDK 亦可在 PyPI 上取得：

```bash
pip install chloros-sdk
```

請參閱 [API：Python SDK](api-python-sdk.md) 及 [SDK 參考手冊](reference/sdk-reference.md) 以獲取相關文件。

{% hint style="info" %}
**Linux 使用者**：`.deb` 套件會安裝 CLI、後端及 Python SDK。Linux 版本沒有圖形介面（GUI）——所有互動皆透過 CLI 或 SDK 進行。
{% endhint %}

***

## 其他資源

### Python SDK


針對開發人員及自動化工作流程，每個安裝程式均附帶 Chloros Python SDK：Windows 安裝程式會將其安裝至您的 Python 環境中，而 Linux 版的 `.deb` 則會為系統安裝 `python3`。 若您的機器上未安裝 Chloros 套件（即該機器僅負責驅動另一台機器的後端），請從 PyPI 安裝：

```bash
pip install chloros-sdk
```

**文件**：[API：Python SDK](api-python-sdk.md)

**需求**：需有可驅動的 Chloros 後端（Windows 安裝程式或 Linux `.deb` 套件會安裝一個），以及付費方案的 Chloros+ 登入帳戶

***

## 包含內容

### Windows 安裝程式

* ✅ **Chloros GUI** - 功能齊全的圖形化介面
* ✅ **Chloros CLI** - 命令列介面（需具備 Chloros+ 授權）
* ✅ **Chloros 後端** - 處理引擎
* ✅ **Python SDK** — 若系統已安裝 Python，則會一併安裝（需具備 Chloros+ 授權）
* ✅ **相機設定檔** — 預先配置的 MAPIR 相機範本

### Linux.deb 套件

* ✅ **Chloros CLI** - 命令列介面（需具備 Chloros+ 授權）
* ✅ **Chloros 後端** - 處理引擎
* ✅ **Python SDK** - 針對系統 `python3` 進行安裝（需具備 Chloros+ 授權）
* ✅ **相機設定檔** — 預先配置的 MAPIR 相機範本
* ❌ 無 GUI — Linux 為無顯示介面的版本，僅支援 CLI/SDK


### Python SDK（PyPI，適用於未安裝 Chloros 套件的機器）

* ✅ **Chloros SDK** — Python API（需具備 Chloros+ 授權）

***

## 升級至 Chloros+

透過 Chloros+ 訂閱解鎖進階功能：

* 🚀 **多執行緒處理** — 並行處理影像
* ⚡ **GPU (CUDA) 加速** - 善用 NVIDIA GPU 運算能力
* 💻 **命令列介面 (CLI) 存取** - 透過命令列工具實現自動化
* 🐍 **Python SDK** - 程式化 API 存取
* 📱 **多裝置支援** - 可在 2 至 10 台以上裝置上使用（視方案而定）
* **🐻 進階紋理感知去拜爾演算法** — 結合 AI/ML 降噪模型的高品質邊緣感知去拜爾演算法，可消除幾乎所有去拜爾過程產生的噪聲。
* 🧮 **自訂公式** — 建立自訂多光譜指標

<p align="center"><a href="https://cloud.mapir.camera/pricing" class="button primary">查看 Chloros+ 方案與定價</a></p>***

## 安裝協助

### 疑難排解

**安裝失敗並顯示錯誤訊息：**

* 請確認您具備系統管理員權限
* 暫時停用防毒軟體
* 確認您的系統符合最低系統需求

**應用程式無法啟動（Windows）：**

* 確認已安裝 Windows 10/11（64 位元）
* 更新顯示卡驅動程式
* 查看 Windows 事件檢視器以了解錯誤詳情
* 附上錯誤日誌聯絡技術支援

**CLI 無法啟動（Linux）：**

* 確認 `.deb` 套件已正確安裝：`dpkg -l | grep chloros`
* 檢查權限：`sudo chmod +x /usr/bin/chloros-cli`
* 執行診斷：`chloros-cli selftest`
* 檢查是否缺少函式庫：`ldd /usr/lib/chloros/backend/chloros-backend | grep "not found"`（自 1.3.0 版本起，`/usr/lib/chloros/chloros-backend` 為一個小型啟動腳本；程式本身位於 `/usr/lib/chloros/backend/`）

**授權啟用問題：**

* 確保網路連線正常
* 請至 [https://cloud.mapir.camera](https://cloud.mapir.camera) 驗證憑證
* 確認防火牆未封鎖 Chloros

* 詳情請參閱 [Chloros+ 登入](chloros+-login.md)

### 取得支援

需要安裝或設定方面的協助嗎？

* 📧 **電子郵件**：info@mapir.camera
* 🌐 **網站**：[https://www.mapir.camera/community/contact](https://www.mapir.camera/community/contact)
* 📚 **文件**：[入門指南](./)
* ❓ **常見問題**：[常見問題](faq.md)***

## 軟體更新

Chloros 會檢查更新、在有新版本時發出通知，並提供此下載頁面的連結 — 您只需執行新的簽名安裝程式即可完成更新。您的設定、專案及登入資料在更新後仍會保留。 在 Linux 和 Jetson 系統上，`chloros-cli update` 會檢查是否有更新版本，並提供下載及安裝對應的 `.deb`（此指令僅限 Linux 系統）。

***

## 變更紀錄**版本 1.3.0（最新版，2026 年 11 月 11 日）**— 完整清單請參閱 [入門指南](./) 頁面中的**Chloros

1.3.0 新增功能**。**版本 1.2.0**— 請參閱 [入門指南](./) 頁面中的**Chloros 1.2.0 新增功能**。

<details>

<summary>版本 1.0.5</summary>

**發行日期：2026 年 2 月 10 日**

**新功能*** **紋理感知去拜爾演算法 \[僅限 Chloros+] -** 紋理感知功能採用高品質的邊緣感知去拜爾演算法，並結合 AI/ML 降噪模型，可消除幾乎所有的去拜爾噪點。
* **支援 T4P 校準標靶*** **Chloros+ GPU 處理速度更快，記憶體管理更佳**

**錯誤修正*** 全新前端介面（GUI），現應可在所有 Windows 電腦上正常運作。

</details>

<details>

<summary>版本 1.0.4</summary>

**發行日期：2026年1月5日**

**新功能*** **影像／元資料切換**：在檔案瀏覽器中新增切換功能，可將選取影像的元資料以表格形式顯示，而非影像網格
* **影像網格縮放滑桿**：新增 UI 滑桿，用於調整縮圖大小 （亦支援 CTRL + 滑鼠滾輪）
* **影像網格匯出按鈕**：頂部列中的按鈕，可將縮圖切換為經處理的匯出格式（目標值、反射率、指數、LUT）
* **地圖分頁**：新增互動式 2D 地圖，顯示影像的 GPS 位置標記
  * 支援 Google 地圖與 ESRI 地圖圖塊（會根據縮放層級的可用性自動選擇最佳圖塊服務）
  * 將滑鼠懸停於地圖標記上時，可預覽縮圖

**錯誤修正*** 改進了在非英文系統電腦上安裝 Chloros 的支援

</details>

<details>

<summary>版本 1.0.3</summary>

**發行日期：2025年12月20日**

**新功能*** 首次發行

**改進*** 首次發行

**錯誤修正*** 首次發行

**已知問題*** 首次發行

</details>

***

## 授權協議**專有軟體** - 版權所有 (c) 2026 MAPIR Inc.

禁止未經授權的使用、散佈或修改。

**免費版本**：可供個人及商業用途，但功能有限**Chloros+**：基於訂閱制的授權，提供進階功能及商業部署
