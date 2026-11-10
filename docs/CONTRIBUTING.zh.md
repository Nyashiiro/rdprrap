# rdprrap 贡献指南

[English](../CONTRIBUTING.md) | [한국어](CONTRIBUTING.ko.md) | **中文**

感谢你有兴趣为 rdprrap 做贡献。

## 开发环境

### 前置条件

- Rust 工具链（stable，MSVC）
- Windows SDK
- 用于集成测试的 Windows 环境

### 构建

```bash
git clone https://github.com/kernalix7/rdprrap.git
cd rdprrap
rustup target add x86_64-pc-windows-msvc
rustup target add i686-pc-windows-msvc
rustup target add aarch64-pc-windows-msvc
cargo build --release
```

### 测试

```bash
cargo test
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

## 工作流程

1. Fork 本仓库
2. 创建功能分支：`git checkout -b feature/my-change`
3. 以 Conventional Commits 风格提交
4. 推送并发起 Pull Request

## Pull Request 清单

- [ ] 变更范围与动机清晰
- [ ] 在适用处新增／更新了测试
- [ ] `cargo build --release` —— 零错误
- [ ] `cargo clippy --all-targets -- -D warnings` —— 零警告
- [ ] `cargo test` —— 全部通过
- [ ] 所有 `unsafe` 块都带有 `// SAFETY:` 注释
- [ ] 非测试代码中没有 `.unwrap()`
- [ ] 行为有变化时更新了 README／文档

## 提交信息规范

使用 [Conventional Commits](https://www.conventionalcommits.org/)：

- `feat:` 新功能
- `fix:` 缺陷修复
- `docs:` 文档变更
- `refactor:` 不改变行为的内部改进
- `test:` 测试变更
- `chore:` 维护性任务

## 安全

安全问题请按 [SECURITY.zh.md](SECURITY.zh.md) 中的流程处理。