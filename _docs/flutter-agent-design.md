# Flutter Agent 运行时设计

**状态**：讨论中（非实施文档）  
**最后更新**：2025-01-08

## 需求背景

将ZeroClaw的部分源码抽取为Rust库，通过flutter_rust_bridge接入Flutter，作为端侧推理Agent。用户已有SQLite文档管理系统，需要与之集成。

## 架构选择

**方案**：无头运行时库（Headless Runtime Library）
- Flutter UI处理所有输入输出
- Rust核心处理LLM交互、工具执行和记忆管理
- 通过flutter_rust_bridge进行跨语言通信

## 模块保留与裁剪

### ✅ 保留模块

| 模块 | 功能 | 必需性 |
|------|------|--------|
| `agent/` | 编排循环、工具调度 | 核心 |
| `providers/` | OpenAI + 兼容API | 核心 |
| `tools/` | 文件操作、网络工具、HTTP、**文档搜索** | 核心 |
| `memory/` | SQLite持久化 | 核心 |
| `skills/` | 技能管理执行 | 增强 |
| `security/` | 策略控制、权限管理 | 安全 |
| `config/` | 配置加载 | 基础设施 |
| `runtime/` | 运行时适配器 | 基础设施 |
| `observability/` | 日志追踪 | 调试 |

### ❌ 裁剪模块

| 模块 | 原因 |
|------|------|
| `channels/` | Telegram、Discord等通信通道由Flutter替代 |
| `gateway/` | 不需要HTTP服务端 |
| `peripherals/` | 无硬件外设需求 |
| `rag/` | 用户已有SQLite文档系统，无需ZeroClaw的硬件专用RAG |
| `cron/` | 定时任务由Flutter端调度 |
| `plugins/` | WASM插件系统非必需 |
| 高级沙箱 | Docker等不适合移动端 |
| 浏览器自动化 | 体积过大 |

## API 设计

```rust
pub struct FlutterAgentRuntime {
    agent: Arc<Agent>,
    config: Config,
}

impl FlutterAgentRuntime {
    // 初始化
    pub async fn new(config: AgentConfig) -> Result<Self>;
    
    // 流式对话
    pub async fn chat_stream(
        &self, 
        message: String,
        session_id: Option<String>
    ) -> Stream<ChatEvent>;
    
    // 一次性对话
    pub async fn chat(&self, message: String) -> Result<String>;
    
    // 技能管理
    pub async fn list_skills() -> Vec<SkillInfo>;
    pub async fn execute_skill(name: &str, params: Value) -> Result<String>;
    
    // 记忆管理
    pub async fn recall(query: &str, limit: usize) -> Vec<MemoryEntry>;
    pub async fn store_memory(content: &str, category: &str);
    
    // 安全策略
    pub fn set_security_policy(policy: SecurityPolicy);
    pub fn get_security_policy() -> SecurityPolicy;
    
    // 工具执行
    pub async fn execute_tool(name: &str, args: Value) -> Result<ToolResult>;
}
```

## 数据流

```
Flutter UI
    ↓ 用户输入
flutter_rust_bridge
    ↓
┌──────────────────────────────────┐
│  FlutterAgentRuntime             │
│  ┌──────────────────────────┐    │
│  │  Agent 编排循环           │    │
│  │  ┌──────────┐            │    │
│  │  │ Provider │ ← OpenAI   │    │
│  │  └──────────┘            │    │
│  │       ↓                  │    │
│  │  ┌──────────┐            │    │
│  │  │ Tool路由 │            │    │
│  │  ├──────────┤            │    │
│  │  │ 文件操作  │            │    │
│  │  │ 网络工具  │            │    │
│  │  │ 文档搜索 ─┼──→ 用户SQLite │    │
│  │  └──────────┘            │    │
│  │       ↓                  │    │
│  │  ┌──────────┐            │    │
│  │  │ Skills   │            │    │
│  │  └──────────┘            │    │
│  └──────────────────────────┘    │
│        ↓                         │
│  ┌──────────┐ ← 策略控制         │
│  │ Security │                    │
│  └──────────┘                    │
│        ↓                         │
│  ┌──────────┐ ← SQLite          │
│  │ Memory   │ ← 对话历史        │
│  └──────────┘                    │
└──────────────────────────────────┘
    ↓
flutter_rust_bridge
    ↓
Flutter UI（流式显示）
```

