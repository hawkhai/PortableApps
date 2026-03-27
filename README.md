# PortableApps.com Launcher (PAL)

> **版本**: 3.0 Development (Package 2.9.0.99)  
> **许可**: GPL v2  
> **官网**: http://PortableApps.com/development

---

## 项目概述

PAL 是一个通用的便携式应用启动器框架，基于 **NSIS**（Nullsoft Scriptable Install System）构建。它的核心思想是：通过一个 INI 配置文件，就能把几乎任意 Windows 应用变成"便携版"——可以从 U 盘等移动设备运行，不留痕迹于宿主系统。

项目包含两个可执行产物：
- **`PortableApps.comLauncherGenerator.exe`** — GUI 向导，用于为某个 App 编译生成专属 Launcher
- **`<AppID>.exe`** — Generator 编译出的实际 Launcher，随每个便携 App 分发

---

## 目录结构

```
PortableApps/
├── App/
│   ├── AppInfo/
│   │   ├── appinfo.ini          # 本项目（Generator）的元数据
│   │   └── appicon.*            # 图标
│   └── NSIS/                    # 捆绑的 NSIS 编译器
├── Other/
│   └── Source/
│       ├── PortableApps.comLauncher.nsi      # Launcher 主模板
│       ├── GeneratorWizard.nsi               # Generator GUI 源码
│       ├── Segments.nsh                      # Segment 系统核心
│       ├── Segments/                         # 40 个功能模块（见下）
│       ├── Include/                          # 工具宏库
│       ├── Languages/                        # 29 种语言文件
│       ├── Languages.nsh                     # 语言加载宏
│       ├── Debug.nsh                         # 调试宏
│       ├── make_release.bat                  # 发布构建脚本
│       └── PortableApps.comLauncher.ini      # 用户配置示例
└── help.html                                 # 帮助文档入口
```

---

## 架构：Segment 系统

Launcher 采用**模块化 Segment 架构**。每个 Segment 是一个 `.nsh` 文件，通过向特定**生命周期钩子**注册宏来实现功能。主流程 `PortableApps.comLauncher.nsi` 在各钩子函数中调用 `${RunSegment}` 来驱动所有 Segment。

### 生命周期钩子（按执行顺序）

| 钩子 | 触发时机 |
|------|----------|
| `.onInit` | NSIS 最早初始化（UAC 提权、OS 检查） |
| `Init` | 加载变量、启动准备 |
| `Pre` | 启动可移植性设置（通用） |
| `PrePrimary` | 主实例：文件/注册表备份移入 |
| `PreSecondary` | 次实例：（默认空） |
| `PreExec` | 执行前（设工作目录、RunBefore） |
| `PreExecPrimary` | 主实例执行前（关闭启动画面） |
| **`Execute`** | **运行目标应用（ExecWait/Exec）** |
| `PostExec` | 执行后（RunAfter） |
| `PostExecPrimary` | 主实例执行后 |
| `PostPrimary` | 主实例：文件/注册表清理移回 |
| `PostSecondary` | 次实例：（默认空） |
| `Post` | 通用收尾 |
| `Unload` | 卸载插件，清理临时数据 |

> **Primary vs Secondary**：通过 Windows Mutex 检测。若 Launcher 已在运行，后续启动为"次实例（Secondary）"，跳过大多数设置/清理操作，`WaitForProgram=false`。

---

## 所有 Segments 一览

