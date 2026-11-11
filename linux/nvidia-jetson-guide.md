# NVIDIA Jetson 指南NVIDIA Jetson 上的

Chloros
可在邊緣端（例如現場、無人機及遠端安裝環境）進行多光譜影像處理。Chloros
1.3.0 會在啟動時偵測您的 Jetson 機型，並根據偵測到的硬體優化其處理策略。**無需手動調整。**
{% hint style="info" %}
**

1.3.0 版 Jetson 套件在 Jetson 的 GPU 上進行處理。** `chloros_1.3.0_arm64_jp6.deb` 是為 JetPack 6 打造，並採用 JetPack 6 的 CUDA 版本 PyTorch，因此處理作業（包含「Texture Aware」去拜爾演算法）皆在 Orin 的 GPU 上執行。 此套件本身並不包含 CUDA：它使用 JetPack 6 所安裝的 CUDA 和 cuDNN 函式庫（若映像檔中缺少這些函式庫，則使用 `sudo apt install nvidia-jetpack`）。當無法使用 CUDA GPU 時，處理作業將在 CPU 上執行，且「Texture Aware」會使用標準去拜爾演算法。
{% endhint %}

***

## 支援的 Jetson 機型

| 機型                | 記憶體            | 處理策略                                     | 建議用途                                          |
| -------------------- | -------------- | ------------------------------------------------------- | -------------------------------------------------------- |
| **Jetson AGX Orin**  | 32-64GB 共用 | `GPU_PARALLEL`（2 個工作執行緒）                              | 最高效能，適用於大型資料集                      |
| **Jetson Orin NX**   | 8-16GB 共用    | `GPU_PARALLEL`（2 個工作節點，16GB） / `GPU_SINGLE`（8GB）   | 現場部署的首選方案              |
| **Jetson Orin Nano** | 8GB 共用     | `GPU_SINGLE` (1 個工作節點，順序執行)                     | 入門級邊緣運算                                 |

{% hint style="info" %}
Linux
arm64 套件需要 **JetPack 6**，Jetson Orin 系列已支援此版本。較舊的機型（Nano、TX2、Xavier NX）無法執行 JetPack 6，且不被當前套件所支援。
{% endhint %}

***

## 需求

* **JetPack 6.x**（建議使用最新版本）
* **NVIDIA CUDA 和 cuDNN**（來自 JetPack 6；若您的映像檔未包含，請安裝 `sudo apt install nvidia-jetpack`）
* **付費版 Chloros
+ 方案** — Copper 級別或更高（所有 CLI
/SDK
存取皆需此條件；由伺服器端強制執行）

## 安裝

```bash
# Install the arm64 .deb package (JetPack 6) -- apt, not dpkg -i, so the
# Python SDK's requirements (python3-numpy, python3-requests) come too
sudo apt install ./chloros_1.3.0_arm64_jp6.deb

# Verify installation
chloros-cli --version    # prints "Chloros CLI 1.3.0"
python3 -c "import chloros_sdk; print(chloros_sdk.__version__)"   # the .deb installs the SDK

# Run system diagnostics
chloros-cli selftest
```

`.deb` 會為系統 `python3` 安裝對應此建置版本的 Python
SDK
。 僅虛擬環境（virtualenv）需要將隨附的 wheel 檔案安裝其中：`pip install /usr/lib/chloros/sdk/chloros_sdk-*.whl`（完全離線）。有關一般 Linux
安裝詳情、檔案位置及疑難排解，請參閱 [Linux
安裝](linux-installation.md)。

{% hint style="info" %}
**啟動時不再進行解壓縮。** 自 1.3.0 起，後端與 CLI
會以未解壓縮的狀態安裝於 `/usr/lib/chloros/` 目錄下，因此不再於每次啟動時將自身解壓縮至臨時目錄。 若該資料夾存在，臨時檔案仍會自動儲存至 `/mnt/ssd/tmp`；否則，您可以將 `TMPDIR` 設定為 NVMe 上的路徑（`export TMPDIR=/mnt/nvme/tmp`）。
{% endhint %}

***

## Jetson 上的動態運算適應

### 運作原理

啟動時，Chloros
會對您的系統進行配置：

1. 透過 `/proc/device-tree/model` **偵測 Jetson 型號**

2.**讀取可用的共用 GPU/CPU 記憶體**（Jetson 採用統一記憶體）
3. **查詢 PyTorch 是否可使用 CUDA** — 若已安裝 JetPack 6 的 CUDA 和 cuDNN，則可使用
4. **自動選擇處理策略**（`GPU_PARALLEL`、`GPU_SINGLE` 或 `CPU_PARALLEL`）及工作節點數量

此決定取決於 **總共享 RAM 容量**，而非模型名稱：

