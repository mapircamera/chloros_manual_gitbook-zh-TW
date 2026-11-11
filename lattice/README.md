# LATTICE 相機

LATTICE 是 MAPIR 專為農業與科學成像所設計的模組化多光譜相機系統。每台 LATTICE 相機均採用 Sony IMX265 全域快門感測器（**

3.1 MP，3.45 µm 像素**）打造，並透過乙太網路作為**GigE Vision** 裝置進行連接。

Chloros 1.3.0 可透過三個介面即時控制 LATTICE 相機——包括裝置偵測、即時預覽、影像擷取以及同步多相機陣列：

| 介面    | 位置                                                          | 支援平台                                                |
| ---------- | -------------------------------------------------------------- | -------------------------------------------------------- |
| 圖形使用者介面 (GUI) | Chloros 側邊欄中的 **相機** 標籤                         | Windows 10/11 x64                                        |
| CLI        | `chloros-cli lattice` 命令系列                           | Windows 10/11 x64、Linux x86\_64、Linux aarch64 (Jetson) |
| Python SDK | `chloros_sdk.connect_camera()` / `chloros_sdk.connect_array()` | Windows 10/11 x64、Linux x86\_64、Linux aarch64 (Jetson) |

> **正在尋找硬體嗎？**攝影機模組、鏡頭、濾光片與波段、機身與安裝支架、纜線、PoE 及觸發線路等相關資訊，皆記載於 [**LATTICE 使用手冊**](https://mapir.gitbook.io/lattice-camera)。本章節將說明如何透過 Chloros 驅動攝影機。

LATTICE 的擷取檔案為標準的 `.tif`/`.tiff` 格式，而 Chloros 始終會從原始擷取檔案開始進行處理。 請參閱 [CLI 參考手冊](../reference/cli-reference.md) 及 [SDK 參考手冊](../reference/sdk-reference.md)，以了解完整的命令與 API 介面。

## 兩種感測器配置

| 配置 | 感測器       | 濾波器                                | 單台攝影機的輸出內容                                          |
| ------------- | ------------ | ------------------------------------- | ----------------------------------------------------------------- |
| **M3C**| 拜耳彩色 | 三波段通濾光片                |**單次曝光即可獲得三個經校準波段**                 |
| **M3M**| 單色            | 單一窄帶干涉濾光片                                |**一個經校準的波段**；結合多台 M3M 相機以獲得指數值 |

由於 M3M 相機在單一濾光片後方為單色，因此每個波段都會有其專屬的曝光。 M3C 相機則透過單次感光器曝光即可涵蓋其所有三個波段。

## 型號字串與命名

每台相機皆將其識別碼儲存於 GenICam `DeviceUserID` 中，作為型號字串：

```
<sensor>-<lens>-F<filter>       e.g.  M3C-L41-FRGN,  M3M-L87-F450
```

Chloros 會以 `LATT-` 作為前綴來顯示（例如 `LATT-M3M-L87-F450`）。 相同的 `LATT-…` 字串會寫入每次匯出檔案的 EXIF `Model` 標籤中，並在已處理的專案中用作相機的輸出資料夾名稱。

| 元件 | 值                                                   | 意義                                                                                            |
| --------- | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| 感光元件    | `M3C` / `M3M`                                            | 拜耳色彩 / 單色                                                                          |
| 鏡頭      | `L41` / `L87`                                            | 該數字代表 **水平視場角（度）**： L41 = 窄視場 (41°)，L87 = 寬視場 (87°)    |
| 濾光片    | `FRGB` / `FRGN` / `FOCN` / `FNGB` (M3C) 或 `F<nm>` (M3M) | 請參閱 [濾光片與光譜波段](https://mapir.gitbook.io/lattice-camera/hardware/filters-and-bands) |

型號字串將驅動後續所有流程：Chloros 會從 `DeviceUserID` + `DeviceSerialNumber` 中解析感測器配置檔、波段佈局及出廠校準。 無需針對每台相機進行任何設定 — 請參閱 [連接相機](connecting.md)。

## 濾光片與光譜帶

光譜帶中心頻率、FWHM邊界以及完整的 23 項 SKU M3M 型錄均屬產品規格，因此收錄於硬體手冊中：[**濾光片與光譜帶**](https://mapir.gitbook.io/lattice-camera/hardware/filters-and-bands)。

軟體層面需注意的是：型號字串中的濾光片代碼決定了 Chloros 能生成哪些產品。 RGB 濾光片相機 (`FRGB`) 僅輸出去拜爾化及預覽產品——對於寬頻感測器而言，各波段的輻照度與反射率並無實質意義，因此 Chloros 會跳過這些參數並明確說明。其餘所有濾光片皆會產生完整的輻照度 → 反射率 → 指數鏈。

## 輻測校正概覽

每台 LATTICE 相機均在工廠針對可追溯至 NIST 的校準鏈進行個別校正，並隨機附上每台相機專屬的校準證書。 校正範圍、測量方式以及可引用的精度，詳見硬體手冊：[**工廠輻量校正**](https://mapir.gitbook.io/lattice-camera/calibration/factory-radiometric-calibration)。

在軟體方面，關鍵在於當相機連接時，Chloros 能解析正確的校正參數，並將所套用的係數固定於每次匯出中 — 請參閱 [連接相機](connecting.md)。

## 本章內容

* [連接相機](connecting.md) — 自動偵測、GUI 連線對話方塊、CLI／SDK 對應操作，以及相機連線時如何解析出廠校準（相機內建包 vs. 雲端）。
* [相機設定](camera-settings.md) — 「相機」分頁中的所有控制項：相機清單、各相機的設定面板，以及陣列設定面板（包含 ROI 編輯器與對齊功能）。
* [擷取設定與模式](capture.md) — 全部擷取、單次／連續／間隔擷取、匯出類型，以及擷取檔案夾的結構。
* [多相機陣列](arrays.md) — 陣列連接精靈、同步線探針、頻寬、PTP 以及即時陣列檢視。
* [單色相機與植被指數](mono-indices.md) — M3M 相機，以及對齊 → 堆疊 → 指數的工作流程。

完整的命令介面詳見 [CLI 參考手冊](../reference/cli-reference.md) 及 [SDK 參考手冊](../reference/sdk-reference.md)。
