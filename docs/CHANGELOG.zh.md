# 变更日志

[English](../CHANGELOG.md) | [한국어](CHANGELOG.ko.md) | **中文**

本项目的重要变更均记录于此文件。

格式基于 [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)，
版本策略遵循 [Semantic Versioning](https://semver.org/lang/zh-CN/)。

## [Unreleased]

### 修复

- 修复 Windows 11 24H2/25H2（`termsrv.dll` 6.2.26100.9549）上 DefPolicy
  patch 未生效的问题。该构建的 `CDefPolicy::Query` 使用间接形式
  `MOV; MOV; CMP r,r; JNE` 门禁，而分析器此前按 SEC-ROBUST-01 直接跳过
  indirect JNZ —— 运行时补丁从未写入，`offset-finder --dry-run` 报告
  `def_policy: NOT FOUND`。现在复用已验证的 direct-JNZ 分支改写：短跳转
  `75 rel8` 改为 `EB` 并保留原 `rel8`，近跳转 `0F 85 rel32` 改为
  `E9 rel32` + NOP；强制 store 则把 `0x100` 写入动态捕获的 `[base+W]`
  字段。已用真实字节码（patch RVA `0xA6896`）验证，并对补丁后的控制流
  做了反汇编回读校验。
- 加固 DefPolicy 分析器：间接形式要求门禁 `CMP` 紧跟第二个 `MOV`（旧实现
  会一直向前扫到 `RET`，可能把重写锚点落在无关的日志调用，以及设置成功
  返回值的 `xor esi,esi` 上）；direct 形式遇到没有匹配前置 load 的相似
  `CMP` 也不再中止整个函数扫描。
- `rdprrap-installer install --force` 重装时复用已保存的
  `OriginalServiceDll`，不再重新快照当前 `ServiceDll`（此时它已指向我们
  自己的 `termwrap.dll`）。此前重装会让 uninstall 再也回不到原版
  `termsrv.dll`。
- TermService cohort 重启改为通过 `EnumDependentServicesW(SERVICE_ACTIVE)`
  动态发现运行中的依赖服务，不再依赖 `-k netsvcs` ImagePath 标记 —— 新版
  Windows 上该标记匹配不到任何服务（TermService 为 `-k NetworkService`，
  同族服务为 `LocalSystemNetworkRestricted`／`SessionEnvSvc`）。这也正是
  `sc stop TermService` 返回 1051 的那一组服务。curated sibling 列表保留
  为兜底，并修正了误导性的注释与安装日志。
- `rdprrap-conf` 配置窗口不再整片空白：4 个 `nwg::Frame` 父容器与静态
  标签原本只是 `build()` 的局部变量（或 `let _ = ...` 临时值），函数返回
  时 Drop 销毁 HWND，连带递归销毁全部子控件。现在由 `App` 持有。该问题
  与提权无关，只读模式窗口同样受影响。

## [0.3.0] - 2026-06-25

### 变更

- patch 位置不再硬编码，而是改为运行时动态推导。新增的 `patcher::analyze`
  模块对目标函数做反汇编，在运行时捕获 `struct` displacement、`register`
  选择与 `branch` 方向；新增的 `patcher::encode` x86/x64 编码器
  （`mov reg, imm32`、`mov [base+disp32], reg`）据此生成 patch bytes。
  移除固定 `struct` offset、`register` 枚举、固定字节码模板这类绑定构建的
  硬编码，使 DefPolicy 与 PropertyDevice patch 能经受 `termsrv.dll`
  `struct` 布局的变动。解决了 issue-#3 那一类回归（Windows 构建更新后
  patch 被静默跳过），并已在 26200.8037／26100.32230（Server 2025）／
  20348.587（Server 2022）上完成验证。
- patch 位置探测现在会跟随 PGO 的 hot/cold split。当目标函数的识别字符串
  只在 cold（拆分出的）片段中被引用时，先沿 x64 chained unwind 信息
  （`UNW_FLAG_CHAININFO`）走到 primary（hot）函数，再寻找 patch 位置。
  没有这一步时，DefPolicy 会被抓到 deny 路径片段上，在 Server 2022
  （20348）这类 cold-split 构建上被静默跳过。
- `termwrap-dll` 的 DefPolicy／PropertyDevice patch 模块现在只是包在
  纯 `patcher::analyze` 分析器外面的一层薄 unsafe wrapper。

### 新增

- `offset-finder --dry-run`（`-d`）：自诊断功能，对每个 patch 报告捕获到
  的 patch 位置，以及实际将写入的 bytes（或 NOT FOUND）。`--assert-all`
  隐含 `--dry-run`，现在只要有一个 patch 位置未找到即失败。

### 移除

- `umwrap-dll` 的 x64 legacy slc.dll 路径不再写入依赖构建的
  `or [rsp+0x40], 1` stack-slot 模板。以 log-and-skip 取代这种脆弱的
  硬编码写入，确保构建不匹配时不会破坏无关的 stack slot。

## [0.2.0] - 2026-05-13

### 新增

- ARM64 Windows 构建 scaffold：`aarch64-pc-windows-msvc` 在整个 workspace
  内通过类型检查，并由 static CRT rustflags 与 CI 的构建／静态产物检查
  任务守护。release 工作流也会在 x64/x86 之外打包 ARM64 ZIP。
- `umwrap-dll` 与 `endpwrap-dll` 的实验性 ARM64 运行时 patching。
  `patcher::arm64` 会探测 ARM64 `.pdata` 函数条目、ADR/ADRP+ADD 的
  policy 字符串引用以及附近的 BL 调用。`umwrap` 把 policy BL 调用改成
  `mov w0,#1`；`endpwrap` 把被引用的音频捕获函数起点改成
  `mov w0,#1; ret`。
- `termwrap-dll` 的实验性 ARM64 运行时 patching。ARM64 路径用 `.pdata`
  解析同一批 termsrv policy 字符串，并对以下逻辑打 patch：DefPolicy
  字段检查、SingleUser／LocalOnly 返回 false、AppServer／NonRDP 返回
  true、PropertyDevice 的 BL 结果返回 false、SL policy 查询 BL 调用返回
  true。要按生产级支持对待，仍需在真实 Windows ARM64 上完成运行时验证。

### 修复

- `rdprrap-installer install --force` 现在会在 race-safe 的 `CREATE_NEW`
  复制之前，真正替换已安装的 wrapper DLL。不带 `--force` 且目标 DLL 已
  存在时，改为尽早失败并提示"use --force"，而不是抛出低层级的 Win32
  创建错误。
- `offset-finder` 不再把 ARM64 PE32+ 镜像当作 x64 处理。纯 ARM64 的
  `termsrv.dll` 镜像现在会输出 ARM64 字符串／函数／BL-site 报告；只有
  ARM64EC/ARM64X 混合镜像在另行验证之前报告为 unsupported。

## [0.1.3] - 2026-04-23

### 修复（许可证合规 —— 补充）

- `NOTICE` 此前不完整。0.1.2 只列出了 9 个 rdpwrap 派生文件，而更深入的
  审计发现实际有 **16 个 Rust 源文件**在移植或镜像 `stascorp/rdpwrap`
  的特定 Delphi 文件。`NOTICE` 已按上游二进制重新组织
  （`RDPWInst.exe → rdprrap-installer`、`RDPConf.exe → rdprrap-conf`、
  `RDPCheck.exe → rdprrap-check`），使评审者一眼就能看清按上游工具
  （而非逐文件）划分的派生边界。
- 在 `CODE_OF_CONDUCT.md` 与 `docs/CODE_OF_CONDUCT.ko.md` 中补充显式注明
  Contributor Covenant 原文所采用的 Creative Commons Attribution 4.0
  International（CC BY 4.0）许可。原有出处块与 contributor-covenant.org
  的 adopter 模板一致，但没有写明 CC BY 本身。
- Copyright 行的一致性：`rdprrap-conf` About 对话框中的 MIT 文本此前是
  "Copyright (c) 2026 rdprrap contributors"，而 `LICENSE`／随附的
  `THIRD_PARTY_LICENSES.txt` 是 "Copyright (c) 2026 Kim DaeHyun" ——
  为避免三处各执一词，About 对话框已对齐到 `LICENSE` 的写法。

## [0.1.2] - 2026-04-23

### 修复

- 随二进制一并分发上游项目的源码归属声明。rdprrap 的部分内容是对三个
  上游项目的重新实现或行为镜像，相应许可要求保留版权声明及／或分发
  许可文本：
    * `stascorp/rdpwrap`（Apache-2.0）—— RDPCheck 断连原因码表、
      安装器／cohort 重启／ACL 契约、HKLM 注册表布局、防火墙规则形态。
      Apache-2.0 §4(a) 要求再分发时随附许可文本，§4(c) 要求保留归属
      声明。
    * `llccd/TermWrap`（MIT）—— patcher 的 pattern 扫描器、PE 基址
      重定位、x64 xref／x86 prologue 扫描的函数定位、DLL 导出契约。
      MIT 要求一切再分发都附带版权声明。
    * `llccd/RDPWrapOffsetFinder`（MIT）—— offset-finder 的算法设计
      （字符串扫描、xref、branch 跟随的 priority queue）。
  0.1.0／0.1.1 发布只打包了 Cargo 依赖的归属声明
  （`THIRD_PARTY_LICENSES.txt`），漏掉了这三项源码级声明 —— 属于
  许可证合规缺口。

### 新增

- **`NOTICE`** 根文件 —— 逐项记录每个上游项目、rdprrap 中由其派生的
  文件、许可证类型与版权全文（MIT 内联给出，Apache-2.0 以引用方式
  给出）。
- **`vendor/licenses/`** —— vendor 上游许可证文本，使标签时点不再依赖
  能否访问上游仓库：
    * `LICENSE.rdpwrap.Apache-2.0`
    * `LICENSE.TermWrap.MIT`
    * `LICENSE.RDPWrapOffsetFinder.MIT`
- release 工作流把 `NOTICE` + `vendor/licenses/` 整棵目录树打包进按
  架构划分的 release ZIP。
- `README.md`／`docs/README.ko.md` 的 "License" 一节改为：上游归属指向
  `NOTICE` 与 `vendor/licenses/`，Cargo 依赖归属指向
  `THIRD_PARTY_LICENSES.txt`。

## [0.1.1] - 2026-04-22

### 修复

- `rdprrap-installer` 的注册表回读可能把过量分配缓冲区尾部的垃圾字节
  带进解码后的字符串 —— 安装前的 `ServiceDll` 值存入
  `HKLM\SOFTWARE\rdprrap\Installer\OriginalServiceDll` 时尾部多出一个
  字符（issue #1 报告了 `termsrv.dll` → `termsrv.dlll`）。在这种状态下
  执行 `uninstall`，回滚路径会把损坏的值还原回去，`TermService` 随即
  指向一个不存在的 DLL，下次开机 RDP 直接不可用。
  现在 `RegKey::get_string` 与 `get_service_dll` 都改为按
  `RegQueryValueExW`／`RegGetValueW` 实际写入的字节数（`lpcbData` out
  参数）截断解码后的缓冲区，不再依赖对整个分配执行
  `while buf.last() == Some(0) { pop }`。

### 测试

- 新增 Windows 专用的注册表 round-trip 回归测试，运行于
  `HKCU\Software\rdprrap-installer-tests\` 之下 —— 无需提升权限，可在
  非特权的 Windows CI runner 上执行。每个测试按调用生成分离的 subkey
  （PID + atomic counter），并以 RAII guard 在 Drop 时删除子树。

### 针对 0.1.0 安装版的缓解措施

在以 0.1.0 安装过的主机上执行 `rdprrap-installer.exe uninstall` 之前，
先核实已保存的 `ServiceDll` 目标：

```
reg query "HKLM\SOFTWARE\rdprrap\Installer" /v OriginalServiceDll
```

若显示的路径并非精确以 `.dll` 结尾，请在 uninstall 之前先覆写：

```
reg add "HKLM\SOFTWARE\rdprrap\Installer" /v OriginalServiceDll ^
  /t REG_EXPAND_SZ /d "%SystemRoot%\System32\termsrv.dll" /f
```

## [0.1.0] - 2026-04-22

### 本版本的范围

0.1.0 正式发布多会话 RDP patch 流水线（`termwrap`）＋安装器＋check＋
conf＋offset-finder。`umwrap`／`endpwrap` 出于与原版 rdpwrap 的布局
延续性随安装载荷一同分发，但目前处于 **dormant** 状态（见下文*已知
限制*）。扩大受支持的 Windows SKU 范围、启用 USB／摄像头／音频
wrapper，留待后续版本跟进。

### 修复

- `rdprrap-installer` 的卸载路径改用 `restart_termservice_with_cohort`
  （与安装相同的基础操作）。普通的 stop/start 在 `UmRdpService` 或
  `SessionEnv` 处于活动状态的宿主上必定撞上
  `ERROR_DEPENDENT_SERVICES_RUNNING`（0x041B），而 `?` 提前返回会让
  状态键＋安装目录残留下来。现在重启失败不再致命 —— step 5/6 的清理
  始终执行，并输出"请重启以完成收尾"的提示。
- `.cargo/config.toml` 对两个 MSVC 目标强制
  `target-feature=+crt-static`。否则二进制会依赖 `VCRUNTIME140.dll`／
  `MSVCP140.dll`，在未装 VC++ Redistributable 的新 Windows 镜像上拒绝
  启动（STATUS_DLL_NOT_FOUND = 0xC0000135）。已用 `llvm-readobj`
  验证：x64 与 i686 的全部产物中 MSVC CRT 导入为 0。
- 文档：音频端点 wrapper 安装时的正式名称是 `endpwrap.dll` 而非
  `rdpendp.dll`。安装器契约从一开始就是 `endpwrap.dll`，是 TESTING
  文档写错了。

### 运行时验证（2026-04-22）

- Windows 11 x64（build 10.0.26200.0）上的安装 → 卸载完整 round-trip
  通过（Linux 宿主机 + winpodx／dockur-windows 容器）。
- 多会话 smoke 测试：两个交互会话可同时使用；DebugView 中确认出现
  `TermWrap:` patch-applied 行，`patch not found` 为 0 条。
- `offset-finder --assert-all C:\Windows\System32\termsrv.dll` 在该
  构建上 7/7 字符串、7/7 函数全部解析成功。
- 卸载后 `ServiceDll` 还原为 `%SystemRoot%\System32\termsrv.dll`，
  删除 `%ProgramFiles%\RDP Wrapper\` 目录树，删除防火墙规则
  `rdprrap-RDP-TCP`／`rdprrap-RDP-UDP`，删除
  `HKLM\SOFTWARE\rdprrap\Installer`，`fDenyTSConnections` 还原为
  安装前的值。

### 已知限制

- **支持的 SKU 范围有限** —— 完成运行时端到端验证的环境只有一台
  Windows 11 x64（build 10.0.26200.0）。Windows Server 2025／2022、
  Windows 11 24H2／23H2、Windows 10 22H2 以及全部 i686 运行时路径仅
  覆盖编译＋单元测试。`offset-finder` 基于 pattern，在未经验证的
  termsrv.dll 构建上可能优雅地失败（drift）—— 若 pattern 不匹配，
  请连同 `--assert-all` 的完整 stdout 与 termsrv.dll 的 VersionInfo
  一并收集上报。
- **`umwrap`／`endpwrap` 在本版本中 dormant** —— DLL 虽进入
  `%ProgramFiles%\RDP Wrapper\`，但 Windows 不会加载它们。
  `UmRdpService` 与音频端点 COM 服务器直接加载 `System32\umrdp.dll`／
  `System32\rdpendp.dll`，而 0.1.0 尚未提供能穿透 WFP／SFC 的 DLL
  重定向机制。因此 USB／摄像头重定向 patch 与音频捕获 patch 在
  0.1.0 中**没有运行时效果** —— `termwrap` 的多会话 patch 是唯一在
  运行时生效的组件。启用工作留待后续版本跟进。
- `%ProgramFiles%\RDP Wrapper\` 的 ACL 加固依赖 Program Files 父目录的
  继承，并未设置显式的 protected DACL。写权限仍仅限 SYSTEM／
  Administrators／TrustedInstaller；`BUILTIN\Users` 持有继承而来的
  read+execute。
- Windows PowerShell 5.1 的 `Start-Transcript` 捕获不到安装器的原生
  控制台输出。要拿到可靠日志，请使用
  `.\rdprrap-installer.exe install 2>&1 | Tee-Object install.log`。

### 新增（2026-04）

- **`rdprrap-installer`**：Rust CLI 安装／卸载器 —— 服务注册、注册表
  （KEY_WOW64_64KEY + SDDL）、防火墙规则（TCP+UDP 3389，对 locale
  稳健）、安装目录 ACL 加固（SetEntriesInAclW + SetNamedSecurityInfoW）、
  `netsvcs` cohort 服务重启、基于 `VerQueryValueW` 的 termsrv.dll
  版本检查、`--force`／`--skip-firewall`／`--skip-restart`／
  `--disable-nla` 参数
- **`rdprrap-check`**：RDP 回环测试器 —— `mstsc /v:127.0.0.2:PORT`、
  `NlaGuard` RAII（备份／还原 SecurityLayer/UserAuthentication 注册表
  值，Drop 安全）、44 个断连原因码
- **`rdprrap-conf`**：配置 GUI —— native-windows-gui 1.0 + Frame 容器
  布局，1 秒定时器驱动诊断（wrapper 状态、TermService SCM、termsrv
  版本、RDP-Tcp 监听器、支持级别）＋运行时设置（Enable RDP、Port、
  SingleSession、HideUsers、AllowCustom、AuthMode、Shadow），无管理员
  权限时进入只读模式
- 补齐原版 rdpwrap 的功能缺口：C1 AllowMultipleTSSessions 备份／还原、
  C2 CheckInstall 预检查 + `--force`、H1 cohort 服务重启
  （EnumServicesStatusExW）、H2 CheckTermsrvVersion、H3 安装目录 ACL、
  H4 CheckTermsrvDependencies 预检查
- 完成三轮安全审计：CRITICAL 0 项，HIGH/MEDIUM 全部处理（SDDL 保护的
  备份键、reparse point 防御、NLA 禁用改为 opt-in、fDenyTSConnections
  备份／还原、路径校验、事务性回滚、DLL 搜索路径加固）
- CI 更新：Windows x64/x86 × debug/release 矩阵，8 个 crate 全部构建＋
  二进制校验（DLL 导出、PE 架构、大小 sanity，以及安装器／check／conf
  的 `--help` 行为）
- 记忆系统：`.priv-storage/memory/` 可移植持久记忆（iced-x86 API 陷阱、
  NWG 1.0.13 API 陷阱、团队编排规则）

### 新增（2026-03 首次发布）

- Cargo workspace（5 个 crate）：`patcher`、`termwrap-dll`、
  `umwrap-dll`、`endpwrap-dll`、`offset-finder`
- `patcher` crate：PE 头／节／导入表解析、4 字节对齐 pattern matching、
  iced-x86 反汇编 wrapper、基于 WriteProcessMemory 的 patching、
  14 个已验证的字节码常量
- `termwrap-dll`：termsrv.dll proxy DLL，7 种 patch 类型
  - DefPolicyPatch（直接／间接 CMP，x64/x86，JZ/JNZ 变体）
  - SingleUserPatch（memset→VerifyVersionInfoW 及 CMP pattern）
  - LocalOnlyPatch（把 TEST→JS/JNS→CMP→JZ 改为无条件 JMP）
  - NonRDPPatch（IsAllowNonRDPStack，含内联的 IsAppServerInstalled 回退）
  - PropertyDevicePatch（SHR+AND 的 PnP 设备过滤，经注册表确认）
  - CSLQuery::Initialize 的 SL policy 变量 patching（bRemoteConnAllowed、
    bFUSEnabled、bAppServerAllowed、bMultimonAllowed、bInitialized）
- `umwrap-dll`：umrdp.dll proxy DLL，USB／摄像头 PnP 重定向
  （legacy + 现代路径，含摄像头辅助 patch）
- `endpwrap-dll`：rdpendp.dll proxy DLL，音频录制重定向
  （TSAudioCaptureAllowed）
- `offset-finder`：基于 pelite 的 termsrv.dll offset 探测 CLI 工具
  （x64 xref + x86 字符串扫描）
- x64 函数解析：exception table 中搜索 xref + 沿 unwind 链回溯
- x86 函数解析：扫描 prologue（8B FF 55 8B EC）+ branch 跟随的
  priority queue
- 为安全的 in-memory patching 而做的挂起线程／恢复线程
- DLL 导出转发（ServiceMain、SvchostPushServiceGlobals 等）
- GitHub CI：Linux 检查 + Windows x64/x86 完整构建并上传产物
- 11 个单元测试（pattern matching + 反汇编）
- 双语文档（英语 + 韩语）：README、SECURITY、CONTRIBUTING、
  CODE_OF_CONDUCT、CHANGELOG
- GitHub 模板：PR、bug 报告、功能请求
- AI 多工具配置（.priv-storage/ v2.0）
- 加密备份工具箱（tmp-igbkp/）# rdprrap
用 Rust 重写的 RDP Wrapper。

**English** | [한국어](README.ko.md) | **中文**

## 主要特性

| 组件 | 说明 |
|-----------|-------------|
| **termwrap-dll** | 核心 RDP patching —— 多会话支持，并绕过 Home／非 Server 版本的 policy。共 7 种 patch：DefPolicy、SingleUser、LocalOnly、NonRDP、PropertyDevice、SLPolicy、CSLQuery::Initialize |
| **umwrap-dll** | 在现代化（非 slc.dll）的 `umrdp.dll` 构建上重定向 USB／摄像头 PnP 设备。legacy slc.dll 路径（较老的 SKU）可被识别但会跳过 —— 它依赖因构建版本而异的 stack-slot displacement（位移量），无法在运行时动态还原 |
| **endpwrap-dll** | 音频录制重定向（TSAudioCaptureAllowed） |
| **patcher** | 共享库 —— PE 解析、x86/x64 反汇编、ARM64 `.pdata` 函数扫描、运行时 pattern matching、运行时生成 patch bytes（`patcher::analyze` + `patcher::encode`），外加 18 个供 static patch 使用的固定字节码模板 |
| **ARM64 支持** | `aarch64-pc-windows-msvc` 构建目标，以及实验性的 ARM64 `termwrap`／`umwrap`／`endpwrap` runtime patcher。仍需在真实 Windows ARM64 上完成验证才能认定为生产级别支持 |
| **offset-finder** | x86/x64/ARM64 独立 CLI offset 探测工具（基于 pelite，无需 PDB）。`--dry-run`（`-d`）逐项报告捕获到的 patch site 与实际将写入的 bytes；`--assert-all` 隐含该行为，任一未命中即以非零码退出 |
| **rdprrap-installer** | Rust CLI 安装／卸载器 —— 服务注册、注册表写入、防火墙规则、cohort 服务重启、安装目录 ACL 加固（替代 Delphi 的 `RDPWInst.exe`） |
| **rdprrap-check** | RDP 连接测试器 —— 通过 `mstsc.exe` 连接 `127.0.0.2` 回环，NLA 守卫 RAII，44 种断连原因码（替代 `RDPCheck.exe`） |
| **rdprrap-conf** | 配置 GUI —— native-windows-gui 面板，提供诊断与运行时 RDP 设置（Enable／Port／SingleSession／HideUsers／AllowCustom／AuthMode／Shadow），替代 `RDPConf.exe` |

## 技术栈

| 层次 | 技术 |
|------|------|
| 语言 | Rust（stable，MSVC 工具链） |
| 反汇编器 | [iced-x86](https://crates.io/crates/iced-x86)（纯 Rust） |
| PE 解析 | [pelite](https://crates.io/crates/pelite) |
| Windows API | [windows-rs](https://crates.io/crates/windows) |
| 目标平台 | x86_64-pc-windows-msvc、i686-pc-windows-msvc、aarch64-pc-windows-msvc |
| CI | GitHub Actions（Linux 检查 + Windows x64/x86 构建 + ARM64 构建／静态产物检查） |

## 快速开始

### 前置条件

- Rust 工具链（stable，MSVC）
- Windows SDK

### 安装

```bash
git clone https://github.com/kernalix7/rdprrap.git
cd rdprrap
rustup target add x86_64-pc-windows-msvc
rustup target add i686-pc-windows-msvc
rustup target add aarch64-pc-windows-msvc
cargo build --release
```

### 使用

在目标 Windows 主机上以管理员身份打开命令提示符：

```powershell
# 安装 —— 复制 DLL、写入注册表、开放防火墙（TCP+UDP 3389）、
# 授予安装目录 ACL（SYSTEM + LocalService）、重启 TermService 的 cohort 服务
rdprrap-installer.exe install --source <放置已构建 DLL 的目录>

# 查看当前状态（ServiceDll、注册表、防火墙、termsrv.dll 版本）
rdprrap-installer.exe status

# 输出安装契约（路径、DLL 名称、注册表键与值、防火墙规则）
# —— 纯文档输出，无 I/O，无需管理员权限
rdprrap-installer.exe plan

# 卸载 —— 还原原始 ServiceDll、AllowMultipleTSSessions、
# fDenyTSConnections、AddIns，并移除防火墙规则
rdprrap-installer.exe uninstall
```

其他可选参数：

| 参数 | 作用 |
|------|------|
| `--source DIR` | 从该目录复制 DLL（默认取安装器自身所在目录） |
| `--force` | 即使 ServiceDll 已指向 wrapper DLL，也强制重装并替换现有 wrapper DLL |
| `--skip-firewall` | 不添加／不删除防火墙规则 |
| `--skip-restart` | 不重启 TermService（可手动应用或等待重启生效） |
| `--disable-nla` | 设置 `UserAuthentication=0`（可选，legacy 客户端需要） |
| `-i` / `-u` | install／uninstall 的 legacy 别名（兼容 RDPWInst） |

安装完成后，可从 `%ProgramFiles%\RDP Wrapper\` 启动两个 GUI 程序：

```powershell
# 配置面板 —— 实时状态 + 运行时设置开关
rdprrap-conf.exe

# RDP 回环测试 —— 在 NLA 守卫 RAII 保护下启动 mstsc /v:127.0.0.2
rdprrap-check.exe
```

可以不使用 `rdprrap-installer.exe` 手动安装 —— 把 DLL 复制到 `%ProgramFiles%\RDP Wrapper\`，再合并对应的注册表文件即可。DLL 接口参考见上游 [TermWrap](https://github.com/llccd/TermWrap)。

要对 `termsrv.dll` 做自检，可运行独立的 offset finder：

```powershell
# 报告每一处捕获到的 patch site，以及实际将写入的 bytes（或 NOT FOUND）
# —— 纯诊断用途，不写入任何内容。
offset-finder.exe --dry-run C:\Windows\System32\termsrv.dll

# 只要有任何必需的字符串／函数／patch site 缺失，即以非零错误码退出。
# --assert-all 隐含 --dry-run。
offset-finder.exe --assert-all C:\Windows\System32\termsrv.dll
```

## 项目结构

```
rdprrap/
├── crates/
│   ├── patcher/            # 共享：PE 解析、反汇编、pattern matching、内存 patch
│   │   └── src/
│   │       ├── pe.rs       # PE 头／节／导入／异常表解析
│   │       ├── pattern.rs  # .rdata 中 4 字节对齐的字符串 pattern matching
│   │       ├── disasm.rs   # iced-x86 解码器封装、xref 搜索、branch 辅助
│   │       ├── arm64.rs    # ARM64 .pdata 函数扫描 + ADR/ADRP/ADD/BL 辅助
│   │       └── patch.rs    # WriteProcessMemory 封装、NOP 填充、18 个字节码常量
│   ├── termwrap-dll/       # cdylib：termsrv.dll proxy（RDP 核心）
│   │   └── src/patches/    # DefPolicy、SingleUser、LocalOnly、NonRDP、PropertyDevice、SLPolicy
│   ├── umwrap-dll/         # cdylib：umrdp.dll proxy（USB／摄像头重定向）
│   ├── endpwrap-dll/       # cdylib：rdpendp.dll proxy（音频录制）
│   ├── offset-finder/      # 二进制：独立 offset 探测 CLI
│   ├── rdprrap-installer/  # 二进制：安装／卸载 CLI（注册表、服务、防火墙、ACL）
│   ├── rdprrap-check/      # 二进制：RDP 回环测试器（mstsc + NLA 守卫）
│   └── rdprrap-conf/       # 二进制：配置 GUI（native-windows-gui）
├── .github/
│   └── workflows/ci.yml   # Linux 检查 + Windows x64/x86 构建矩阵 + ARM64 静态检查
└── docs/                   # 韩语文档
```

## 工作原理

1. wrapper DLL 作为原始系统 DLL（`termsrv.dll`、`umrdp.dll`、`rdpendp.dll`）的 proxy
2. 在 `DLL_PROCESS_ATTACH` 时加载原始 DLL，并转发其导出函数
3. 挂起所有线程 → 通过 `WriteProcessMemory` 应用 in-memory patch → 恢复线程
4. 在运行时定位 patch site：
   - **x64**：在 `.rdata` 中扫描已知字符串 → 在 exception table 中搜索 LEA xref → 沿 unwind 链回溯到函数起点
   - **x86**：在 `.text` 中扫描函数 prologue（`8B FF 55 8B EC`）→ 跟随 branch → 把 PUSH/MOV 立即数与字符串 RVA 匹配
   - **ARM64**：用 `.pdata` 函数范围配合 ADR/ADRP+ADD 字符串引用定位 policy 检查。`termwrap` 以 ARM64 专用字节码对 DefPolicy、SingleUser、LocalOnly、AppServer/NonRDP、PropertyDevice 与 SL policy 路径打 patch；`umwrap` 把 PnP／摄像头相关的 BL 调用改成 `mov w0,#1`；`endpwrap` 把被引用的音频捕获函数起点改成 `mov w0,#1; ret`

### 健壮性 / 为什么选择 rdprrap

rdprrap **既不需要 PDB 符号，也不需要手工维护的逐构建 offset 表**。
因为其 patch site 定位完全符号无关（字符串扫描 + 反汇编 + xref）；
DefPolicy／PropertyDevice，`struct` displacement、`register` 与 分支判断方向
都在 *运行时动态* 捕获，patch字节码则由 `patcher::encode` 指令编码器生成。
由于没有任何环节绑定在固定 offset 或固定字节码模板上，即便新 Windows
构建带来的 `termsrv.dll` `struct` 布局发生变动，patch 依然有效 ——
微软挪动字段时，我们不需要更新 offset 表

## patch 类型（termsrv.dll）

| patch | 用途 | 机制 |
|-------|---------|-----------|
| DefPolicyPatch | 允许多个 RDP 会话 | `patcher::analyze` 反汇编 policy 函数，在运行时捕获 `struct` displacement、`register` 与 `branch` 方向；`patcher::encode` 再按捕获到的布局生成 `mov reg, 0x100` + `mov [base+disp32], reg` bytes，因此能经受 `struct` offset 变动 |
| SingleUserPatch | 关闭按用户的会话数限制 | 用 NOP 抹掉 VerifyVersionInfoW 调用或 CMP 指令 |
| LocalOnlyPatch | 解除 local-only 许可限制 | 把 JZ 改成无条件 JMP |
| NonRDPPatch | 允许非 RDP 协议栈 | 用 `inc [ecx]; xor eax,eax` 替换 CALL |
| PropertyDevicePatch | 启用 PnP 设备重定向 | `patcher::analyze` 在运行时捕获目标 `register`；`patcher::encode` 针对该 `register` 生成 `mov reg, 0`，因此能经受跨构建的指令／`register` 变化 |
| SLPolicyPatch | 把 SL policy 变量置为 1 | 直接向 bRemoteConnAllowed、bFUSEnabled 等写内存 |

## 测试

```bash
cargo test                                          # 单元测试
cargo clippy --all-targets -- -D warnings           # 静态检查
cargo fmt --check                                   # 格式检查
```

推送／PR 时 CI 自动运行 Linux 检查 + Windows x64/x86 完整构建 + ARM64 构建／静态产物检查。

## 故障排查

| 症状 | 可能原因 | 处理方式 |
|---------|--------------|------------|
| 多会话 RDP 原本可用，Windows 更新／换新构建后失效 | `termsrv.dll` 变动导致某处 patch site 移位，patcher 在新构建上无法解析（即 issue-#3 那一类回归） | 运行 `offset-finder --dry-run C:\Windows\System32\termsrv.dll`，查看 `[Patch Dry-Run]` 区块。出现 `NOT FOUND` 的那一行，正好指出哪个 patch site 解析失败。把该输出与 `termsrv.dll` 版本一并留存，供排查。 |

## 参与贡献

开发环境与工作流见 [CONTRIBUTING.zh.md](CONTRIBUTING.zh.md)。

## 安全

安全相关问题请按 [SECURITY.zh.md](SECURITY.zh.md) 中的流程处理。

## 参考项目

- [stascorp/rdpwrap](https://github.com/stascorp/rdpwrap) —— 原始 RDP Wrapper
- [llccd/TermWrap](https://github.com/llccd/TermWrap) —— 集成 offset finder 的 C++ 重写版
- [llccd/RDPWrapOffsetFinder](https://github.com/llccd/RDPWrapOffsetFinder) —— 基于 PDB 的 offset finder

## 许可证

MIT 许可证 —— 详见 [LICENSE](../LICENSE)。

rdprrap 的部分代码是对上游项目的移植<!--Note: 歧义更少-->或重新实现（rdpwrap 为 Apache-2.0；
TermWrap 与 RDPWrapOffsetFinder 为 MIT）。归属声明见 [NOTICE](../NOTICE)，
上述上游许可证全文见 [vendor/licenses/](../vendor/licenses/)；编译期 Rust
依赖的归属声明见各发布 ZIP 内的 `THIRD_PARTY_LICENSES.txt`（发布时自动生成）。