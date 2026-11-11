---
description: Frequently Asked Questions
metaLinks:
  alternates:
    - https://app.gitbook.com/s/o044KN3Ws0uIDvOmSkcR/faq
---

# 常見問題

<details>

<summary>我能否使用 Chloros 處理非 MAPIR 品牌的相機所拍攝的影像？</summary>

不行，Chloros 僅支援處理 MAPIR 相機的影像 — 即 Survey3 和 LATTICE 系列。 請參閱 [支援的相機型號清單](supported-cameras.md) 以獲取更多資訊。我們確實提供在 MAPIR Cloud 上處理其他相機影像的服務，完整清單請參閱 [此處](https://mapir.gitbook.io/mapir-cloud/supported-cameras)。

</details>

<details>

<summary>Chloros 是否支援 LATTICE 相機？</summary>

是的。Chloros 1.3.0 支援 LATTICE M3C 和 M3M 相機模組的端到端處理： **即時控制**— 可透過 GUI 的「相機」分頁（`chloros-cli lattice`）或 Python SDK 進行偵測、連線、預覽及擷取影像，包含透過 PTP 時間同步的同步多相機陣列 — 以及對擷取影像進行**完整的輻射度處理** （原始影像 → 去拜耳化 → 輻照度 → 反射率 → 指數）。請參閱 [支援的相機](supported-cameras.md) 及 [LATTICE 指南](lattice/README.md)。

</details>

<details>

<summary>我可以在沒有校準標靶的情況下，對影像進行反射率校準嗎？</summary>

**Survey3

：** 不行。若在拍攝非標靶影像時，未同步拍攝校準標靶的影像，您將無法將影像的像素值與已知的反射率百分比建立關聯。 若您亦未包含來自 MAPIR

光感測器的記錄檔，則無法測量環境光譜，反射率結果亦將不準確。

**LATTICE：** 可以。反射率可參照由 DAQ 光感測器（而非面板）測得的下行輻照度來計算（ρ = π·L/E）。 當畫面中確實存在通過品質保證（QA）的目標時，它預設會成為絕對參考（`--reflectance-source auto`）；除非該目標是帶有標記通過的，此時目標與 DAQ 鏈將根據測量分數相互競爭。 有一項例外：「F988 反射率是使用場景內的反射率面板進行校準的：該波段超出 DAQ 光感測器的校準範圍，因此 Chloros

會套用您最近一次的面板擷取數據，並在兩次面板觀測之間維持該值。」 請參閱 [校準目標](calibration-targets.md)。

</details>

<details>

<summary>我需要 DAQ 光感測器嗎？</summary>

若為輻度則不需要：LATTICE 輻度匯出資料源自各相機的出廠輻度校準，既不需要 DAQ 感測器，也不需要標靶。至於 **反射率**，您需要一個環境光的參考值——無論是 DAQ 光感測器的下行輻度測量值，或是畫面內的校準標靶。 透過 DAQ 感測器，您可**無需在場景中放置任何面板** 即可產生經校正的反射率。記錄的 `.daq` 檔案會根據時間戳記自動與您的影像配對。 請參閱 [校準標靶](calibration-targets.md) 及 [CLI

參考手冊](reference/cli-reference.md)。

</details>

<details>

<summary>我可以使用 AI 助理（如 Claude、ChatGPT 等）搭配 Chloros 嗎？</summary>

可以——本手冊以及 CLI/SDK 皆為此目的而設計：

* 完整手冊索引位於 `https://mapir.gitbook.io/chloros/llms.txt`，以便 AI 助理能檢索到每一頁。
* 每個頁面的原始 Markdown 檔案皆可透過該頁面的小寫 URL 加上 `.md` 來取得（例如 `https://mapir.gitbook.io/chloros/reference/cli-reference.md`）。
* [CLI 參考手冊](reference/cli-reference.md) 與 [SDK 參考手冊](reference/sdk-reference.md) 皆針對大型語言模型（LLM）的使用而撰寫：包含精確的參數、預設值、退出語義，以及可直接複製貼上的指令。

請參閱 [AI 助理](ai-assistants.md)，了解如何將您的助理指向 Chloros。

</details>

<details>

<summary>處理後的輸出檔案會儲存到哪裡？</summary>

產出檔案會寫入專案資料夾下，先按相機分類，再按檔案格式分組：

```
<project>/<camera-folder>/<format-folder>/<Product>_Images/
```

* **相機資料夾** — LATTICE 為 `LATT-<sensor>-<lens>-F<filter>`，Survey3 為 `<model>_<filter>`（例如 `Survey3N_RGN`）
* **格式資料夾** — `tiff16`、`tiff8`、`png8`、 `jpg8`，或 `tiff32`
* **產品資料夾** — `Reflectance_Calibrated_Images/`、`Debayered_Images/`、 `Preview_Images/`、`Radiance_Images/`（始終位於 `tiff32` 之下）、`<INDEX>_Index_Images/`
* **`clamp_masks/`** — 僅限 LATTICE 多光譜影像：產品資料夾旁附有逐像素遮罩，詳見下文**匯出檔案將保留原始檔案名稱 — 資料夾名稱用以識別產品，而非檔案名稱後綴。**使用CLI時，專案資料夾會建立在您的預設專案資料夾下（除非您已變更，否則為`Documents/MAPIR/Chloros Projects`），除非您傳入`-o`； 若傳入相對路徑 `-o`，則會以執行命令的資料夾為基準。 請注意，若執行 `chloros-cli process` 並請求產品但未寫入任何內容，系統會輸出 `Processing finished but wrote no image products.` 並**以非零狀態退出**，因此腳本可偵測此情況。 請參閱 [輸出影像格式](output-image-formats.md) 以及 [CLI 參考手冊](reference/cli-reference.md)。

</details>

<details>

<summary>什麼是 clamp_masks 資料夾和 *_clampmask.tif 檔案？</summary>

此為 1.3.0 版本的新功能。多光譜 LATTICE M3C 相機的輻照度與反射率產品會執行 3×3 波段解混，若某波段產生負值，Chloros 會將該值裁剪為零。 若產出中至少有一個像素被裁剪，則會產生一個單波段的 `uint8` 遮罩，用以標記哪些像素和波段被裁剪（位元 0 = 產出的 R 通道，位元 1 = G，位元 2 = B）。 遮罩會儲存於產品資料夾**旁邊**的 `clamp_masks/` 資料夾中 — 例如 `tiff16/clamp_masks/Reflectance_Calibrated_Images/<name>_clampmask.tif` ——而產品的 `Chloros:UnmixClampMask` XMP 標籤會標示其遮罩名稱（或標示為 `none`）。

它們並非場景的影像：Chloros 自身的匯入功能會跳過它們，而且由於它們從未位於產品資料夾內，因此您可以直接將產品資料夾傳遞給 Pix4D 或 Metashape。請參閱 [夾具遮罩](output-image-formats.md#clamp-masks)。

</details>

<details>

<summary>為什麼 Chloros 會顯示「此電腦的專案資料夾層級過深」？</summary>

此問題僅發生於 Windows 系統。Windows 預設關閉長路徑功能，因此任何檔案路徑長度不得超過 260 個字元。 Chloros 產品通常位於專案資料夾下方約 130 個字元處，因此 Chloros 會在處理前拒絕過深的專案資料夾 — 而不是像 1.3.0 版本之前那樣，執行後卻什麼都寫不出來。該訊息會顯示最長且符合要求的專案資料夾路徑。

解決方法如下：

* 使用較短的專案資料夾（在「專案設定」中的 **工作目錄**，或透過 CLI 指定 `-o` / `chloros-cli set-project-folder`）。
* 啟用 Windows 長路徑並重新啟動 Chloros。Chloros 安裝程式的 **啟用 Windows 長路徑** 選項（預設為勾選）會執行此設定；若您曾取消勾選，請重新執行安裝程式。

請參閱 [開始處理](processing-images-gui/starting-the-processing.md#the-project-folder-is-too-deep-for-this-computer)。

</details>

<details>

<summary>我在 Chloros 進行處理前，可以編輯影像嗎？</summary>

不可以。Chloros 假設輸入資料未經修改。請勿變更檔案名稱。

</details>

<details>

<summary>我能否將 MAPIR Survey3 相機設定為自動曝光，並在 Chloros 中處理影像？</summary>

不可以。Survey3影像資料集必須具有固定／鎖定的曝光參數，因此不得使用自動快門速度或自動 ISO。同一相機型號的所有影像必須具有相同的快門速度和 ISO（曝光）。

LATTICE 相機則無此限制：Chloros 會即時控制其曝光（Smart AE），且每次拍攝都會記錄實際使用的曝光值與增益，輻射計量處理流程會將此納入考量。

</details>

<details>

<summary>Chloros 能否處理或分析正射鑲嵌影像？</summary>

不行。僅支援單張MAPIR相機影像，不支援正射鑲嵌圖這類拼接影像。

</details>

<details>

<summary>如何加快 Chloros 的目標偵測步驟？</summary>

在檔案瀏覽器表格中，預先選取右側欄位中的目標影像，將指示 Chloros 僅在這些影像中搜尋校準目標，從而大幅加快處理速度。

</details>

<details>

<summary>如果我要將影像上傳至 <a href="https://www.mapir.camera/collections/software/products/mapir-cloud-subscription">MAPIR Cloud</a>，是否應在上傳前先在 Chloros 中進行處理？</summary>

若您計劃將影像上傳至我們的線上處理平台 [MAPIR

Cloud](https://www.mapir.camera/collections/software/products/mapir-cloud-subscription)，請勿在上傳前編輯影像。Cloud 將執行所有相同的處理步驟，甚至更多。

</details>

<details>

<summary>MAPIR 未來會支援 X 功能嗎？我真的很希望 MAPIR 能提供 X 功能。</summary>

我們始終樂於接收關於產品的回饋。若您發現產品問題，或對如何改進產品有任何建議，請 [聯絡我們](https://www.mapir.camera/community/contact) 分享您的想法。 我們的大部分研發工作，都是以傾聽客戶最迫切的需求為導向。

</details>

<details>

<summary>Chloros 是否支援 Linux？</summary>

是的！Chloros 1.3.0 透過 `.deb` 套件，支援 Linux amd64 (x86_64) 及 arm64 (NVIDIA Jetson JetPack 6)。 CLI 和 Python SDK 在 Linux 上均獲得完整支援，包括對 LATTICE 攝影機和 DAQ 感測器的即時控制。Linux 版本沒有 GUI — 所有互動皆透過 [CLI](CLI.md) 或 [Python SDK](api-python-sdk.md) 進行。 詳情請參閱 [Linux 概覽](linux/linux-overview.md)。

</details>

<details>

<summary>我可以在 NVIDIA Jetson 上執行 Chloros 嗎？</summary>

可以！Chloros 支援運行 JetPack 6 的 NVIDIA Jetson **Orin 系列** —— 包含 Orin Nano、Orin NX 及 AGX Orin。 較舊的型號（Jetson Nano、TX2、Xavier NX）無法執行 JetPack 6，且不受當前套件支援。Chloros 會自動偵測您的 Jetson 型號，並優化其處理策略。 Jetson 套件採用 JetPack 6 的 CUDA 版本 PyTorch，因此處理作業將在 Jetson 的 GPU 上執行。有關設定與部署說明，請參閱 [NVIDIA Jetson 指南](linux/nvidia-jetson-guide.md)。

</details>

<details>

<summary>Chloros 會自動針對我的硬體進行優化嗎？</summary>

是的！Chloros 內建 [動態運算適應](processing-architecture/dynamic-compute-adaptation.md) 功能，可自動偵測您的 CPU、GPU、RAM 以及（在 Jetson 上）溫度感測器。接著，它會選擇最佳的處理策略 ——從高記憶體 GPU 上的 `GPU_PARALLEL`，到較小規格 GPU 上的 `GPU_SINGLE`，再到不具備 PyTorch 可用的 CUDA GPU 的系統上所採用的 `CPU_PARALLEL`。 無需手動配置。

</details>

<details>

<summary>何謂 4 執行緒處理管線？</summary>

Chloros
為 Chloros
+ 使用者採用 4 執行緒管線式架構： 第 1 執行緒（偵測）負責載入影像並偵測校準目標；第 2 執行緒（校準）負責計算反射率校準；第 3 執行緒（處理）執行 GPU 加速的去拜耳化與指數計算；第 4 執行緒（匯出）則負責寫入輸出檔案。 多張影像可同時在不同執行緒中處理，以達到最大吞吐量。詳情請參閱 [處理管線](processing-architecture/processing-pipeline.md)。

</details>

<details>

<summary>如何對我的 Chloros 安裝執行診斷？</summary>

請使用 `selftest` 指令執行 7 步驟的煙霧測試：版本、埠可用性、後端啟動、API 連線狀態 (`/api/test`)、系統資訊 (`/api/system-info` — GPU/CUDA/PyTorch)、去噪器模型是否存在，以及 CUDA + 去噪器就緒狀態：

```bash
chloros-cli selftest
```

此功能在 Linux 系統上特別有用，可藉此驗證 GPU 和 CUDA 的設定。在 Jetson 裝置上，`CUDA: False` 通常表示 JetPack 6 的 CUDA 和 cuDNN 函式庫缺失：請使用 `sudo apt install nvidia-jetpack` 進行安裝。

</details>
