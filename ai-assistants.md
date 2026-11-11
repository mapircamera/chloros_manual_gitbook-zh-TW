# 搭配 AI 助理使用 Chloros

本手冊的撰寫對象有兩類：人類，以及人類日益倚賴的 AI 助理。每頁皆提供精確數值、預設值及可直接複製貼上的指令，以便助理（例如 Claude、ChatGPT、Copilot、程式設計代理等）能一次成功撰寫可運作的 Chloros 自動化腳本。

Chloros 版本：**

1.3.0**。CLI/SDK 平台：Windows 10/11 x64 及 Linux（x86_64 / Jetson aarch64）。

## 該提供什麼給您的助理

| 資源 | URL | 用途 |
| --- | --- | --- |
| **llms.txt** | `https://mapir.gitbook.io/chloros/llms.txt` | 本手冊中每頁的機器可讀索引。 |
| **命令列介面（CLI）參考手冊** | `https://mapir.gitbook.io/chloros/reference/cli-reference` | 完整的 `chloros-cli` 命令介面：包含所有命令、參數、預設值、退出代碼及輸出資料夾規則。專為大型語言模型（LLM）處理而撰寫。 |
| **SDK 參考** | `https://mapir.gitbook.io/chloros/reference/sdk-reference` | 完整的 `chloros_sdk` Python API：類別、簽名、例外情況及實作範例。專為大型語言模型（LLM）處理而撰寫。 |
| **將任何頁面轉為原始 Markdown 格式** | 在頁面 URL 後方附加 `.md` | 例如：`https://mapir.gitbook.io/chloros/reference/sdk-reference.md` 會將頁面以原始 Markdown 格式返回 — 非常適合貼入上下文視窗或由代理程式擷取。 |

手冊內連結：[CLI 參考](reference/cli-reference.md) · [SDK 參考](reference/sdk-reference.md)。

{% hint style="info" %}
這兩頁參考文件內容自成一體：已閱讀其中一頁的助理，無需參閱手冊其餘部分即可撰寫正確的腳本。
{% endhint %}

## 讓您的助理直接執行 Chloros：`chloros-mcp`

