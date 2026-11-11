# 動態運算適應

Chloros
1.3.0 採用硬體偵測與自動處理策略選取機制。處理引擎會根據您的硬體進行適應——從 Jetson Orin Nano 到多 GPU 工作站——無需任何手動設定。

***

## 運作原理

Chloros
啟動時，會對您的系統進行分析：

1. **偵測作業系統** — Windows
或 Linux
2. **識別 CPU 核心數與總記憶體容量**

3.**偵測 GPU 存在狀況** — NVIDIA CUDA 功能、VRAM、模型
4. **識別 Jetson 型號**（如適用）— 透過 `/proc/device-tree/model`
5. **檢查溫度感測器**（Jetson）— 用於溫度感知處理
6. **選擇運算策略** — 基於所有偵測到的硬體
7. **自動配置工作執行緒數、管線類型及記憶體分配**

偵測到的設定檔會於記憶體及磁碟中為該工作階段進行快取，因此後續執行時能更快啟動：

| 平台 | 快取設定檔 |
| --- | --- |
| **Linux
/ Jetson** | `~/.config/chloros/system_config.json`（優先採用 `XDG_CONFIG_HOME`） |
| **Windows** | `%LOCALAPPDATA%\Chloros\config\system_config.json` |

刪除該檔案可強制進行重新偵測 —— 此操作在新增 GPU 或增加記憶體後特別有用。當快取是由不相容的舊版本寫入時，Chloros
也會自動重新偵測。

***

## 運算策略

Chloros
會根據您的硬體，從三種運算策略中選取一種：

| 策略 | 選取條件 | 工作節點 | 執行器 | 處理管線 |
| --- | --- | --- | --- | --- |
| **`GPU_PARALLEL`**| CUDA GPU 報告**12GB+ VRAM**（在 Jetson 統一記憶體上，還需具備 12GB+ 總共享 RAM） | `min(4, ⌊(VRAM − 0.5GB) × 0.9 ÷ 3.5GB⌋)`，最少 2 —**在 Jetson 上上限為 2** | 工作程序（獨立的 CUDA 執行上下文） | `fused_gpu` |
| **`GPU_SINGLE`**| 具備**2-12GB VRAM**的 CUDA GPU | 3（I/O 重疊；GPU 存取由信號量串行化）。**在 RAM 低於 12GB 的 Jetson 上為 1（順序執行）** | 工作執行程序（獨立的 CUDA 上下文）；在低 RAM Jetson 上為順序的同程序執行 | `fused_gpu` / `tiled_gpu` |
| **`CPU_PARALLEL`** | 無 CUDA GPU，或 VRAM 低於 2GB | `max(2, physical cores − 1)` | `ThreadPoolExecutor` | `cpu_fallback` |

`GPU_PARALLEL` 工作節點公式的實際範例：12GB 視訊記憶體 → 2 個工作節點，16GB → 3 個工作節點，20GB+ → 4 個工作節點。 每個工作節點的預算為 3.5GB，此為「Texture Aware」任務的實際需求，且須控制在預留 0.5GB 後剩餘 VRAM 的 90% 範圍內。

{% hint style="info" %}
**GPU 策略需要能使用 CUDA 的 PyTorch。** 若機器上的 PyTorch 沒有可用的 CUDA 裝置，則無論其 GPU 型號為何，都會被分配為 `CPU_PARALLEL`。 1.3.0 版的 Jetson 套件採用 JetPack 6 的 CUDA 版本 PyTorch，因此只要安裝了 JetPack 的 CUDA 和 cuDNN 函式庫，Jetson 裝置即符合資格。RAM 低於 12GB 的 Jetson 裝置始終一次僅處理一張影像，且在同一執行程序中處理 （記錄為 `GPU_SINGLE`）。
{% endhint %}

GPU 策略會運行一組工作程序，每個工作程序皆為獨立的作業系統程序，並擁有專屬的 CUDA 執行上下文；Chloros
會透過再次啟動其已安裝的後端程式來啟動這些工作程序（而非透過 Python
的 `multiprocessing`，因為該工作程序的啟動機制在套件應用程式內部無法正常運作）。 CPU 策略則使用執行緒池（Python
的 `ThreadPoolExecutor`）。Chloros
未採用任何第三方分散式框架（例如 Ray）。

### 處理管線類型

