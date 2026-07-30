# Coding to Rust — Multi-Language Migration Skills Collection

> [中文版](README.md)

## Introduction

**Coding to Rust** is a Claude Code skills collection covering complete migration guides from 16 mainstream programming languages to Rust. Each language provides:

- **Architecture Mapping** — Runtime differences, paradigm shifts, memory model comparisons
- **Type System Tables** — 30-80 precise type/syntax mappings
- **Framework Migration Guides** — Popular framework to Rust ecosystem mappings (e.g., Django→Axum, Spring→Actix)
- **Canonical Patterns** — Before/after code comparisons
- **Common Mistakes** — Language-specific pitfalls and fixes
- **FFI & Incremental Migration** — Phased replacement strategies
- **Reference Implementations** — Real-world migration cases and projects

## Languages Covered

| # | Language | Directory | Core Paradigm Shift |
|---|----------|-----------|---------------------|
| 1 | Python | `python-to-rust/` | GIL→true parallelism, asyncio→tokio, numpy/pandas→ndarray/polars |
| 2 | JavaScript/TypeScript | `nodejs-to-rust/` | Event loop→tokio, V8→AOT, Express→Axum |
| 3 | Go | `go-to-rust/` | goroutines→tokio tasks, channels→mpsc, GC→ownership |
| 4 | Java | `java-to-rust/` | JVM→native binary, Spring→Axum, JPA→sqlx |
| 5 | C# | `csharp-to-rust/` | CLR→native, LINQ→iterators, ASP.NET→Axum |
| 6 | PHP | `php-to-rust/` | Dynamic typing→static types, FPM→long-lived process, Laravel→Axum |
| 7 | C | `c-to-rust/` | malloc/free→ownership, pointers→references, headers→modules |
| 8 | C++ | `cpp-to-rust/` | Templates→generics, virtual→traits, move semantics→ownership |
| 9 | Zig | `zig-to-rust/` | comptime→proc macros, allocator→ownership, error sets→Result |
| 10 | Lua | `lua-to-rust/` | Table→struct/enum, metatable OOP→traits, coroutines→async |
| 11 | R | `r-to-rust/` | data.frame→polars, formula→builder pattern, apply→iterators |
| 12 | Julia | `julia-to-rust/` | Multiple dispatch→traits, JIT→AOT, Array→ndarray |
| 13 | Kotlin | `kotlin-to-rust/` | Coroutines→tokio tasks, data class→struct, sealed class→enum, Gradle→Cargo |
| 14 | Swift | `swift-to-rust/` | ARC→ownership, actor→Mutex, protocol→trait, SwiftUI→Leptos |
| 15 | Ruby | `ruby-to-rust/` | GC→ownership, blocks→closures, Rails→Axum, Bundler→Cargo |
| 16 | Vue | `vue-to-rust/` | :warning: **(Frontend / WebAssembly — NOT backend)** SFC→component functions, ref()→RwSignal, Vite→Trunk |

## Project Architecture

```
coding-to-rust/
├── README.md                 # This file — project overview (Chinese)
├── README.en.md              # English version
├── SKILL.md                  # Master entry point (loaded by Claude Code)
│                               - Quick selector for all 16 languages
│                               - Universal concepts (ownership/async/errors/build)
│                               - 6-8 core quick-reference entries per language
│                               - Cross-language common mistakes
│                               - Language-agnostic 5-phase migration strategy
│
├── python-to-rust/           # Python → Rust detailed guide
├── php-to-rust/              # PHP → Rust detailed guide
├── nodejs-to-rust/           # JS/TS → Rust detailed guide
├── csharp-to-rust/           # C# → Rust detailed guide
├── cpp-to-rust/              # C++ → Rust detailed guide
├── zig-to-rust/              # Zig → Rust detailed guide
├── java-to-rust/             # Java → Rust detailed guide
├── r-to-rust/                # R → Rust detailed guide
├── go-to-rust/               # Go → Rust detailed guide
├── julia-to-rust/            # Julia → Rust detailed guide
├── c-to-rust/                # C → Rust detailed guide
├── lua-to-rust/              # Lua → Rust detailed guide
├── kotlin-to-rust/           # Kotlin → Rust detailed guide
├── swift-to-rust/            # Swift → Rust detailed guide
├── ruby-to-rust/             # Ruby → Rust detailed guide
└── vue-to-rust/              # Vue → Rust (WASM) detailed guide
```

**Design Principles:**

- **Index Entry** — `SKILL.md` serves as the master entry, auto-loaded by Claude Code via keyword matching
- **Standalone Readable** — Each language directory's `SKILL.md` is self-contained with no external dependencies
- **Progressive Depth** — Quick reference (index layer) → Architecture mapping (concept layer) → Code examples (practice layer) → Reference projects (validation layer)
- **Cross-Referenced** — Languages reference each other for shared patterns (e.g., Python's asyncio and Go's goroutines both map to tokio)

## Usage

### Triggering

In a Claude Code conversation, any of the following will auto-load this skill:

- "Migrate this Python code to Rust"
- "Rewrite this Go service in Rust"
- "How to implement Spring Boot patterns in Rust"
- "Port this C++ class hierarchy to Rust"
- "Convert this Node.js Express app to Rust"

### Workflow

1. **Auto-load** — Claude Code detects a migration need, loads `coding-to-rust/SKILL.md`
2. **Quick Lookup** — Browse 6-8 core mappings for your language to understand the high-level correspondence
3. **Deep Dive** — When code examples or detailed framework mappings are needed, Claude automatically loads the language-specific `SKILL.md`
4. **Cross-Reference** — Multi-language shared patterns link to related sections in other languages

### Recommended Migration Order

Regardless of source language, follow this order:

| Phase | What | How |
|-------|------|-----|
| 1 | **Data types** | Define corresponding Rust structs/enums; share via JSON/Protobuf schema |
| 2 | **Pure functions** | Port stateless logic first — no external dependencies, easiest to test |
| 3 | **I/O boundary** | Replace HTTP handlers, DB queries, file I/O behind same interfaces |
| 4 | **Concurrency** | Convert threads/coroutines/async tasks to tokio tasks |
| 5 | **Full cutover** | Remove source runtime; keep FFI bridges for legacy compatibility |

### Size Reference

| Tier | File | Size | Best For |
|------|------|------|----------|
| Light | Index quick-reference tables | ~8 lines/lang | Quick lookup, confirm basic mappings |
| Medium | Index universal concepts | ~60 lines | Understanding core Rust paradigms |
| Detailed | Per-language SKILL.md | 600-1000 lines | Deep migration, code examples, framework mapping |

## Contributing

To add a new language, fix errors, or improve examples: edit the corresponding language's `SKILL.md` file, then update the quick-reference table and links in `coding-to-rust/SKILL.md`.
