---
description: This page lists some multispectral indices that Chloros uses
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/o044KN3Ws0uIDvOmSkcR/multispectral-index-formulas
---

# 多光譜指數公式

以下指數公式採用Survey3濾光片平均透射率範圍的組合：

<table><thead><tr><th align="center">Survey3 濾鏡顏色</th><th width="196.199951171875" align="center">Survey3 濾光片名稱</th><th width="159.800048828125" align="center">透射率範圍（FWHM）</th><th align="center">平均透射率</th></tr></thead><tbody><tr><td align="center">Blue</td><td align="center">NGB - Blue</td><td align="center">468-483nm</td><td align="center">475 奈米</td></tr><tr><td align="center">Cyan</td><td align="center">OCN- Cyan</td><td align="center">476-512 奈米</td><td align="center">494 奈米</td></tr><tr><td align="center">Green</td><td align="center">RGN | NGB - Green</td><td align="center">543-558 奈米</td><td align="center">547 奈米</td></tr><tr><td align="center">Orange</td><td align="center">OCN - Orange</td><td align="center">598-640奈米</td><td align="center">619 奈米</td></tr><tr><td align="center">Red</td><td align="center">RGN - Red</td><td align="center">653-668 奈米</td><td align="center">661 奈米</td></tr><tr><td align="center">RedEdge</td><td align="center">Re - RedEdge</td><td align="center">712-735 奈米</td><td align="center">724 奈米</td></tr><tr><td align="center">NIR1</td><td align="center">OCN - NIR1</td><td align="center">798-848 奈米</td><td align="center">823 奈米</td></tr><tr><td align="center">NIR2</td><td align="center">RGN | NGB | NIR - NIR2</td><td align="center">835-865 奈米</td><td align="center">850 nm</td></tr></tbody></table>

使用這些公式時，名稱可能會以「\_1」或「\_2」結尾，這代表所使用的NIR濾光片類型，即NIR1或NIR2。

對於 LATTICE M3C（拜耳三波段通濾光片）相機，相同的索引引擎會採用 M3C 濾光片波段：

| M3C 濾光片 | 波段 1（中心波長／FWHM） | 波段 2（中心波長／FWHM） | 波段 3（中心波長／FWHM） |
| --- | --- | --- | --- |
| FRGB | Blue 475 nm / 30 nm | Green 550 nm / 30 nm | Red 625 nm / 30 nm |
| FRGN | Red 660 nm / 21 nm | Green 550 nm / 30 nm | NIR 850 nm / 30 nm |
| FOCN | Orange 615 nm / 21 nm | Cyan 490 nm / 38 nm | NIR 808 nm / 14 nm |
| FNGB | Blue 475 nm / 30 nm | Green 550 nm / 30 nm | NIR 850 nm / 30 nm |

LATTICE M3M 相機為單波段（每台相機配備一個窄帶濾光片），因此無法針對單張 M3M 影像計算多波段指數。 若要使用 M3M 計算指數，請將兩台或更多相機的影像組合成對齊的多波段疊加影像，並使用 LATTICE 指數引擎（`chloros-cli lattice index`，或 GUI 中的即時指數計算器）。

***

## 各索引名稱的適用範圍

Chloros擁有 **三個** 索引表面，且其預設清單並不完全相同。請利用本節確認您計畫使用的名稱是否適用於該處。

| 您所在的位置 | 適用清單 | 計數 |
| --- | --- | --- |
| 專案設定 → 索引 → 新增索引 (GUI) | 表面 1 | 27 |
| 影像檢視器 [索引/LUT 沙盒](../image-viewer-gui/index-lut-sandbox.md) (GUI) | 表面 1 | 27 |
| `chloros-cli process --indices NDVI,NDRE` | 表面 2 | 27 |
| SDK `process_folder(indices=[...])` | 表面 2 | 27 |
| `chloros-cli lattice index --preset` | 表面 3 | 22（另一個 22） |
| 「相機」分頁即時索引計算器 | 表面 3 | 22（另一個 22） |