| Segment | 功能 |
|---------|------|
| `Core` | 读 `appinfo.ini`，确定要启动的 EXE，检测 32/64 位 |
| `Variables` | 设置所有 `PAL:*` 路径环境变量 |
| `DriveLetter` | 跟踪当前/上次驱动器盘符，供路径替换用 |
| `Language` | 自动检测/设置语言环境变量 |
| `OperatingSystem` | 检查 `MinOS`/`MaxOS` 兼容性 |
| `RunAsAdmin` | UAC 提权（`force`/`try`/`compile-force`） |
| `InstanceManagement` | 多实例管理（Mutex、SingleAppInstance） |
| `ExecString` | 构造最终命令行字符串 |
| `RunLocally` | "Live 模式"：将 App/Data 复制到本地临时目录运行 |
| `RunBeforeAfter` | 在主程序前后执行额外命令（`RunBefore1`/`RunAfter1`…） |
| `SplashScreen` | 显示启动画面（`splash.jpg`） |
| `Temp` | 管理 `%TEMP%`（清洁的隔离临时目录） |
| `Environment` | 写入 `[Environment]` 段中定义的自定义环境变量 |
| `LastRunEnvironment` | 保存/恢复上次运行的环境（用于跨运行路径替换） |
| `FileWrite` | 向配置文件写值（INI/ConfigWrite/Replace/XML） |
| `FilesMove` | Pre 时将文件从 Data 移到目标路径，Post 时移回 |
| `DirectoriesMove` | Pre 时将目录从 Data 移到目标路径，Post 时移回 |
| `DirectoriesCleanup` | 清理 Move 后遗留的空目录 |
| `DirectoryMoving` | 目录移动辅助逻辑 |
| `Registry` | 初始化 registry 插件 |
| `RegistryKeys` | 备份/恢复/删除整个注册表键 |
| `RegistryKeysDisableRedirect` | 同上，禁用 WOW64 重定向 |
| `RegistryValueBackupDelete` | 备份/删除单个注册表值 |
| `RegistryValueBackupDeleteDisableRedirect` | 同上，禁用重定向 |
| `RegistryValueWrite` | 写注册表值 |
| `RegistryValueWriteDisableRedirect` | 同上，禁用重定向 |
| `RegistryCleanup` | 清理空注册表键 |
| `RegistryCleanupDisableRedirect` | 同上，禁用重定向 |
| `Java` | 查找 Java/JDK（便携 CommonFiles 或系统安装），设 `JAVA_HOME` |
| `DotNet` | 检查所需 .NET Framework 版本 |
| `Ghostscript` | Ghostscript 支持 |
| `Qt` | Qt 框架支持（清理注册表残留） |
| `PathChecks` | 路径有效性检查 |
| `Settings` | 读取用户配置文件 |
| `Integrity` | 完整性校验 |
| `RefreshShellIcons` | 刷新 Shell 图标缓存 |
| `WorkingDirectory` | 设置工作目录 |
| `Services` | Windows 服务管理（**当前禁用**） |
| `RegisterDLL` | DLL 注册（**当前注释掉**） |
| `XML` | XML 插件卸载 |

---

## 配置文件说明

### 1. `App\AppInfo\appinfo.ini`（应用元数据）

```ini
[Details]
Name=MyApp Portable
AppID=MyAppPortable
Publisher=...
Category=...
Description=...

[Version]
PackageVersion=1.0.0.0
DisplayVersion=1.0

[Dependencies]
UsesJava=yes|optional|no          ; Java 依赖
UsesDotNetVersion=4.5             ; .NET 版本要求

[Control]
Start=MyAppPortable.exe
```

### 2. `App\AppInfo\Launcher\<AppID>.ini`（Launcher 核心配置）

