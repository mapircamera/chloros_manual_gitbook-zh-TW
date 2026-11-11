# 處理管線

Chloros 1.3.0 採用 4 執行緒的處理管線，運作方式類似分階段的組裝線。每個執行緒負責工作流程中的不同階段，因此可同時有多張影像處於不同階段的處理過程中。

<figure><img src="../.gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>

***

## 處理管線架構

```

Images In → [Thread 1: Detection] → [Thread 2: Calibration] → [Thread 3: Processing] → [Thread 4: Export] → Files Out
```

每張影像都會依序流經這四個執行緒。透過 Chloros+ 的多執行緒處理機制，多張影像可同時佔用不同的執行緒——當執行緒 3 正在處理一張影像時，執行緒 1 可以正在檢測下一張，執行緒 2 正在校準另一張，而執行緒 4 則將已處理完成的影像寫入磁碟。

各執行緒的進度會分別回報，並透過「伺服器發送事件」（Server-Sent Events）串流傳輸（後端會將其發佈至 `/api/events`）。在CLI的即時進度顯示中，這四個階段分別標示為 **偵測、分析、處理、匯出**。***

## 執行緒詳細資訊

### 執行緒 1：偵測

**目的**：載入影像並偵測校準目標。

* 從磁碟讀取影像檔案 — Survey3 的 `.raw`+`.jpg` 配對檔， LATTICE `.tif`/`.tiff` 擷取影像，以及 `.dng`
* 擷取 EXIF 元資料（GPS、相機型號、時間戳記、曝光時間）
* 偵測校準標靶：針對 LATTICE 擷取影像的 ArUco 標記標靶幾何形狀，以及針對 Survey3 校準標靶照片的經典面板偵測器
* 輸出：影像資料 + 元資料 + 標靶偵測結果

主要為 I/O 和 CPU 密集型執行緒。

### 執行緒 2：校正

**目的**：根據偵測到的標靶計算校正參數。

* 根據標靶影像計算反射率校正係數
* 計算暗角校正參數
* 確定各波段的校正曲線
* 輸出：每張影像的校正參數

此為受 CPU 限制的運算執行緒。在 Survey3 或混合執行任務中，若啟用反射率校正，執行緒 3 會等待此執行緒完成，以確保在處理任何影像前，其係數已準備就緒。在僅執行 LATTICE 的任務中，執行緒 3 會在偵測期間運行：每幀影像會在其目標錨點確定後開始處理。 每個幀會選取時間上最接近的目標（無論在前後），因此會等待直到該相機中所有時間點早於當前最近目標的幀皆已掃描完畢。產出結果相同，只是更早產生。

### 執行緒 3：處理（GPU）

**目的**：套用校正並計算植被指數。**這是運算需求最龐大的執行緒。*** **去拜耳化**：將 RAW 拜耳資料轉換為多通道影像
  * 標準模式（快速、中等品質）— 預設，`--debayer standard`
  * 紋理感知模式（緩慢、最高品質）— 僅限 Chloros+，`--debayer texture-aware`，採用 AI/ML 去噪模型
  * LATTICE 單色 (M3M) 擷取為單波段：其去馬賽克與白平衡步驟將被跳過（並會顯示一行日誌訊息），而同一執行程序中的任何 M3C/拜耳影像仍會進行這些步驟
* **暗角校正**：對整個影像套用鏡頭暗角校正
* **反射率校正**：套用校正係數以轉換為反射率值
* **指數計算**：計算植被指數（NDVI、NDRE、GNDVI 等）
* 輸出：已處理且可直接匯出的影像資料

此執行緒最能受益於 GPU 加速，且正是 [動態運算適應](dynamic-compute-adaptation.md) 所調校的執行緒。

### 執行緒 4：匯出

**目的**：將處理後的影像寫入磁碟。