* **`fused_gpu`** — 全 GPU 處理路徑。去拜爾演算、校正及索引運算在 GPU 上以單一融合處理階段執行。吞吐量最高，但需佔用最多 VRAM。
* **`tiled_gpu`** — 記憶體高效能的 GPU 處理路徑。將影像分割為圖塊進行處理，以適應有限的 GPU 記憶體。吞吐量較低，但適用於記憶體受限的裝置。
* **`cpu_fallback`** — 採用多執行緒並行處理的純 CPU 處理路徑。當無 NVIDIA GPU 可用時採用此路徑，並作為兩條 GPU 路徑皆失敗時的最終備用方案。

執行時備用鏈始終為 `fused_gpu` → `tiled_gpu` → `cpu_fallback`。

***

## 手動策略覆寫

設定 `CHLOROS_STRATEGY` 環境變數以強制採用特定策略——這是一項專家級的應急方案，適用於自動偵測選取的策略不適合您的情況時（例如，需將 GPU 保留給其他工作）：

```bash
# Valid values: CPU_PARALLEL, GPU_SINGLE, GPU_PARALLEL
CHLOROS_STRATEGY=CPU_PARALLEL chloros-cli process ~/datasets/flight001
```

此變數的比對不區分大小寫；凡非這三個名稱之一的值均會被忽略，自動偵測將照常進行。在覆寫狀態下，Chloros
仍會為您選擇工作節點數量：

| 覆寫值 | 使用的工節點數量 |
| --- | --- |
| `CPU_PARALLEL` | `max(2, physical cores − 1)` |
| `GPU_SINGLE` | 3 |
| `GPU_PARALLEL` | `min(4, physical cores)` |

若在沒有可用 CUDA 裝置的機器上進行 `GPU_SINGLE` 或 `GPU_PARALLEL` 的覆寫設定，系統會拒絕該設定，並在日誌中顯示 `[STRATEGY]` 這一行， 且執行時會使用 `CPU_PARALLEL`。

建議按指令個別設定，而非永久設定，以便正常執行時能持續自動適應。

***

## 平台特定行為

| 平台 | 策略 | 工作執行緒 | 管線 | 備註 |
| --- | --- | --- | --- | --- |
| **Jetson Orin Nano 8GB** | `GPU_SINGLE` | 1 | `tiled_gpu` (順序執行) | 記憶體高效模式，每次處理一個影像 |
| **Jetson Orin NX 8GB** | `GPU_SINGLE` | 1 | `tiled_gpu` (順序式) | 共享 RAM 低於 12GB 導致必須採用順序式處理 |
| **Jetson Orin NX 16GB** | `GPU_PARALLEL` | 2 | `fused_gpu`（並行） | 建議的邊緣裝置 — Jetson 限制為 2 個工作執行緒 |
| **Jetson AGX Orin 32-64GB** | `GPU_PARALLEL` | 2 | `fused_gpu` (並行) | 最高邊緣運算效能（Jetson 同樣限制為 2 個工作執行緒） |
| **配備 8GB GPU 的桌上型電腦** | `GPU_SINGLE` | 3 | `fused_gpu` / `tiled_gpu` | 3 個工作執行緒在 I/O 處理上重疊，同時由信號量對 GPU 存取進行串行化 |
| **配備 12GB+ GPU 的桌面電腦** | `GPU_PARALLEL` | 2-4 | `fused_gpu` (並行) | 最佳桌面效能：12GB → 2 個工作執行緒，16GB → 3 個，20GB+ → 4 個 |
| **純 CPU 系統** | `CPU_PARALLEL` | 實體核心數 − 1（至少 2 個） | `cpu_fallback` | 無需 GPU，使用執行緒池 |

{% hint style="info" %}
**Jetson 統一記憶體**：Jetson 裝置共享 GPU 與 CPU 記憶體。一台 16GB 的 Jetson Orin NX 報告的 VRAM 約為 15.3GB，但這與作業系統及 CPU 程序所使用的實體 RAM 相同。 這就是為什麼 16GB 以上的 Jetson 裝置能像 12GB 以上的桌上型 GPU 一樣符合 `GPU_PARALLEL` 的資格，卻僅限於 2 個工作執行緒——因為 GPU、工作執行緒及其每個工作執行緒的 CUDA 執行上下文，全都取用同一個共享記憶體池。
{% endhint %}

### 根據 VRAM 設定的 GPU 預算（獨立 GPU）