Surface 1 和 2 會**一次處理來自單一攝影機的一張影像**， 並使用與該相機濾光片通道綁定的符號槽 `x`/`y`/`z`(/`a`)。 Surface 3 處理的是**對齊的多波段堆疊** —— 將多台 LATTICE 相機的影像配準整合為一個立方體 —— 並透過小寫名稱來引用通道。

### 1. GUI 專案設定 / 影像檢視器沙盒下拉選單 — 27 個公式

下拉選單按以下順序列出這些公式 （此為插入順序，非字母順序）：

`NDVI, GNDVI, CVI, ENDVI, EVI, MSR, OSAVI, TDVI, LAI, FCI1, FCI2, GARI, GCI, GEMI, GLI, GOSAVI, GRVI, GSAVI, LCI, MNLI, MSAVI2, NDRE, NLI, RDVI, SAVI, VARI, WDRVI`

在 GUI 中，您可將相機的濾光片通道拖曳至公式的波段插槽中，因此任何公式皆可搭配相機所支援的任何波段配置使用。您已儲存的自訂公式會附加於此清單下方。

那 **五個僅限 GUI 使用** 的公式 —— 也就是 CLI/SDK 的 `--indices` 清單不接受的那些 —— 其實作方式如下：

| 僅限 GUI 的預設值 | 公式（其實作方式） | 插槽 |
| --- | --- | --- |
| FCI1 | `x*y` | x, y |
| FCI2 | `x*y` | x, y |
| GARI | `(y-(x-1.7*(z-a)))/(y+(x-1.7*(z-a)))` | x, y, z, a（四個參數槽） |
| GEMI | `((2*(y*y-x*x)+1.5*y+0.5*x)/(y+x+0.5))*(1-0.25*((2*(y*y-x*x)+1.5*y+0.5*x)/(y+x+0.5)))-((x-0.125)/(1-x))` | x, y |
| LCI | `(y-x)/(y+z)` | x, y, z |

各公式的預期對應關係將於本頁後續的獨立章節中說明（例如 GARI 預期 x=Green、y=NIR、z=Blue、a=Red）。GARI 是 Chloros 中唯一使用第四個參數槽的公式。

### 2. CLI／SDK `--indices` 名稱擴展 — 22 個預設值

`chloros-cli process --indices` 選項（以及 SDK `indices` 參數）接受以下預設名稱：

`NDVI, GNDVI, NDRE, OSAVI, SAVI, MSAVI2, EVI, MSR, TDVI, LAI, GCI, GRVI, GSAVI, GOSAVI, NLI, MNLI, RDVI, WDRVI, CVI, ENDVI, GLI, VARI`

{% hint style="warning" %}
**未列出的索引名稱將被靜默跳過。** 此清單以外的名稱（包括五個僅限 GUI 使用的公式 `FCI1`、`FCI2`、`GARI`、`GEMI`、 `LCI`，以及您在 GUI 中儲存的任何自訂公式）將僅以日誌通知的方式被捨棄——執行程序將跳過該索引繼續進行，且執行結果仍會回報成功。該通知將顯示為：

```
[INDEX_EXPAND] skipping unknown preset 'LCI'; known: ['CVI', 'ENDVI', 'EVI', ...]
```

名稱在去除空格後進行匹配，且不區分大小寫，因此 `ndvi`、`NDVI` 和 ` NDVI ` 屬於同一個預設值。 若某個預設需要您的相機濾鏡未提供的波段，該預設也會被跳過。
{% endhint %}

實際實作的精確公式（符號 `x`/`y`/`z` 代表波段槽位；各預設值對應的預設映射如下所示）：

