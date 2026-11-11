# Chloros+ 登入

## 圖形化介面 (GUI) 登入

使用者 <img src=".gitbook/assets/icon_user.JPG" alt="" data-size="line"> 側邊欄選單可讓您登入 Chloros+ 帳戶並解鎖更多功能。

**每台機器只需登入一次。** 圖形化介面 (GUI)、命令列介面 (CLI) 以及 Python SDK 共用同一個快取的會話 — 透過桌面 GUI 登入，也會在該台機器上啟用 CLI 和 SDK（反之亦然，透過 `chloros-cli login`）。

登入後，您的帳戶詳細資訊將顯示如下：

<figure><img src=".gitbook/assets/user_account.JPG" alt="" width="375"><figcaption></figcaption></figure>
<!-- SCREENSHOT-UPDATE: re-shoot the logged-in user account panel in Chloros 1.3.0 — plan name display and the registered-device list UI may have changed; must show plan name, expiration, and device list. -->

## 方案等級

| 方案 | `plan_id` | 類型 |
| --- | --- | --- |
| 鐵級 | `0` | 免費 |
| 銅級 | `1` | 付費（Chloros

+） |
| 青銅級 | `2` | 付費（Chloros

+） |
| 銀級 | `3` | 付費（Chloros

+） |
| 黃金 | `4` | 付費（Chloros

+） |

請參閱 [方案與定價](https://cloud.mapir.camera/pricing)，了解各付費層級的包含內容。

### 存取 CLI

／SDK

需具備付費層級

使用 CLI

和 Python

SDK

需具備 **任何付費 Chloros

+ 方案（銅級或更高）**。此限制由**伺服器端** 強制執行 — 每個 CLI

/SDK

請求都必須同時包含有效會話和付費方案：

| HTTP

狀態 | `error_code` | 含義 | 解決方法 |
| --- | --- | --- | --- |
| `401` | `AUTH_REQUIRED` | 未在此機器上登入 | `chloros-cli login <email> <password>` |
| `403` | `PLAN_UPGRADE_REQUIRED` | 已登入，但方案等級過低（免費 Iron 等級） | 升級至任何付費的 Chloros

+ 方案 |

`chloros-cli status` 在免費方案下仍可存取，因此您隨時都能查看當前方案及遭拒的理由。

### 各方案的連線硬體限制

每個方案皆設有同時可即時連線的 LATTICE 攝影機與 DAQ 光感測器的數量上限：

| 方案 | LATTICE 攝影機 | DAQ 光感測器 |
| --- | --- | --- |
| Iron（免費／未登入） | 4 | 2 |
| Copper／Bronze | 6 | 3 |
| Silver | 10 | 6 |
| Gold | 20 | 12 |

## CLI

登入

請使用您的 Chloros

+ 憑證登入以啟用 CLI

處理功能。在 Linux

（無 GUI）環境中，這是啟用授權的唯一方式。

**語法：**

```bash
chloros-cli login <email> <password>
```

{% hint style="info" %}
**SDK

使用者**：Python

SDK

亦提供程式化方法 `logout()` 來清除快取的憑證。詳情請參閱 [SDK

參考手冊](reference/sdk-reference.md)。
{% endhint %}

**範例：**

```powershell
chloros-cli login user@example.com 'MyP@ssw0rd123'
```

{% hint style="warning" %}
**特殊字元**：若密碼包含如 `$`、`!` 或空格等字元，請使用單引號將其包圍。
{% endhint %}

**輸出：**

<figure><img src=".gitbook/assets/cli login_w.JPG" alt=""><figcaption></figcaption></figure>
<!-- SCREENSHOT-UPDATE: re-shoot the CLI login output — the banner now prints "Chloros CLI 1.3.0"; capture a successful login with the current output format. -->

### 憑證儲存

在 **所有平台** 上，快取的憑證和設定皆儲存於您的使用者家目錄中的 `.chloros` 資料夾內：

| 平台 | 憑證快取路徑 |
| --- | --- |
| **Windows** | `%USERPROFILE%\.chloros\` |
| **Linux** | `~/.chloros/` |

若在現有版本上安裝新版，您的登入資訊將被保留，此情況適用於 Windows

（安裝程式）及 Linux

（`.deb`，包含 `chloros-cli update`）。 若解除安裝 Chloros

—— 或移除 Linux

套件 —— 將清除快取的登入資訊，因此重新安裝後需再次登入。

### 方案到期與離線寬限期

GUI 中顯示的方案到期日期，代表您的授權將於該日期失效。 對於每月自動續訂的訂閱，到期日為當月月底；對於年度訂閱，則為您開始訂閱後的一年。

Chloros

會線上驗證您的授權，但在寬限期內仍支援離線運作：

* 成功的伺服器驗證結果會被快取 **5 分鐘**，因此正常使用時幾乎不會發出授權查詢請求。
* 經簽名且與機器綁定的授權快取可涵蓋更長的離線時段：**月費方案為 30 天**，**年費方案則持續至訂閱到期日（最長 365 天）**。
* 當寬限期屆滿時，方案將自動降級為免費的 Iron 層級，直至該裝置能成功連線至授權伺服器一次為止；下次驗證成功後，即可恢復存取權限。

### 裝置數量限制

每種 Chloros

+ 方案提供的註冊裝置數量各不相同。您使用 Chloros

+ 帳戶登入的每台裝置，都會計入您的註冊裝置總數。您可以在 MAPIR

雲端帳戶頁面中重新命名或移除裝置。

<table><thead><tr><th width="168.5999755859375" align="right">Chloros+ 方案</th><th align="center">銅級</th><th align="center">BRONZE</th><th align="center">銀級</th><th align="center">GOLD</th></tr></thead><tbody><tr><td align="right">支援裝置</td><td align="center">2</td><td align="center">2</td><td align="center">5</td><td align="center">10</td></tr></tbody></table>

您帳戶的確切裝置配額會顯示在您的 MAPIR Cloud 帳戶頁面中。從裝置登出可確實釋放該裝置的配額，且已註冊的裝置即使在帳戶達到裝置上限時，仍可隨時重新登入。