* **總 RAM 容量低於 12GB**（所有 8GB Jetson 裝置）：**1 個工作執行緒 — 刻意採用序列處理**。由於記憶體過於緊繃，無法支援並行工作執行緒，因此影像將逐一處理。 在**8GB 或以下** 的 Jetson 裝置上，Thread 3 會完全跳過工作執行緒池，並在進程內執行每張影像的處理工作。後端日誌將此策略命名為 `GPU_SINGLE`。
* **12GB 或以上**（Orin NX 16GB、AGX Orin）：統一記憶體雖符合 `GPU_PARALLEL` 的條件，但**Jetson 上的工作執行緒數量上限為 2** — 由於 GPU、工作執行程序的 RAM 以及每個工作執行程序的 CUDA 執行上下文皆共用同一個記憶體池，因此增加工作執行程序數量將增加發生記憶體不足錯誤的風險。
* **無法使用 CUDA**（缺少 JetPack 的 CUDA 函式庫）：12GB 或以上時適用 `CPU_PARALLEL`，以下每次僅限一個映像檔。

您可以透過 `CHLOROS_STRATEGY` 環境變數覆寫自動選擇 — 請參閱 [動態運算適應](../processing-architecture/dynamic-compute-adaptation.md#manual-strategy-override)。

### 各模型的行為

| Jetson 模型                | 策略       | 工作節點數 | 執行方式                                      |
| --------------------------- | -------------- | ------- | ---------------------------------------------- |
| **Jetson Orin Nano 8GB**    | `GPU_SINGLE`   | 1       | 順序式進程內迴圈（在記憶體壓力下採用 `tiled_gpu`） |
| **Jetson Orin NX 8GB**      | `GPU_SINGLE`   | 1       | 序列化進程內迴圈                     |
| **Jetson Orin NX 16GB**     | `GPU_PARALLEL` | 2       | 並行工作程序，`fused_gpu` 路徑  |
| **Jetson AGX Orin 32-64GB** | `GPU_PARALLEL` | 2       | 並行工作程序，`fused_gpu` 路徑  |

各平台之間的關鍵差異在於 **記憶體**。當負載較高時，8GB 的 Jetson 必須採用節省記憶體的拼接方式逐張處理影像；而 16GB 以上的 Orin 則能透過高吞吐量的融合管線，讓 GPU 同時處理 2 張影像。

### 各模型的 GPU 預算

每個 Jetson 模型都附帶一個硬體配置檔，該配置檔會限制共享池處理可佔用的資源量，並調整批次大小：

| 模型 | GPU 預算上限 | 批次大小倍數 | 預留給系統／顯示器 |
| --- | --- | --- | --- |
| **Jetson Orin Nano** | 70% | ×0.8 | 2.0 GB |
| **Jetson Orin NX** | 75% | ×1.0 | 3.0 GB |
| **Jetson AGX Orin** | 80% | ×1.5 | 4.0 GB |

系統會根據偵測到的 RAM 容量調整設定檔：若 Jetson 報告的 RAM 容量為 **16GB 或以上**，其批次倍數將提高至 ×1.2。未經倍數調整前的基礎批次大小為 8 張影像。

完整的運算適應參考，請參閱 [動態運算適應](../processing-architecture/dynamic-compute-adaptation.md)。

***

## Nano 和 Orin Nano 上「Texture Aware」的 GPU 頻率上限

「Texture Aware」去拜爾演算法會執行 GPU 神經網路推論，這可能在低功耗 Jetson 機型（10-15W 級）以全速 GPU 時脈運行時觸發 **過流警告**。當您在**Jetson Nano 或 Orin Nano** 上傳入 `--debayer texture-aware` 時，CLI
會檢查 GPU 的最高頻率，若當前頻率高於此值，則將其限制在 **510 MHz** (510000000)：

* 若CLI
能寫入 GPU 頻率的 sysfs 節點，則會 **自動套用** 限制並顯示確認訊息。
* 若無法寫入（需 root 權限），CLI
會輸出確切的 `sudo` 指令供您手動設定上限，並稍作等待以便您閱讀，隨後繼續執行 — 處理程序仍會運行，但可能會顯示過電流警告。

若要在處理前自行套用上限：

```bash
echo 510000000 | sudo tee /sys/devices/platform/bus@0/17000000.gpu/devfreq/17000000.gpu/max_freq
```

高功耗機型（Orin NX 25W、AGX Orin 60W）會以全速運行 GPU；不會套用任何上限。標準去拜爾演算法在任何機型上都不會觸發上限。

{% hint style="info" %}
**在 Jetson 上，「Texture Aware」模式始終一次僅處理一張影像。** 每個工作執行緒都需要專屬的 CUDA 執行上下文（約 1GB）以及專屬的去噪模型副本，而統一記憶體無法負擔此需求——因此在 Jetson 上，「Texture Aware」路徑會被固定在單一工作執行緒上，並將 GPU 存取進行序列化。預期在任何 Jetson 裝置上，「Texture Aware」的執行速度都會明顯慢於「標準」模式。 「Texture Aware」適用於Survey3
影像：LATTICE擷取的影像始終會採用「Standard」去拜爾演算法。若無可用的CUDA GPU，執行時將使用「Standard」，且後端日誌會記錄一行`[DEBAYER] … needs a CUDA GPU …`說明原因（如`chloros-cli --verbose`所示）。
{% endhint %}

***

## 熱管理

Jetson 裝置的熱餘裕有限，特別是在封閉式或機載部署環境中。Chloros
會監控 SoC 溫度並自動調控批次大小：

| 溫度                 | 動作                                            |
| ------------------- | ------------------------------------------------- |
| **&lt; 70°C**          | 正常運作 — 全速處理          |
| **70°C** (警告)  | 批次大小逐步縮減（70°C 至 80°C 之間從 100% 降至 50%） |
| **80°C** (危急) | 嚴格限速（80°C 至 90°C 之間從 50% 降至 0%） |
| **90°C** (關機) | 完全停止 GPU 處理 — 需進行冷卻 |

{% hint style="warning" %}
為確保持續運算，**請確保有充足的通風與散熱**，特別是在密閉的現場機箱或機載系統中。為保護硬體，熱限速機制會降低處理吞吐量。
{% endhint %}

***

## 記憶體管理

Jetson 裝置採用 **統一記憶體** —— GPU 與 CPU 共用相同的實體 RAM。所顯示的 VRAM（例如 Orin NX 16GB 上的 ~15.3GB）並非專用的 GPU 記憶體；它是作業系統及其他所有程序所共用的同一塊 RAM。

### 交換空間警告與建議

在 Jetson 上進行處理之前，CLI
會統計輸入資料夾中的 RAW 影像數量 （`.tif`、`.tiff`、`.raw`、`.dng` — JPG 預覽圖不計入其中），估算執行所需的高峰記憶體用量，並在 **開始執行前發出警告**，若 RAM + 交換空間可能不足。該警告標題為 `LOW MEMORY WARNING - Jetson Detected`，會列出您的影像數量、 RAM、當前交換空間以及預估峰值，接著會提供針對 專案量身打造的具體命令（絕不小於 8GB）。系統會暫停數秒，以確保訊息不會因螢幕捲動而遺失，隨後處理程序將繼續進行。**警告所使用的記憶體估算值：**

| 去拜爾模式 | 基礎值 | 每張影像 |
| --- | --- | --- |
| 標準 | ~1.5 GB | ~10 MB |
| 紋理感知 | ~2.5 GB（模型 + Python 執行時環境） | ~15 MB |

當預估峰值超過「RAM + 交換空間」減去 1GB 安全餘裕時，此警告便會觸發，且僅計算 **基於檔案的** 交換空間 — 若僅使用 zram 的設定，仍會被標記。

若要手動新增交換空間（範例：8 GB）：

```bash
# Check current memory and swap
free -h

# Create a swap file
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make persistent across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

<!-- SCREENSHOT-NEEDED: Terminal on a Jetson Orin (SSH session) showing the full "LOW MEMORY WARNING - Jetson Detected" block printed by `chloros-cli process` on a large folder: the image count and debayer mode line, RAM / current swap / estimated peak figures, and the fallocate/chmod/mkswap/swapon command block it recommends -->


### OOM（記憶體不足）處理

處理過程中，Chloros
會監控 GPU 記憶體，並採取平穩降級措施而非直接當機：

1. 當 GPU 記憶體使用率超過 **85%** 時，會預先減少批次大小
2. 若仍發生記憶體不足事件，批次大小將 **減半**，且每次連續發生 OOM 時再減半一次；隨後每個成功的批次會將此懲罰措施回退一步
3. 若持續面臨記憶體壓力，處理管線將從 `fused_gpu` 切換至記憶體效率較高的 `tiled_gpu` 路徑，並在萬不得已時轉為 CPU 處理

由於 GPU 與 Jetson 共用 RAM，因此 RAM 加上交換空間仍會限制執行範圍——上述的交換空間警告即是需特別留意的檢查項目。

***

## 現場部署

### 電源考量

| Jetson 型號     | 典型功耗 | 備註                   |
| ---------------- | ------------------ | ----------------------- |
| Jetson Orin Nano | 7-15W              | 直流圓柱插頭          |
| Jetson Orin NX   | 10-25W             | 直流圓柱插頭          |
| Jetson AGX Orin  | 15-60W             | USB-C PD 或圓柱插頭 |

請針對持續運算規劃電源預算 — 峰值功耗會出現在 GPU 負載較重的第 3 執行緒（處理）期間。

### 儲存裝置建議

* **NVMe SSD** 強烈建議用於 arm64 部署
* SD 卡速度過慢，不適合用於處理 — 僅作為開機媒體使用
* 請預留 2 至 3 倍於原始影像資料大小的空間，以存放處理後的輸出結果

### 透過 SSH
進行無顯示器運作

Chloros
CLI
非常適合無顯示器的 Jetson 部署：

```bash
# SSH into the Jetson
ssh user@jetson-hostname

# Process a dataset
chloros-cli process /data/datasets/flight001 --format "TIFF (32-bit, Percent)"

# Monitor export progress
chloros-cli export-status
```

### LATTICE / DAQ-E 時間同步的常駐後端

若您的 Jetson 以無頭模式控制 LATTICE 攝影機或 DAQ-E 光感測器，請以 **您的使用者身分** 啟用後端 systemd 服務，以確保 PTP 主時鐘持續運行，並讓 SDK
腳本能偵測到正在運行的後端（該組件預設已安裝但未啟用）：

```bash
sudo systemctl enable --now chloros-backend@$USER.service    # $USER expands to your user name
chloros-cli time-sync status
```

此後，後端將以您的身分運行：其寫入的擷取資料屬於您，且會使用您的 `chloros-cli login`。 詳情請參閱 [Linux
安裝指南](linux-installation.md#always-on-ptp-for-headless-hosts)，內容包含 root 權限的 `chloros-backend.service`，以及套件如何讓 PTP 埠 319/320 無需 root 權限即可綁定。

### 透過 systemd 進行自動化處理

建立一個 systemd 服務以進行自動化處理：

```ini
# /etc/systemd/system/chloros-process.service
[Unit]
Description=Chloros Automated Processing
After=network.target

[Service]
Type=oneshot
User=chloros
ExecStart=/usr/bin/chloros-cli process /data/incoming --output /data/processed
StandardOutput=append:/var/log/chloros-process.log
StandardError=append:/var/log/chloros-process.log

[Install]
WantedBy=multi-user.target
```

當某次執行請求產出結果卻未寫入任何影像時，`chloros-cli process` 會以非零狀態碼退出，因此 systemd 的失敗狀態對監控具有實質意義。

搭配 systemd 定時器進行排程處理：

```ini
# /etc/systemd/system/chloros-process.timer
[Unit]
Description=Run Chloros Processing Every Hour

[Timer]
OnCalendar=hourly
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable chloros-process.timer
sudo systemctl start chloros-process.timer
```

***

## 工作流程範例

### 基本 Jetson 處理

```bash
#!/bin/bash
# Process a drone flight dataset on Jetson
chloros-cli process /data/flights/flight_042 \
    --output /data/processed/flight_042 \
    --format "TIFF (32-bit, Percent)" \
    --indices NDVI GNDVI OSAVI
```

### Jetson 上的 Python
SDK


```python
from chloros_sdk import ChlorosLocal

with ChlorosLocal() as chloros:
    chloros.create_project("field_survey_042")
    chloros.import_images("/data/flights/flight_042")
    chloros.configure(
        indices=["NDVI", "GNDVI", "OSAVI"],
        export_format="TIFF (32-bit, Percent)",
        reflectance_calibration=True
    )
    chloros.process(mode="parallel")

print("Processing complete!")
```

### 批次處理多趟飛行任務

```bash
#!/bin/bash
# Process all flight datasets in a directory
for flight in /data/flights/*/; do
    name=$(basename "$flight")
    echo "Processing $name..."
    chloros-cli process "$flight" \
        --output "/data/processed/$name" \
        --format "TIFF (32-bit, Percent)" \
        --indices NDVI OSAVI
    echo "Completed $name"
done
```

***

## 建議用於現場應用的 Jetson 系統

針對現場及機載部署，請考慮以下 Jetson Orin NX 16GB 載板選項：

* **機載／無人機**：具備抗震等級（MIL-STD）、輕量化（300 克以下）及被動散熱的系統
* **嚴苛野外環境**：配備 IP67/IP69K 防水外殼，並支援 PoE GigE 攝影機連接
* **精簡/經濟型**：附帶擴充外殼的開發套件

如需針對您的部署情境獲得具體硬體建議，請聯絡 [MAPIR
技術支援](https://www.mapir.camera/community/contact)。

***

## 後續步驟

* [Linux
安裝](linux-installation.md) — Linux
安裝的一般詳細資訊
* [動態運算適應](../processing-architecture/dynamic-compute-adaptation.md) — 完整的運算策略參考
* [處理管線](../processing-architecture/processing-pipeline.md) — 了解 4 執行緒管線
* [CLI：命令列](../CLI.md) — CLI
指南
* [API
：Python
SDK
](../api-python-sdk.md) — SDK
指南
* [CLI
參考](../reference/cli-reference.md) 與 [SDK
參考](../reference/sdk-reference.md) — 1.3.0 版本的完整命令／API
清單