```ini
[Launch]
ProgramExecutable=App\MyApp.exe         ; 要启动的程序（相对 EXEDIR）
ProgramExecutable64=App\MyApp64.exe     ; 64 位系统下的可执行文件
CommandLineArguments=--portable         ; 默认命令行参数
RunAsAdmin=force|try|compile-force      ; UAC 提权模式
WaitForProgram=true|false               ; 是否等待程序退出
CleanTemp=true|false                    ; 是否使用隔离临时目录
HideCommandLineWindow=true|false        ; 隐藏命令行窗口
SingleAppInstance=true|false            ; 禁止多实例
SinglePortableAppInstance=true|false    ; 禁止多便携实例
MinOS=XP|Vista|7|...                    ; 最低 OS 要求
MaxOS=7|2008R2|...                      ; 最高 OS 要求
RunBefore1="$PAL:AppDir$\pre.exe"       ; 启动前执行
RunAfter1="$PAL:AppDir$\post.exe"       ; 退出后执行
SplashTime=1200                         ; 启动画面显示时间（ms）

[Activate]
Registry=true                           ; 启用注册表模块
Java=require|find                       ; 启用 Java 查找
XML=true                                ; 启用 XML 操作

[Environment]
MY_VAR=some_value                       ; 自定义环境变量
MY_DIR~=%PAL:AppDir%\data               ; ~结尾表示目录变量（生成多格式版本）

[FilesMove]
settings.cfg=%APPDATA%\MyApp\config.cfg ; 文件：Data目录 -> 本地路径

[DirectoriesMove]
profile=%APPDATA%\MyApp                 ; 目录：Data目录 -> 本地路径

[RegistryKeys]
MyAppKey=HKCU\Software\MyApp            ; 整个键备份/恢复

[FileWrite1]
Type=INI
File=%AppData%\MyApp\config.ini
Section=Paths
Key=DataDir
Value=%PAL:DataDir%

[LanguageFile]
Type=INI
File=%AppData%\MyApp\config.ini
Section=Settings
Key=Language
```

### 3. `<AppID>.ini`（用户级运行时配置，放在 App 根目录）

```ini
[<AppID>]
AdditionalParameters=--extra-arg     ; 额外命令行参数
DisableSplashScreen=false            ; 禁用启动画面
RunLocally=false                     ; 启用 Live 模式
```

---

## PAL 环境变量速查

| 变量 | 含义 | 示例值 |
|------|------|--------|
| `%PAL:AppDir%` | App 目录 | `E:\PortableApps\MyApp\App` |
| `%PAL:DataDir%` | Data 目录 | `E:\PortableApps\MyApp\Data` |
| `%PAL:PortableAppsDir%` | PortableApps 根目录 | `E:\PortableApps` |
| `%PAL:PortableAppsBaseDir%` | 驱动器根的基目录 | `E:\` |
| `%PAL:Drive%` | 当前驱动器 | `E:` |
| `%PAL:DriveLetter%` | 驱动器盘符 | `E` |
| `%PAL:DrivePath%` | 驱动器根路径 | `E:\` |
| `%PAL:LastDrive%` | 上次运行的驱动器 | `D:` |
| `%PAL:Bits%` | 系统位数 | `32` 或 `64` |
| `%PAL:AppID%` | 应用 ID | `MyAppPortable` |
| `%PAL:LanguageCode%` | 语言代码 | `zh` |
| `%PAL:LanguageName%` | 语言名称 | `SIMPCHINESE` |
| `%PAL:LanguageLCID%` | Windows LCID | `2052` |
| `%PAL:LanguageCustom%` | 应用自定义语言值 | `zh_CN` |
| `%JAVA_HOME%` | Java 目录（Java 模式） | `E:\PortableApps\CommonFiles\Java` |

> 每个路径变量还有三个后缀变体：`:Forwardslash`、`:DoubleBackslash`、`:java.util.prefs`

---

## FileWrite 支持的操作类型

| Type | 用途 |
|------|------|
| `INI` | 写 INI 文件的 `[Section]\Key=Value` |
| `ConfigWrite` | 写 `Key=Value` 格式配置（可选大小写敏感） |
| `Replace` | 文件内字符串查找替换（支持 UTF-16LE/ANSI） |
| `ReplaceCommon` | 替换上次/当前路径（Data/App/Drive 等） |
| `ReplaceAll` | 同上，额外替换 PortableApps 文档/图片等路径 |
| `XML attribute` | 修改 XML 属性（需 `[Activate]:XML=true`） |
| `XML text` | 修改 XML 文本节点（需 `[Activate]:XML=true`） |

---

## 注册表映射规则

注册表键名会被自动规范化：

| 原始键名 | 规范化为 |
|----------|----------|
| `HKEY_CLASSES_ROOT\...` | `HKCU\Software\Classes\...` |
| `HKCR\...` | `HKCU\Software\Classes\...` |
| `HKEY_CURRENT_USER\...` | `HKCU\...` |
| `HKEY_LOCAL_MACHINE\...` | `HKLM\...` |

备份位置：`HKCU\Software\PortableApps.com\Keys\<原始键路径>`

---

## 自定义与调试

### 自定义代码（`App\AppInfo\Launcher\Custom.nsh`）

可以在任意钩子注入代码，或覆盖整个 Execute 函数：

```nsis
; 禁用某个 Segment 的某个钩子
${DisableHook} FilesMove PrePrimary

