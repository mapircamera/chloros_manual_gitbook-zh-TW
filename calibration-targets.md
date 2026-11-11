---
description: Lab-measured panels used to calibrate captured data in post processing
metaLinks:
  alternates:
    - https://app.gitbook.com/s/o044KN3Ws0uIDvOmSkcR/calibration-targets
---

# 校準標靶

MAPIR 提供多種校準標靶，以滿足各類應用需求。下圖所示的緊湊型 T4-R50 包含 4 塊面板，其光反射率已針對 250 至 2,500 奈米波長範圍進行測量。

<figure><img src=".gitbook/assets/t4-r50_2.jpg" alt=""><figcaption><p>MAPIR T4-R50</p></figcaption></figure>T4 漫反射參考標靶具有以下反射率曲線，[點此下載資料](https://cdn.shopify.com/s/files/1/0972/5566/files/MAPIR_Diffuse_Reflectance_Standard_Calibration_Target_Data_T4.xlsx?v=1741759157)：

<figure><img src=".gitbook/assets/MAPIR Diffuse Reflectance Standard Calibration Target Data T4 (250-2500nm).png" alt=""><figcaption><p>MAPIR T4 反射率 :: 250-2500nm</p></figcaption></figure>

<figure><img src=".gitbook/assets/MAPIR Diffuse Reflectance Standard Calibration Target Data T4 (400-1000nm).png" alt=""><figcaption><p>MAPIR T4 反射率 :: 400-1000nm</p></figcaption></figure>T4P 漫反射參考靶的反射率曲線如下，[資料請由此下載](https://cdn.shopify.com/s/files/1/0972/5566/files/MAPIR_Diffuse_Reflectance_Standard_Calibration_Target_Data_T4.xlsx?v=1741759157)：

<figure><img src=".gitbook/assets/MAPIR Diffuse Reflectance Standard Calibration Target Data T4P -- 350-2500nm.jpg" alt=""><figcaption><p>MAPIR T4P 反射率 :: 250-2500nm</p></figcaption></figure>

<figure><img src=".gitbook/assets/MAPIR Diffuse Reflectance Standard Calibration Target Data T4P -- 400-1000nm.jpg" alt=""><figcaption><p>MAPIR T4P 反射率 :: 400-1000nm</p></figcaption></figure>觀察反射率圖表可知，其數值呈現波長（x 軸）對應反射率百分比（y 軸）的關係。當我們拍攝校準標靶的影像時，便會在相機各感光元件波段所感應的波長範圍內，建立像素值與反射率百分比之間的對應關係。

這意味著，每當您使用我們的相機拍攝影像時，都可以利用我們的反射率標靶照片（例如 [T4-R50](https://www.mapir.camera/collections/calibration-targets/products/diffuse-reflectance-standard-calibration-target-package-t3-r50) 或 [T4-R125](https://www.mapir.camera/collections/multispectral-reflectance-reference-calibration-targets/products/diffuse-reflectance-standard-calibration-target-package-t4-r125)，對影像進行反射率校正。校正完成後，影像中的每個像素值即等同於反射率百分比。

針對 **Survey3** 的輸出，若您在 Chloros

中將校正後的影像輸出為常見的 JPG 或 TIFF

格式，則反射率百分比是透過將像素值除以該影像格式的位元深度來計算。因此，JPG 格式需除以 255，TIFF

格式則需除以 65,535。 您也可以在 Chloros

中選擇「PERCENT」格式輸出，此時每個像素的百分比值範圍將介於 0.0 至 1.0 之間（即 0% 至 100% 反射率）。請注意，部分影像應用程式無法處理百分比（浮點數）格式的影像，且此類檔案在儲存空間上佔用較大。

{% hint style="info" %}
**LATTICE 反射率採用不同的像素比例。** LATTICE 反射率的儲存方式為 DN 32768 = 100% 反射率（而非 65535），且每個檔案都附帶一個 XMP `Chloros:PixelScale` 標籤，用以標示其比例。 請讀取該標籤並以此數值進行除法運算，而非假設為恆定值 — 詳見 [輸出影像格式](output-image-formats.md)。
{% endhint %}

## 使用 LATTICE 相機的校準標靶

使用 LATTICE 相機時，反射率的校準標靶為 **可選**：Chloros

亦可改為參考由 DAQ 光感測器測得的下行輻照度來計算反射率（ρ = π·L/E）。 該參考值由專案的「反射率來源」設定決定，該設定可透過 `chloros-cli process --reflectance-source` 進行調整 （圖形使用者介面（GUI）無此控制項，將採用專案儲存的值，預設為 `auto` — 請參閱 [專案設定](project-settings/project-settings.md)）：

| 值 | 行為 |
| --- | --- |
| `auto` *(預設)* | 通過品質保證（QA）的幀內目標即為 **絕對參考**；若無目標存在或 QA 失敗，Chloros

將回退至 DAQ 下行分界值。 若目標雖通過 QA 但帶有標記（例如面板遺漏或頻段異常），則不會自動被優先選用：系統會依其在保留面板上測得的評分順序，依序嘗試「目標」、「目標+DAQ」及「僅 DAQ」這三種鏈路。 |
| `target` | 嚴格僅限目標 — 不進行 DAQ 替換。 |
| `daq` | DAQ 權威模式 — 下行測量始終作為參考。 |

LATTICE 的額外目標行為：

* **目標幾何形狀** — 支援 ArUco 標記面板、固定 ROI 面板及條狀目標；幾何形狀取自專案的目標配置。
* **各單位的測量目標數據** — `--target-reflectance-dir DIR` 指向包含各單位測量目標反射率掃描資料的目錄（`<serial>.csv`，透過目標單位的序列號/QR 進行查詢）。 若未能偵測到目標，Chloros

將回退至標稱的 T3/T4P 光譜。
* **時間錨定** — 偵測到的目標會校準其周圍的幀，並在目標出現期間保持錨定狀態。

完整的標誌語義與範例請參閱 [CLI

參考手冊](reference/cli-reference.md)（請參閱「按產品導出切換開關」）。

### F988

「F988 的反射率是透過場景內的反射率面板進行校準：由於該波段超出 DAQ 光感測器的校準範圍，因此 Chloros

會套用您最近一次的面板擷取資料，並在兩次面板觀測之間維持該值。」

若以「僅 DAQ」校準模式執行 F988，Chloros

將拒絕該波段基於 DAQ 的反射率並說明原因（跳過原因 `dls-uncalibrated-band-988`）；標準的反射板工作流程才是受支援的處理方式。

<div><figure><img src=".gitbook/assets/t3-125.jpg" alt=""><figcaption><p>T4-R125</p></figcaption></figure> <figure><img src=".gitbook/assets/t3-125_2.jpg" alt=""><figcaption><p>T4-R125</p></figcaption></figure> <figure><img src=".gitbook/assets/t3-125_closed.jpg" alt=""><figcaption><p>T4-R125</p></figcaption></figure></div>