| 預設值 | 公式（實作版本） | 預設濾光片 | 槽位 (x, y, z) |
| --- | --- | --- | --- |
| NDVI | `(y-x)/(y+x)` | RGN | Red、NIR |
| GNDVI | `(y-x)/(y+x)` | RGN | Green、NIR |
| NDRE | `(y-x)/(y+x)` | RE | RE、NIR |
| OSAVI | `(y-x)/(y+x+0.16)` | RGN | Red、NIR |
| SAVI | `1.5*(y-x)/(y+x+0.5)` | RGN | Red, NIR |
| MSAVI2 | `(2*y+1-sqrt((2*y+1)*(2*y+1)-8*(y-x)))/2` | RGN | Red, NIR |
| EVI | `2.5*(y-x)/(y+6*x-7.5*z+1)` | RGN | Red、NIR、Blue |
| MSR | `((y/x)-1)/(sqrt(y/x)+1)` | RGN | Red、NIR |
| TDVI | `1.5*(y-x)/sqrt(y*y+x+0.5)` | RGN | Red、NIR |
| LAI | `3.618*(2.5*(y-x)/(y+6*x-7.5*z+1))-0.118` | RGN | Red、NIR、Blue |
| GCI | `(y/x)-1` | RGN | Green、NIR |
| GRVI | `y/x` | RGN | Green、NIR |
| GSAVI | `1.5*(y-x)/(y+x+0.5)` | RGN | Green, NIR |
| GOSAVI | `(y-x)/(y+x+0.16)` | RGN | Green, NIR |
| NLI | `((y*y)-x)/((y*y)+x)` | RGN | Red, NIR |
| MNLI | `((y*y-x)*(1+0.5))/((y*y)+x+0.5)` | RGN | Red, NIR |
| RDVI | `(y-x)/sqrt(y+x)` | RGN | Red, NIR |
| WDRVI | `(0.2*y-x)/(0.2*y+x)` | RGN | Red, NIR |
| CVI | `(z/y)*(x/y)` | RGN | Red, Green, NIR |
| ENDVI | `((x+y)-(2*z))/((x+y)+(2*z))` | NGB | NIR, Green, Blue |
| GLI | `((y-x)+(y-z))/((2*y)+x+z)` | RGB | Red、Green、Blue |
| VARI | `(y-x)/(y+x-z)` | RGB | Red、Green、Blue |

#### 預設名稱如何轉化為波段位置

當您傳入如 `NDVI` 這樣的純名稱時，Chloros 必須決定每個符號讀取的是哪個檔案的哪個通道。它會使用這張對照表，將濾波器代碼映射至各通道的陣列位置：

| 濾波器代碼 | 通道 → 陣列索引 |
| --- | --- |
| OCN | Orange 0、Cyan 1、NIR 2（`Red` 被視為 Orange 的別名，同樣為 0） |
| RGN | Red 0、Green 1、NIR 2 |
| NGB | NIR 0, Green 1, Blue 2 |
| RGB | Red 0,  若無，Chloros 會依 `RGN, OCN, NGB, RGB, RE, NIR` 的順序掃描專案中實際存在的濾光片，並選取第一個能提供預設值所需所有通道的濾光片。若無任何濾光片符合條件，則該預設值將在該執行階段中被捨棄。 這就是為什麼在僅含 OCN 的資料集上請求 `NDVI` 仍能產生合理結果的原因——它會綁定至 OCN 的 Orange 和 NIR 位置。

LATTICE M3C 模型字串會以 `F` 前綴攜帶濾光片 (`LATT-M3C-L41-FRGN`)，但當從影像讀取濾鏡代碼時，該前綴會被省略，因此 FRGN 相機可透過上方的 `RGN` 行進行解析，無需特殊處理。

### 3. LATTICE 索引引擎 (`lattice index --preset`，即時索引計算器) — 22 組預設值

LATTICE 引擎適用於對齊的多波段堆疊影像（即時陣列或匯出的多波段 TIFF 檔案），並使用小寫通道名稱（`red`、`green`、 `blue`、`red_edge`、`nir`）。其預設清單與上述兩者不同：

