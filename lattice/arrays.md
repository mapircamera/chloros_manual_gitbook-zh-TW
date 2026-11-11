# 多鏡頭陣列

LATTICE **陣列**是指將兩台或更多台 LATTICE 相機連接成一個同步單元。其中一台相機擔任**主機**：它會在共用同步線（預設為**Line2**）上發射硬體 GPIO 觸發脈衝， 使每個成員都能在同一瞬間進行曝光。Chloros

增加了 PTP 時間同步、即時預覽（可選每台相機的圖塊或單一對齊的多波段合成影像），以及同步擷取功能——每次擷取循環都會產生一個 **幀組**，其中所有相機共享相同的時間戳記和幀 ID （在擷取輸出中顯示為 `fid:N`）。

陣列是單色（M3M）相機生成植被指數的方式——每台相機貢獻一個波段，而陣列將它們對齊成多波段堆疊。 請參閱 [單色相機與植被指數](mono-indices.md)。

連接陣列有三種等效方式，且所有方式皆執行相同的「smart-prep」流程：

| 介面 | 進入點 |
| --- | --- |
| GUI | 「相機」分頁 → **連接陣列**（藍色按鈕） |
| 命令列介面 (CLI

) | `chloros-cli lattice array-connect --serials SN1,SN2,…`（第一個序列號 = 主節點） |
| Python

SDK

| `connect_array(serials=[…])` → `ArraySession` (第一個序列號 = 主節點) |

