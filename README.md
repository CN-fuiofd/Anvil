# Anvil

> [!NOTE]
> 文档版本 v1.1 ｜ 对应程序版本 0.11.0-alpha ｜ 最后更新 2026-10-22

Anvil 是一个 .NET 10 + Avalonia 12 的 Windows 系统锻造与修复工具。它**不依赖** `sfc.exe`、`DISM`、`bcdedit.exe`、`reg.exe` 等系统内置修复工具——所有修复逻辑通过 Win32 API 和内置规则库自实现。

## 下载与更新

- 最新版本：<https://github.com/CN-fuiofd/Anvil/releases>
- 单文件版（自包含，双击即用）：`Anvil.<版本>.exe`
- 程序内置更新检查，读取仓库根目录的 `update.json`；下载后会做 SHA-256 强校验，不匹配即拒绝安装。

**系统要求**：Windows 10 / 11（x64）。主程序为 `asInvoker`，不主动提权；需要写入 HKLM、加载 hive、改系统文件时会按需拉起一次 UAC 提权副本。数据与配置都在 `%LOCALAPPDATA%\Anvil\`，不写注册表持久化配置。

> 这是 Anvil 2.0，较上一代版本重写实现；旧代码仅作参考保留。
>
> 说明：主修复路径全部自实现（进程内 P/Invoke + 规则库），仅**离线修复页的 SFC / DISM 兜底**仍会调用系统工具 —— 那是内置扫描修不好时由用户显式触发的最后手段（`OfflineSfcService` 优先走内置扫描，`sfc.exe` 不可用时自动降级）。`DllRepairService` 里曾经残留的 SFC/DISM 调用已于 0.11.0 删除。

## 核心设计原则

1. **不依赖系统修复工具**：注册表操作走 `advapi32` P/Invoke，文件完整性走 SHA-256 比对 + 用户提供的健康源，启动项走直接注册表/文件操作。
2. **MVVM + DI**：CommunityToolkit.Mvvm + Microsoft.Extensions.Hosting，所有服务通过构造注入，可测试。
3. **无空壳项目**：每个项目都有实际代码，没有占位 csproj。
4. **平台注解**：Windows-only API 用 `[SupportedOSPlatform("windows")]` 显式标注，核心逻辑库可跨平台编译/测试。

## 修复引擎技术方案

| 修复领域 | 系统工具（禁用） | Anvil 自实现方案 |
|---|---|---|
| 注册表 | `reg.exe` | `RegLoadKeyW`/`RegUnLoadKeyW` + 内置 `RegistryRepairRules` 规则库 |
| 系统文件完整性 | `sfc.exe` / `DISM` | SHA-256 比对，从用户提供的健康源复制替换 |
| 启动项 | — | 直接操作 Run 键 + 扫描 Startup 文件夹快捷方式 |
| 驱动签名 | — | `CryptCATAdmin*` 目录清单哈希扫描驱动目录 |
| 离线系统修复 | `sfc /offbootdir` | 挂载目标 hive 后按目标路径读写，写前备份到目标盘 |

## 离线修复（WinPE / WinRE）

在 PE 里启动后，可在「离线修复」页选定一个离线 Windows 安装，**全站**随之把该安装当作修复目标：

- 系统文件还原、DLL 修复、字体、网络、WMI、启动项、注册表规则、服务配置、用户配置文件都按目标盘寻址；
- 目标注册表以 `RegLoadKey` 挂载，写入前会把原值记进 `<目标盘>\AnvilBackup\registry-undo.jsonl`；
- 只在本机才有意义的能力（显示设置、桌面与任务栏、Windows 更新重置、文件关联、系统还原）在离线时明确禁用并写明原因；
- **要执行的工具仍取宿主自己那一份**（PE 里的 `regsvr32`），只有「检测与修复的目标」指向目标系统。

> 已知限制：离线的「写回 + 写前备份落在目标盘」尚未在真机 WinPE 环境实测。

## 构建

```powershell
dotnet restore Anvil.sln
dotnet build Anvil.sln --no-restore
```

## 测试

```powershell
dotnet test Anvil.sln
```

## 运行

```powershell
dotnet run --project src/Anvil.App
```

> 应用清单为 `asInvoker`（不主动请求提权）。需要管理员权限的操作（HKLM 注册表写入、注册表 hive 加载、系统文件修复等）会按需通过 UAC 提权拉起提权副本，用户取消授权不会让程序出错。

## 关于开发方式

本项目采用 Vibe Coding 方式开发：需求定义、方案取舍与验收由作者负责，代码由 AI 助手起草，
每一轮改动都须通过编译、全量测试（2600+ 条）与真机验证后才进入版本。

- 变更记录（`CHANGELOG.md`）保留每轮的判断依据、已知限制与未验证项，不隐藏未完成部分；
- 发布包附带 SHA-256 校验和，更新链路强制校验，不匹配即拒绝安装；
- 涉及系统写入的操作在测试模式下只记录、不执行，避免测试污染本机。

## 与上一代版本的对比

| 维度 | 上一代 (1.0) | Anvil (2.0) |
|---|---|---|
| UI 框架 | WPF | Avalonia 12 |
| MVVM | 手写 + 未连接到 App | CommunityToolkit.Mvvm + 真正绑定 |
| DI | 无，到处 `new` | Microsoft.Extensions.Hosting |
| 日志 | 静态 `Logger`，无文件 | `ILogger<T>` + 控制台 + 文件 |
| 空壳项目 | 5 个 | 0 个 |
| 依赖安装 | 假实现（写 marker 文件） | 已移除该入口 |
| WinPE 检测 | 逻辑错误 | 修正（X:\ + 环境变量 + Windows 目录） |
| P/Invoke | `RegLoadKey` 返回值误判为 bool | 正确的 `int` + `Win32Errors.ThrowIfFailed` |
| 修复引擎 | 调用系统工具 | 完全自实现 |
| 测试覆盖 | 3 个 | 2600+ 个，覆盖核心修复逻辑 |
