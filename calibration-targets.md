---
description: 實驗室測量面板用於校准後處理中捕獲的數據
metaLinks:
  alternates:
    - https://app.gitbook.com/s/o044KN3Ws0uIDvOmSkcR/calibration-targets
---

# 校準目標

MAPIR 提供各種校準目標來涵蓋一系列應用。下面看到的緊湊型 T4-R50 包含 4 個面板，這些面板已在 250 - 2,500 nm 範圍內測量光反射率。

<figure><img src=".gitbook/assets/t4-r50_2.jpg" alt=""><figcaption><p>MAPIR T4-R50</p></figcaption></figure>

T4 漫反射參考目標具有以下反射率曲線 [數據在這裡下載](https://cdn.shopify.com/s/files/1/0972/5566/files/MAPIR_Diffuse_Reflectance_Standard_Calibration_Target_Data_T4.xlsx?v=1741759157)：

<figure><img src=".gitbook/assets/MAPIR Diffuse Reflectance Standard Calibration Target Data T4 (250-2500nm).png" alt=""><figcaption><p>MAPIR T4 反射率::250-2500nm</p></figcaption></figure>

<figure><img src=".gitbook/assets/MAPIR Diffuse Reflectance Standard Calibration Target Data T4 (400-1000nm).png" alt=""><figcaption><p>MAPIR T4 反射率::400-1000nm</p></figcaption></figure>

查看反射率圖，您可以看到這些值是波長（x 軸）與反射率百分比（y 軸）。當我們捕獲校準目標的圖像時，我們會在每個相機傳感器波段敏感的光譜內創建像素值和反射率百分比之間的關係。

這意味著，對於您使用我們的相機拍攝的每張圖像，您都可以使用我們的反射率目標（例如 [T4-R50](https://www.mapir.camera/collections/calibration-targets/products/diffuse-reflectance-standard-calibration-target-package-t3-r50) 或 [T4-R125](https://www.mapir.camera/collections/multispectral-reflectance-reference-calibration-targets/products/diffuse-reflectance-standard-calibration-target-package-t4-r125)）的照片來校準圖像的反射率。校准後，圖像中的每個像素都等於反射率百分比。

如果在 Chloros 中將校準圖像輸出為典型的 JPG 或 TIFF，則反射率百分比是通過將像素值除以圖像格式的位深度來計算的。因此，對於 JPG，則除以 255；對於 TIFF，則除以 65,535。您還可以在 Chloros 中選擇 PERCENT 格式輸出，然後每個像素的百分比值範圍為 0.0 到 1.0（0% 到 100% 反射率）。請記住，某些圖像應用程序無法接受百分比（浮點）圖像，並且它們的存儲尺寸很大。

<div><figure><img src=".gitbook/assets/t3-125.jpg" alt=""><figcaption><p>T4-R125</p></figcaption></figure> <figure><img src=".gitbook/assets/t3-125_2.jpg" alt=""><figcaption><p>T4-R125</p></figcaption></figure> <figure><img src=".gitbook/assets/t3-125_closed.jpg" alt=""><figcaption><p>T4-R125</p></figcaption></figure></div>
