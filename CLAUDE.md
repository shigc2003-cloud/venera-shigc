# venera

A Flutter comic reader app (v1.6.3). Supports local comics and network sources defined by JS scripts.

## Tech stack
- Flutter 3.41.4 / Dart SDK >=3.8.0
- SQLite3 for local databases (history, favorites, cookies)
- `flutter_qjs` (QuickJS) — executes JS comic source scripts
- `dio` for HTTP, `zip_flutter` for archive handling

## Key architectural conventions

### Comic sources
Comic sources are JS files in `App.dataPath/comic_source/`. Each defines a class extending `ComicSource` with a `key` property (e.g. `key = "nhentai"`).

`lib/foundation/comic_source/` is the core source system:
- `comic_source.dart` — ComicSourceManager, find/register/lifecycle
- `parser.dart` — parses JS source files, validates key format (`^[a-zA-Z0-9_]+$`)
- `models.dart` — ComicDetails, ComicChapters, Comment, HistoryMixin
- `types.dart` — typedefs for JS↔Dart bridge functions
- `category.dart`, `favorites.dart` — category browsing and favorites integration

### ComicType and type values
`ComicType` (`lib/foundation/comic_type.dart`) wraps an `int` that equals a source key's `String.hashCode`. `ComicType(0)` is reserved for local comics.

The `type` column in DB tables (`history`, `local_favorite`) stores this hashCode.
To resolve: `ComicSourceManager().fromIntKey(value)` looks up `sources.where(s => s.key.hashCode == value)`.

Known type→source mappings (Dart `String.hashCode`):

| type value | source key | 来源 |
|---|---|---|
| `0` | `local` | 本地漫画 |
| `264196719` | `nhentai` | nhentai |
| `385625716` | `ehentai` | E-Hentai |
| `769844263` | `jm` | 禁漫天堂 |
| `823512256` | picacg (旧版 key) | 皮卡漫画旧 key |
| `553570794` | picacg (新版 key) | 皮卡漫画新 key |
| `258019538` | hitomi | Hitomi.la |

## File formats

### `.venera` export/backup file
A **ZIP archive** containing:

| 内部路径 | 格式 | 说明 |
|---|---|---|
| `history.db` | SQLite | 表 `history` — 浏览历史（id, title, cover, time, type, ep, page, readEpisode, max_page） |
| `local_favorite.db` | SQLite | 表结构按 folder 组织，默认在 `default` 表 |
| `appdata.json` | JSON | 应用全部设置 + 搜索历史 |
| `cookie.db` | SQLite | 表 `cookies` — 网络认证 Cookie |
| `comic_source/*.js` | JS 文本 | 各漫画源的脚本文件 |
| `comic_source/*.data` | JSON | 漫画源持久化数据（登录 token、账号等） |

Export logic: `lib/utils/data.dart` → `exportAppData()` creates ZIP via Isolate.
Import logic: `importAppData()` extracts ZIP, replaces DB files, merges settings via `appdata.syncData()`.
WebDAV sync: `lib/utils/data_sync.dart` manages versioned upload/download.

### `appdata.json` structure
```json
{
  "settings": { ... },   // all app settings, key-value
  "searchHistory": [...] // search keyword history
}
```

### Sync exclusion
`appdata.syncData()` never syncs these fields: `proxy`, `authorizationRequired`, `customImageProcessing`, `webdav`, `disableSyncFields`, `deviceId`. Users can add custom exclusion fields via `disableSyncFields` setting.

## Database schemas

### history.db
```sql
CREATE TABLE history (
  id text PRIMARY KEY,
  title text,
  subtitle text,
  cover text,
  time int,
  type int,       -- ComicType value = sourceKey.hashCode
  ep int,         -- 1-based chapter index
  page int,       -- 1-based page index
  readEpisode text, -- comma-separated set of read episode numbers
  max_page int,
  chapter_group int
);
```

`readEpisode` stores either chapter numbers or `"group-chapter"` format for grouped chapters.

### local_favorite.db
Tables are dynamically named by folder. The `default` table holds unfiled favorites. Standard columns: `id`, `name`, `author`, `type`, `tags`, `cover_path`, `time`, `display_order`.

## Build environment