在配備獨立 NVIDIA GPU 的 x86_64 主機上，偵測到的 VRAM 也會決定該顯示卡可佔用的處理資源量，以及批次大小能擴增至何種程度：

| 偵測到的 VRAM | GPU 預算上限 | 批次大小倍數 |
| --- | --- | --- |
| **8GB+** | 90% | ×2.0 |
| **6-8GB** | 85% | ×1.75 |
| **

3.5-6GB** | 80% | ×1.5 |
| **2-3.5GB** | 75% | ×1.25 |
| **低於 2GB** | 70% | ×1.0 |

獨立 GPU 僅為系統預留 0.5GB 記憶體，因為它們不與系統 RAM 共用。Jetson 設定檔預留的記憶體遠多於此，且上限較低 — 請參閱 [NVIDIA Jetson 指南](../linux/nvidia-jetson-guide.md#per-model-gpu-budget)。

***

## 動態 GPU 記憶體分配

Chloros
採用 [4 執行緒處理管線](processing-pipeline.md)：

* **執行緒 1**（偵測）— 載入影像、解析 EXIF 資料、目標偵測
* **執行緒 2**（校正） — 反射率校正運算
* **執行緒 3**（處理）— GPU 去拜爾演算法、暗角校正、索引計算
* **執行緒 4**（匯出）— 檔案寫入、元資料嵌入

執行緒 1、2 和 4 對 GPU 的消耗較輕； 第 3 執行緒則是資源消耗最重的。隨著前期處理流程中的執行緒完成，其 GPU 資源配額將**重新分配給剩餘的活躍執行緒**，因此隨著執行進程推進，第 3 執行緒將獲得越來越多的記憶體。

### 分配階段

| 階段 | 活躍執行緒 | GPU 記憶體分配 |
| --- | --- | --- |
| **早期** | 1、2、3、4 | 分配給所有執行緒，其中大部分分配給執行緒 3 |
| **中期-早期** | 2、3、4 | 執行緒 1 的配額重新分配 |
| **中後期** | 3、4 | 執行緒 1 和 2 的配額轉移至 3 和 4 |
| **後期** | 3 或 4 | 最後一個活躍的執行緒將獲得其最大配額 |

這些數字受兩項規則規範：

* 若某執行緒是**唯一**的活躍執行緒，則會獲得其配置檔的最大分配量。
* 當有多個*高負載*的 GPU 任務處於活躍狀態時，每個高負載任務的基礎分配量將在它們之間分配（絕不會低於其設定的最低值）。

執行時實際使用的數值是平台設定檔的分配值與 GPU 記憶體監控器所提供的即時建議值中的 **較低者**，因此繁忙的顯示卡總是優先於過於樂觀的設定檔。***

## 紋理感知處理

「紋理感知」去拜爾演算法（**僅限 Chloros
+** — `--debayer texture-aware`）會執行一個 AI/ML 去噪模型，該模型每份副本約需 1.75GB 的 FP16 格式 VRAM，因此其 GPU 記憶體使用量遠高於標準方法：

* **VRAM 低於 7GB**的系統會以**同步迴圈、一次處理一張影像** 的方式執行「紋理感知」—— 無法容納多個模型副本，且工作池只會增加競態
* **VRAM 容量 7GB 以上** 的系統可並行處理「Texture Aware」，但相較於「標準」方法，工作執行緒數量會減少
* 在 **Jetson** 上，Texture Aware 始終被固定在單一工作執行緒上
* 若 **無可用 CUDA GPU**，執行時將改用標準去拜爾演算法，且後端日誌會記錄一行 `[DEBAYER] … needs a CUDA GPU …` 訊息說明原因
* 在低功耗 Jetson 裝置（Nano、Orin Nano）上，當您要求使用「Texture Aware」時，CLI
會對 GPU 頻率施加上限 — 請參閱 [NVIDIA Jetson 指南](../linux/nvidia-jetson-guide.md#gpu-frequency-cap-for-texture-aware-on-nano-and-orin-nano)

***

## 熱管理（Jetson）

Jetson 裝置存在熱限制，特別是在封閉式或機載部署環境中。Chloros
會監控 Jetson 的板載溫度感測器，並自動調整批次大小：

| 溫度 | 反應 |
| --- | --- |
| **&lt; 70°C** | 正常運作 — 全速 |
| **70°C** (警告) | 批次大小逐步縮減（70°C 至 80°C 之間從 100% 降至 50%） |
| **80°C**（危急）| 嚴格限速（80°C 至 90°C 之間從 50% 降至 0%）|
| **90°C**（關機）| 完全停止 GPU 處理 |

在散熱充足的桌面系統上，極少會觸發熱限速。

***

## 記憶體壓力處理

Chloros
在處理過程中會持續監控 GPU 記憶體，並依三種層級做出反應。

**批次大小設定。** 批次起始大小為 8 張影像乘以上表中的平台倍數。Chloros
接著檢查可用 VRAM，預留其中 20% 供 PyTorch 自身開銷使用，並假設每張 12MP 影像約佔用 100MB 的 GPU 記憶體——批次大小取記憶體推算出的上限與平台基礎值中的較小值。 批次大小絕不會低於 1。

**預先縮減。**當**VRAM 使用率超過 85%** 時，系統會在發生任何故障前自動縮減批次大小。**按執行緒分配的削減機制。** 隨著即時利用率攀升，每個執行緒的 GPU 配額將被縮減：利用率超過 80% 時縮減至 0.75 倍，超過 90% 時縮減至 0.5 倍。 監控閾值分別為 70%（保守值）、85%（正常運作上限）和 95%（記憶體不足風險）。**記憶體不足的退避與恢復機制。** 若仍發生記憶體不足事件：

* 批次大小將被**減半**，且每發生一次連續的 OOM 事件便再次減半——隨後每次成功的批次處理，該懲罰級別將回退一步
* 活躍執行緒的分配量將削減至當前值的 70%，且分配器將切換至保守策略；在連續成功分配後，該策略將再次放寬
* 在極度壓力下，處理管線會從 `fused_gpu` 降級至 `tiled_gpu`，並在萬不得已時降級至 `cpu_fallback`

**主機 RAM（Jetson）。** 在處理之前，CLI
會根據您的影像數量和去拜爾模式估算主機記憶體的峰值，若 RAM 加上檔案支援的交換空間可能不足，則會發出警告，並列印出新增交換空間的精確指令 — 請參閱 [NVIDIA Jetson 指南](../linux/nvidia-jetson-guide.md#swap-warning-and-recommendations)。

***

## 監控運算層適應狀況

### 系統診斷

`chloros-cli selftest` 是確認運算層所偵測狀態的最快速方式：

```bash
chloros-cli selftest
```

其 7 項檢查涵蓋版本、埠可用性、後端啟動狀態、`/api/test`、系統資訊、去噪器模型存在性，以及 CUDA 與去噪器的就緒狀態。檢查 5 會直接輸出硬體行：

```
      GPU: NVIDIA RTX A4000, CUDA: True, PyTorch: 2.7.0
```

檢查 7 會輸出 `CUDA: <bool>, Denoiser: <bool>` —— 這兩項都必須為真，Texture Aware 才可能被使用。

### 後端日誌

策略與工作執行緒數量是在每次執行開始時由後端內部選定的 — 並無CLI
標語會顯示這些資訊。當發生意外行為時（例如 GPU 路徑回退、記憶體不足 (OOM)、去噪器無法載入），相關資訊會顯示在該執行階段的後端日誌中：

| 平台 | 日誌位置 |
| --- | --- |
| **Linux
/ Jetson** | `~/.cache/chloros/logs/backend_<YYYYMMDD_HHMMSS>.log`（每次啟動一個檔案） |
| **Linux
，透過 CLI
啟動的後端** | 同樣為 `~/.chloros/backend.log` |
| **Windows** | `%LOCALAPPDATA%\Chloros\logs\` |

### 即時進度

執行期間，CLI
會透過 Server-Sent Events 串流顯示各執行緒的即時進度（偵測、分析、處理、匯出）——這可作為判斷執行緒 3 是否為瓶頸的實用參考。請參閱 [處理管線](processing-pipeline.md)。

***

## 後續步驟

* [處理管線](processing-pipeline.md) — 了解 4 執行緒管線架構
* [NVIDIA Jetson 指南](../linux/nvidia-jetson-guide.md) — 針對 Jetson 的部署與優化
* [CLI
：命令列](../CLI.md) — CLI
指南
* [CLI
參考手冊](../reference/cli-reference.md) — 1.3.0 版本的完整命令清單
