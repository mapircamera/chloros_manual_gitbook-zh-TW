# Chloros+ 登錄

## Chloros 和 Chloros（瀏覽器）登錄

用戶 <img src=".gitbook/assets/icon_user.JPG" alt="" data-size="line"> 側邊欄菜單允許您登錄 Chloros+ 帳戶並解鎖其他功能。

登錄後，將顯示您的帳戶詳細信息：

<figure><img src=".gitbook/assets/user_account.JPG" alt="" width="375"><figcaption></figcaption></figure>

## 命令行登錄

使用您的 Chloros+ 憑據登錄以啟用 CLI 處理。

**句法：**

```bash
chloros-cli login <email> <password>
```

**例子：**

```powershell
chloros-cli login user@example.com 'MyP@ssw0rd123'
```

{% 提示樣式=“警告”%}
**特殊字符**：在包含 `$`、`!` 或空格等字符的密碼周圍使用單引號。
{% 結束提示 %}

**輸出：**

<figure><img src=".gitbook/assets/cli login_w.JPG" alt=""><figcaption></figcaption></figure>

### 計劃到期

GUI 中的計劃到期時間顯示您的許可證何時失效。對於每月定期訂閱，到期日為月底。對於按年訂閱，是指您開始訂閱後的一年。許可證檢查需要每月連接互聯網進行驗證，並有 30 天的寬限期。

### 設備限制

每個 Chloros+ 計劃提供不同數量的註冊設備。您使用 Chloros+ 帳戶登錄的每台設備都將計入您的註冊設備數量。您可以在 MAPIR 雲帳戶頁面上重命名和刪除設備。

<table><thead><tr><th width="168.5999755859375" align="right">Chloros+ 計劃</th><th align="center">COPPER</th><th align="center">BRONZE</th><th align="center">SILVER</th><th align="center">GOLD</th></tr></thead><tbody><tr><td align="right">設備支持</td><td align="center">2</td><td align="center">2</td><td align="center">5</td><td align="center">10</td></tr></tbody></table>