## 工具集设计

### 核心工具
- `file_read` / `file_write` / `file_edit` - 文件操作
- `web_search` / `web_fetch` - 网络工具
- `http_request` - HTTP/API调用

### 自定义工具：文档搜索
由于用户已有SQLite文档管理系统，将其封装为Tool供Agent调用：

```rust
pub struct DocumentSearchTool {
    db: SqlitePool,  // 用户的SQLite连接
}

impl Tool for DocumentSearchTool {
    fn name(&self) -> &str { "search_documents" }
    fn description(&self) -> &str { "从文档库中搜索相关内容" }
    
    fn parameters_schema(&self) -> Value {
        json!({
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "搜索关键词"},
                "limit": {"type": "integer", "default": 5}
            },
            "required": ["query"]
        })
    }
    
    async fn execute(&self, args: Value) -> Result<ToolResult> {
        let query = args["query"].as_str().unwrap();
        let limit = args["limit"].as_u64().unwrap_or(5) as usize;
        
        // 查询用户的SQLite文档库
        let results = self.db.search(query, limit).await?;
        
        Ok(ToolResult::text(format!(
            "找到 {} 个相关文档：\n{}", 
            results.len(),
            results.iter().map(|r| format!("- {}: {}", r.title, r.content)).collect::<Vec<_>>().join("\n")
        )))
    }
}
```

**优势**：
- 复用现有SQLite架构，无需数据迁移
- Agent通过自然语言即可调用（"帮我查找关于XXX的文档"）
- 保持文档权限和访问控制的一致性

### 核心依赖
- `tokio` - 异步运行时
- `reqwest` - HTTP客户端
- `serde` - 序列化
- `rusqlite` - SQLite存储
- `anyhow` - 错误处理

### 可选依赖（Feature控制）
- `observability-prometheus` - 指标监控
- 用户自定义依赖：您的SQLite连接库

## FAQ

### Q: RAG模块是基于SQLite还是实际文档？
**A**: ZeroClaw的RAG是**基于实际文档**（Markdown、txt、PDF文件）的专用实现，专为硬件数据手册设计。

**当前方案**：
- ❌ 裁剪ZeroClaw的RAG模块（硬件专用，不匹配需求）
- ✅ 将用户现有的SQLite文档系统封装为`search_documents`工具

这样Agent可以通过工具调用访问文档库，无需额外的RAG基础设施。

### Q: 为什么不用ZeroClaw的Memory来存文档？
**A**: 
- **Memory**（`src/memory/`）：用于**对话历史和Agent记忆**，SQLite后端，结构简单
- **您的文档系统**：专门的文档管理，可能包含全文搜索、标签、权限等

两者职责分离：
- Memory = Agent的"短期记忆"
- 您的SQLite = "知识库"
- 通过Tool让Agent调用知识库，架构更清晰

## 集成工作流

### 方案：Fork镜像 + 项目内精简

**核心原则**：
- **Fork保持纯净**：仅作为上游镜像，不修改代码
- **精简在用户项目**：通过自动化脚本完成裁剪
- **更新一键完成**：脚本处理复制+精简全过程

### 目录结构

```
# ZeroClaw Fork（保持不变）
~/zeroclaw-fork/          # 上游镜像
├── .git/
├── src/                  # 完整源码
├── Cargo.toml
└── ...

# 您的项目
your-project/
├── Cargo.toml
├── src/
├── vendor/
│   └── zeroclaw/         # 精简后的代码（提交到git）
│       ├── src/          # 裁剪后的源码
│       └── Cargo.toml    # 精简后的配置
├── scripts/
│   └── update-zeroclaw.sh  # 更新脚本
└── flutter_rust_bridge/
    └── ...
```

### 更新脚本