| 預設值 | 公式 | 通道 |
| --- | --- | --- |
| NDVI | `(nir - red) / (nir + red)` | 紅光、近紅外 |
| GNDVI | `(nir - green) / (nir + green)` | 紅光、近紅外 |
| BNDVI | `(nir - blue) / (nir + blue)` | 藍光、近紅外 |
| NDRE | `(nir - red_edge) / (nir + red_edge)` | 紅邊、近紅外 |
| ENDVI | `((nir + green) - 2*blue) / ((nir + green) + 2*blue)` | 藍、綠、近紅外 |
| SAVI | `1.5 * (nir - red) / (nir + red + 0.5)` | 紅、近紅外 |
| OSAVI | `(nir - red) / (nir + red + 0.16)` | 紅, 近紅外 |
| MSAVI | `(2*nir + 1 - sqrt((2*nir + 1)**2 - 8*(nir - red))) / 2` | 紅, 近紅外 |
| EVI | `2.5 * (nir - red) / (nir + 6*red - 7.5*blue + 1)` | 藍, 紅, 近紅外 |
| EVI2 | `2.5 * (nir - red) / (nir + 2.4*red + 1)` | 紅光、近紅外光 |
| CVI | `nir * red / green**2` | 紅光、綠光、近紅外光 |
| MSR | `((nir/red) - 1) / (sqrt(nir/red) + 1)` | 紅光、近紅外光 |
| TDVI | `1.5 * (nir - red) / sqrt(nir**2 + red + 0.5)` | 紅光、近紅外光 |
| LAI | `3.618 * (2.5 * (nir - red) / (nir + 6*red - 7.5*blue + 1)) - 0.118` | 藍色、紅色、近紅外線 |
| GLI | `(2*green - red - blue) / (2*green + red + blue)` | 紅色、綠色、藍色 |
| NGRDI | `(green - red) / (green + red)` | 紅、綠 |
| VARI | `(green - red) / (green + red - blue)` | 紅、綠、藍 |
| TGI | `green - 0.39*red - 0.61*blue` | 紅色、綠色、藍色 |
| EXG | `2*green - red - blue` | 紅色、綠色、藍色 |
| CIRE | `(nir / red_edge) - 1` | 紅_邊緣, 近紅外 |
| CIGREEN | `(nir / green) - 1` | 綠, 近紅外 |
| NDWI | `(green - nir) / (green + nir)` | 綠色、近紅外線 |

執行 `chloros-cli lattice index --list-presets` 可從您已安裝的版本中列印此表格， 並執行 `--list-gradients` 以取得可用的色彩漸層。通道符號區分大小寫，且必須與預設值的小寫名稱完全一致（例如：`--channel red=Red_660 --channel nir=NIR_850`）。

***

## CVI - 葉綠素植被指數

此指數對廣泛樹冠密度範圍內的葉片葉綠素含量相當敏感。它使用NIR、紅光和綠光波段。

$$
CVI = NIR * {Red \over Green^{2}}
$$

在圖形使用者介面（GUI）以及命令列介面（CLI）／軟體開發套件（SDK）的預設清單中，其符號公式為 `(z/y)*(x/y)`，其中 x=Red、y=Green、z=NIR （即RGN相機的波段）；LATTICE 指數引擎的預設值為 `nir * red / green**2`。每個表面皆計算相同的指數。

_參考文獻：Vincini, M.、E. Frazzi 及 P. D'Alessio。《冠層尺度的寬頻葉綠素植被指數》。《精準農業》第 9 卷（2008）：303-319。_

***

## ENDVI - 增強型歸一化差值植被指數

此指數除NIR與綠色通道外，亦使用藍色通道，並廣泛應用於藍色波段取代紅色波段的NGB濾光相機。

$$
ENDVI = {(NIR + Green) - (2 * Blue) \over (NIR + Green) + (2 * Blue)}
$$

其實作採用符號公式 `((x+y)-(2*z))/((x+y)+(2*z))` — 請將相機的NIR與Green通道分配至x/y槽位，並將Blue分配至z槽位（以NGB相機為例：x=NIR，y=Green，z=Blue）。

***

## EVI - 增強植被指數

此指數最初是為配合 MODIS 數據而開發，旨在透過優化葉面積指數（LAI）較高的區域中的植被訊號，來改進 NDVI 的表現。它在 NDVI 可能飽和的高 LAI 區域中最具實用價值。該指數利用藍色反射率區域來校正土壤背景訊號，並減少大氣影響（包括氣溶膠散射）。

