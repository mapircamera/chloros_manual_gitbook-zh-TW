# 連接攝影機

<figure><img src="../.gitbook/assets/image (37).png" alt=""><figcaption><p>在連接任何裝置之前，「攝影機」分頁的顯示狀態</p></figcaption></figure>

Chloros
會自動在連線中偵測到 LATTICE 相機 — 無論是透過 GUI 的「相機」分頁、`chloros-cli lattice`，或是 Python
SDK
。 相機的型號字串將驅動後續所有流程：Chloros
會從相機的 `DeviceUserID` + `DeviceSerialNumber` 檔案中解析感測器規格、波段配置及出廠校準參數，因此 **無需針對每台相機進行任何設定**。

連線前，請確保已設定好主機網路 — 包含鏈路本地域址、巨型幀，以及針對陣列的網卡接收緩衝區設定。 上述為硬體端的設定，詳見 LATTICE 手冊：[**網路設定**](https://mapir.gitbook.io/lattice-camera/setup/network-setup)。

## 透過 GUI 進行連線

請在 Chloros
側邊欄中開啟 **攝影機**標籤頁（後端啟動完成後，硬體標籤頁即會顯示），或透過主選單 →**連線至攝影機**。這兩種方式都會開啟**連線攝影機** 對話方塊。

### 「連接攝影機」對話方塊

對話方塊一開啟便會掃描網路（「正在掃描網路...」），並列出所有偵測到的攝影機。 每行會顯示攝影機的 **型號**（例如 `LATT-M3M-L41-F550`）、**序號**以及**IP 位址**。

* **點擊一行即可選取該行**（以綠色高亮顯示）。 您可以選取**多台攝影機** 並一次連接 — Chloros
會依序進行連接。
* 標有 **「已連接」** 標記的列表示該攝影機已連接，無法再次選取。
* 標有 **「在陣列中」** 標記的列表示該攝影機屬於目前已連接的攝影機陣列。 若要將該攝影機以獨立模式使用，請先斷開該陣列的連線。
* **連線** — 連線所選的攝影機；當選取多台時，按鈕會顯示數量，例如「連線 (3)」。
* **重新掃描** — 再次執行偵測。
* **關閉** — 關閉對話方塊。
* 若掃描完成後無結果，對話方塊將顯示 **「網路中未找到任何攝影機」** — 請參閱下方的 [疑難排解](connecting.md#troubleshooting)。

<figure><img src="../.gitbook/assets/image (38).png" alt=""><figcaption><p>「連接攝影機」對話方塊 — 此處顯示為網路中無任何攝影機</p></figcaption></figure>

### 首次連線：校準包下載

當某台攝影機在電腦上 **首次** 連線時，Chloros
會透過 GigE 從攝影機本身擷取其出廠校準包（約 3.8 MB）。 在此過程中，對話方塊會顯示一個綠色的 **「正在從攝影機下載校準資料」**面板，並附有按序號劃分的進度條 — 每台攝影機約需**70 秒**。若攝影機數量超過六台，進度條會於面板內滾動顯示。 校準包會緩存於主機上，因此同一台攝影機後續連接時將完全跳過下載步驟（且不會顯示該面板）。

### 系統分析

對話方塊中的 **分析系統** 按鈕會偵測主機與網路（執行期間標籤顯示為「分析中...」），並產生一份診斷報告：

* **主機** — CPU 核心數與 RAM；GPU 名稱與記憶體，或顯示「GPU：未偵測到」。
* **網路介面** — 每個網路介面卡（NIC）的名稱、鏈路速度、MTU（若啟用則標示「jumbo」標籤）、上行/下行狀態，以及是否位於 USB 匯流排上。
* **攝影機**— 序列號、型號、IP 位址，以及**每台攝影機所連接的網路介面卡**。
* **效能** — 每台攝影機在該像素格式下的當前幀率與理想幀率比較；當理想幀率高於當前幀率時，會顯示一條綠色的「潛在提升空間：可提升 N 倍」標線。
* **警告與編號建議** — 若無需修正之處，則顯示「系統狀態良好，可支援當前攝影機數量」。

當偵測或串流行為異常時，請隨時執行此功能 — 它能在不離開對話方塊的情況下，識別出多數網路介面卡端的問題（錯誤的 MTU、攝影機連接至錯誤的介面、USB 轉接器限制）。

### 連接陣列

若要將兩台或更多攝影機連接為 **同步陣列**，請改用陣列連接精靈（**連接攝影機陣列**）：它會引導您完成主從機選擇（透過同步線纜偵測預先填入 — 任何已連接線纜的攝影機均可為主機， 且當RGB
`-FRGB`攝影機連接線纜時會預先選中），顯示模式選擇（獨立與合併拼圖），以及陣列設定畫面——在您確認前，該畫面會即時預覽可達到的每秒幀數（fps）與線路頻寬。 本手冊的「多攝影機陣列」章節中詳述了此精靈與陣列工作流程；其 CLI
對應操作則是 [CLI
參考手冊](../reference/cli-reference.md) 中的「LATTICE 攝影機首次連接工作流程」。

## 透過 CLI
和 SDK
進行連線使用

CLI
和 SDK
需具備付費的 Chloros
+ 服務等級並已登入；此限制由伺服器端強制執行（未登入時為 `401 AUTH_REQUIRED`，免費服務等級為 `403 PLAN_UPGRADE_REQUIRED`）。

```bash
# List cameras on the network (vendor, model, serial, IP, MAC)
chloros-cli lattice info

# Single-camera smoke test: capture one frame (saves every applicable export type)
chloros-cli lattice capture -o output/

# Connect a synchronized array — same smart-prep flow as the GUI
chloros-cli lattice array-connect --serials 213800234,214000533
```

```python
import chloros_sdk

# Persistent live-camera session through the backend
with chloros_sdk.connect_camera("213800234") as cam:
    ...

# Array session (smart-prep: network probe, tier auto-pick, PTP, AE seeding, trigger config)
with chloros_sdk.connect_array(["213800234", "214000533"]) as array:
    ...
```

完整的簽名、選項及擷取工作流程：[CLI
參考手冊](../reference/cli-reference.md) § `chloros-cli lattice`， [SDK
參考](../reference/sdk-reference.md) § `connect_camera()` / `connect_array()`。

## 連線時如何處理校準

每台 LATTICE 攝影機皆在 **機身內** 儲存其出廠校準包，且當攝影機連線時，Chloros
也會檢查 MAPIR
的雲端：

| 情況   | Chloros
使用的內容                                                                                                                                                                                                          |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **線上**|**該序號所發布的最新校準資料** —— 雲端副本優先於相機內建副本。因此，經由 MAPIR
重新校準或更新的相機將自動更新；無需使用者操作。 |
| **離線**|**相機內的校準包**，維持原狀。完全離線的工作流程仍可正常運作；它們僅會在相機首次連線（或進行工廠級重新燒錄）後，才會採用較新的校準參數。                                                  |

在拍攝時，實際套用的係數會被**凍結在每張影像的 XMP 元資料中**。後續的校準更新絕不會在您不知情的情況下更改您已拍攝的影像——重新處理舊影像時，系統會使用其 XMP 中標記的係數，而非當下的最新版本。

## 疑難排解

* **「「網路中未找到任何攝影機」**— 請驗證 [網路設定](https://mapir.gitbook.io/lattice-camera/setup/network-setup) 中的鏈路本地域設定：主機網卡設定為靜態 IP `169.254.x.x/16`，攝影機位於同一鏈路上，預期不使用 DHCP 或閘道。 接著請在連線對話方塊中使用**分析系統**功能，檢查每台攝影機在哪些網路介面上可見（或不可見）。進行任何線路連接或網路介面卡變更後，請執行**重新掃描**。
* **先前可正常運作的系統無法連線**（陣列面板閘口編號為 `FRAMES WILL DROP` / `Reduce ROI to enable`）— 網卡驅動程式更新已默默重置了接收環設定。 請重新套用設定，或從具有管理權限的終端機執行 `chloros-cli lattice network --fix`；詳見 [網路設定](https://mapir.gitbook.io/lattice-camera/setup/network-setup)。
* **某台攝影機顯示「In Array」** — 表示該攝影機屬於已連線的陣列工作階段。若要將該攝影機作為獨立裝置使用，請先斷開陣列連線。
