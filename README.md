# copicseal-static

Copicseal 的静态资源仓库：桌面端「设置 → 插件」里那些**按需下载的本机组件**放在这里。
App 默认把本仓库 `main` 分支当作下载源：

```txt
https://raw.githubusercontent.com/copicseal/copicseal-static/main
```

用户可在设置里追加镜像源；每个源按同样的目录结构组织，谁先给出可用清单就用谁。

## 目录约定

一个组件一个目录，目录名 = 组件 id：

```txt
<id>/
  manifest.json            清单：元数据 + 每个资产的体积与校验和
  <id>-<platform>.zip      平台资产
```

例如：

```txt
libheif/    manifest.json + libheif-windows-{x86_64,aarch64,x86}.zip
exiftool/   manifest.json + exiftool-windows-{x86_64,x86}.zip + exiftool-perl.zip
```

App 读取的是 `<源>/<id>/manifest.json`，资产也从同一个源下载 —— 镜像只要复刻同样的目录结构即可。

## manifest.json 格式

```json
{
  "id": "libheif",
  "version": "1.23.1",
  "notes": "Windows HEIC/HEIF 转码（libheif 1.23.1）",
  "assets": [
    { "name": "libheif-windows-x86_64.zip", "size": 12910157, "sha256": "EBF1FAEA8D98DCEB359174A97679C6DE60567A67521B872905F809BC68DD4978" }
  ]
}
```

| 字段 | 说明 |
|------|------|
| `id` | 组件 id，需与 App 内置目录（`src-tauri/src/plugin.rs` 的 `catalog()`）里的 id 一致 |
| `version` | 版本号；安装目录按版本存放，可并存与回滚 |
| `notes` | 可选说明，显示在设置面板里 |
| `assets[]` | 每个资产一条：`name` 文件名、`size` 字节数、`sha256` **大写十六进制** |

> 「平台 → 文件」的对应关系在 **App 的目录里**，不在清单里：清单只声明「有哪些文件、多大、摘要多少」。
> 多个平台可以指向同一个文件（exiftool 的 Perl 包同时服务 macOS 与 Linux）。
> 下载前后 App 都会按 `size` / `sha256` 校验，不匹配就丢弃。

## 平台标识

| 平台 | 标识 | 说明 |
|------|------|------|
| Windows | `windows-x86_64` / `windows-aarch64` / `windows-x86` | 细分到架构 |
| macOS | `macos` | 只到系统，不区分架构 |
| Linux | `linux` | 同上 |

macOS 上 libheif 那类 HEIC 解码组件不需要（系统自带能力，面板显示「已内置该能力，无需安装」）。

## 为什么统一用 zip

所有资产都是 zip，**不要用 tar.gz**：App 侧已经有 zip 解包（`plugin.rs::extract_zip`），
换成 tar.gz 就得额外引入 `flate2` + `tar` 并新增一条解包分支。

- Windows 的 zip 直接来自上游发行包（重命名为 `<id>-windows-<arch>.zip`）
- macOS/Linux 用的 Perl 版由上游 `Image-ExifTool-<版本>.tar.gz` 重新打包（见下），
  用支持「按扩展名选格式」的打包器（`tar -a -cf x.zip …`、7-Zip）**以保证 zip 内路径分隔符是 `/`**，
  并把上游的版本目录拍平，让解压结果直接是 `exiftool` + `lib/`

## 组件形态可以很不一样（以 exiftool 为例）

组件不是「一种东西」，探测与安装逻辑按组件分别定义，`exiftool` 就和 `libheif` 完全不同：

| | libheif | exiftool |
|---|---|---|
| 适用平台 | 仅 Windows | Windows + macOS + Linux |
| 资产形态 | 自包含 zip（含 `heif-dec.exe`）| Windows：自包含 exe（内含 Perl 运行时）<br>macOS/Linux：**Perl 版**（依赖系统自带的 `perl`）|
| 安装前探测 | 系统是否已有 HEIC 解码能力 | **系统是否已有 `exiftool`** —— 有则直接显示「已安装」，不下载 |
| 调用方式 | 直接执行解压出的 `heif-dec.exe` | 经 `perl` 调用脚本（见下）|
| 兜底 | — | macOS 上若系统连 `perl` 都没有，提示用户到 <https://exiftool.org/> 自行下载 |

对应到资产：

| 文件 | 用途 |
|------|------|
| `exiftool-windows-x86_64.zip` | Windows 64 位：解压即用，内含 Perl 运行时 |
| `exiftool-windows-x86.zip` | Windows 32 位：同上 |
| `exiftool-perl.zip` | macOS / Linux：上游 Perl 发行包（`exiftool` 脚本 + `lib/Image/ExifTool*`），需要系统有 `perl` |

Perl 版的调用方式（App 侧照此实现；`-I` 显式带上 `lib`，不依赖脚本自身的路径探测）：

```sh
perl -I <安装目录>/lib <安装目录>/exiftool -json -n -ShutterCount <照片>
```

> 上游没有 Windows aarch64 版本：该架构下目前没有可用资产，面板会显示「当前系统没有该组件的版本」。

## 新增或更新一个组件

1. 把自包含资产放进 `<id>/`，命名 `<id>-<platform>.zip`
2. 算体积与校验和，写进 `manifest.json`：

   ```powershell
   Get-ChildItem <id>\*.zip | ForEach-Object {
     "{0}  {1}  {2}" -f $_.Name, $_.Length, (Get-FileHash $_.FullName -Algorithm SHA256).Hash
   }
   ```

3. 提交并推送到 `main` —— App 直接读 `raw.githubusercontent.com`，不需要额外的发布流程

## 约定

- **资产必须自包含**：除「Perl 版 exiftool 依赖系统 perl」这种已声明的例外，不依赖系统里已有的东西
- **只放发行产物**：上游源码包只用于本地重建时，放 `<id>/src/` 并加进 `.gitignore`，不随仓库分发
- 新增平台或组件时，先在 App 的 `catalog()` 里登记「平台 → 文件名」和该组件的探测/安装逻辑，再往这里放包