Everything under `D:\applications\` — one place, easy to manage and backup.

### Environment checklist

| # | 组件 | 版本要求 | 安装路径 | 状态 |
|---|---|---|---|---|
| 1 | **Git** | 2.x+ | 已有 (scoop) | ✅ |
| 2 | **Java JDK** | 21 (Temurin) | `D:\scoop\apps\temurin21-jdk\current` | ✅ |
| 3 | **Flutter SDK** | 3.41.4 | `D:\applications\flutter` | ✅ |
| 4 | **Rust** (gnu) | 1.85.1 | `D:\applications\cargo` + `D:\applications\rustup` | ✅ |
| 5 | **Android cmdline-tools** | latest | `D:\applications\android-sdk\cmdline-tools\latest` | ✅ |
| 6 | **Android SDK Platform** | API 36 | `D:\applications\android-sdk\platforms\android-36` | ✅ |
| 7 | **Android Build-Tools** | 36.0.0 | `D:\applications\android-sdk\build-tools\36.0.0` | ✅ |
| 8 | **Android NDK** | 28.0.13004108 | `D:\applications\android-sdk\ndk\28.0.13004108` | ✅ |
| 9 | **Visual Studio** | 2026 Community (C++ workload) | `D:\codeSpace\VisualStudio26` | ✅ |
| 10 | **Windows 开发者模式** | | 系统设置 | ✅ |

> **第 9 项说明**: `rhttp` 和 `flutter_qjs` 的 Rust 编译过程脚本需要 `link.exe`（MSVC 链接器）。`rustup run stable` 在 Windows 上硬编码使用 msvc 工具链。即使设置了 gnu 为默认，`stable` 别名仍指向 msvc。装 VS 2022 Build Tools 后即可解决。

### 环境变量

需在编译前设置（用户级/会话级）：

```powershell
$env:JAVA_HOME = "D:\scoop\apps\temurin21-jdk\current"
$env:ANDROID_HOME = "D:\applications\android-sdk"
$env:ANDROID_NDK_HOME = "D:\applications\android-sdk\ndk\28.0.13004108"
$env:CARGO_HOME = "D:\applications\cargo"
$env:RUSTUP_HOME = "D:\applications\rustup"
$env:PATH = "D:\applications\flutter\bin;D:\applications\cargo\bin;$env:PATH"
```

### 国内网络配置

```powershell
# Dart/Flutter 镜像
$env:PUB_HOSTED_URL = "https://pub.flutter-io.cn"
$env:FLUTTER_STORAGE_BASE_URL = "https://storage.flutter-io.cn"

# 代理 (Clash)
$env:HTTP_PROXY = "http://127.0.0.1:7897"
$env:HTTPS_PROXY = "http://127.0.0.1:7897"

# Gradle 代理 (Java 不认 HTTP_PROXY)
$env:GRADLE_OPTS = "-Dhttp.proxyHost=127.0.0.1 -Dhttp.proxyPort=7897 -Dhttps.proxyHost=127.0.0.1 -Dhttps.proxyPort=7897"
```

Git 代理：
```bash
git config --global http.proxy http://127.0.0.1:7897
git config --global https.proxy http://127.0.0.1:7897
git config --global http.sslBackend openssl   # schannel 连代理有问题
```

### 编译命令

```powershell
cd E:\workSpace\codeProject\code2026\venera-shigc
flutter clean
flutter build apk --release
```

APK 输出: `build\app\outputs\flutter-apk\app-release.apk`

### 关键配置文件

| 文件 | 说明 |
|---|---|
| `android/key.properties` | APK 签名配置（指向 debug.keystore） |
| `android/app/debug.keystore` | 调试签名密钥 |
| `rust-toolchain.toml` | 锁定 Rust 1.85.1-gnu + Android targets |
| `D:\applications\cargo\config.toml` | Android NDK 交叉编译链接器路径 |
| `D:\applications\install-sdk.bat` | Android SDK 组件安装脚本 |

### 项目修改记录

- **2026-06-09**: 修复 `wnacg.js` — HTML 选择器适配新版 wnacg.com（`#comicName`, `#Cover`, `a.ImgA`, `a.txtA`, `span.info`, `a.dv-tag`, `div.dv-desc-text`, `div.block-pagination`）
- **2026-06-09**: 创建 `android/key.properties` + `android/app/debug.keystore`（调试签名，非原版发布密钥）
- **2026-06-09**: 修改 `rust-toolchain.toml` channel 从 `1.85.1` 改为 `1.85.1-x86_64-pc-windows-gnu`