上述所有內容皆要求您的助理 *撰寫* 腳本或指令。 透過 `chloros-mcp`，任何能支援 [模型上下文協定](https://modelcontextprotocol.io)（MCP）的助手（例如 Claude Desktop、Claude Code 或 Cursor），都能直接驅動 Chloros 本身。 您只需以自然語言下達指令，它便會調用伺服器的工具：

* **影像處理：** 建立專案、匯入擷取影像的資料夾、選擇設定、進行處理，並回報執行結果。
* **操作硬體：** 連接 LATTICE 相機、陣列及 DAQ 光感測器，預覽畫面、設定曝光與色彩、進行擷取、錄製，並更新韌體。

```text
You:  Process the flight in D:\Flights\2026-10-17 to reflectance and NDVI.

You:  Show me what the array sees, then capture reflectance every 10 seconds
      for 5 minutes.
```

伺服器的工具涵蓋了 [Python SDK](api-python-sdk.md) 的功能。 這些工具會隨 SDK 一併安裝，因此通常無需額外安裝：在 Windows 系統上，Chloros 安裝程式會將 `chloros-mcp.exe` 放置於您的 Python 的 `Scripts` 資料夾中， 而 Linux 的 `.deb` 則會將 `chloros-mcp` 放置於 `/usr/bin` 中。

**Claude 程式碼：**

```bash
claude mcp add chloros -- chloros-mcp
```

**Claude Desktop：**將此內容新增至 `claude_desktop_config.json`（透過**設定 > 開發者 > 編輯設定**可開啟該檔案），然後重新啟動 Claude Desktop。**游標** 則在 `~/.cursor/mcp.json` 中採用相同的設定項。

```json
{
  "mcpServers": {
    "chloros": {"command": "chloros-mcp"}
  }
}
```

若客戶端顯示伺服器無法啟動，可能是因為 `chloros-mcp` 未出現在其所偵測到的 `PATH` 中： 請改為提供完整路徑（Windows 系統為 `where chloros-mcp`，Linux 系統為 `which chloros-mcp`）。

您需要在同一台電腦上安裝 Chloros，並透過應用程式或 `chloros-cli login` 登入任何付費的 Chloros+ 方案（Copper 或更高階）。伺服器無法代您登入。

{% hint style="info" %}
各應用程式、選項、每項工具的客戶端設定，以及執行行為說明：請參閱 SDK 參考文件中的 [MCP 伺服器 (AI 助理)](reference/sdk-reference.md#mcp-server-ai-assistants)。
{% endhint %}

## 提示語範本

複製、填入 `<placeholders>`，並貼上至您的助理中。

### 1. 將飛行資料夾處理為 NDVI

```

Read https://mapir.gitbook.io/chloros/reference/cli-reference.md.
Then write a script for <Windows PowerShell | bash> that:
1. logs in with `chloros-cli login <email> '<password>'` (only needed once per machine),
2. processes the folder <path/to/flight_001> with reflectance and the NDVI index,
3. prints where each output product landed, using the reference's
   "Where the outputs land" folder rules.
```

### 2. 批次監控擷取目錄

```

Read https://mapir.gitbook.io/chloros/reference/sdk-reference.md (sections
"Quickstart" and "Post-Run Summary & Hints"). Write a Python script that
watches <path/to/captures> for new flight subfolders and runs
chloros_sdk.process_folder() with indices=["NDVI"] on each new one.
After each run, print every hint from result["summary"]["hints"] and treat
a run with zero image products as a failure for that folder.
```

### 3. 連接 LATTICE 陣列並進行擷取

```

Read https://mapir.gitbook.io/chloros/reference/sdk-reference.md (section
"connect_array"). Write a Python script that connects my LATTICE cameras
with serials <213800234, 214000533, ...> as one synchronized array, captures
a reflectance image set into <output/> every 10 seconds for one hour, and
disconnects cleanly when done (use the context-manager form).
```

### 4. 記錄 DAQ 光感測器的光譜

```

Read https://mapir.gitbook.io/chloros/reference/cli-reference.md (section
"chloros-cli daq" — use only the pool-* commands). Write a script that:
1. connects my DAQ-E sensor with `chloros-cli daq pool-connect --eth-host <daq-e-xxxxxx.local>`,
2. lists the pool with `pool-list` to get the sensor id,
3. records a 10-minute calibrated .daq file named "<field-A>" with `pool-record`,
4. disconnects with `pool-disconnect`.
```

{% hint style="warning" %}
從命令列執行 DAQ 腳本時，總是會透過 `daq pool-*` 系列指令（`pool-connect`、`pool-list`、`pool-latest`、 `pool-stream`、`pool-record`、`pool-set-cap`、`pool-disconnect`）。 您的助理可能發明的其他 `daq` 子指令，在正式發行版本中並不可用，執行時會因錯誤而終止。
{% endhint %}

## 為何 AI 撰寫的腳本能與 Chloros 完美配合

這些皆為 Chloros 1.3.0 經實測驗證的真實行為——它們消除了機器撰寫自動化腳本常見的經典失敗模式：

* **無需繁瑣的設定流程。**SDK 的智慧連線輔助程式（`connect_camera`、`connect_array`、`connect_daq_sensor`）以及處理入口點 (`ChlorosLocal`、`process_folder`) 會**自動啟動本地後端**。生成的腳本無需開啟 GUI 或手動啟動伺服器——只需安裝桌面版／CLI 套件即可。
* **整個處理流程僅需一次呼叫。** `chloros_sdk.process_folder("path", indices=["NDVI"])` 會端到端執行匯入 → 校正 → 反射率 → 折射率匯出。流程更簡潔，生成的腳本出錯的可能性也更低。
* **無輸出執行會進行自我診斷。** `process()` 執行完畢後，執行摘要會附加於結果中，且每項處理提示（例如 *為何*運行未產生輸出）也會以Python `UserWarning` 格式重新輸出——因此，即使是從不檢查結果字典的腳本，也能顯示診斷資訊。
* **CLI 會明確報錯。**若某次 `chloros-cli process` 執行請求了產出但未寫入任何內容，系統會輸出 `Processing finished but wrote no image products.` 並**以非零狀態退出**，因此 shell 腳本和 CI 只需透過簡單的退出代碼檢查即可偵測到此狀況。 成功執行的任務會回報 `Image products written: N`，若寫入遮罩檔案則會額外回報 `(plus M clamp masks)`。擷取指令同樣會列出所有無法寫入的檔案名稱，並以 `1` 退出。
* **相對路徑的解析方式取決於腳本的預期行為。** 若使用本機後端，相對輸出路徑（CLI 中為 `-o out/`，SDK 中為 `output_dir="out/"`）將以命令或腳本執行的資料夾為基準進行解析。

助理應知曉的一項差異：SDK 的 `process()` 在無產出結果的執行中會刻意 **不** 拋出異常 — 而是透過摘要/提示進行回報。 若 Python 管線在執行結果為空時必須停止，請檢查摘要（食譜 2 會這麼做）。

## 注意事項

* **需具備 Chloros+ 登入權限。**CLI 和 SDK 需要**付費** 的 Chloros+ 方案，此限制由伺服器端強制執行：若未登入，請求將因 `401 AUTH_REQUIRED` 錯誤而失敗；若使用免費方案，則會出現 `403 PLAN_UPGRADE_REQUIRED` 錯誤。 在執行生成的腳本之前，請先在每台機器上執行一次 `chloros-cli login`。請參閱 [Chloros+ 登入](chloros+-login.md)。
* **擷取指令可驅動實體硬體。** `lattice` / `daq` / `project` 指令、SDK 工作階段物件以及 `chloros-mcp` 的工具，可連接、串流並觸發實體攝影機和感測器。在首次執行生成的腳本前請先審閱內容，並在有人值守的狀態下執行該腳本。
* **抽查輸出結果。** 在發布結果前，請驗證產品資料夾及若干像素值。 特別是，反射率 TIFF 檔案會根據光源進行縮放 — 請參閱 `Chloros:PixelScale` 的 XMP 標籤（LATTICE 檔案包含此標籤：32768 = 1.0 反射率； Survey3檔案則不包含此標籤，且採用 65535 = 1.0）來進行縮放，而非直接假設除數。兩份參考頁面均在「讀取反射率像素」章節中記載此內容。
* **可能導致生成的程式碼出錯的小陷阱：**`pool-record` 會寫入**後端主機** 的檔案系統（預設為 `~/Documents/DAQ Live View/`）； 在具備多個網路介面的機器上，建議優先使用 `daq pool-connect --eth-host <ip-or-hostname>` 而非自動偵測；凡出現後端 URL 的位置，請使用 `http://127.0.0.1:5000`（切勿使用 `localhost`）； 任何遍歷專案輸出目錄樹的腳本應跳過 `clamp_masks/` 資料夾（其中的 `*_clampmask.tif` 檔案是遮罩，而非影像 — 參見 [輸出影像格式](output-image-formats.md#clamp-masks))；此外，在已關閉長路徑功能的 Windows 系統上，請保持專案資料夾名稱簡短——若專案資料夾路徑長度超過約 130 個字元，系統將在處理開始前拒絕該專案。
