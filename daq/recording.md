# 記錄與 .daq 格式

`.daq` 檔案是 Chloros 的光感測器記錄格式：一個由單一 DAQ 感測器所產生的、經過校準的譜幀所組成的 **SQLite 資料庫**。 在擷取作業期間記錄一個檔案，反射率處理流程便能於事後將每張影像除以該精確時刻測得的下行輻照度。

## .daq 檔案的內容

| 屬性 | 值 |
| --- | --- |
| 容器 | SQLite 資料庫，每個感測器每次記錄各佔一個檔案 |
| 檔案名稱 | 包含 **感測器 ID**及**時間戳記**，例如 `daq_data_daq-e-def330_2026_04_13_18h30m00.daq` |
| 每幀光譜 | 135 個數據點，波長範圍 340–1010 nm，間隔 5 nm，外加 CIE XYZ 三刺激值 |
| 單位 | 經校正的光譜輻照度，**W/m²/nm**（已套用工廠校正套件 + 遮光罩校正曲線） |
| 標記的元資料 | 感測器 ID（用於擷取該裝置工廠校正的金鑰）以及當前生效的蓋板特徵曲線 — 請參閱 [蓋板特徵曲線與校正範圍](caps-and-range.md) |

此格式在 DAQ-U、DAQ-M、DAQ-E 及 DAQ-E-S 之間完全一致，因此後續處理無需關心是透過哪種傳輸方式進行記錄。DAQ-E-S 的記錄還會將該裝置的傾斜角、滾轉角和俯仰角儲存於每個光譜旁。

**錄製資料可在斷電後保留（1.3.0）。**即時錄製會在每次提交時透過日誌和完整磁碟同步進行寫入，因此斷電或程序被終止時，僅會損失約最後 3 秒的資料，而非整個檔案。 若記錄在寫入過程中中斷，可能會在 `.daq` 檔案旁留下一個 `<name>.daq-journal` 檔案：**請將這兩者一併保留並複製。** Chloros 在首次讀取檔案時會將日誌回滾。 磁碟繁忙會延遲檔案寫入，但絕不會延遲感測器的讀數。

**若 DAQ-E 停止傳輸資料，系統會重新連線（1.3.0）。** 當 DAQ-E 的資料流中斷（例如其交換機重新啟動或連線中斷），後端會以其原有的設定重新連線，且該裝置上的記錄將在同一個 `.daq` 檔案中繼續進行，並在檔案中標記出中斷的區段。

校準錄製需要感測器的原廠校準套件。對於 DAQ-U 和 DAQ-M，後端會根據感測器 ID 從 MAPIR 的雲端取得該套件（若無法取得則拒絕錄製）；DAQ-E 裝置則不受此限，因為它們將校準資料儲存於裝置內。

## 透過 GUI 進行錄製

透過 GUI 進行錄製需先 **開啟專案**（否則「錄製」按鈕將處於停用狀態）：

* **全部錄製 / 全部停止** — 位於「光感測器」側邊欄頂端；可同時啟動或停止所有已連接感測器的 `.daq` 錄製。
* **錄製 / 停止錄製** — 針對每個感測器，位於齒輪設定模態視窗中。錄製期間，感測器的即時資訊列中會顯示紅色的「REC」指示符。

檔案會寫入 `<project>/light_sensor/`，當錄製停止時——無論是透過「停止」、「全部停止」，或是斷開錄製感測器的連接——完成的 `.daq` 檔案將 **自動新增至開啟中的專案**。 該檔案會直接出現在專案的檔案清單中，無需手動新增步驟，且已準備就緒以進行反射率處理。

<!-- SCREENSHOT-NEEDED: Light Sensors tab with one DAQ sensor connected and recording: sidebar showing the red "Stop All" state of the Record All button, the sensor row, and the settings modal open with the red "REC" indicator visible in the live info rows. -->

<!-- SCREENSHOT-NEEDED: File Browser / project file list immediately after stopping a DAQ recording, showing the .daq file auto-added to the open project alongside imagery. -->



## 透過 CLI

進行錄製

CLI 透過後端的感測器池進行錄製（後端必須正在運行 — 這些指令是輕量級的 HTTP 客戶端）：

```bash
# Connect the sensor into the backend pool
chloros-cli daq pool-connect --eth-host daq-e-def330.local

# Record for 150 seconds, with a human-friendly device label
chloros-cli daq pool-record --sensor-id daq-e-def330 --duration 150 \
    -o ./out --device-name "rooftop-A"

# Or run open-ended and stop explicitly
chloros-cli daq pool-record --sensor-id daq-e-def330            # --duration defaults to 0 = run until --stop
chloros-cli daq pool-record --sensor-id daq-e-def330 --stop
```

從 `chloros-cli daq pool-list` 取得 `--sensor-id` 的數值。有兩項值得了解的預設值：

| 選項 | 預設值 |
| --- | --- |
| `--duration` | `0` — 記錄至 `pool-record --stop` |
| `--output` / `-o` | `~/Documents/DAQ Live View/` 位於 **後端** 的檔案系統上，而非 CLI 的 |

當 CLI 指向另一台機器上的後端時，輸出目錄的區別至關重要：檔案會存放在後端執行的位置。 若後端位於同一台電腦上，相對路徑的 `-o`（例如上方的 `./out`）將以您執行該指令的資料夾為基準進行解析。

## 來自 Python

的錄製`DAQSensorSession`（由 `chloros_sdk.connect_daq_sensor()` 返回）揭露了相同的彙總錄製內容： `record_start(output_dir=None, device_name=None)` 會傳回檔案路徑，`record_stop()` 則會傳回 `{path, rows}`。 請參閱 [SDK 參考手冊](../reference/sdk-reference.md) 以了解完整的會話 API。 SDK 的直接硬體類別（僅限桌面安裝版）預設會將錄製資料寫入 `~/Documents/DAQ/`；對於已發布的建置版本，上述彙總路徑是受支援的途徑。