$$
EVI = 2.5 *  {(NIR - Red) \over (NIR + 6 * Red - 7.5 * Blue + 1)}
$$

對於植被像素，EVI 值應介於 0 至 1 之間。雲層和白色建築物等明亮特徵，以及水體等暗色特徵，都可能導致 EVI 影像中的像素值出現異常。在建立 EVI 影像之前， 應先從反射率影像中遮罩掉雲層與高亮度特徵，並可選擇性地將像素值閾值化為 0 至 1 之間。

_參考文獻：Huete, A. 等。「MODIS 植被指數的輻測與生物物理性能綜述」。《遙感環境》83 (2002):195–213._

***

## FCI1 - 森林覆蓋指數 1

_僅限 GUI — 無法作為 CLI/SDK 的 `--indices` 預設值使用._

此指標利用包含紅邊波段的多光譜反射率影像，將森林樹冠與其他類型的植被區分開來。

$$
FCI1 = Red * RedEdge
$$

由於樹木的反射率較低，且樹冠內存在陰影，林地區域的FCI1值會較低。

_參考文獻：Becker, Sarah J.、Craig S.T. Daughtry 及 Andrew L. Russ。 〈多光譜影像的穩健森林覆蓋指數〉。《攝影測量工程與遙感》第 84 卷第 8 期（2018 年）：505-512 頁。_

***

## FCI2 - 森林覆蓋指數 2

_僅限 GUI — 無法作為 CLI/SDK 的 `--indices` 預設值使用._

此指標利用不含紅邊波段的多光譜反射率影像，將森林樹冠與其他類型的植被區分開來。

$$
FCI2 = Red * NIR
$$

由於樹木的反射率較低，且樹冠內存在陰影，林地區域的FCI2值會較低。

_參考文獻：Becker, Sarah J.、Craig S.T. Daughtry 及 Andrew L. Russ。 〈多光譜影像的穩健森林覆蓋指數〉。《攝影測量工程與遙感》第 84 卷第 8 期（2018 年）：505-512 頁。_

***

## GEMI - 全球環境監測指數

_僅限 GUI — 無法作為 CLI/SDK 的 `--indices` 預設值使用。_

此非線性植被指數用於透過衛星影像進行全球環境監測，並試圖校正大氣效應。 它與NDVI類似，但對大氣效應的敏感度較低。由於會受到裸土的影響，因此不建議用於植被稀疏或中等密度的區域。

$$
GEMI = eta (1 - 0.25 * eta) - {Red - 0.125 \over 1 - Red}
$$

其中：

$$
eta = {2(NIR^{2}-Red^{2}) + 1.5 * NIR + 0.5 *  Red \over NIR + Red + 0.5}
$$

_參考文獻：Pinty, B. 及 M. Verstraete。《GEMI：一種透過衛星監測全球植被的非線性指數》。《植被》101 (1992)：15-20._

***

## GARI — 抗大氣干擾綠色指數 (Green)

_僅限 GUI — 無法作為 CLI/SDK 的 `--indices` 預設值使用。_

相較於 NDVI，此指數對廣泛範圍的葉綠素濃度更為敏感，且對大氣效應的敏感度較低。

$$
GARI = {NIR - [Green - \gamma(Blue - Red)] \over NIR + [Green - \gamma(Blue - Red)]   }
$$

伽瑪常數是一種取決於大氣中氣溶膠狀況的加權函數。ENVI 採用 1.7 這個數值，此為 Gitelson、Kaufman 及 Merzylak 所建議的數值 （1996 年，第 296 頁）所建議的數值。

_參考文獻：Gitelson, A., Y. Kaufman 及 M. Merzylak。《利用綠色通道透過 EOS-MODIS 進行全球植被遙感》。《環境遙感》第 58 卷（1996 年）： 289-298._

***

## GCI - 綠色葉綠素指數（Green Chlorophyll Index）

此指數用於估算廣泛植物物種的葉片葉綠素含量。

$$
GCI = {NIR \over Green} - 1
$$

具備廣泛的NIR與綠色波長範圍，不僅能更精準地預測葉綠素含量，同時也能提升靈敏度並獲得更高的訊號雜訊比。

