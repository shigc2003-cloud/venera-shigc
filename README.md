# venera-shigc

venera 个人维护分支，基于 [venera-app/venera](https://github.com/venera-app/venera) 继续改进。

原项目已停止维护，本分支持续修复漫画源解析、优化编译流程。

## 与原版差异

- **wnacg.js 修复**：适配 wnacg.com 新版网页结构（2026-06 改版），重写全部 HTML 选择器
- **rhttp 升级**：0.15.1 → 0.18.0，修复 flutter_rust_bridge codegen 版本不匹配导致的运行崩溃
- **CLAUDE.md**：完整的项目文档，包含环境搭建清单、.venera 文件格式、数据库结构、ComicType 映射等

## 技术栈

| 层 | 技术 |
|---|---|
| UI 框架 | Flutter 3.41.4 / Dart 3.11 |
| 漫画源引擎 | QuickJS (`flutter_qjs`) — JS 脚本运行时 |
| 本地存储 | SQLite3 |
| 网络层 | Dio + Rust `rhttp`（底层 HTTP） |
| 跨语言 | `flutter_rust_bridge` (Dart ↔ Rust) |

## 编译环境

详见 `CLAUDE.md` 中的完整清单。简要版本：

- Flutter 3.41.4 + Dart 3.11
- Rust 1.85.1 (msvc) + Android NDK 28
- Java 21 (Temurin)
- Visual Studio 2026 Community（C++ 桌面开发）
- Android SDK API 36

国内开发需配置 pub 镜像和代理，具体命令见 CLAUDE.md。

## 快速开始

```powershell
# 编译 APK
flutter pub get
flutter build apk --release
# 输出：build\app\outputs\flutter-apk\app-release.apk
```

## .venera 文件格式

应用导出的 `.venera` 文件是一个 ZIP 压缩包，内部结构：

| 文件 | 格式 | 说明 |
|---|---|---|
| `history.db` | SQLite | 浏览历史 |
| `local_favorite.db` | SQLite | 本地收藏 |
| `appdata.json` | JSON | 应用设置 + 搜索历史 |
| `cookie.db` | SQLite | 网络 Cookie |
| `comic_source/*.js` | JS | 漫画源脚本 |
| `comic_source/*.data` | JSON | 漫画源持久化数据（token、账号） |

## 致谢

原项目 [venera-app/venera](https://github.com/venera-app/venera)，感谢原作者。
