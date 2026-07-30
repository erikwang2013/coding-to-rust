# Coding to Rust — 多语言迁移到 Rust 的技能集合

> [English](README.en.md)

## 项目介绍

**Coding to Rust** 是一个 Claude Code 技能集合，覆盖从 16 种主流编程语言迁移到 Rust 的完整指南。每种语言提供：

- **架构映射** — 运行时差异、范式转换、内存模型对照
- **类型系统对照表** — 30-80 条精确的类型/语法映射
- **框架迁移指南** — 主流框架到 Rust 生态的对照（如 Django→Axum、Spring→Actix）
- **标准模式对照** — 带代码示例的 before/after 对比
- **常见错误** — 该语言开发者在 Rust 中最容易犯的错误及修复方法
- **FFI 渐进迁移策略** — 分阶段替换方案
- **参考实现** — 真实世界的迁移案例和项目

## 覆盖语言

| # | 语言 | 目录 | 核心范式转换 |
|---|------|------|-------------|
| 1 | Python | `python-to-rust/` | GIL→真并行、asyncio→tokio、numpy/pandas→ndarray/polars |
| 2 | JavaScript/TypeScript | `nodejs-to-rust/` | 事件循环→tokio、V8→AOT、Express→Axum |
| 3 | Go | `go-to-rust/` | goroutine→tokio task、channel→mpsc、GC→所有权 |
| 4 | Java | `java-to-rust/` | JVM→原生二进制、Spring→Axum、JPA→sqlx |
| 5 | C# | `csharp-to-rust/` | CLR→原生、LINQ→迭代器、ASP.NET→Axum |
| 6 | PHP | `php-to-rust/` | 动态类型→静态类型、FPM→长驻进程、Laravel→Axum |
| 7 | C | `c-to-rust/` | malloc/free→所有权、指针→引用、头文件→模块 |
| 8 | C++ | `cpp-to-rust/` | 模板→泛型、虚函数→trait、移动语义→所有权 |
| 9 | Zig | `zig-to-rust/` | comptime→宏、分配器→所有权、错误集→Result |
| 10 | Lua | `lua-to-rust/` | table→struct/enum、元表OOP→trait、coroutine→async |
| 11 | R | `r-to-rust/` | data.frame→polars、formula→builder、apply→迭代器 |
| 12 | Julia | `julia-to-rust/` | 多重分派→trait、JIT→AOT、Array→ndarray |
| 13 | Kotlin | `kotlin-to-rust/` | Coroutines→tokio、data class→struct、sealed class→enum、Gradle→Cargo |
| 14 | Swift | `swift-to-rust/` | ARC→ownership、actor→Mutex、protocol→trait、SwiftUI→Leptos |
| 15 | Ruby | `ruby-to-rust/` | GC→ownership、blocks→closures、Rails→Axum、Bundler→Cargo |
| 16 | Vue | `vue-to-rust/` | SFC→组件函数、ref()→RwSignal、Vite→Trunk |

## 项目架构

```
coding-to-rust/
├── README.md                 # 本文件 — 项目说明（中文）
├── README.en.md              # 英文版项目说明
├── SKILL.md                  # 总入口（Claude Code 加载此文件）
│                               - 16 语言快速选择器
│                               - 通用迁移概念（所有权/异步/错误处理/构建系统）
│                               - 各语言 6-8 条核心快速对照表
│                               - 跨语言通用错误
│                               - 语言无关的 5 阶段迁移策略
│
├── python-to-rust/           # Python → Rust 详细指南
├── php-to-rust/              # PHP → Rust 详细指南
├── nodejs-to-rust/           # JS/TS → Rust 详细指南
├── csharp-to-rust/           # C# → Rust 详细指南
├── cpp-to-rust/              # C++ → Rust 详细指南
├── zig-to-rust/              # Zig → Rust 详细指南
├── java-to-rust/             # Java → Rust 详细指南
├── r-to-rust/                # R → Rust 详细指南
├── go-to-rust/               # Go → Rust 详细指南
├── julia-to-rust/            # Julia → Rust 详细指南
├── c-to-rust/                # C → Rust 详细指南
├── lua-to-rust/              # Lua → Rust 详细指南
├── kotlin-to-rust/           # Kotlin → Rust 详细指南
├── swift-to-rust/            # Swift → Rust 详细指南
├── ruby-to-rust/             # Ruby → Rust 详细指南
└── vue-to-rust/              # Vue → Rust (WASM) 详细指南
```

**设计原则：**

- **索引入口** — `SKILL.md` 作为总入口，Claude Code 根据关键词自动加载
- **独立可读** — 每个语言目录的 `SKILL.md` 可独立使用，不依赖其他文件
- **渐进深度** — 快速对照表（索引层）→ 架构映射（概念层）→ 代码示例（实操层）→ 参考项目（验证层）
- **交叉引用** — 各语言互相引用共享模式（如 Python 的 asyncio 和 Go 的 goroutine 都映射到 tokio）

## 使用说明

### 触发方式

在 Claude Code 对话中，以下任何一种说法都会自动加载此技能：

- 「把这段 Python 代码迁移到 Rust」
- 「用 Rust 重写这个 Go 服务」
- 「Java 的 Spring Boot 怎么用 Rust 实现」
- 「Port this C++ class hierarchy to Rust」
- 「Migrate Node.js Express app to Rust」

### 使用流程

1. **自动加载** — Claude Code 识别到迁移需求，加载 `coding-to-rust/SKILL.md`
2. **快速对照** — 查看你所用语言的 6-8 条核心映射，快速理解大致对应关系
3. **深入查阅** — 需要代码示例、框架详细对照时，Claude 自动读取对应语言的详细 `SKILL.md`
4. **交叉参考** — 涉及多语言共享模式时，通过交叉引用链接到其他语言的对应章节

### 推荐的迁移顺序

无论哪种源语言，建议按以下顺序执行迁移：

| 阶段 | 内容 | 方法 |
|------|------|------|
| 1 | **数据类型** | 定义对应的 Rust struct/enum，通过 JSON/Protobuf schema 共享 |
| 2 | **纯函数** | 先迁移无状态逻辑，无外部依赖，最容易编写测试 |
| 3 | **I/O 边界** | 替换 HTTP handler、数据库查询、文件读写 |
| 4 | **并发模型** | 将线程/协程/异步任务统一转换为 tokio task |
| 5 | **完全切换** | 移除源语言运行时，仅保留 FFI 桥接做兼容过渡 |

### 文件大小参考

| 层级 | 对应文件 | 规模 | 适用场景 |
|------|----------|------|----------|
| 轻量 | 索引层快速对照表 | ~8 行/语言 | 快速查询，确认基本映射 |
| 中等 | 索引层通用概念 | ~60 行 | 理解 Rust 核心范式 |
| 详细 | 各语言 SKILL.md | 600-1000 行 | 深度迁移、代码示例、框架对照 |

## 贡献

如需新增语言、修复错误或改进示例代码，请编辑对应语言的 `SKILL.md` 文件，并同步更新 `coding-to-rust/SKILL.md` 索引中的快速对照表和链接。

---

