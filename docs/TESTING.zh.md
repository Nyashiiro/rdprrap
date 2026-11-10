# Windows 运行时验证清单

[English](TESTING.md) | [한국어](TESTING.ko.md) | **中文**

Linux CI 与 Windows CI 构建已覆盖 x64/x86 的编译 + clippy + 单元测试，
以及 ARM64 构建／静态检查。但有些环节是 CI 覆盖不到的：

- 把 wrapper DLL 真正加载进 `svchost.exe`／`umrdp.dll`／`rdpendp.dll`
  宿主并确认其行为。
- 针对 Windows 10、11、Server 2022、Server 2025 实际自带的
  `termsrv.dll` 字节运行 patch 路径。
- 以 `SYSTEM` 权限端到端地验证安装器／卸载器。

本文记录用于填补上述空缺的手动验证流程。请在一次性 VM 快照上执行，每轮
之间还原快照，确保 ACL、注册表与服务状态都被干净地复位。

如果想在 Linux 上、不另起 VM 就把 x64 各行至少跑一遍，请参见
[TESTING_WINPODX.zh.md](TESTING_WINPODX.zh.md) —— 其中介绍了把
[winpodx](https://github.com/kernalix7/winpodx) 容器（dockur/windows +
FreeRDP）当作目标 Windows 宿主来复用的流程。

## 前置准备

- 默认未启用 RDP 的 Windows VM。
- 由 `cargo build --release` 生成、与目标架构匹配的构建产物（x64 宿主 →
  `x86_64-pc-windows-msvc`，x86 宿主 → `i686-pc-windows-msvc`，ARM64
  宿主 → `aarch64-pc-windows-msvc`）。
- 管理员账户，另一台机器上的远程桌面客户端（`mstsc.exe`）。
- 可选：DebugView（SysInternals）—— 用于捕获 `OutputDebugString`。

## 待验证的构建矩阵

下表每一行都必须在该目标 OS 支持的每种运行时架构上验证。较新的 Windows
SKU 已不再官方提供 x86，但旧款 x86 VM（Win10 32 位、Windows 7 实验室
镜像）仍是 i686 路径获得真实覆盖的唯一场所。ARM64 仍属实验性质：
`termwrap`／`umwrap`／`endpwrap` 都已有 ARM64 runtime patcher，但完整的
RDP 路径仍须在真实 ARM64 硬件／VM 上验证。

| OS                 | x64 | x86 | ARM64 | 备注                                               |
|--------------------|-----|-----|-------|----------------------------------------------------|
| Windows 10 22H2    | ✅  | ⚠️  | —     | x86 覆盖只能借助 legacy 镜像                        |
| Windows 11 23H2    | ✅  | —   | ⚠️    | ARM64 实验性运行时检查                              |
| Windows 11 24H2    | ✅  | —   | ⚠️    | 最新消费级 SKU；ARM64 实验性运行时                  |
| Server 2022        | ✅  | —   | —     | 与 `windows-latest` runner 匹配                     |
| Server 2025        | ✅  | —   | —     | 与 `windows-2025` runner 匹配                       |

某 OS／架构组合只有在下方**安装／运行时／卸载**各节全部通过之后，才能
勾选其运行时行。在 `termwrap` 真正打出 ARM64 patch-applied 日志、且
并发会话 smoke 测试通过之前，不要把 ARM64 标记为生产级支持。

## 1. 安装

```powershell
# 在目标 VM 的管理员 PowerShell 中执行。
.\rdprrap-installer.exe plan   # 预览契约，不做任何更改
.\rdprrap-installer.exe install
```

通过标准：

- [ ] 退出码为 0。
- [ ] 创建了 `%ProgramFiles%\RDP Wrapper\`，其中包含 `termwrap.dll`、
      `umwrap.dll`、`endpwrap.dll`。
- [ ] `HKLM\SYSTEM\CurrentControlSet\Services\TermService\Parameters\ServiceDll`
      指向 `%ProgramFiles%\RDP Wrapper\` 内部。
- [ ] `HKLM\SOFTWARE\rdprrap\Installer` 存在，且仅 SYSTEM + Administrators
      可写（见 `icacls` 输出）。
- [ ] 存在防火墙规则 `rdprrap-RDP-TCP` 与 `rdprrap-RDP-UDP`，放行
      TCP/UDP 3389。
- [ ] `sc query TermService` 显示 `STATE : 4 RUNNING`。

## 2. 运行时 —— termwrap（x64 + x86 + 实验性 ARM64）

```powershell
# 在另一台机器上：
mstsc /v:<target-ip>
```

通过标准：

- [ ] 当本地已有另一名管理员登录时，仍能用非管理员凭据完成 RDP 连接
      （并发会话 smoke 测试）。
- [ ] DebugView 中能看到 `TermWrap:` patch-applied 日志，且没有
      `patch not found` 警告。
- [ ] 在 ARM64 上，DebugView 显示 ARM64 DefPolicy、SingleUser、
      LocalOnly、AppServer/NonRDP、PropertyDevice、SL policy 的 patch
      日志。
- [ ] （在目标机上运行的）`rdprrap-check` 报告回环 RDP OK。
- [ ] （在目标机上运行的）`rdprrap-conf` 将 Wrapper、TermService、
      termsrv 版本、RDP-Tcp 监听器显示为绿色。

在 Windows 11 上也要重复一遍，以捕捉 termsrv.dll 的布局变化。
`rdprrap-conf` 中任何一行变红，都意味着 patcher 未能解析出 offset ——
请运行 `offset-finder --assert-all C:\Windows\System32\termsrv.dll`，
依据其报告进行排查。

## ARM64 支持状态

在 ARM64 Windows 上，`termwrap`、`umwrap`、`endpwrap` 都可能打出 ARM64
patch-applied 日志。`termwrap` 应显示 DefPolicy、SingleUser、LocalOnly、
AppServer/NonRDP、PropertyDevice、SL policy 的 ARM64 日志。在这些日志与
真实 Windows ARM64 并发会话 smoke 测试一起通过之前，ARM64 仍属实验性
支持。

## 3. 运行时 —— umwrap（PnP 重定向）

目标：证明 i686 与 ARM64 的设备重定向路径不只是编译通过，而是真的
patch 了东西。

- [ ] 在 RDP 客户端重定向 USB 存储设备（`mstsc` → 本地资源 → 详细信息
      → 驱动器）。
- [ ] 设备出现在 RDP 会话内的"此电脑"中。
- [ ] DebugView 显示 `UmWrap:` patch-applied 日志，且没有
      `PnpRedirection patch not found`。
- [ ] 在 ARM64 上，DebugView 显示 `UmWrap: ARM64 PnP patch applied`。

摄像头重定向（Win10+）：

- [ ] USB 摄像头重定向可用。
- [ ] 当 `.rdata` 中存在 `CameraRedirectionAllowed` 字符串时，DebugView
      显示 camera-secondary patch 已应用的日志。
- [ ] 在 ARM64 上，若该字符串存在于 `.rdata`，DebugView 显示
      `UmWrap: ARM64 Camera patch applied`。

## 4. 运行时 —— endpwrap（音频捕获）

- [ ] 在 `.rdp` 文件中写入 `audiocapture:i:1`（或在本地资源 → 远程音频
      → 录音里选择"从此计算机录音"），使 RDP 客户端把麦克风音频捕获进
      远程会话。
- [ ] DebugView 显示 `EndpWrap:` patch-applied 日志。
- [ ] 在 ARM64 上，DebugView 显示
      `EndpWrap: ARM64 AllowAudioCapture patched`。

## 5. 安装器 preflight 与失败用例

- [ ] 在 `offset-finder --assert-all` 未覆盖的 OS/termsrv 构建上运行
      安装器 ⇒ 以版本不匹配错误中止（CheckTermsrvVersion）。
- [ ] 以 `--skip-firewall` 运行 ⇒ 不创建防火墙规则，其余步骤照常完成。
- [ ] 以 `--skip-restart` 运行 ⇒ TermService 保持原状态，重启由用户
      自行负责。
- [ ] 以 `--disable-nla` 运行 ⇒ `HKLM\...\RDP-Tcp\UserAuthentication`
      被设为 0，卸载时还原为先前的值。
- [ ] 在既有安装上重复运行安装器（幂等性）：成功，重新登记相同状态，
      `HKLM\SOFTWARE\rdprrap\Installer` 备份子树没有 drift。
- [ ] 模拟备份被篡改：手动把
      `HKLM\SOFTWARE\rdprrap\Installer\OriginalServiceDll` 改成
      `C:\ProgramData\Evil.dll`，然后执行卸载 ⇒ 卸载拒绝还原（走 SDDL
      保护的校验路径）。

## 6. 卸载

```powershell
.\rdprrap-installer.exe uninstall
```

通过标准：

- [ ] 退出码为 0。
- [ ] `ServiceDll` 还原为 `%SystemRoot%\System32\termsrv.dll`。
- [ ] `%ProgramFiles%\RDP Wrapper\` 已删除。
- [ ] 防火墙规则已删除。
- [ ] `HKLM\SOFTWARE\rdprrap\Installer` 已删除。
- [ ] `fDenyTSConnections` 还原为安装前的值（如果安装器记录过）。
- [ ] AddIns 子树只在安装器最初创建过 AddIns 父键时才删除；既有的
      AddIns 配置不受影响。
- [ ] 卸载后 TermService 仍能正常启动；RDP 行为与全新系统完全一致。

## 7. offset-finder 运行时 smoke

在每个 OS 镜像上：

```powershell
offset-finder --assert-all C:\Windows\System32\termsrv.dll
```

- [ ] 退出码为 0。
- [ ] 所有具名字符串都解析成功（没有 `NOT_FOUND`）。
- [ ] 所有具名函数都经 xref 解析成功（没有 `NOT_FOUND`）。
- [ ] 用 `--dry-run` 运行（或依赖隐含它的 `--assert-all`），确认
      `[Patch Dry-Run]` 区块中出现已解析的 DefPolicy 行，例如：
      `def_policy: base=... value=... write_disp=... is_jnz=... patch_rva=... bytes=[...]`
      —— 而**不是** `def_policy: NOT FOUND`。
- [ ] `property_device` 的 dry-run 会被有意跳过（它需要运行时的 GUID
      遍历），因此出现
      `property_device dry-run: needs runtime walk — skipped`
      行属正常现象，不算失败。

如果在受支持的 OS 上以上检查失败，请开一个 patcher-team 工单，附上完整
stdout 与 termsrv.dll 版本（`(Get-Item termsrv.dll).VersionInfo`）。

## 报告

每次执行清单后，请按下列字段记录结果：OS build 号、termsrv.dll 版本、
架构、日期、各节的通过／失败情况，以及能展示 `not found` 消息的
DebugView 摘录。offset drift 是最主要的失败原因 —— 一份完整的
DebugView 捕获就是最快的排查工具。