* 以選定的格式寫入輸出檔案 — `TIFF (16-bit)`、`TIFF (8-bit)`、 `TIFF (32-bit, Percent)`、`PNG (8-bit)`、`JPG (8-bit)`
* 在輸出檔案中嵌入元資料（GPS、時間戳記、處理參數）——包括 `Chloros:*` 處理標籤，自 1.3.0 版本起，此標籤會在所有處理模式下寫入，無論是 Survey3 還是 LATTICE 皆然
* 將輸出檔案組織於專案資料夾下，命名格式為 `<camera>/<format>/<Product>_Images/` — 例如 `LATT-M3M-L41-F550/tiff16/Reflectance_Calibrated_Images/`。**匯出檔案保留原始檔案名稱；資料夾名稱則標示產品類型。**
* 對於 LATTICE 擷取資料，一個原始幀可拆分為多個產出檔（去拜耳化、預覽、輻照度、反射率、索引），每個產出檔皆存放於獨立的產出檔資料夾中。索引產出檔是根據該執行程序所產生的反射率產出檔計算得出（若無反射率產出檔，則取自該執行程序產生的其他產出檔，例如輻照度）。 多光譜 M3C 相機的輻照度與反射率產出檔，亦可在 `<format>/clamp_masks/` 中取得每像素的夾緊遮罩（參見 [輸出影像格式](../output-image-formats.md#clamp-masks))
* 輸出：磁碟上的最終檔案

主要為 I/O 受限的執行緒 — 使用 SSD 儲存裝置可顯著提升效能。

***

## 幕後運作：執行器

在執行緒 3 中，每張影像的處理工作會依照以下方式進行平行化：

* **GPU 策略**（`GPU_SINGLE`、`GPU_PARALLEL`）使用一組**工作程序**，每個工作程序皆為獨立的作業系統程序，並擁有專屬的 CUDA 執行上下文。 Chloros 透過再次啟動其已安裝的後端程式來啟動每個工作程序——而非透過 Python 的 `multiprocessing`，因為該功能在封裝的應用程式內部無法正常啟動工作程序
* **`CPU_PARALLEL`** 使用執行緒池（Python 的 `ThreadPoolExecutor`）——由於 NumPy 和 OpenCV 會釋放 GIL，因此僅需執行緒即可
* 共享 RAM 為 8GB 或以下的 Jetson 裝置會完全跳過執行器，並在同一個進程中依序處理
* 在 VRAM 低於 7GB 的 GPU 上，Texture Aware 也會依序執行 — 去噪模型無法進行多次擬合

Chloros 未使用任何第三方分散式框架（例如 Ray）。 有關策略與工作節點數量的選擇方式，請參閱 [動態運算適應](dynamic-compute-adaptation.md)。

***

## 順序處理與管線化處理

### 自由模式（順序處理）

在 Chloros 的免費版本中，影像會 **一次處理一張**，依序經過所有四個階段：

```

Image 1: [Detect] → [Calibrate] → [Process] → [Export]
                                                         Image 2: [Detect] → [Calibrate] → [Process] → [Export]
```

在免費模式下，GUI 會顯示一個簡化的進度條；其串行階段會顯示為 **目標偵測**，接著是**處理**。

### Chloros+ 模式（管線式）

若擁有 Chloros+ 授權，所有四個執行緒將針對不同的影像 **並行** 運作：

```

Thread 1: [Image 1] [Image 2] [Image 3] [Image 4] ...
Thread 2:           [Image 1] [Image 2] [Image 3] ...
Thread 3:                     [Image 1] [Image 2] ...
Thread 4:                               [Image 1] ...
```

GUI 進度條會顯示這四個階段；將游標懸停其上即可查看各執行緒的進度。在CLI 中，這四個階段會以 **偵測、分析、處理、匯出** 的形式即時串流顯示。

{% hint style="info" %}
**同一標籤，兩種名稱。** CLI將第三階段稱為 _處理_。後端的高階模式進度串流（即 GUI 進度條所顯示的內容）則將同一階段標示為 _校準_。這兩者實為同一執行緒執行相同的工作（執行緒 3：去拜爾濾波、校正、索引）。
{% endhint %}

{% hint style="success" %}
**使用 Chloros+ 進行管線化處理** 的速度可比順序處理快 3 至 5 倍，具體取決於您的硬體配置與資料集大小。 在配備高速 GPU 和 SSD 的系統上，速度提升最為顯著。
{% endhint %}

***

## 執行緒 4 匯出進度

匯出執行緒擁有獨立的進度追蹤機制，您可以單獨查詢其狀態：

**命令列介面 (CLI)：**

```bash
chloros-cli export-status
```

**軟體開發套件 (SDK)：**

```python
status = chloros.get_status()
print(f"Export: {status['export']['percent']}% - Phase: {status['export']['phase']}")
```

當第 4 執行緒達到 100% 時，即表示處理完成。

{% hint style="info" %}
**若執行過程未寫入任何影像，即視為失敗。**成功時，`chloros-cli process` 會回報已寫入的影像產品數量（`Image products written: N`）。 若已請求產品但**未**寫入任何產品——僅有 `project.json` 和 `calibration_data.json`——CLI 會輸出 `Processing finished but wrote no image products.` 並**以非零狀態退出**， 並會列出專案資料夾名稱及常見原因（例如輸入資料夾未被識別為擷取來源——請檢查佈局及 `--input-level`——或所有請求的產品均不適用於該批攝影機）。腳本可依據此退出代碼進行判斷。
{% endhint %}

***

## 與動態運算適應的關聯

[動態運算適應](dynamic-compute-adaptation.md) 主要影響 **第 3 執行緒（處理）**：

* **`GPU_PARALLEL`**：執行緒 3 透過 `fused_gpu` 管線，同時將多張影像傳送至 GPU 處理
* **`GPU_SINGLE`**：執行緒 3 透過信號量將 GPU 存取序列化，同時工作程序重疊執行 I/O，並使用 `fused_gpu` 或記憶體效率較高的 `tiled_gpu` 管線
* **`CPU_PARALLEL`**：第 3 個執行緒採用基於 CPU 的處理方式，並運用多執行緒並行處理

隨著執行緒 1 和 2 完成執行，執行緒 3 的 GPU 記憶體分配也會隨之增加 — 請參閱 [動態 GPU 記憶體分配](dynamic-compute-adaptation.md#dynamic-gpu-memory-allocation)。

***

## 下一步

* [動態運算適應](dynamic-compute-adaptation.md) —— Chloros 如何為您的硬體選擇最佳策略
* [NVIDIA Jetson 指南](../linux/nvidia-jetson-guide.md) —— Jetson 平台上管線的特定行為
* [處理進度監控](../processing-images-gui/monitoring-the-processing.md) — 透過 GUI 監控進度
* [命令列介面 CLI](../reference/cli-reference.md) 參考資料 — `process`、`export-status`、結束代碼及輸出佈局