_參考文獻：Gitelson, A., Y. Gritz, 及 M. Merzlyak. 「葉片葉綠素含量與光譜反射率之間的關係，以及高等植物葉片非破壞性葉綠素評估演算法。」《植物生理學期刊》160 (2003): 271-282._

***

## GLI - 綠葉指數 (Green Leaf Index)

此指數最初是為配合數位 RGB 攝影機測量小麥覆蓋率而設計，其中紅、綠、藍三色的數位數值 (DN) 範圍為 0 至 255。

$$
GLI = {(Green - Red) + (Green - Blue)  \over (2 * Green) + Red + Blue }
$$

GLI的數值範圍為-1至+1。負值代表土壤及非生物特徵，而正值則代表綠葉與莖稈。

_參考文獻：Louhaichi, M.、M. Borman 及 D. Johnson。 〈利用定位平台與航空攝影記錄放牧對小麥的影響〉。《Geocarto International》第 16 卷第 1 期（2001）：65-70。_

***

## GNDVI —— Green 歸一化差值植被指數

此指數與NDVI相似，惟其測量的是540至570 nm的綠色光譜，而非紅色光譜。相較於NDVI，此指數對葉綠素濃度的敏感度更高。

$$
GNDVI = {(NIR - Green) \over (NIR + Green)  }
$$

_參考文獻：Gitelson, A. 及 M. Merzlyak。《高等植物葉片葉綠素濃度的遙感測量》。《太空研究進展》第 22 卷（1998）：689-692。_

***

## GOSAVI — 綠色優化土壤校正植被指數（Green Optimized Soil Adjusted Vegetation Index）

此指數最初是針對彩色紅外線攝影設計，用於預測玉米的氮需求量。它與 OSAVI 相似，但以綠色波段取代紅色波段。

$$
GOSAVI = {NIR - Green \over NIR + Green + 0.16)  }
$$

_參考文獻：Sripada, R. 等。「利用航空彩色紅外線攝影測定玉米生長季氮需求量。」博士論文，北卡羅來納州立大學，2005年。_

***

## GRVI - 綠比植被指數（Green Ratio Vegetation Index）

此指數對森林樹冠層的光合作用速率相當敏感，因為綠色與紅色的反射率會受到葉片色素變化顯著影響。

$$
GRVI = {NIR \over Green }
$$

_參考文獻：Sripada, R. 等。「利用航空彩色紅外線攝影測定玉米生長初期氮需求量」。《農藝學期刊》98 (2006)：968-977。_

***

## GSAVI — 經綠色土壤校正的植被指數（Green Soil Adjusted Vegetation Index）

此指數最初是配合彩色紅外線攝影技術設計，用於預測玉米的氮需求。它與 SAVI 相似，但將紅色波段替換為綠色波段。

$$
GSAVI = 1.5 * {(NIR - Green) \over (NIR + Green + 0.5)  }
$$

_參考文獻：Sripada, R. 等。「利用航空彩色紅外線攝影測定玉米生長季氮需求量」。博士論文，北卡羅來納州立大學，2005年。_

***

## LAI - 葉面積指數

此指數用於估算葉片覆蓋率，並預測作物生長與產量。ENVI 採用 Boegh 等人（2002）提出的以下經驗公式來計算綠色 LAI：

$$
LAI = 3.618 * EVI - 0.118
$$

其中 EVI 定義如下：

$$
EVI = 2.5 *  {(NIR - Red) \over (NIR + 6 * Red - 7.5 * Blue + 1)}
$$

LAI 的高值通常約介於 0 至 3.5 之間。然而，當影像中包含雲層及其他會產生飽和像素的明亮特徵時，LAI 值可能會超過 3.5。在建立 LAI 影像之前，您最好先將雲層和明亮特徵從影像中遮罩掉。

_參考文獻：Boegh, E.、H. Soegaard、N. Broge、C. Hasager、N. Jensen、K. Schelde 及 A. Thomsen。《用於量化農業中葉面積指數、氮濃度及光合作用效率的機載多光譜資料。」《環境遙感》第 81 卷，第 2-3 期（2002）：179-193。_

