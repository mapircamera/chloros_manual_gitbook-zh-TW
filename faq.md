---
description: 常見問題解答
metaLinks:
  alternates:
    - https://app.gitbook.com/s/o044KN3Ws0uIDvOmSkcR/faq
---

# 常問問題

<details>

<summary>我可以使用 Chloros 處理來自非 MAPIR 品牌的相機的圖像嗎？ </summary>

不，Chloros 僅支持處理 MAPIR 相機圖像。請參閱 [支持的相機型號](supported-cameras.md) 列表了解更多信息。我們確實在 MAPIR 雲上提供其他攝像機的處理，請參閱完整列表 [這裡](https://mapir.gitbook.io/mapir-cloud/supported-cameras)。

</details>

<details>

<summary>我可以在沒有校準目標的情況下校準圖像的反射率嗎？ </summary>

不能。如果在捕獲非目標圖像時沒有捕獲校準目標的圖像，您將無法將圖像的像素值與已知的反射率百分比聯繫起來。如果您也不包含來自 MAPIR 光傳感器的日誌，則將不會測量環境光譜，並且反射率結果將不准確。

</details>

<details>

<summary>我可以在 Chloros 中處理之前編輯圖像嗎？ </summary>

不會。 Chloros 假設輸入數據未被修改。不要更改文件名。

</details>

<details>

<summary>我可以將 MAPIR Survey3 相機設置為自動曝光並在 Chloros 中處理圖像嗎？ </summary>

不可以。 Survey3 圖像數據集必須具有固定/鎖定曝光，因此沒有自動快門速度或自動 ISO。同一相機型號的所有圖像必須具有相同的快門速度和 ISO（曝光）。

</details>

<details>

<summary>Chloros 可以處理或分析正射馬賽克圖像嗎？ </summary>

不。僅支持單個 MAPIR 相機圖像，而不支持像正射馬賽克地圖那樣的拼接圖像。

</details>

<details>

<summary>如何加快 Chloros 的目標檢測步驟？ </summary>

在文件瀏覽器表中，在右列中預先選擇目標圖像將告訴 Chloros 僅在這些圖像中查找校準目標，從而大大加快處理速度。

</details>

<details>

<summary>如果我要將圖像上傳到<a href="https://www.mapir.camera/collections/software/products/mapir-cloud-subscription">MAPIR Cloud</a>我應該在上傳之前在 Chloros 中進行處理嗎？ </summary>

如果您打算上傳到我們的在線處理平台[馬皮爾雲](https://www.mapir.camera/collections/software/products/mapir-cloud-subscription)，請勿在上傳前編輯圖像。雲將執行所有相同的處理甚至更多。

</details>

<details>

<summary>MAPIR 會支持 X 功能嗎？我真的希望 MAPIR 提供 X.</summary>

我們始終有興趣收到有關我們產品的反饋。如果您發現我們的產品存在問題，或者對我們如何改進產品有建議，請[聯繫我們](https://www.mapir.camera/community/contact) 分享您的想法。我們的大部分研發都是以傾聽客戶的最大需求為指導的。

</details>