; 禁用整个 Segment
${DisableSegment} SplashScreen

; 覆盖 Execute 函数
${OverrideExecute}
    Exec "$ExecString"
!macroend
```

### 调试（`App\AppInfo\Launcher\Debug.nsh`）

```nsis
!define DEBUG_ALL          ; 调试全部
!define DEBUG_GLOBAL       ; 调试全局段
!define DEBUG_SEGMENT_Core ; 调试特定 Segment
!define DEBUG_SEGWRAP      ; 显示 Segment 进出包装信息
```

调试输出默认写入 `Data\debug.log`，也可设为弹出消息框。

---

## 构建流程

### 生成单个 App 的 Launcher

1. 运行 `PortableApps.comLauncherGenerator.exe`
2. 选择 App 根目录（含 `App\AppInfo\appinfo.ini`）
3. Generator 调用 `App\NSIS\makensis.exe` 编译 `PortableApps.comLauncher.nsi`
4. 输出：`<Package>\<AppID>.exe`

命令行方式（静默编译）：
```
PortableApps.comLauncherGenerator.exe "X:\PortableApps\MyAppPortable"
```

### 构建 Generator 本身

```bat
Other\Source\make_release.bat
```

---

## 支持的语言（29 种）

Armenian · Bulgarian · Danish · Dutch · **English** · Estonian · Finnish · French · Galician · German · Hebrew · Hungarian · Indonesian · Italian · Japanese · Polish · Portuguese · PortugueseBR · Romanian · **SimpChinese（简体中文）** · Slovenian · Spanish · Sundanese · Swedish · Thai · **TradChinese（繁体中文）** · Turkish · Vietnamese

---

## Include 工具库速查

| 文件 | 功能 |
|------|------|
| `ProcFunc.nsh` | 进程查找/等待（`ProcessExists`、`ProcessWaitClose`） |
| `ForEachINIPair.nsh` | 遍历 INI 段的所有键值对 |
| `ForEachPath.nsh` | 支持通配符的文件/目录遍历 |
| `SetEnvironmentVariable.nsh` | 设置环境变量 |
| `ReplaceInFileWithTextReplace.nsh` | 文件内文本替换（ANSI/UTF-16LE） |
| `DotNet.nsh` | .NET Framework 版本检测 |
| `MyXML.nsh` | XML 读写操作 |
| `NullByte.nsh` | Null 字节处理 |
| `TrimWhite.nsh` | 字符串首尾空白修剪 |
| `EmptyWorkingSet.nsh` | 释放工作内存集 |
| `CallANSIPlugin.nsh` | 调用 ANSI 版 NSIS 插件 |

---

## 运行时数据文件

| 文件 | 位置 | 用途 |
|------|------|------|
| `PortableApps.comLauncherRuntimeData-<AppID>.ini` | `Data\` | 崩溃恢复数据（正常退出后删除） |
| `<AppID>Settings.ini` | `Data\settings\` | 持久化设置（LastDrive、Language等） |
| `*.reg` | `Data\settings\` | 注册表键快照 |
| `debug.log` | `Data\` | 调试日志（仅 Debug 构建） |
| `PortableApps.comLauncherGeneratorLog.txt` | Generator `Data\` | 编译日志 |
