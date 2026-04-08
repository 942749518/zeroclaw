# ZeroClaw 框架分析

> Zero overhead. Zero compromise. 100% Rust.

## 概览

ZeroClaw 是高性能 AI 代理运行时，采用特质驱动架构，支持多提供商、多通道和硬件外设。

## 技术栈

| 层级 | 技术 |
|------|------|
| 语言 | Rust 2024 (≥1.87) |
| 异步 | Tokio (multi-thread) |
| HTTP | Axum + Reqwest |
| 存储 | SQLite + Markdown |
| 安全 | ChaCha20-Poly1305, Ring |
| 序列化 | Serde + TOML |

## 架构

### 核心特质

```
Provider ──→ AI 模型接口 (OpenAI, Claude, Gemini...)
Channel  ──→ 消息通道 (Telegram, Discord, Slack...)
Tool     ──→ 工具执行 (shell, file, memory...)
Memory   ──→ 存储后端 (SQLite, Qdrant...)
Sandbox  ──→ 沙箱隔离 (Docker, Firejail...)
```

### 模块结构

```
src/
├── agent/        # 代理编排循环
├── gateway/      # HTTP/WebSocket 服务器
├── channels/     # 20+ 通信平台
├── providers/    # 15+ AI 提供商
├── tools/        # 60+ 工具
├── memory/       # 多后端存储
├── security/     # 策略与沙箱
└── config/       # 配置管理
```

## 核心能力

### 多通道
Telegram · Discord · Slack · Matrix · WhatsApp · iMessage · Email · Signal · Nostr · Lark · DingTalk

### 多提供商
OpenAI · Anthropic · Gemini · Ollama · OpenRouter · Azure · Bedrock · 智谱 · 月之暗面 · 通义千问

### 安全沙箱
- Docker / Firejail / Bubblewrap
- Landlock (Linux) / Seatbelt (macOS)
- 工作区边界 · OTP · 紧急停止

### 可观测性
Prometheus 指标 · OpenTelemetry 追踪 · 成本追踪

## 扩展点

| 扩展类型 | 实现方式 | 注册位置 |
|---------|---------|---------|
| Provider | 实现 `Provider` trait | `providers/mod.rs` |
| Channel | 实现 `Channel` trait | `channels/mod.rs` |
| Tool | 实现 `Tool` trait | `tools/mod.rs` |
| Memory | 实现 `Memory` trait | `memory/mod.rs` |

## 构建配置

```toml
[profile.release]
opt-level = "z"
lto = "fat"
codegen-units = 1
strip = true
panic = "abort"
```

## 特性标志

- **通道**: `channel-nostr`, `channel-matrix`, `channel-lark`
- **硬件**: `hardware`, `peripheral-rpi`, `probe`
- **功能**: `browser-native`, `whatsapp-web`, `plugins-wasm`

## 部署模式

```bash
zeroclaw agent        # 交互式会话
zeroclaw daemon       # 长期运行守护进程
zeroclaw gateway      # HTTP 服务器
zeroclaw acp          # JSON-RPC over stdio
```

---

*文档基于 v0.6.8 生成*
