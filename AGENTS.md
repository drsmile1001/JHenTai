# AGENTS.md

本文件指引 AI agent 在本 repo 的建置、依賴處理與本機發佈工作。

## 專案概述

JHenTai 是以 Flutter 開發的 E-Hentai/EXHentai 瀏覽器。master 追蹤上游 jiangtian616/JHenTai。本分支 build/fedora 為 Fedora Linux 桌面建置分支，使用 Flutter stable SDK。版本以 pubspec.yaml 為準。

## 建置

必要系統套件：

```bash
sudo dnf install gtk3-devel webkit2gtk4.1-devel
```

另需 clang、cmake、ninja-build、pkgconfig。

標準建置流程：

```bash
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter build linux --release -t lib/src/main.dart
```

- 入口為 lib/src/main.dart
- Drift schema 或依賴變更後執行 build_runner
- 產物位於 build/linux/x64/release/bundle/

## 依賴與 SDK 相容

pub get 失敗或編譯錯誤時，先比對 SDK 內建套件的鎖定版本。flutter_test 鎖定 path，flutter_localizations 鎖定 intl。鎖定版本查 SDK 目錄 packages 下對應套件的 pubspec.yaml。專案依賴低於鎖定版本時，直接調整 pubspec.yaml 至鎖定版本，勿以 dependency_overrides 硬蓋。

本機 SDK 位於 /home/<user>/repos/flutter，為 git clone 的 stable channel。pub get 提示依賴要求較新 Dart SDK 時，將 SDK 更新至最新 stable：`git -C /home/<user>/repos/flutter reset --hard origin/stable`，再執行 flutter --version 觸發工具鏈下載。

git ref 的 plugin 在解析新版本後，檢查 linux 平台 pluginClass 宣告。pluginClass: none 會讓 Flutter 生成無效 include，移除該行即可。

升級 SDK 後出現 API 錯誤時，以 SDK 現有 API 為準修正程式碼。

## 生成檔案

linux、windows、macos 的 generated_plugin_registrant 由工具鏈在 pub get 與 build 時生成，已列入 .gitignore。

## 本機發佈

佈署 bundle：

```bash
rm -rf /home/<user>/apps/JHenTai/bundle
cp -r build/linux/x64/release/bundle /home/<user>/apps/JHenTai/bundle
```

desktop entry 位於 ~/.local/share/applications/JHenTai.desktop，Exec 指向 bundle 內 jhentai 執行檔，Icon 指向應用目錄 icon.png。GNOME 即時偵測變更，不需登出。

驗證：

```bash
file /home/<user>/apps/JHenTai/bundle/jhentai
timeout 5 /home/<user>/apps/JHenTai/bundle/jhentai
```

- file 預期輸出 ELF 64-bit LSB executable
- timeout 終止碼 124 代表應用持續運行未崩潰
- Atk-CRITICAL 與 Gdk-Message 為 GTK 主題警告，不影響功能