Smart-prep 會執行以下操作：網路功能探測（ICMP DF ping + GVSP 探測）、同步層選取、根據請求的幀大小進行幀率預測、決定是否採用壓縮式 12 位元傳輸 （參見 [12 位元封裝傳輸](#packed-12-bit-transport)）、啟用 PTP、針對每台攝影機自動選擇像素格式、根據各攝影機儲存的狀態進行自動曝光初始化，以及在 Line2 上配置 GPIO 觸發。

**預設為全感測器模式 （1.3.0 版新增功能）。** Connect 會維持您所要求的幀尺寸——完整的 2048×1536 感光元件，不進行像素合併也不進行裁切，除非您另行選擇 ——並讓幀率降至主機與網路所能提供的水準。它絕不會自行進行像素合併或裁切（1.2.0 版本會在幀率低於 1.5 fps 時自動啟用像素合併）。 是否進行像素合併或設定較小的 ROI 由您決定：在「連線」對話方塊中選擇其中一項，「預估結果」欄位將顯示對應的幀率；`array-connect` 會列印較快的選項及其預估幀率。

{% hint style="info" %}
在執行任何操作之前，必須確保攝影機在連線中可被存取 — 請參閱 [連接攝影機](connecting.md) 以了解裝置偵測、位址設定及首次連線校準下載的相關資訊。對於多攝影機架設，主機網路介面卡（NIC）的接收環設定與鏈路速度同等重要；完整的「症狀→解決方案」對照表請參閱 [CLI

參考 § 主機 NIC 設定與調校](../reference/cli-reference.md#host-nic-setup--tuning-lattice-arrays) 中。
{% endhint %}

## 「陣列連線」對話方塊

在「攝影機」分頁中點選 **連線陣列**，將開啟一個三步驟精靈：**選取 → 顯示模式 → 設定**。

### 步驟 1 — 選取主相機與從相機

<!-- SCREENSHOT-NEEDED: Array Connect dialog, Select scene, with 3-4 LATTICE cameras discovered (an FRGB among them). Table showing Camera / Serial / IP / Master radio / Slave checkbox columns in camera order (M3C first, FRGB at the top), with the FRGB pre-selected as master and the green "Sync cable found on N cameras. Any of them can be master; the RGB camera (else the first) is pre-selected." probe banner visible above the table. -->



對話方塊一開啟便會立即掃描網路（「正在掃描網路...」），接著偵測同步線纜（「正在偵測同步線纜...」），以找出哪些相機連接到 M8 同步線路上。您至少需要 **2 台相機** 才能建立陣列。

由於 M8 同步線纜匯聚於單一接點，因此 **任何已連接線纜的攝影機皆可擔任主機**。探測程序會預先填入角色——當RGB

（`-FRGB`）相機已連接線纜時即設為主機，否則則為第一台連接線纜的相機，其餘連接線纜的相機則設為從機——並顯示以下三種標語之一：

| 標語 | 含義 |
| --- | --- |
| 「在 {n} 台相機上偵測到同步線纜。其中任一台皆可擔任主機；RGB

相機（否則為清單中第一台）已預先選定。」（綠色） | 探測器已偵測到同步線上的相機；主機與從機的核取方塊已預先勾選。 |
| 「未偵測到主相機 — 請檢查 GPIO 線纜」 (橘色) | 沒有任何相機偵測到觸發脈衝；請檢查同步線纜連接。角色仍預先填入（RGB

相機，否則為清單中的第一台相機，作為主相機），且您可以變更這些設定。 |
| 「無同步線纜：{serials}」（橘色） | 清單中的相機未連接同步線纜。 |

相機表格包含 **相機 / 序列號 / IP / 主相機 (無線電) / 從相機 (核取方塊)**這些欄位。列的排列順序依**相機順序** 排列 — M3C (拜耳) 相機排在 M3M (單色) 之前；在 M3C 之中，依序為 FRGB、FRGN、FOCN，然後是 FNGB； M3M 型號則依濾光片波長排序，波長最短者優先（F450 排在 F808 之前）。側邊欄的相機清單與網格的預設佈局均採用相同排序。

* 請精確選擇 **一個主相機**和**一個或多個從相機**。再次點擊當前主相機的單選按鈕可取消選取。選擇不同主相機只需一次點擊：新主相機將帶走其從相機，而先前的主相機則轉為從相機（除非其未連接同步線）。
* 標記為 **「無同步線」** 的攝影機絕不能被選為從機 — 若從機未連接觸發線路，將永遠在同步線路上等待，並傳輸無效訊號。請改將該攝影機作為獨立攝影機連接。
* 已以獨立模式連接的攝影機 *不會* 被停用：陣列連接功能會釋放獨立模式會話，並將該攝影機重新納入陣列中。

**下一步：顯示模式 →**需在選取主機及至少一台從機後才會啟用。**重新掃描** 會重新執行裝置偵測與接線檢測。

{% hint style="warning" %}
當掃描或檢測正在進行時，**取消** 按鈕將被停用 — 若在檢測過程中取消，可能會導致 LATTICE 攝影機韌體上的攝影機 SDK

當機。請等待轉圈圖示完成。
{% endhint %}

### 步驟 2 — 顯示模式

<!-- SCREENSHOT-NEEDED: Array Connect dialog, Display Mode scene, showing the two selectable cards ("Separate Cameras" and "Combined Cameras") with Combined selected/highlighted as the default. -->



| 模式 | 顯示效果 |
| --- | --- |
| **獨立相機** | 每台相機各佔一個即時磁磚，所有相機同步觸發以確保畫面保持同步。每台相機皆保留其專屬色彩與設定。 |
| **合併相機** *(預設)* | 單一動態磚渲染對齊後的多波段 NDVI

/指數合成影像。各相機共用陣列顏色。 |

顯示模式僅改變即時預覽的呈現方式 — 兩種模式下的擷取行為皆相同。

### 步驟 3 — 陣列設定與預估結果

<!-- SCREENSHOT-NEEDED: Array Connect dialog, Settings scene, healthy state: left column with ROI / Binning / Trigger Rate / Wire Budget controls (1.3.0 has no "Pin resolution" checkbox), right "Projected Outcome" column showing green "Simultaneous capture" tier, an fps range, the NIC line, the "Sim-emit burst" line, and the "Wire budget" line with a checkmark. -->



進入此場景時，Chloros

會向後端請求 **建議**並加以應用：**完整感測器 — 不進行像素合併，不進行 ROI 裁切 — 只要任何幀率能支援**，無論主機與傳輸線提供何種幀率。僅當沒有任何幀率能支援完整感測器時，建議才會縮小畫面， 此時系統會優先採用像素合併而非 ROI 裁切，因為像素合併能保留完整的視野範圍。從此處開始，您將在解析度與幀率之間進行權衡。每次變更設定後，系統都會即時重新執行分析，並更新右側的**預估結果** 面板。

左欄 — 設定：

| 控制項 | 選項 | 預設值 | 備註 |
| --- | --- | --- | --- |
| **ROI（視野）** | 完整 (2048×1536) / 半幅 (1024×768) / 四分之一 (512×384) | 完整 | 感測器裁切：以原生像素間距將畫面裁切為半幅或四分之一幅的較小區域。 |
| **像素合併** | 1× / 2×（2×2 合併） / 4×（4×4 合併） | 1× | 硬體像素合併：2×2 = 以四分之一的傳輸成本獲得完整 FoV；4×4 = 以 1/16 的傳輸成本獲得完整 FoV。 若相機不支援像素合併，則隱藏此欄位。 |
| **線路端影像**（讀出） | — | — | 實際透過線路傳輸的像素合併後寬度×高度，會被截斷為 16 的倍數（最小 64）。 |
| **觸發率** | 0.5–60 fps，步長 0.1 | 留空 = 自動 | 主控端的觸發發射率。 留空讓 Chloros

自動推算。 |
| **線路預算**| 20–2000 MB/s，步長 10 | 留空 = 自動 | 主機實際可吸收的流量，單位為 MB/s —**這是整個陣列分配的關鍵數值。** 由網路介面卡自動偵測。若陣列回報封包損毀，請降低此數值：偵測到的數值往往高估了 USB 介面卡和共用交換器的實際傳輸能力。變更此數值將即時重新執行預測。 此欄位預設顯示上次連接陣列時使用的預算值（隨專案儲存），因此您曾修正過的預算值會被保留。 |

自 1.2.0 版本起，**Pin 解析度** 核取方塊已移除：連接時系統會始終維持您選擇的幀大小，並相應降低幀率。

右側欄 — **預測結果**：

* **同步層級** — 「同步擷取」（綠色）、「同步擷取（FTD 交錯發射）」（綠色）、「交錯擷取（100 毫秒漂移）」（琥珀色），或「配置過大」 （紅色）。Chloros

會自動選擇層級：只要線路能以相機可跟上的速率共享，便採用同步擷取（隨著每台相機的共享比例縮減，幀率會隨之下降）；若無法共享，則採用交錯擷取——因此陣列可在任何網路環境下連接，並以該網路允許的幀率運作。
* **fps 預設值** — 陣列預期運作的幀率。 在同時擷取模式下，這是一個單一數值：幀率優先，自動曝光會填滿其所指定的時間段，然後提高增益，因此光線昏暗的場景不會降低幀率（參見 [自動曝光如何分配曝光時間與增益](camera-settings.md)）。 在交錯擷取模式下，此數值會以範圍形式顯示（「暗 → 亮」），因為在光線昏暗的場景中，每台攝影機的曝光時間會拉長交錯間隔。
* **網路介面卡 (NIC) 線路** — 鏈路速度與持續傳輸量（「NIC {mbps} Mbps · 持續 {N} MB/s」）。
* **同步發射突發檢查** — 主機的 NIC 環路能否接收來自所有攝影機的單次同步突發傳輸（「同步發射突發：X MB · NIC 環路可用空間：Y MB ✓/✗」）。
* **線路預算檢查** — 穩態總需求與防碰撞線路上限的比對（「線路預算：{n} 台攝影機所需的 {demand} MB/s · 上限 {ceiling} MB/s ✓/✗ 超額訂閱」）。
* **「此線路最大相機數量：{n} — 由每台相機的頻寬下限決定，因此分組處理不會提高此數值。」** — 當您接近（或超過）相機數量上限時顯示。
* **「在此設定下將發生幀丟失。&quot;**— 紅色警告訊息，附帶後端系統的理由說明，以及阻礙清單和藍色的**解決方案建議**（「讓此陣列適應網路」／「解鎖同時擷取」）。**「套用並連線」** 按鈕在存在預測結果前將被鎖定，其標籤會告知您被拒絕的原因：

| 按鈕標籤 | 含義 | 實際解決方案 |
| --- | --- | --- |
| “分析中...” | 分析仍在進行中。 | 請等待。 |
| **“此網路的攝影機數量過多&quot;**| 該線路甚至無法一次傳輸該陣列中的一台攝影機（聚合檢查失敗）：攝影機數量遠超其處理能力，或該線路上已有其他攝影機正在串流。 | 減少攝影機數量、端到端使用巨型幀，或使用更快的網路介面卡。**縮小 ROI 並無幫助** — 請參閱下方說明。 |
| **「縮小 ROI 以啟用」** | 此設定下畫面會掉幀（突發/環形檢查失敗）。 | 縮小 ROI、提高像素合併比，或修復網路介面卡的接收環。 |

<!-- SCREENSHOT-NEEDED: Array Connect dialog, Settings scene, over-subscribed state: red "Wire budget ... over-subscribed" line, the "Max cameras on this wire" hint, and the Apply button reading "Too many cameras for this network". Reproduce by configuring more cameras than the 1 GbE ceiling (e.g. 7+ cams at 1500 MTU) or with CHLOROS-simulated models via `lattice analyze-array`. -->



連接時，可能會出現一個帶有逐序列進度條的綠色 **校準下載面板**：當機器首次連接攝影機時，Chloros

會透過 GigE 從攝影機下載約 3.8 MB 的出廠校準包（每台攝影機約需 70 秒）。 已快取的攝影機不會顯示此面板；當攝影機數量超過六台時，其行會滾動顯示。請參閱 [連接攝影機](connecting.md)。

### 壓縮式 12 位元傳輸

12 位元格式（`BayerRG12`、`Mono12`）通常以 16 位元字的形式傳輸。**壓縮**傳輸方式會將相同的 12 位元以每像素 1.5 位元組的形式傳送，並在主機端進行解壓縮，因此每個畫面的傳輸時間可減少 25% — 在感測器全載情況下，每秒畫面數約可增加三分之一。**所節省的資料內容完全相同。** 陣列連接會自動決定此模式（`auto`）：僅在壓縮有助益且主機有閒置 CPU 資源進行解壓縮時才會進行壓縮。圖形使用者介面（GUI）中沒有對應的切換選項；若要透過CLI

設定，請使用 `array-connect --packed {auto,on,off}` （預設為 `auto`）。8 位元與 16 位元格式絕不會進行封裝。

## 頻寬：可支援多少台攝影機

陣列能承載的數量取決於網路線路，而非Chloros

，因此規劃數值請參閱硬體手冊：**[陣列頻寬規劃](https://mapir.gitbook.io/lattice-camera/setup/array-bandwidth-planning)**。

Chloros

如何運用這些數值：連線對話方塊會執行網路探測，預估可達成的幀率，並選擇符合條件的層級。陣列可在任何網路中連線：若遇低速連線，每台攝影機的傳輸速率會根據其佔用的頻寬比例進行調控，幀率隨之降低，而非直接拒絕陣列連線或壓縮其幀資料。 唯有當線路連單一攝影機的流量都無法承載時，陣列才會被拒絕（**此網路的攝影機數量過多**），而非默默丟棄封包——請參閱上文所述的「預測結果」面板。

## 當幀資料遺失時

攝影機可能因兩種截然不同的原因而未出現在已發佈的群組中，
且兩者需要截然相反的解決方案。Chloros

會將它們分別計數，而非報告一個
既未指明原因的「不完整」數字：

| 發生情況 | 代表意義 | 檢查位置 |
| --- | --- | --- |
| **損毀**— 幀已送達但結構異常 | 網路路徑上的 GVSP 封包遺失 |**線路預算**、網卡接收環、巨型幀、交換機 |
| **未送達**— 完全沒有幀傳來 | 攝影機未觸發，或未傳出任何資料 |**M8 同步線纜**、同步線路、所有成員是否已啟用 |

在陣列串流期間，系統每 10 秒重新評估此分佈。若超過 5%，系統會
將兩項數值一併記錄於日誌中；每個損壞的緩衝區將在首次發生時
針對每台攝影機進行通報，之後每分鐘彙總一次，以確保長時間的會話記錄仍可讀取。

**若損壞幀數為零且「未送達」數值為零，表示觸發與線纜同步皆完美**且所有遺失的幀皆發生於網路路徑上。解決方法是降低**線路預算** 並
重新連線。

{% hint style="warning" %}
**降低觸發率對解決畫面損壞問題無濟於事。** 攝影機的封包
步調是在連線時一次性設定的。降低觸發率僅會改變突發傳輸
的發生頻率，並不會影響突發傳輸本身傳輸至線路的速率。 在經過測量的 4 台攝影機系統中，
將觸發率降低 5 倍並未產生任何改變，而將線路預算從 240 降至
200 MB/s，則使同一套系統的損壞率從 10.4% 降至零。
{% endhint %}

正在運行的陣列無法自行重新規劃 — 請先斷開連接並重新連接，以便連線時
的選取器能根據新的頻寬預算進行運作。

### USB 網路適配器存在傳輸速率上限

USB 以太網適配器雖標示其 *以太網* 鏈路速率，但實際
可維持的速率受限於 USB 匯流排及其驅動程式。一款 USB 10GbE 隨身碟曾被認為
具備約 1000 MB/s 的吞吐量 ——這個數字從未被實際測量過——而將
四台攝影機的傳輸速率設定為該虛幻餘裕值，會導致 6–18 % 的幀資料損壞，儘管陣列
仍報告目標幀率正常。目前 USB 連接的網卡傳輸速率上限為
Windows

和 NVIDIA Jetson 系統上為 **200 MB/s**，在 Linux

x86_64 系統上則為 **400 MB/s**。此上限為絕對值而非百分比，因為限制因素在於
匯流排：USB 1 GbE 轉接器約可達到 80 MB/s，且不受此影響。

若您的主機速度確實快於此上限，請將 **Wire Budget** 調高以反映實際情況。

## PTP 時間同步

幀的 *同步* 來自硬體觸發；**PTP**（IEEE 1588 PTPv2）則為所有裝置提供可比的 *時間戳記*。 此功能在陣列連線時預設為啟用狀態：

* **Chloros

主機後端執行PTP主時鐘**。LATTICE攝影機與DAQ-E光感測器在第0域中作為其從屬裝置，因此影像時間戳記與DAQ光譜皆以同一時鐘（約1毫秒）為基準。
* `--no-ptp`（CLI

）可於實驗台作業時停用此功能——此時不同相機之間的時間戳記將**無法**相互比對。
* 透過 CLI

檢查同步狀態：

```bash
chloros-cli time-sync status     # grandmaster state, clock identity, health (OK, or why its packets are not reaching the network)
chloros-cli time-sync peers      # slaves seen (cameras + DAQ-E sensors)
chloros-cli time-sync cameras    # per-camera PtpStatus / PtpOffsetFromMaster / PtpMeanPathDelay
```

「相機」分頁本身沒有 PTP 指示器；該處顯示的各相機同步資訊包含唯讀的 **角色**（主機/從機）、**同步線**，以及陣列的「功能」層級。DAQ-E 的 PTP 狀態則顯示於「光感測器」分頁的感測器詳細資訊中。

## 即時陣列檢視

<!-- SCREENSHOT-NEEDED: Cameras tab with a connected combined array: sidebar showing the ARRAY row (color badge, array name, "DAQ · on" pill) with indented member camera rows in camera order (M3C before M3M, monos by wavelength), and the main area showing the combined index composite tile with the LUT-colored NDVI render, top-left array name pill, and top-right fps readout. -->

主畫面區域提供兩種佈局（可透過頂部工具列切換）：**網格檢視**（每個方塊皆為一個單元格； 當網格鎖定圖示解鎖時可拖曳重新排序）以及**清單檢視**（全寬列：**合併**陣列佔一列——即其複合圖塊——而**獨立**陣列則讓**每個成員各佔一列**，並擁有各自的即時串流及校準目標表格； 陣列下方則顯示一台獨立相機——即當前活躍的相機）。**畫面縮放**滑桿可調整圖塊大小；當單元格寬度低於 200 像素時，名稱／幀率疊加標籤會自動隱藏。成員相機會依相機順序顯示於側邊欄及網格的預設佈局中 （M3C 先於 M3M，FRGB 優先，單色相機依波長排序）；解鎖網格並拖曳仍會覆寫網格順序。

<!-- SCREENSHOT-NEEDED: list view of a connected SEPARATE array (1.3.0): one full-width row per member camera, each with its live feed and its own Calibration Target table below it. -->



**獨立模式** 會以每台相機一塊圖磚的方式顯示。每塊圖磚會疊加顯示：

* 相機名稱（左上角），
* **幀率讀數**（右上角）——此為後端回報的相機 *實際擷取率*，而非預覽輪詢率（即時預覽無論擷取率為何，皆上限為 30 fps），
* 狀態點 — 綠色（串流中）／琥珀色（載入中）／紅色（錯誤），
* 若 2 秒內未收到新幀，則顯示 **過期幀旋轉圖示** — 連線或斷線後約 5 秒內屬正常現象，此時後端正在各相機間重新平衡頻寬分配。**組合模式**會顯示單一複合圖塊：後端會執行去拜爾化、縮放、對齊、降噪，轉換為各波段輻照度（若綁定光感測器則加上 DLS 反射率），評估陣列的索引表達式， 套用 LUT，並將結果以 MJPEG 格式串流傳輸。在渲染出第一個對齊畫面之前，該圖塊會顯示其狀態：「準備陣列…」、 「校準對齊中…」、「等待首幀…」，或——若磁磚的自動對齊重試（約一分鐘）時間用盡——則顯示「需手動對齊」並附有**校準對齊** 按鈕。

陣列圖塊的 fps 讀數也會顯示串流狀態：當設定變更（格式、像素合併、ROI）導致相機重新啟動時顯示 **「套用設定中…」**；當陣列仍運作但已停止傳送幀組時顯示**「停滯」**；以及當串流停止時顯示**「已停止”**表示已停止串流——此時請重新連接陣列。當光譜解混過程必須對合成圖中某個波段的大部分像素（25% 或以上）進行裁剪時，該圖塊會顯示**“低可信度波段資料 · N%&quot;**：基於該波段計算的索引值可能有誤（通常表示某個波段接收的光線過少）。

混合模式實用須知：

* 合成影像會與 **主**相機的幀進行對齊。在合成影像上進行 AE-ROI 對焦與點測光時，對主相機而言是精確的，對從屬相機而言則是近似的；若要查看各相機的像素精確圖塊，請使用**分割視圖**（陣列設定 → 「顯示成員相機」），無需開啟額外的相機連線。
* ****顯示圖層**（陣列設定；預設為關閉）可讓您選擇前景與背景圖層——任何成員相機或**索引**。當前景設定為**索引** 時，超出 LUT 最小/最大範圍的像素將顯示背景圖層。
* **渲染解析度**（預設 720p）會同時設定直播畫面高度 *以及* 儲存的合成影像輸出尺寸。 各攝影機的影像始終以全解析度匯出。
* 對齊設定是針對每個會話計算的，且絕不會被永久儲存。Chloros

會在 **連接後自動對齊每個陣列，無論處於何種顯示模式**，前提是成員的自動曝光已穩定下來（系統會持續嘗試長達 5 分鐘）。陣列設定面板的**對齊**區段會顯示 RMS 殘差，並提供**重新校準對齊** 按鈕 — 此功能同樣適用於未參與合成拼貼的獨立陣列。

## 擷取：監控與分析

陣列擷取表面可明確區分為 **監測級**（記錄所見畫面）與**分析級**（記錄原始資料，稍後校正）：

| 工作流程 | 等級 | 儲存內容 | GUI | CLI

|
| --- | --- | --- | --- | --- |
| **擷取**（靜態影像） | 分析級 | 每輪掃描一個同步幀組；每台相機在每個選定的匯出層級（原始/去拜耳化/輻射度/反射率/預覽/索引）的檔案 + `.daq` 輔助檔案 |****擷取全部** 按鈕 + 擷取設定 | `lattice array-capture` |
| **錄製索引影片** | 監控 | 顯示中的即時複合索引合成影像 — 8 位元、預覽解析度、內嵌 LUT。 若在未開啟 GUI 視窗的情況下透過 CLI

操作，後端會在陣列對齊後渲染該陣列的 NDVI

（或 `--expr`）；若錄製過程中未寫入任何幀，則會失敗 | ● 錄製指數影片（組合陣列） | `lattice array-record` |
| **原始連拍 → 建立影片**| 分析 | 以全抓取速率擷取原始感測器幀 + 清單 + `.daq`，隨後離線重建為經校正的輻照度／反射率／指數影片，並與 DAQ 讀數進行時間對齊 | ⦿ 錄製原始連拍 →**建立影片** | `lattice array-burst` → `lattice array-build-video` |

經驗法則：若像素資料將用於 *測量*，請使用擷取或連拍模式（分析級）；若僅需 *觀看或展示* 感測陣列所捕捉的內容，則錄製索引影片（監控級）。

### 擷取設定 (GUI)



<!-- SCREENSHOT-NEEDED: Capture Settings pane (gear next to Capture All) with a connected array: capture-mode buttons (Single/Continuous/Interval), the bulk export-type toggle row, the Fastest Capture toggle, and the per-array group card showing the Aligned checkbox and the "Record index video" / "Record raw burst" buttons. -->

**擷取全部** 旁邊的齒輪圖示可開啟「擷取設定」窗格（需開啟專案 — 擷取資料將儲存至該專案中）：

* **擷取模式**：**單次**（一次通過）／**連續**（接連進行；受擷取次數限制，預設為 1 次，或受時間長度限制，預設為 10 秒）／**間隔** （定時攝影：每 X 間隔擷取 N 張，共 Y 張；預設為每 5 秒擷取 1 張，持續 1 分鐘）。
* **每台相機的匯出類型**：原始檔、去拜耳化、輻照度、反射率、預覽、索引 — 所有適用選項預設皆為開啟。對於配備RGB

濾鏡的相機，輻照度／反射率選項將被隱藏； **「反射率」僅在相機具備 DAQ 光感測器時才會顯示**（無論是相機自身具備或從陣列繼承而來）；「索引」則需要配置索引表達式。
* **對齊**（按陣列設定，預設**開啟**）：將成員匯出檔案變形至陣列的對齊設定檔，以確保匯出檔案的像素對齊。 原始檔始終保持未變形狀態，但會在元資料中攜帶變換資訊。陣列成員檔案會以各相機的原生方向儲存（任何鏡像/旋轉設定僅記錄於檔案中，不會實際套用）。
* **最快擷取**（切換）：僅原始資料 + 指派的 DAQ 讀數 + 免費的組合索引合成，在擷取時跳過校準運算以達到最高速率 — 稍後可從儲存的 `.daq` 檔案重建輻照度/反射率/索引。
* 選取範圍會隨專案保留。 隱藏或暫停的相機將被跳過。
* 陣列中的 **連續** 模式在 Chloros

後端內運行（1.3.0 版新增）：當清單中的每台相機均屬於同一陣列時，後端會自行執行擷取迴圈，在寫入一組資料的同時選取下一組，且無需針對每次擷取發出請求。 磁碟空間已滿將停止執行（「已停止：磁碟空間已滿。已儲存 N 筆，F 筆失敗」）。 有關 Microsoft Defender 對高擷取率的注意事項，請參閱 [擷取設定與模式](capture.md#capture-modes)。

等效的 CLI

（相同後端端點、相同語義）：

```bash
# One synced group, every applicable export level per camera (the default)
chloros-cli lattice array-capture -o output/

# Interval timelapse: one reflectance pass every 10 s for 5 minutes
chloros-cli lattice array-capture --interval 10 --duration 300 --processing reflectance -o timelapse/

# Fastest grab for a moving rig — raw + .daq now, calibrate later
chloros-cli lattice array-capture --fastest -o flightline/

# 30-second monitoring clip of the combined index view, plus a GIF
chloros-cli lattice array-record --duration 30 --fps 10 --gif -o monitoring/

# 5-second analysis-grade raw burst, then build the combined index video
chloros-cli lattice array-burst --duration 5 --build --products combined:index --fps 10 -o capture/
```

擷取的 TIFF

壓縮設定為 `deflate`（無損，預設）或 `none` — 完整的標誌表、擷取資料夾結構及重新處理規則詳見 [CLI

參考手冊](../reference/cli-reference.md#capture-modes-recorders--offline-reprocess) 中。

## 配對 DAQ 光感測器

反射率與照度校正的預覽畫面需要來自 DAQ 感測器（於 **光感測器** 索引標籤中連接）的下射光數據：

* 側邊欄的 **陣列行**會顯示一個**「DAQ · 開/關」圓點**—— 當設定了陣列層級的光線感測器**或** 任何成員相機擁有自己的感測器時，該圓點會顯示為 *開啟*；其工具提示會明確列出哪個感測器為哪台相機提供資料。
* 在陣列設定中進行全陣列分配 → **環境光感測器**→**光感測器** 下拉選單。此設定將隨專案保留，並套用至每個成員相機；個別相機仍可透過其自有感測器覆寫此設定。
* 其下方的狀態列會顯示即時狀態： **關閉**→ 「等待第一個光譜…」 →**「啟用 — 陣列中所有攝影機均已進行照度校正」** → 或者，若過去 3 秒內未收到新的光譜，則會顯示過期通知 — 系統將繼續使用最後一次讀數（在擷取路徑上，讀數永不過期）。

當已指派感測器時：「反射率」匯出類型將可使用，即時預覽會進行照度校正，預測性自動曝光可使用光譜，且每次反射率擷取都會將實際使用的 DAQ 讀數寫入擷取檔案的 `daq/` 資料夾中，以便日後重新處理該擷取。此外，一個名為 `.daq.cameras.json` 的檔案會標示該讀數所對應的相機；重新處理時，Chloros

僅會將該讀數與來自這些相機的影像進行配對。

## `array-connect` CLI

選項

| 標誌 | 預設值 | 說明 |
| --- | --- | --- |
| `--serials SN1,SN2,…` | 自動偵測所有 LATTICE 攝影機（需 ≥2 台） | **序列號最前者為主控（MASTER）。** |
| `--line {Line0,Line2,Line3}` | `Line2` | GPIO 同步線。 |
| `--target-fps F` | auto | 主控器觸發率。 |
| `--binning {1,2,4}` | auto | 硬體像素合併。當任何幀率支援此功能時，「auto」即代表使用完整感測器（不進行像素合併）；若設定為「never bins」，則僅為維持幀率而不進行像素合併。 |
| `--packed {auto,on,off}` | `auto` | [封裝式 12 位元傳輸](#packed-12-bit-transport): `auto` 僅在有助益且主機能負擔解包作業時才進行封裝。 |
| `--force-tier {sim-capture-sim-emit, sim-capture-ftd-stagger, slip-emit-and-capture}` | 自動 | 由專家覆寫同步層選擇器。 |
| `--wire-ceiling-mbps MB_PER_S` | 自動偵測 | 主機線路預算（單位：MB/s）——此為 **線路預算** 欄位的 CLI

形式。若陣列回報幀損壞，請降低此值。此設定會隨專案儲存，因此日後重新連線時將恢復此設定。 |
| `--no-recommend` | 關閉 | 跳過網路分析步驟。 |
| `--no-ptp` | 關閉 | 停用 PTP（此時不同攝影機之間的時間戳記將無法相互比較）。 |

`lattice array-list`、`array-status` 及 `array-disconnect` 用於管理持久性連線。 完整的子指令參考，包括對齊（`align-calibrate` / `align-apply`）及網路工具，請參閱 [CLI

參考 § chloros-cli lattice](../reference/cli-reference.md#chloros-cli-lattice)；對應的 SDK

指令（`connect_array`、`ArraySession`、`attach_array`、`analyze_array_network`）則收錄於 [SDK

參考手冊](../reference/sdk-reference.md) 中。 在 Python

中，線路預算為 `connect_array(..., wire_ceiling_mbps=120)`，而「有效/損毀」與「未送達」的分類則位於 [`/api/camera/array/<id>/capability`](../reference/sdk-reference.md#array-health--which-subsystem-is-losing-frames)。