***

## 葉綠素指數（LCI）

_僅限 GUI — 無法作為 CLI/SDK 的 `--indices` 預設值使用._

此指數用於估算高等植物中的葉綠素含量，對葉綠素吸收所引起的反射率變化具有敏感性。

$$
LCI = {NIR2 - RedEdge \over NIR2 + Red}
$$

_參考文獻：Datt, B. 「桉樹葉片含水量的遙感測量。」《植物生理學期刊》154 卷，第 1 期 (1999)：30-36._

***

## MNLI - 修正非線性指數

此指標是對非線性指標（NLI）的改進版本，其整合了土壤校正植被指數（SAVI）以考量土壤背景的影響。ENVI 採用 0.5 作為冠層背景校正因子（_L_）的數值。

$$
MNLI = {(NIR^{2} - Red) * (1 + L) \over (NIR^{2} + Red + L)  }
$$

_參考文獻：Yang, Z.、P. Willis 及 R. Mueller。《帶比增強型 AWIFS 影像對作物分類精確度的影響》。《Pecora 17 遙感研討會論文集》（2008），科羅拉多州丹佛市。_

***

## MSAVI2 - 改良型土壤校正植被指數 2

此指數是 Qi 等人 （1994）所提出的 MSAVI 指數的簡化版本，該指數對土壤校正植被指數（SAVI）進行了改進。它能降低土壤噪聲並增加植被訊號的動態範圍。MSAVI2 基於一種歸納法，不使用恆定的 _L_ 值（如 SAVI 所做）來突顯健康的植被。

$$
MSAVI2 = {2 * NIR + 1 - \sqrt{(2 * NIR + 1)^{2} - 8(NIR - Red)} \over 2}
$$

_參考文獻：Qi, J.、A. Chehbouni、A. Huete、Y. Kerr 及 S. Sorooshian。《改良土壤調整植被指數》（A Modified Soil Adjusted Vegetation Index）。《遙感環境》第 48 卷 (1994)：119-126。_

***

## MSR - 改良簡單比值

此指數是對簡單NIR/Red比值的改良版本，旨在使其與生物物理參數的關係線性化，且在植被密度較高的情況下，其敏感度高於NDVI。

$$
MSR = {(NIR / Red) - 1 \over \sqrt{NIR / Red} + 1}
$$

_參考文獻：Chen, J.〈植被指數與改良簡單比值在北方林區應用之評估〉。《加拿大遙感期刊》22 (1996)：229-242._

***

## NDRE — 歸一化差值RedEdge

此指標與NDVI相似，但將NIR與RedEdgeEdge進行對比，而非Red，這通常能更早偵測到植被壓力。

$$
NDRE = {NIR - RedEdge \over NIR + RedEdge  }
$$

***

## NDVI — 歸一化差值植被指數

此指標用於衡量健康、綠色的植被。其標準化差值公式結合了葉綠素吸收率與反射率最高的波段，使其在廣泛的環境條件下均表現穩健。然而， 在植被密集且葉面積指數（LAI）偏高的情況下，該指數可能會飽和。

$$
NDVI = {NIR - Red \over NIR + Red  }
$$

此指數的數值範圍為 -1 至 1。綠色植被的常見數值範圍為 0.2 至 0.8。

_參考文獻：Rouse, J.、R. Haas、J. Schell 及 D. Deering。《利用 ERTS 監測大平原的植被系統》。第三屆 ERTS 研討會，美國國家航空暨太空總署 (1973)：309-317。_

***

## NLI - 非線性指數

此指標假設許多植被指標與地表生物物理參數之間的關係為非線性。它將傾向於呈現非線性特徵的地表參數關係線性化。

$$
NLI = {NIR^{2} - Red \over NIR^{2} + Red  }
$$

_參考文獻：Goel, N. 及 W. Qin。「冠層結構對各種植被指數與LAI及Fpar之間關係的影響：電腦模擬。」《遙感評論》10 (1994)：309-347._

***

## OSAVI — 優化土壤校正植被指數

