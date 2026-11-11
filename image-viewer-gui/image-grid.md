# 圖片網格

將圖片匯入專案後，您會在主區域看到它們以網格形式排列。此網格是您選擇 **要檢視每張圖片的哪個版本** 的位置——網格上方的按鈕可一次切換所有縮圖，在原始檔案與各處理後成果之間切換。

## 縮圖大小

請使用右上角的縮放滑桿來調整圖片縮圖的大小。滑桿的範圍為 **64 像素 至 1200 像素**。

* **Ctrl + 滑鼠滾輪** 亦可調整縮圖大小。
* **Ctrl + `+`**/**Ctrl + `=`**以及**Ctrl + `−`** 每次按下會將大小以 4 像素為單位逐步調整。 鍵盤操作的路徑在小端以 64 像素為限，而在大端則以當前視窗中每行能精確容納兩個縮圖的尺寸為上限。
* 您設定的大小會與專案一同儲存（位於 `UI → Grid thumbnail size` 中的 `project.json`，預設為 `160`），因此重新開啟專案時會恢復該設定。

<figure><img src="../.gitbook/assets/chloros_grid_zoom.gif" alt=""><figcaption></figcaption></figure>縮圖的 *解析度* 與 *大小* 是兩個獨立的設定：請參閱 [專案設定](../project-settings/project-settings.md) 中的 **顯示 → 圖片縮圖解析度**（預設為長邊 512 像素）。 「大小」是指磁磚的繪製尺寸；「解析度」則是填滿該磁磚所擷取的細節程度。***

## 網格工具列

網格上方的按鈕列最多分為三個群組，由左至右依序為：

1. **按觸發器 / 按相機** — 分組模式。僅在包含 LATTICE 擷取資料的專案中顯示。
2. **相機篩選按鈕** — 每個 LATTICE 相機對應一個。僅在「按相機」模式下顯示。
3. **匯出／檢視模式按鈕** — 每張縮圖所顯示的產品。

當視窗過窄而無法同時顯示所有按鈕時，各群組會由右至左收合為懸停下拉選單：首先收合的是匯出／檢視按鈕，接著是攝影機按鈕。 收合後的群組會留下一個標示當前選項的按鈕，將游標懸停其上即可將完整組別向下滑出。**「按觸發器 / 按相機」群組絕不會收合。**<!-- SCREENSHOT-NEEDED: Image grid toolbar of a LATTICE array project at full width, showing all three button groups inline: Per Trigger / Per Camera, three camera filter buttons labelled "LATT-M3M (serial)", and the export/view buttons including TIFF, RAW (Original), RAW (Radiance), RAW (Reflectance). -->***

## 匯出檢視按鈕

這些按鈕用於在網格縮圖中切換影像類型。**只要該按鈕所對應的產品存在，按鈕便會立即顯示** —— 對於原始檔案而言，這意味著在匯入時即刻顯示，而非處理完成後。Chloros

會在處理任務進行期間重新掃描專案中的產品，因此當每個產品開始寫入磁碟時，對應的按鈕便會在處理過程中顯示。

### 基本按鈕

最左側的匯出按鈕標示的是 **您實際匯入的內容**：

| 匯入內容 | 按鈕標籤 |
| --- | --- |
| Survey3

RAW+JPG | `JPG` |
| LATTICE 擷取檔（附帶與 RAW 畫框並列的顯示預覽） | `PNG` 或 `TIFF`（視預覽內容而定） |
| LATTICE 擷取且基礎檔案 **即為** RAW 影像的畫面 | *無按鈕* — `RAW (Original)` 已顯示該檔案 |

在混合專案中，標籤將採用多數影像所使用的副檔名。

### 產品按鈕

| 按鈕 | 顯示內容 | 出現時機 |
| --- | --- | --- |
| **標靶** | 已偵測到校準標靶的影像 | 在偵測到標靶的運行結束後 |
| **反射率** | 已校正的反射率影像 | 僅限 Survey3

專案 — LATTICE 專案改用 `RAW (Reflectance)`，因此網格中絕不會同時顯示兩個反射率按鈕 |
| **白平衡** | 白平衡後產品（RGB

相機） | 處理完成後 |
| **暗角校正** | 未經校正的暗角校正備用產品 | 在無法套用反射率校正且 *暗角校正* 已開啟的執行任務後 |
| **感測器響應** | 未經校準的感測器響應備用選項 | 情況相同，但 *暗角校正* 已關閉 |
| **`RAW (<INDEX> Index)`** | 每個計算出的指標對應一個按鈕 | 配置指標後執行一次 |
| **`<INDEX> LUT`** | 每個色彩映射指數對應一個按鈕 | 已配置 LUT 並執行後 |
| **`<Index> <Index\|LUT> <NNN>`** | 每個 [索引/LUT 沙盒](index-lut-sandbox.md) 匯出執行一次按鈕 | 沙盒匯出完成之際 |

### LATTICE 層級按鈕

包含 LATTICE 擷取資料的專案會新增這些按鈕，其標籤為層級名稱而非產品名稱：

| 按鈕 | 層級 |
| --- | --- |
| **RAW (原始)** | 導入時的原始原始畫格 |
| **RAW (輻照度)** | Float32 光譜輻照度，W/m²/sr/nm |
| **RAW (反射率)** | uint16 反射率，32768 = ρ 1.0 |

`RAW (Original)` 自匯入當下即已可用 — 無需任何處理。當 LATTICE 匯入完全沒有基礎按鈕時（每個擷取的基礎檔案即為其原始幀），網格會自動移動至第一個可用的層級按鈕，以便工具列的高亮顯示與您所見內容相符。

有兩種層級的 Chloros

匯出檔 **不會擁有專屬的網格按鈕**：

* **去拜耳化** — `RAW (Original)` 檢視畫面已渲染為去拜耳化狀態，因此在視覺上完全相同的影像上再增加第二個按鈕只會造成干擾。 `RAW (Debayered)` 產出檔仍會寫入磁碟，且仍可從全螢幕圖層下拉選單中選取。
* **預覽** — 在RGB

相機上，預覽會註冊為 `White Balanced` 圖層，該圖層確實具有按鈕。 在多光譜相機上，預覽會註冊為 `RAW (Preview)`，並可透過全螢幕圖層下拉選單存取。

{% hint style="info" %}
這些層級按鈕僅會針對實際包含 LATTICE 幀的專案進行渲染。Survey3

專案會註冊部分相同的內部圖層名稱，系統會對其過濾掉這些按鈕，因此 Survey3

網格會保留其熟悉的 `JPG / Targets / Reflectance` 圖層集。
{% endhint %}

點擊網格縮圖將開啟全螢幕 [影像檢視器](opening-an-image-full-screen.md)，顯示 **與該網格所呈現之產品相同** 的內容 ——若網格設定為 `Targets`，則縮圖會開啟已匯出的目標影像。

<figure><img src="../.gitbook/assets/chloros_grid_mode.gif" alt=""><figcaption></figcaption></figure>
<!-- SCREENSHOT-UPDATE: This GIF predates the LATTICE level buttons and the toolbar group separators. Reshoot on a LATTICE project cycling base -> RAW (Original) -> RAW (Radiance) -> RAW (Reflectance) -> an index button, so the new button set and the level names are visible. -->



***

## 群組化 LATTICE 專案：依觸發器 vs 依相機

陣列擷取會從不同的相機模組產生同一瞬間的多張影像。 分組方式決定了網格如何堆疊這些影像。兩種模式都會顯示全寬的可收合標題欄；**每個群組預設為展開狀態**，且 Chloros

會記住您關閉的群組。收合狀態會依模式分別追蹤，因此若在「按相機」模式下關閉某個群組，並不會影響「按觸發」模式中的任何群組。

### 按相機（預設）

每個相機模組對應一個群組。標題列會顯示相機型號與序號（`LATT-M3M — <serial>`）以及照片數量。群組內的圖塊會依照擷取事件的時間順序排列。

在此模式下，工具列還會針對每台相機顯示一個 **相機篩選按鈕**，標籤為 `MODEL (SERIAL)`。所有相機預設皆為選取狀態；點擊按鈕可取消選取該相機，並將其群組從網格中移除。這是快速檢視整趟飛行中單一波段影像的便捷方式。

### 按觸發器

每個擷取事件對應一個群組——即所有模組在同一觸發器下拍攝的幀組。頁首顯示擷取時間、參與拍攝的相機數量，以及群組中每種相機型號的標記。 群組內的圖塊依相機序列號排序，因此每個觸發事件中，同一波段的圖塊都會位於同一欄位。

混合專案中的<!-- SCREENSHOT-NEEDED: Image grid in Per Trigger mode for a 3-camera LATTICE array, showing two consecutive trigger groups with their header bars (chevron, capture timestamp, "3 cameras", and the three model badges) and one group collapsed to show the closed state. -->

非 LATTICE 影像不會被分組 — 它們會在群組之後以普通圖塊的形式呈現。

***

## 網格縮圖遵循 GSD 區塊大小

若您已在「影像」分頁的側邊欄中設定了 **GSD (px)** 區塊大小，網格縮圖將以相同的地面解析度呈現——而不僅限於全螢幕檢視模式。 當區塊大小設定為 8 時，應用程式中所有顯示影像的位置，每個顯示像素皆為 8 × 8 原始像素區塊的平均值。

由於磁磚的原始寬度通常只有幾百像素，因此較粗糙的區塊大小在網格中變得難以察覺的時機，遠早於全螢幕檢視中顯現差異的時機：將一個 4000 像素的圖框繪製到 160 像素的磁磚中時，每個顯示像素已相當於約 25 個原始像素。 關於該控制項本身，請參閱 [以全螢幕開啟圖片](opening-an-image-full-screen.md#gsd-block-size)。

***

## 相關頁面

* [**以全螢幕開啟影像**](opening-an-image-full-screen.md) — 全螢幕檢視器、游標數值與直方圖
* [**影像圖層**](image-layers.md) — 全螢幕檢視器內的圖層下拉選單
* [**索引/LUT 沙盒**](index-lut-sandbox.md) — 建立與匯出索引視覺化效果
* [**專案設定**](../project-settings/project-settings.md) — 決定哪些產品會存在的匯出切換開關