## 在處理階段使用 .daq 檔案

要從影像中產生反射率，Chloros 需要與每個曝光相匹配的向下輻照度：

* **請將 `.daq` 與影像一併保存。**在處理階段，處理流程會自動從已記錄的 `.daq`（任何 DAQ 型號）——或從與影像一同存放的 DAQ-M 原生 `.csv` ——從影像旁找到的資料中自動解析出**時間戳記匹配的向下輻射強度**。GUI 錄製檔會自動滿足此條件，因為它們在停止錄製的瞬間即會被加入專案中。
* **校準資料會按需取得。** 若尚未在本地快取中儲存各相機或各 DAQ 的出廠校準套件，Chloros 將在首次使用時自動從 MAPIR 的雲端擷取（需連線一次；快取於 `~/.chloros/` 中）。
* **即時擷取會寫入其專屬的 sidecar 檔案。** 對於任何即時擷取的反射率幀，實際使用的 DAQ 讀數會以 `.daq` 檔案的形式儲存於該擷取檔的 `daq/` 資料夾中，因此日後即使沒有原始錄製檔，仍可重新處理該擷取檔。
* **來自 `chloros_scripts` 錄影機的錄影檔（1.3.0）。**由公開版 [`chloros_scripts`](https://github.com/mapircamera/chloros_scripts) 記錄器所產生的 `.daq` 檔案，會被列為生成該檔案的 DAQ 型號，其上限值可像其他記錄檔一樣在**「專案設定 → DAQ 光感測器」** 中設定上限，與其他錄製檔案無異。這些記錄器會寫入原始感測器計數值；Chloros 會透過序列埠擷取感測器的出廠校準資料並加以套用。
* **讀數來源（1.3.0）。**在每個擷取項目 `.daq` 旁邊，會有一項名為 `.daq.cameras.json` 的項目，用以標示該讀數所對應的攝影機（或多台攝影機），而處理程序會將其**僅與來自這些攝影機的影像** 進行配對。 因此，未配備自有光感測器的獨立相機將不再借用其他相機的讀數——請為其綁定感測器，或自行記錄讀數。 使用 DAQ 本身所做的錄製（GUI 錄製，`daq pool-record`），以及任何未包含該檔案的 `.daq`（包括較舊的擷取檔），仍適用於所有攝影機。

## 提取輻照度

處理專案時，系統也會將其中包含的所有光感測器記錄匯出至
`Light Sensor/` 資料夾中，該資料夾位於影像產品旁邊。此過程**無需**影像資料：單獨記錄的
光感測器資料即為完整的擷取結果，而僅包含 `.daq`
檔案的資料夾亦為有效的輸入。 僅包含光感測器記錄的專案，無論透過應用程式或CLI 執行，都會列出這些記錄並
進行處理，且執行完成後這些記錄仍會保留在清單中。執行報告
會顯示已寫入多少個光感測器產品。

| 產品 | 內容說明 |
| --- | --- |
| `<name>_calibrated.daq` | 一個可重新處理的存檔，其結構與即時記錄相同，但會聲明產生該檔案的校準套件。重新匯入該檔案時，**不會**再次進行校準。 |
| `<name>_calibrated.csv` | 以 W/m²/nm 為單位、基於感測器自身波長網格的光譜輻照度，每筆讀數佔一行，並附有光度學欄位：總功率、明視與暗視勒克斯、包含藍/綠/紅分拆的 PPFD，以及峰值波長。 |

若 DAQ-U 或 DAQ-M 無法取得其校準套件 —— 您處於離線狀態，或
該感測器檔案中無校準資料 —— 系統會 **附帶理由跳過該感測器**，絕不會將其
寫入為包含原始計數值的「已校準」檔案。請連線至網際網路並重新執行。 DAQ-E
攜帶自身的校準資料，因此僅在裝置未連線且
本地端無任何快取資料時才需要此步驟。

### DAQ-A：原始計數，以及為何這是正確的解答

**DAQ-A** 早於「每條串列埠對應一個校準包」的系統問世，因此沒有校準包
可供擷取。 這並非疏漏：DAQ-A 是在現場透過
反射率標靶進行校準的，而基於標靶的校準僅需感測器的 *相對*
響應——這正是其原始計數的本質。Chloros 至今仍以此進行校準。

因此，DAQ-A 的記錄資料雖可匯出，但會以不同的名稱儲存：

```
<project>/
└── Light Sensor/
    ├── <name>_raw.daq
    └── <name>_raw.csv
```

`_raw`，而非 `_calibrated` —— 採用不同的檔案名稱而非檔案內的標記，
因為該標示必須在檔案以純文字名稱透過電子郵件傳送時仍能保留。 `.csv`
的標頭顯示為 `raw spectral sensor counts (NOT irradiance)`，並警告這些數值僅
在檔案**內部**可相互比較，而非跨感測器比較。 僅對
真實輻照度具有意義的欄位 — 總功率、勒克斯、PPFD — 會留空，而非
根據計數值計算得出。

較舊的 DAQ-A-SD 記錄（資料結構 v1.01 / v1.02）僅記錄檔案的寫入時間，而非
每筆讀數的時戳記。 Chloros 無法將影像與這些記錄進行比對——若未經核對便將幀與
寫入時間配對，結果雖看似無誤實則有誤——但匯出功能能正常讀取這些資料，且
CSV 檔案會標明所對應的時鐘。

關於反射率的完整說明——單感測器搭配相機，以及雙感測器（環境光／物體）——請參閱 [反射率工作流程](reflectance.md)。
