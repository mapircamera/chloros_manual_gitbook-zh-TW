# 圖形使用者介面 (GUI)：導覽

首次啟動 Chloros 時，系統會先啟動其處理後端。一旦後端準備就緒，左上角的主選單圖示便會顯示出來 <img src=".gitbook/assets/image (1) (1) (1) (1).png" alt="" data-size="line">，而左側邊欄中的「攝影機」與「光感測器」分頁也會解鎖（在此之前，這些分頁會呈現灰色）。

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

頂部標題列從左至右包含：

### <img src=".gitbook/assets/image (1) (1) (1) (1).png" alt="" data-size="line"> 主選單

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>透過主選單，您可以：

* **新建專案**— 建立新專案。若您已儲存專案範本，將出現**選取範本** 下拉選單，讓新專案能基於範本的設定開始建置。
* **開啟專案**— 開啟現有專案。 清單中包含一個**開啟專案資料夾** 按鈕，可於您的檔案總管中開啟專案資料夾。
* **複製專案** — 將目前開啟的專案以新名稱複製（建議使用自由命名，例如「MyProject (2)」），並開啟該副本。 _(僅在開啟專案後顯示)_
* **新增檔案** — 將單一影像檔案新增至當前專案 _(僅在開啟專案後顯示)_
* **新增資料夾** — 將一個或多個影像資料夾新增至當前專案 _(僅在開啟專案後顯示)_
* **開始處理 / 停止處理** — 啟動或停止影像處理流程 _(在新增檔案後啟用)_
* **連接至相機** — 跳轉至 [「相機」分頁](lattice/) 以連接 LATTICE 相機或陣列。無需開啟專案即可使用。
* **連接光感測器** — 跳轉至 [光感測器分頁](daq/) 以連接 DAQ 光感測器。無需開啟專案即可使用。

{% hint style="info" %}
**僅限 Windows**：Chloros 桌面 GUI 僅在 Windows 系統上提供。Linux 使用者應參閱 [CLI](CLI.md) 及 [Python SDK](api-python-sdk.md) 文件，以進行無頭處理。
{% endhint %}

### <img src=".gitbook/assets/image (2) (1) (1).png" alt="" data-size="line">

播放／開始按鈕

啟用後，開始處理按鈕將啟動影像處理流程。

### <img src=".gitbook/assets/image (4).png" alt="" data-size="line">

進度條 <img src=".gitbook/assets/image (5).png" alt="" data-size="line">



在免費的 Chloros 模式下，系統會依序處理所有檔案，進度條將顯示 2 個階段：目標偵測與處理。

在付費的 Chloros+ 授權模式下，系統會同時處理所有檔案，進度條將顯示 4 個階段：偵測、分析、校準、匯出。 若將滑鼠游標懸停在Chloros+ 進度條上，將會下拉顯示擴展的 4 階段進度條面板，以便您追蹤進度。點擊頂部的進度條將凍結下拉面板，再次點擊則解除凍結。

<figure><img src=".gitbook/assets/plus_prog.JPG" alt=""><figcaption></figcaption></figure>

## 側邊選單

左側邊欄選單包含各種可互動的圖示，由上而下的順序如下：

#### <img src=".gitbook/assets/icon_project-settings.JPG" alt="" data-size="line"> [專案設定](project-settings/project-settings.md)

「專案設定」分頁可讓您調整專案的全域設定及處理設定。 請在開始處理檔案前先調整這些設定。

#### <img src=".gitbook/assets/icon_file-browser.JPG" alt="" data-size="line"> 檔案瀏覽器

可將檔案／資料夾新增至專案，或從專案中移除檔案。系統會忽略重複的檔案。若勾選任何目標影像旁的欄位，處理過程將僅針對已勾選的影像進行目標比對，大幅加快處理速度。 使用「影像／元資料」切換鈕，可在檢視選取影像的縮圖網格與詳細元資料表格之間切換。

#### <img src=".gitbook/assets/icon_image-viewer.JPG" alt="" data-size="line"> [影像檢視器](image-viewer-gui/opening-an-image-full-screen.md)

當在主影像檢視器中點擊某張影像時，該影像將在「影像檢視器」分頁中以全螢幕模式開啟。

#### <img src=".gitbook/assets/image (3) (1).png" alt="" data-size="line"> [地圖檢視器](image-viewer-gui/map-markers.md)

根據影像的 GPS 座標，在互動式 2D 地圖上檢視您的影像。 支援 Google 地圖與 ESRI 圖塊供應商，並會根據您的位置自動選擇最佳服務。將游標懸停於標記上即可查看影像縮圖預覽。

#### <img src=".gitbook/assets/image (17).png" alt="" data-size="line"> [攝影機](lattice/)

即時連接並控制 LATTICE 攝影機——可單台操作，或作為同步的多攝影機陣列使用。 此分頁顯示帶有疊加層與直方圖的即時預覽圖塊、各攝影機及陣列的設定，以及「擷取設定」——該設定可選擇「擷取全部」功能將產出的攝影機與匯出類型。待後端準備就緒後即可使用；請參閱 [LATTICE 專區](lattice/) 以獲取完整操作指南。

#### <img src=".gitbook/assets/image (23).png" alt="" data-size="line"> [光感測器](daq/)

連接 DAQ 光感測器 — DAQ-U（USB）、DAQ-M（藍牙）及 DAQ-E（乙太網路） — 並在 W/m²/nm 單位下檢視其即時校準光譜圖。 在此處，您可以將 `.daq` 檔案記錄至開啟中的專案、重新命名感測器、選擇遮光校正設定檔，以及更新 DAQ-E 韌體。此功能將於後端就緒後開放；請參閱 [DAQ 區段](daq/) 以獲取完整操作指南。

#### <img src=".gitbook/assets/icon_log.JPG" alt="" data-size="line"> 除錯日誌

發生問題時，請檢視日誌中的除錯輸出。複製／下載日誌並寄送至 [MAPIR 技術支援](https://www.mapir.camera/community/contact) 以獲取協助。

1.3.0 版新功能：日誌分頁頂端的錯誤圖示（**回報錯誤**）可將包含您的描述、最多 10 張圖片、Chloros 日誌及系統詳細資訊的報告傳送至 MAPIR。您必須先登入才能發送報告。 請參閱 [從日誌分頁回報錯誤](processing-images-gui/monitoring-the-processing.md#reporting-a-bug-from-the-log-tab)。

#### <img src=".gitbook/assets/icon_user.JPG" alt="" data-size="line"> [使用者登入](chloros+-login.md)

透過使用者登入側邊欄，您可以登入您的 Chloros+ 帳戶以解鎖進階功能。您亦可查看當前應用程式版本，並調整 Chloros 圖形使用者介面 (GUI) 及命令列介面 (CLI) 中顯示文字的語言。
