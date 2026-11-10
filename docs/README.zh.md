# rdprrap

用 Rust 重写的 RDP Wrapper。

[English](../README.md) | [한국어](README.ko.md) | **中文**

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