```bash
#!/bin/bash
# scripts/update-zeroclaw.sh

set -e

ZEROCAWK_FORK="$HOME/zeroclaw-fork"
PROJECT_DIR="$(dirname "$(dirname "$(realpath "$0")")")"
VENDOR_DIR="$PROJECT_DIR/vendor/zeroclaw"

echo "🔄 更新 ZeroClaw..."

# 1. 拉取上游更新
cd "$ZEROCAWK_FORK"
git checkout main
git pull origin main

echo "📦 复制到项目..."

# 2. 复制完整代码
rm -rf "$VENDOR_DIR"
mkdir -p "$VENDOR_DIR"
rsync -av --exclude='.git' --exclude='target' \
    "$ZEROCAWK_FORK/" "$VENDOR_DIR/"

echo "✂️  开始精简..."

# 3. 删除不需要的模块
rm -rf "$VENDOR_DIR/src/channels"
rm -rf "$VENDOR_DIR/src/gateway"
rm -rf "$VENDOR_DIR/src/peripherals"
rm -rf "$VENDOR_DIR/src/rag"
rm -rf "$VENDOR_DIR/src/cron"
rm -rf "$VENDOR_DIR/src/plugins"

# 4. 删除CLI相关（可选，如需纯库）
rm -f "$VENDOR_DIR/src/main.rs"

# 5. 修改Cargo.toml
# 使用sed删除不需要的features和依赖
cd "$VENDOR_DIR"

# 删除channels相关的features
sed -i '/channel-/d' Cargo.toml

# 删除不需要的依赖行
sed -i '/matrix-sdk/d' Cargo.toml
sed -i '/axum/d' Cargo.toml
sed -i '/hyper/d' Cargo.toml
sed -i '/tokio-tungstenite/d' Cargo.toml
sed -i '/nostr-sdk/d' Cargo.toml
# ... 其他不需要的依赖

# 6. 清理[[bin]]部分（如不需要CLI）
# 保留[lib]部分

# 7. 更新lib.rs，移除已删除模块的声明
# 需要手动调整或使用更复杂的脚本

echo "✅ 更新完成！"
echo ""
echo "精简统计："
echo "  源码文件: $(find "$VENDOR_DIR/src" -name '*.rs' | wc -l) 个"
echo "  目录大小: $(du -sh "$VENDOR_DIR" | cut -f1)"
echo ""
echo "运行 cargo check 验证..."
cd "$PROJECT_DIR"
cargo check
```

### 使用方法

```bash
# 首次设置
git clone https://github.com/zeroclaw/zeroclaw.git ~/zeroclaw-fork
git remote add upstream https://github.com/original/zeroclaw.git

# 日常更新
./scripts/update-zeroclaw.sh

# 提交精简版本
git add vendor/zeroclaw/
git commit -m "Update zeroclaw to v0.6.9 (trimmed)"
```

### 优势

| 优势 | 说明 |
|------|------|
| **Fork零维护** | 只是一个镜像，永远不改代码 |
| **更新简单** | 一键脚本，自动处理 |
| **版本可控** | vendor/zeroclaw可提交到git，锁定版本 |
| **构建快速** | 精简后代码量小，编译快 |
| **可追溯** | 脚本即文档，清晰明确 |

### 注意事项

1. **首次需要手动调整**：`lib.rs`中的模块声明需要手动移除已删除的模块
2. **Cargo.toml调整**：脚本使用sed处理，复杂情况需手动验证
3. **版本对齐**：确保您的项目Cargo.toml与ZeroClaw的依赖版本一致

### 依赖版本对齐

```toml
# 您的项目的 Cargo.toml
[dependencies]
zeroclaw = { path = "vendor/zeroclaw" }

# 确保这些与ZeroClaw一致
tokio = { version = "1.50", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
anyhow = "1.0"
reqwest = { version = "0.12", features = ["json"] }
```

## 讨论要点

1. **文档搜索工具**：确认用户SQLite的表结构，设计Tool参数
2. **Memory隔离**：明确区分Agent Memory（对话）与文档库（知识）
3. **安全策略**：定义文档搜索的权限边界（能否跨用户访问？）
4. **Cargo features**：确定条件编译的feature粒度
5. **脚本完善**：补充lib.rs模块声明的自动处理逻辑

## 体积估算

- **最小运行时（方案A）**：约 5-8MB
- **当前设计（+skills +security +文档工具）**：约 8-12MB
- **完整ZeroClaw**：约 25-40MB
