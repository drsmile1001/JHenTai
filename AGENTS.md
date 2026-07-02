# AGENTS.md

本文件指引 AI agent 處理本 repo 中的程式碼時使用。

## 專案概述

JHenTai 是一個以 Flutter 開發的 E-Hentai/EXHentai 瀏覽器，目標平台為 Android、iOS、Windows、macOS 與 Linux。版本 `8.0.14+317`（取自 pubspec.yaml）。

## 建置與開發指令

```bash
# 取得依賴
flutter pub get

# 執行程式碼生成（Drift DB 等）
dart run build_runner build

# 在已連接裝置上執行
flutter run

# 靜態分析
flutter analyze

# 執行測試
flutter test
```

目前無測試目錄檔案，`flutter test` 只在新增測試時使用。

## 架構

### 依賴框架：GetX
狀態管理、路由、依賴注入與 i18n 皆使用 GetX。Widget 使用 `GetBuilder<T>`（手動呼叫 `update()`），不使用 reactive `.obs` 串流。僅在設定值需廣播變更至跨 view 時，頁面才使用 `.obs`。

### 生命週期：`JHLifeCircleBean` 模式

每個單例服務與設定都實作 `lib/src/service/jh_service.dart` 中的 `JHLifeCircleBean`：

- `initBean()` — 非同步初始化（`runApp` 前呼叫）
- `afterBeanReady()` — runApp 後的設定
- `initDependencies` — 此 bean 需先完成初始化的其他 bean 列表

所有 bean 在 `main.dart` 中依拓扑排序註冊並依序初始化。三個 mixin 簡化常見模式：
- `JHLifeCircleBeanErrorCatch` — 用 try/catch 包裹 init/ready 並記錄日誌
- `JHLifeCircleBeanWithConfigStorage` — 在 `local_config` DB 表中加入 JSON 序列化/反序列化
- 服務 bean 位於 `lib/src/service/`；設定單例位於 `lib/src/setting/`

### 路由

所有路由定義在 `lib/src/routes/routes.dart`，為 `class Routes` 上的靜態常數字串。`EHPage` 類別繼承 `GetPage` 並新增：
- `side` — `left`/`right`/`fullScreen`（平板 layout 使用左右分割）
- `offAllBefore` — 是否彈出先前的右側路由

巢狀設定路由使用 `settingPrefix` 慣例（`/setting_*`）。

### 頁面模式

頁面遵循統一的 GetX 結構，位於 `lib/src/pages/`：

- `*_page.dart` — Widget
- `*_logic.dart` — `GetxController` 子類別（商業邏輯）
- `*_state.dart` — 可變狀態物件

基底類別為 `BasePageLogic`（`pages/base/base_page_logic.dart`），處理 gallery 列表頁面的下拉刷新、分頁（prev/next gid）、搜尋設定持久化與標籤封鎖/過濾。子類別覆寫 `getGalleryPage()` 以提供特定頁面的 API 呼叫。

### 網路層（`lib/src/network/`）

使用自訂 Dio fork（`dio` 來自 `jiangtian616/dio`，`append-mode` ref）。主要請求類別：
- `EHRequest` — 所有 E-Hentai API 呼叫，含 cookie 管理、快取、EX 的 domain fronting
- `JHRequest` — 後端 API 呼叫（標籤翻譯、應用更新檢查、內建封鎖列表）
- `ArchiveBotRequest` — archive.org 解析

HTML 回應解析器位於 `lib/src/utils/eh_spider_parser.dart`。

### 資料庫（`lib/src/database/`）

Drift（SQLite），schema 版本 23，含有大量遷移鏈。表格在 `database/table/`，DAO 在 `database/dao/`。全域 `appDb` 單例在 `database.dart` 底部宣告。生成的程式碼在 `database.g.dart`。

### 服務（`lib/src/service/`）

管理核心功能的獨立單例：
- `gallery_download_service.dart` / `archive_download_service.dart` — 下載引擎，含並列佇列、續傳、優先級
- `tag_translation_service.dart` — 從 EhTagTranslation 取得並快取標籤翻譯
- `local_block_rule_service.dart` — 使用者設定的 gallery 封鎖規則
- `cloud_service.dart` — 設定同步
- `storage_service.dart` — JSON/GetStorage NoSQL 持久化
- `super_resolution_service.dart` — 圖片放大中繼資料追蹤
- `path_service.dart` — 平台感知的目錄解析

### 設定（`lib/src/setting/`）

每個設定模組為獨立單例（如 `ehSetting`、`styleSetting`、`preferenceSetting`），使用 `JHLifeCircleBeanWithConfigStorage`。設定序列化為 JSON 並儲存在 `local_config` DB 表中。當設定需觸發跨應用 UI 重建時使用 reactive `.obs`。

### Layout 系統（`lib/src/pages/layout/`）

三種 layout 模式：
- `mobile_v2` — 底部導覽列搭配分頁式頁面
- `tablet_v2` — 主從式分割（左右路由側）
- `desktop` — 側邊欄導覽搭配持續顯示的細節面板

首頁（`home_page.dart`）根據螢幕寬度選擇 layout。

### i18n（`lib/src/l18n/`）

透過 `LocaleText`（GetX `Translations`）自訂翻譯系統。每個語言一個 `.dart` 檔案，內容為鍵值對。新增語言需要：語言檔案、`locale_text.dart` 中的項目、`locale_consts.dart` 中的項目。

