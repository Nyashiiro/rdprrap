# 用 winpodx 做运行时测试

[English](TESTING_WINPODX.md) | [한국어](TESTING_WINPODX.ko.md) | **中文**

[winpodx](https://github.com/kernalix7/winpodx) 在 Linux 上运行真正的
Windows 容器（dockur/windows + KVM/QEMU，通过 FreeRDP RemoteApp 暴露在
`127.0.0.1:3390`）。由于它启动的是未经修改的 Windows 镜像 —— 不是
WINE —— 容器内的 `termsrv.dll`、`umrdp.dll`、`rdpendp.dll` 全都是
货真价实的原版文件。因此，winpodx 是在 Linux 开发机上、不另备 VM 就能
最快验证 [TESTING.zh.md](TESTING.zh.md) 中 **x64 各行** 的手段。

本文是 [TESTING.zh.md](TESTING.zh.md) 的一个子集，按容器环境做了适配。
winpodx 覆盖不到的 x86 覆盖、ARM64 构建／静态检查、ARM64 运行时验证、
多 OS 矩阵，请回到 CI 或独立 VM。

## 覆盖范围

| 可覆盖                                                        | 覆盖不到                                                     |
|---------------------------------------------------------------|--------------------------------------------------------------|
| Win10/11/Server 2022/2025 上的 x64 `termwrap`／`umwrap`／`endpwrap` | x86（i686）构建与 ARM64 运行时验证 —— dockur/windows 仅支持 x64 |
| SYSTEM 权限下的安装／卸载往返                                  | 多架构 USB 重定向（取决于宿主的 USB passthrough）             |
| 对容器 termsrv.dll 执行 `offset-finder --assert-all`           | 一次运行并行覆盖多个 Windows 版本的矩阵                       |
| 用第二个 RDP 客户端做多会话 smoke                              | Podman/KVM 未暴露的物理外设                                   |
| 用 DebugView 捕获 `OutputDebugString` 日志                     | 依赖物理显示器的触控／DPI 缩放怪癖                            |

## 0. 前置准备

- 按 winpodx README 的说明装好 KVM + Podman（或 Docker）的 Linux 宿主机。
- winpodx 已能正常工作（`winpodx run notepad` 成功）。
- `rdprrap` 的 release 产物。在 Linux 上请使用
  [`cargo-xwin`](https://github.com/rust-cross/cargo-xwin) —— 它会自动
  取来 `cargo build --target x86_64-pc-windows-msvc` 自己找不到的
  MSVC SDK + CRT 并完成链接：
  ```bash
  cargo install cargo-xwin            # 一次性
  sudo zypper install lld             # openSUSE（或：apt install lld）
  cargo xwin build --release --target x86_64-pc-windows-msvc --workspace
  ```
  产物生成在 `target/x86_64-pc-windows-msvc/release/` 下：
  `termwrap_dll.dll`、`umwrap_dll.dll`、`endpwrap_dll.dll`、
  `rdprrap-installer.exe`、`rdprrap-check.exe`、`rdprrap-conf.exe`、
  `offset-finder.exe`。安装器会在安装时把 DLL 改名为正式名称
  （`termwrap.dll`、`umwrap.dll`、`endpwrap.dll`）。
  > 如需 i686 构建（winpodx 本身只支持 x64，因此仅当你还计划按
  > [TESTING.zh.md](TESTING.zh.md) 另起一台 32 位 VM 时才需要），
  > cargo-xwin 默认不会拉取 x86 SDK，需显式添加：
  > ```bash
  > XWIN_ARCH=x86,x86_64 cargo xwin build --release \
  >   --target i686-pc-windows-msvc --workspace
  > ```
  > ARM64 产物与实验性 ARM64 patcher 请在 CI 或支持 ARM64 的 MSVC
  > 工具链中构建。但 winpodx 仍仅支持 x64，ARM64 的运行时证据请到真实的
  > Windows ARM64 硬件或 ARM64 VM 上获取。
- 宿主机上使用的第二个 RDP 客户端：`xfreerdp` 或 `Remmina`。
- 把 **DebugView**（`Dbgview.exe`）复制进容器，日志分析会省力得多
  （可选，但强烈建议）。

## 1. 准备容器

1. 在 winpodx 配置中固定一个确定的 Windows 镜像 —— 例如 Windows 11
   24H2 或 Server 2022。把所选的 `version=` 值记下来，稍后要写进测试
   报告。
2. 测试期间关闭 `auto_suspend`：
   ```toml
   # winpodx.toml
   [pod]
   auto_suspend = false
   ```
   重启 TermService 会让 RemoteApp 会话短暂掉线；如果 winpodx 判定 pod
   已空闲，可能在测试中途把它 suspend。
3. 启动容器：
   ```bash
   winpodx pod start
   winpodx pod status   # 确认 RDP 已在 127.0.0.1:3390 就绪
   ```
4. 给容器存储打快照（视 winpodx 配置而定：Podman 的
   `podman container commit`，或底层 qcow2 overlay）。安装／卸载各轮
   之间会还原到这份快照。

## 2. 把产物送进容器

winpodx 默认把 Linux 家目录挂载为 `\\tsclient\home`，所以最简单的办法是：

```powershell
# 在 Windows RDP 会话内的管理员 PowerShell 中：
New-Item -ItemType Directory -Path C:\rdprrap -Force
Copy-Item \\tsclient\home\<you>\…\target\x86_64-pc-windows-msvc\release\*.exe C:\rdprrap\
Copy-Item \\tsclient\home\<you>\…\target\x86_64-pc-windows-msvc\release\*.dll C:\rdprrap\
```

或者，如果你配置过 winpodx 的 SSH 端口，也可以走 scp。无论哪种方式，
最终状态都应是：`C:\rdprrap\` 下齐备全部 7 个产物。

## 3. Preflight

在动 TermService 之前，先确认 patcher 真的认得容器里的 termsrv.dll：

```powershell
cd C:\rdprrap
.\offset-finder.exe --assert-all C:\Windows\System32\termsrv.dll
```

- [ ] 退出码为 0。
- [ ] 报告中没有 `NOT_FOUND` 行。
- [ ] `[Patch Dry-Run]` 区块（`--assert-all` 隐含 `--dry-run`）中出现
      已解析的 `def_policy: base=... patch_rva=... bytes=[...]` 行，而
      不是 `def_policy: NOT FOUND`。`property_device` 会被有意跳过
      （`needs runtime walk — skipped`）。

如果这一步失败，请**就此停手**，不要运行安装器。请留存完整 stdout 以及
termsrv 版本（`Get-Item C:\Windows\System32\termsrv.dll | Select-Object
-Expand VersionInfo` 的输出）—— 这说明 patcher 的 pattern 集对该构建
已经过时，强行运行安装器会让服务停留在损坏状态。

## 4. 安装

```powershell
# 开启 transcript —— 重启 TermService 时 RDP 会话一定会断开。
Start-Transcript -Path C:\rdprrap\install.log -Append
.\rdprrap-installer.exe plan     # 预览契约
.\rdprrap-installer.exe install  # RDP 会话在此断开 → 自动重连
Stop-Transcript
```

- [ ] install transcript 显示退出码为 0。
- [ ] 重连之后，`Get-Service TermService` 显示 `Running`。
- [ ] `%ProgramFiles%\RDP Wrapper\` 存在，且包含 `termwrap.dll`、
      `umwrap.dll`、`endpwrap.dll`。
- [ ] `reg query "HKLM\SYSTEM\CurrentControlSet\Services\TermService\Parameters" /v ServiceDll`
      指向 `%ProgramFiles%\RDP Wrapper\` 内部。
- [ ] `icacls %ProgramFiles%\RDP Wrapper` 显示 SYSTEM + Administrators
      拥有完全控制，其他任何人都无写权限。

## 5. 多会话 smoke

winpodx 的 RemoteApp 会话占据 session 1。要证明多会话 patch 真的生效，
请在 Linux 宿主机上向 `127.0.0.1:3390` 打开**第二个** RDP 连接，用另一个
本地 Windows 账户登录：

```bash
xfreerdp /u:user2 /p:<password2> /v:127.0.0.1:3390
```

- [ ] 第二个会话完成登录，且不把 winpodx 会话踢下线。
- [ ] 两个会话可同时使用（切换焦点、各自输入）。
- [ ] DebugView 显示 `TermWrap:` patch-applied 行，`patch not found`
      为 0 条。
- [ ] 会话内运行的 `rdprrap-conf.exe` 将 Wrapper、TermService、termsrv
      版本、RDP-Tcp 监听器全部显示为绿色。
- [ ] `rdprrap-check.exe` 报告回环 RDP OK。

> **注意**：如果第二个会话被以 `CONNECTION_TERMINATED` 拒绝，请先查
> DebugView —— 最常见的原因是 `NonRDPPatch` 或 `DefPolicyPatch` 缺失。

## 6. umwrap —— PnP + 摄像头（可选，需要 Podman USB passthrough）

winpodx 会原样继承 Podman／libvirt 栈暴露的 USB passthrough。如果你把
U 盘或摄像头接进了容器：

- [ ] 在 RDP 客户端启用 USB 驱动器重定向
      （`xfreerdp /drive:usbstick,/media/usb`）。
- [ ] U 盘以驱动器盘符出现在 RDP 会话内。
- [ ] DebugView 显示 `UmWrap:` patch-applied 行。
- [ ] （若可行）摄像头重定向也能用。

如果容器没有物理 USB 访问能力，跳过本节并在最终报告中注明 —— 完整清单
中的对应行将保持未覆盖状态。

## 7. endpwrap —— 音频捕获

- [ ] 以 `/microphone`（xfreerdp）或含 `audiocapture:i:1` 的 `.rdp`
      文件启动第二个 RDP 客户端。
- [ ] 会话内的"声音设置 → 输入"中能看到被重定向的麦克风。
- [ ] DebugView 显示 `EndpWrap:` patch-applied 行。

## 8. 卸载

```powershell
Start-Transcript -Path C:\rdprrap\uninstall.log -Append
.\rdprrap-installer.exe uninstall   # 会话在此断开 → 重连
Stop-Transcript
```

- [ ] 退出码为 0。
- [ ] `ServiceDll` 还原为 `%SystemRoot%\System32\termsrv.dll`。
- [ ] `%ProgramFiles%\RDP Wrapper\` 已删除。
- [ ] 防火墙规则 `rdprrap-RDP-TCP`、`rdprrap-RDP-UDP` 已删除。
- [ ] `HKLM\SOFTWARE\rdprrap\Installer` 已删除。
- [ ] 第二个会话的登录不再成功（多会话恢复为 Windows 默认行为）。

## 9. 快照回滚 + 换下一个镜像

1. 停止容器：
   ```bash
   winpodx pod stop
   ```
2. 还原到步骤 1.4 拍下的快照。
3. 要覆盖其他 Windows 版本，改一下 winpodx 配置里的 `version=`，从
   步骤 1 重新开始。

## 报告

每次运行记录一行：

```
容器镜像         : <winpodx 配置中的 version=...>
termsrv.dll 版本 : <(Get-Item ...).VersionInfo 的输出>
架构             : x64
日期             : <YYYY-MM-DD>
通过的章节       : 3、4、5、7、8    （没有 USB passthrough 则不含 6）
DebugView 摘录   : <出现的 "not found" 行，或 "none">
备注             : <winpodx 配置差异、异常情况>
```

请在报告中附上 Start-Transcript 输出与 DebugView 日志。有这些材料，日后
排查时就无需再把容器重新启动一遍。