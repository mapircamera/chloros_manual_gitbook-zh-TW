# 支援的語言

Chloros 提供 **全球 38 種語言** 的完整介面支援，讓世界各地的使用者都能輕鬆使用。您可以在桌面圖形介面 (GUI) 和命令列介面 (CLI) 中即時切換語言。

Chloros 支援以下語言：

| # | 語言 | 原生名稱 | CLI 代碼 |
|---|----------|-------------|----------|
| 1 | 🇺🇸 英語 | English | `en` |
| 2 | 🇪🇸 西班牙語 | Español | `es` |
| 3 | 🇵🇹 葡萄牙語 | Português | `pt` |
| 4 | 🇫🇷 法語 | Français | `fr` |
| 5 | 🇩🇪 德語 | Deutsch | `de` |
| 6 | 🇮🇹 義大利語 | Italiano | `it` |
| 7 | 🇯🇵 日語 | 日本語 | `ja` |
| 8 | 🇰🇷 韓語 | 한국어 | `ko` |
| 9 | 🇨🇳 中文（簡體） | 簡體中文 | `zh` |
| 10 | 🇹🇼 中文（繁體） | 繁體中文 | `zh-TW` |
| 11 | 🇷🇺 俄語 | Русский | `ru` |
| 12 | 🇳🇱 荷蘭語 | Nederlands | `nl` |
| 13 | 🇸🇦 阿拉伯語 | العربية | `ar` |
| 14 | 🇵🇱 波蘭語 | Polski | `pl` |
| 15 | 🇹🇷 土耳其語 | Türkçe | `tr` |
| 16 | 🇮🇳 印地語 | हिंदी | `hi` |
| 17 | 🇮🇩 印尼語 | Bahasa Indonesia | `id` |
| 18 | 🇻🇳 越南語 | Tiếng Việt | `vi` |
| 19 | 🇹🇭 泰語 | ไทย | `th` |
| 20 | 🇸🇪 瑞典語 | Svenska | `sv` |
| 21 | 🇩🇰 丹麥語 | Dansk | `da` |
| 22 | 🇳🇴 挪威語 | Norsk | `no` |
| 23 | 🇫🇮 芬蘭語 | Suomi | `fi` |
| 24 | 🇬🇷 希臘語 | Ελληνικά | `el` |
| 25 | 🇨🇿 捷克語 | Čeština | `cs` |
| 26 | 🇭🇺 匈牙利語 | Magyar | `hu` |
| 27 | 🇷🇴 羅馬尼亞語 | Română | `ro` |
| 28 | 🇺🇦 烏克蘭語 | Українська | `uk` |
| 29 | 🇧🇷 巴西葡萄牙語 | Português Brasileiro | `pt-BR` |
| 30 | 🇭🇰 粵語 | 粵語 | `zh-HK` |
| 31 | 🇲🇾 馬來語 | Bahasa Melayu | `ms` |
| 32 | 🇸🇰 斯洛伐克語 | Slovenčina | `sk` |
| 33 | 🇧🇬 保加利亞語 | Български | `bg` |
| 34 | 🇭🇷 克羅埃西亞語 | Hrvatski | `hr` |
| 35 | 🇱🇹 立陶宛語 | Lietuvių | `lt` |
| 36 | 🇱🇻 拉脫維亞語 | Latviešu | `lv` |
| 37 | 🇪🇪 愛沙尼亞語 | Eesti | `et` |
| 38 | 🇸🇮 斯洛維尼亞語 | Slovenščina | `sl` |

## 如何變更語言

### 在 Chloros 桌面版中

1. 開啟應用程式設定
2. 導航至語言選擇選單
3. 從清單中選擇您偏好的語言
4. 介面將立即更新

### 在 Chloros CLI 中

使用 `language` 指令來檢視或變更 CLI 介面語言：

```bash
# View current language
chloros-cli language

# Change to Spanish
chloros-cli language es

# Change to Chinese (Simplified)
chloros-cli language zh

# Change to Brazilian Portuguese
chloros-cli language pt-BR

# List all available languages
chloros-cli language --list
```

更多詳細資訊，請參閱 [CLI 文件](CLI.md)。

## 支援範圍

以下所有平台均完整支援全部 38 種語言：

* **Chloros Desktop** - 完整的 GUI 翻譯
* **Chloros CLI** - 命令列介面與輸出訊息

Python SDK API 及其 [參考文件](reference/sdk-reference.md) 均以英文提供。

語言支援確保全球使用者都能以母語高效工作，不受語言障礙所限。