### Widget 函式庫（`lib/src/widget/`）

以 `eh_` 為前綴的可重用 widget — 對話框、卡片、圖片元件、標籤顯示等。`app_manager.dart` widget 包裹整個應用處理全域事務。`loading_state_indicator.dart` 提供標準的載入/錯誤/空/閒置狀態 widget。

### 主要依賴（超出 Flutter 標準）

- `get` 4.6.6 — 狀態管理、路由、i18n、NoSQL 儲存
- `dio`（自訂 fork）— HTTP 客戶端
- `drift` 2.21.0 — SQLite ORM
- `extended_image` — 含快取的圖片載入
- `photo_view` / `zoom_view`（自訂 fork）— 閱讀頁面圖片檢視器
- `desktop_webview_window` — 桌面的 webview cookie 登入

## Fedora Linux 建置注意事項

本分支 `build/fedora-flutter-3.41` 用於在 Fedora 44 搭配 Flutter 3.41.9 從原始碼建置 JHenTai。

### 必要系統套件

```bash
sudo dnf install gtk3-devel webkit2gtk4.1-devel
```

以下為開發工作站通常已預裝但為 Flutter Linux 工具鏈所必須：
- `clang`、`cmake`、`ninja-build`、`pkgconfig`

### Flutter SDK

Flutter SDK 可透過 VS Code Flutter 延伸套件安裝，或下載官方 tarball 解壓。本機 SDK 路徑為 `/home/<user>/repos/flutter`。

### Flutter 3.41.x 已知相容問題

每次升級到新的上游 JHenTai 版本後，若上游原始使用的 Flutter 版本較舊，需重新套用以下修正。

```yaml
- 套件: path
  症狀: 'flutter_test' from SDK requires path 1.9.1，專案鎖定 1.9.0
  修正: 在 pubspec.yaml dependencies 中將 path: 1.9.0 改為 path: ^1.9.0

- 套件: intl
  症狀: 'flutter_localizations' from SDK pins intl 0.20.2，專案鎖定 0.19.0
  修正: 在 pubspec.yaml dependency_overrides 中加入 intl: 0.20.2

- API: DialogTheme
  症狀: 'DialogTheme' can't be assigned to 'DialogThemeData?'
  修正: 在 lib/src/config/theme_config.dart 中將 DialogTheme 改為 DialogThemeData

- Plugin: system_network_proxy_linux
  症狀: fatal error: 'system_network_proxy_linux/none.h' file not found
  原因: plugin pubspec 宣告 pluginClass: none，Flutter 3.41 將其作為真實類名產生無效 include
  修正: 在系統快取中找到對應版本的 pubspec.yaml，移除 linux 區塊下的 pluginClass: none 行
  快取路徑範例: /home/<user>/.pub-cache/git/system_network_proxy-<hash>/system_network_proxy_linux/pubspec.yaml
  注意: 每次 flutter pub get 解析到新版本的 system_network_proxy_linux 時都需重新修正
```

### Generated plugin registrant 檔案

`linux/flutter/generated_plugin_registrant.*`、`windows/flutter/generated_plugin_registrant.*` 以及 `macos/Flutter/GeneratedPluginRegistrant.swift` 皆由 Flutter 工具鏈在 pub get 或 build 時重新產生。本分支已將其加入 `.gitignore`，避免追蹤隨 Flutter SDK 版本變動的平台特定生成程式碼。

### 建置順序

```bash
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter build linux --release -t lib/src/main.dart
```

建置產物位於 `build/linux/x64/release/bundle/`，包含 `jhentai` 執行檔、`data/` 目錄與 `lib/` 目錄（內含共用函式庫）。

## 本地發佈流程

### 佈署執行檔與資源

```bash
# 複製建置 bundle 到應用目錄
cp -r build/linux/x64/release/bundle /home/<user>/apps/JHenTai/bundle
```

若要更新現有佈署，可直接覆蓋目錄：

```bash
rm -rf /home/<user>/apps/JHenTai/bundle && \
cp -r build/linux/x64/release/bundle /home/<user>/apps/JHenTai/bundle
```

### Desktop entry

GNOME 啟動器項目位於 `~/.local/share/applications/JHenTai.desktop`，內容如下：

```ini
[Desktop Entry]
Name=JHenTai
Exec=/home/<user>/apps/JHenTai/bundle/jhentai
Icon=/home/<user>/apps/JHenTai/icon.png
Type=Application
Categories=Utility;
Terminal=false
```

- `Exec=` 需指向 bundle 內的 `jhentai` 執行檔
- `Icon=` 使用應用目錄下的 `icon.png`（來自舊版 AppImage 保留的圖示）

更新 desktop entry 後不需登出，GNOME 會即時偵測變更。可立即在啟動器中搜尋 JHenTai 啟動。

### 佈署後驗證

```bash
# 確認執行檔類型
file /home/<user>/apps/JHenTai/bundle/jhentai
# 預期輸出: ELF 64-bit LSB executable, x86-64

# 快速執行測試（5 秒後自動終止）
timeout 5 /home/<user>/apps/JHenTai/bundle/jhentai
# 若應用未崩潰即表示正常。Atk-CRITICAL 與 Gdk-Message 為 GTK 主題相關警告，不影響功能。
```