此指數基於土壤校正植被指數（SAVI）。其將冠層背景校正因子設定為標準值 0.16。 Rondeaux（1996）指出，在植被覆蓋率較低的情況下，此數值能比SAVI提供更大的土壤變異性，同時對植被覆蓋率超過50%的情況展現出更高的敏感度。此指數最適合應用於植被相對稀疏、且土壤可透過樹冠層看見的區域。

$$
OSAVI = {(NIR - Red) \over (NIR + Red + 0.16)  }
$$

_參考文獻：Rondeaux, G., M. Steven 及 F. Baret。《土壤校正植被指數的優化》。《環境遙感》第 55 卷（1996）：95-107。_

***

## RDVI - 重歸一化植被差異指數

此指數利用近紅外線與紅光波長之間的差異，並結合NDVI，以突顯健康的植被。它對土壤及太陽觀測幾何效應不敏感。

$$
RDVI = {(NIR- Red) \over \sqrt{(NIR + Red)}  }
$$

_參考文獻：Roujean, J. 及 F. Breon。《根據雙向反射率測量估算植被吸收的有效光合輻射》。《環境遙感》第 51 卷（1995）：375-384。_

***

## SAVI - 土壤校正植被指數

此指數與 NDVI 相似，但能抑制土壤像素的影響。它採用冠層背景校正因子 _L_，該因子是植被密度的函數，通常需要事先了解植被量。 Huete（1988）建議將 _L_ 設為 0.5 作為最佳值，以考量一階土壤背景變異。此指數最適用於植被相對稀疏、且土壤可透過樹冠層看見的區域。

$$
SAVI = {1.5 * (NIR- Red) \over (NIR + Red + 0.5)  }
$$

_參考文獻：Huete, A. 「經土壤校正的植被指數（SAVI）。」《遙感環境》25 (1988): 295-309._

***

## TDVI - 轉換差值植被指數

此指標適用於監測都市環境中的植被覆蓋狀況。它不會像NDVI和SAVI那樣產生飽和現象。

$$
TDVI = 1.5 * {(NIR- Red) \over \sqrt{NIR^{2} + Red + 0.5}  }
$$

_參考文獻：Bannari, A.、 H. Asalhi 及 P. Teillet。《用於植被覆蓋製圖的轉換差值植被指數（TDVI）》，收錄於《地球科學與遙感研討會論文集》（IGARSS '02），IEEE 國際，第 5 卷（2002）。_

***

## VARI - 可見光抗大氣干擾指數

此指數基於 ARVI，用於估算場景中的植被比例，且對大氣效應的敏感度較低。

$$
VARI = {Green - Red \over Green + Red - Blue  }
$$

_參考文獻：Gitelson, A., 等人。「可見光光譜空間中的植被與土壤分界線：植被佔比遙感估算的概念與技術」。《國際遙感期刊》第 23 卷（2002）：2537−2562。_

***

## WDRVI — 寬動態範圍植被指數

此指數與 NDVI 相似，但採用權重係數 (_a_) 來降低近紅外線與紅光訊號對 NDVI 貢獻之間的差異。當 NDVI 值超過 0.6 時，WDRVI 在植被密度中等至較高的場景中特別有效。 當植被覆蓋率與葉面積指數（LAI）增加時，NDVI 往往會趨於平穩，而 WDRVI 則對更廣泛的植被覆蓋率範圍以及 LAI 的變化更為敏感。

$$
WDRVI = {(\alpha * NIR- Red) \over (\alpha * NIR + Red)}
$$

加權係數（_a_）的範圍可介於 0.1 至 0.2 之間。Henebry、Viña 及 Gitelson（2004）建議採用 0.2 作為該值。

_參考文獻_

_Gitelson, A. 〈用於遙感量化植被生物物理特性的寬動態範圍植被指數〉。《植物生理學期刊》161 卷，第 2 期（2004）：165-173。_

_Henebry, G., A. Viña 及 A. Gitelson。「寬動態範圍植被指數及其在缺口分析中的潛在應用。」《缺口分析公報》第 12 期：50-56 頁。_
