---
name: coding-to-rust
description: Use when migrating code from any source language to Rust — master index covering 16 languages (Python, JS/TS, Go, Java, C#, PHP, C, C++, Zig, Lua, R, Julia, Kotlin, Swift, Ruby, Vue) with quick-reference tables, shared ownership/async/error-handling concepts, and pointers to per-language deep-dive skills. Trigger on "migrate X to Rust", "rewrite in Rust", "port from X to Rust", or Rust-equivalent questions.
updated: 2026-07-30
---

# Coding to Rust — Master Migration Index

## Overview

This is the entry point for migrating code from any source language to Rust. It covers 16 languages with quick-reference tables, shared concepts that apply across all migrations, and pointers to detailed per-language skills.

**How to use this skill:**
1. Find your source language in the [Quick Selector](#quick-selector) below
2. Scan the shared concepts section for universal Rust patterns
3. Read your language's quick-reference table for the most common mappings
4. Follow the link to the per-language skill for deep-dive patterns, code examples, and FFI strategies

## Quick Selector

| Source Language | Skill | Key Paradigm Shift |
|-----------------|-------|--------------------|
| **Python** | [python-to-rust](#python-to-rust) | GIL to true parallelism, duck typing to traits, asyncio to tokio |
| **JavaScript / TypeScript** | [nodejs-to-rust](#javascript--typescript-to-rust) | Event loop to tokio, V8 to AOT, npm to Cargo |
| **Go** | [go-to-rust](#go-to-rust) | Goroutines to tokio tasks, channels to mpsc, GC to ownership |
| **Java** | [java-to-rust](#java-to-rust) | JVM to native binary, Spring to Axum, JPA to sqlx |
| **C#** | [csharp-to-rust](#c-to-rust-1) | CLR to native, LINQ to iterators, EF Core to Diesel/sqlx |
| **PHP** | [php-to-rust](#php-to-rust) | Dynamic typing to static types, FPM to long-lived process, Composer to Cargo |
| **C** | [c-to-rust](#c-to-rust) | malloc/free to ownership, pointers to references, headers to modules |
| **C++** | [cpp-to-rust](#c-to-rust-2) | Templates to generics, virtual to trait objects, move semantics to ownership |
| **Zig** | [zig-to-rust](#zig-to-rust) | comptime to proc macros, allocator to ownership, error sets to Result |
| **Lua** | [lua-to-rust](#lua-to-rust) | Table to struct/enum, metatable OOP to traits, coroutine to async |
| **R** | [r-to-rust](#r-to-rust) | data.frame to polars, formula to builder pattern, apply to iterators |
| **Julia** | [julia-to-rust](#julia-to-rust) | Multiple dispatch to traits, JIT to AOT, Array to ndarray |
| **Kotlin** | [kotlin-to-rust](#kotlin-to-rust) | Coroutines to tokio, data class to struct, sealed class to enum, Gradle to Cargo |
| **Swift** | [swift-to-rust](#swift-to-rust) | ARC to ownership, actor to Mutex, protocol to trait, SwiftUI to Leptos |
| **Ruby** | [ruby-to-rust](#ruby-to-rust) | GC to ownership, blocks to closures, Rails to Axum, Bundler to Cargo |
| **Vue** | [vue-to-rust](#vue-to-rust-wasm) | SFC to component functions, ref() to RwSignal, Vite to Trunk |

## Universal Migration Concepts

These concepts apply regardless of source language. Master them once.

### Ownership vs. Garbage Collection

Every source language except C, C++, Zig, and Swift (which uses ARC) has a garbage collector. Rust replaces the GC with compile-time ownership rules.

| Source Pattern | Rust | Rule |
|---------------|------|------|
| Pass object, modify in place | `&mut T` | Exclusive mutable borrow — only one at a time |
| Pass object, read only | `&T` | Shared immutable borrow — many allowed |
| Shared mutable state across threads | `Arc<Mutex<T>>` | Atomic reference count + mutual exclusion |
| GC cleans up unreachable objects | `Drop` trait | Deterministic cleanup at scope exit |
| Lazy initialization | `OnceLock<T>` / `LazyLock<T>` | Thread-safe, exactly-once init |
| Circular references | `Weak<T>` | Breaks reference cycles explicitly |

### Error Handling

| Source Pattern | Rust | Rule |
|---------------|------|------|
| Exceptions (try/catch/throw) | `Result<T, E>` + `?` operator | Errors are values, not control flow |
| Null / nil / None / undefined | `Option<T>` | Type-level absence; `None` is a variant |
| Checked exceptions (Java) | `Result<T, E>` with `thiserror` | Explicit error enums |
| errno (C) / error sets (Zig) | `Result<T, io::Error>` / custom enum | Structured, composable |
| panic / fatal errors | `panic!()` (unrecoverable only) | Use `Result` for expected failures |

### Async Runtime

| Source Pattern | Rust | Notes |
|---------------|------|-------|
| async/await (JS, Python, C#, etc.) | `async fn` + `.await` | Postfix syntax; futures are lazy |
| Promise.all / asyncio.gather | `tokio::join!(a, b, c)` | Concurrent execution |
| Promise.race / asyncio.wait(FIRST) | `tokio::select!` | First-to-complete |
| setTimeout / sleep | `tokio::time::sleep(dur)` | Async; never blocks OS thread |
| Thread pool / executor | `tokio::runtime::Runtime` | Work-stealing M:N scheduler |
| CPU-bound parallelism | `rayon::par_iter()` | Data parallelism; no GIL |

### Build System & Dependencies

| Source Tool | Rust Equivalent |
|-------------|-----------------|
| npm / pip / Maven / NuGet / Composer / Bundler / Gradle / SPM / LuaRocks / CRAN / Pkg | `cargo add` + `Cargo.toml` |
| package.json / requirements.txt / pom.xml / .csproj / Gemfile / build.gradle.kts / Package.swift | `Cargo.toml` |
| lockfile (all forms) | `Cargo.lock` |
| node_modules / vendor / .m2 / .venv / Pods / Carthage | `~/.cargo/registry` (global cache) |
| eslint / pylint / checkstyle | `cargo clippy` |
| prettier / black / gofmt / swiftlint / RuboCop | `cargo fmt` |
| jest / pytest / JUnit / PHPUnit / XCTest / RSpec / kotlin.test | `cargo test` |

---

## Language Quick-Reference Tables

### Python to Rust

| Python | Rust | Quick Note |
|--------|------|------------|
| `def fn():` / `async def` | `fn fn()` / `async fn` | `.await` postfix; futures are lazy |
| `class` | `struct` + `impl` + `trait` | Data + behavior separated |
| `list` / `dict` / `set` | `Vec<T>` / `HashMap<K,V>` / `HashSet<T>` | Typed, homogeneous |
| `None` | `Option::None` | Type-level, not a standalone value |
| `try/except` | `Result<T, E>` + `?` | Errors are values, not control flow |
| `asyncio.gather()` | `tokio::join!(a, b, c)` | Concurrent execution |
| `numpy` / `pandas` | `ndarray::Array2` / `polars::DataFrame` | N-dim arrays; lazy + eager DataFrame |
| FastAPI / Flask / Django | `axum` / `actix-web` | Function-based handlers, Tower middleware |

→ Full guide: `python-to-rust`

### JavaScript / TypeScript to Rust

| JS / TS | Rust | Quick Note |
|---------|------|------------|
| `async function` / `await` | `async fn` / `.await` | Nearly identical syntax |
| `Array` / `T[]` | `Vec<T>` / `&[T]` | Owned vs. borrowed slice |
| `object` / `Record<K,V>` | `HashMap<K,V>` / struct | Typed; prefer struct |
| `null` / `undefined` | `Option::None` | Single sentinel |
| `throw new Error()` | `Err(...)` / `?` | Values, not exceptions |
| Express / Fastify | `axum` | Tower middleware stack |
| Prisma / TypeORM | `sqlx` / `diesel` / `sea-orm` | Compile-time checked SQL |
| `JSON.parse` / `JSON.stringify` | `serde_json::from_str` / `to_string` | Derive-based |

→ Full guide: `nodejs-to-rust`

### Go to Rust

| Go | Rust | Quick Note |
|----|------|------------|
| goroutine | `tokio::spawn(async {})` | M:N scheduling |
| channel (buffered) | `tokio::sync::mpsc::channel(n)` | Multi-producer, single-consumer |
| interface | `trait` | Static dispatch by default |
| `defer` | `Drop` trait + RAII | Scope-bound cleanup |
| error return `(val, err)` | `Result<T, E>` | `?` operator for propagation |
| `nil` | `Option<T>` | Explicit absence |
| `context.Context` | `CancellationToken` | Explicit propagation |
| `go mod` / `go build` | `Cargo.toml` / `cargo build` | Same declarative approach |

→ Full guide: `go-to-rust`

### Java to Rust

| Java | Rust | Quick Note |
|------|------|------------|
| JVM | Native binary (rustc + LLVM) | AOT compilation, no warmup |
| `class` | `struct` + `impl` | No inheritance; composition |
| `interface` | `trait` | Explicit impl; static or dynamic dispatch |
| `Optional<T>` | `Option<T>` | Exhaustive matching |
| Checked exception | `Result<T, E>` | `thiserror` for ergonomics |
| Spring Boot | Axum / Actix-Web | No DI container; constructor injection |
| JPA / Hibernate | Diesel / sqlx | Explicit SQL or type-safe DSL |
| `synchronized` | `Mutex<T>` / `RwLock<T>` | Data-inside-lock pattern |

→ Full guide: `java-to-rust`

### C# to Rust

| C# | Rust | Quick Note |
|----|------|------------|
| CLR / CoreCLR | Native binary | AOT, no JIT, no GC pauses |
| `class` / `interface` | `struct` + `trait` | Composition over inheritance |
| `Nullable<T>` / `T?` | `Option<T>` | Explicit absence |
| `async` / `await` / `Task<T>` | `async fn` → `impl Future` | Stackless coroutines |
| LINQ | `Iterator` combinators | `map`, `filter`, `fold` chains |
| ASP.NET Core | Axum / Actix-Web | Tower middleware pipeline |
| EF Core | Diesel / sqlx | No change tracker; explicit SQL |
| NuGet | crates.io + Cargo | Same declarative dependency model |

→ Full guide: `csharp-to-rust`

### PHP to Rust

| PHP | Rust | Quick Note |
|-----|------|------------|
| Zend Engine / FPM | Native binary + tokio | Long-lived process, no per-request isolation |
| Dynamic typing | Static typing + generics | All types known at compile time |
| `array` (list / dict) | `Vec<T>` / `HashMap<K,V>` / struct | Typed containers |
| `null` / `mixed` | `Option<T>` / concrete types | No universal "any" type |
| `try/catch/finally` | `Result<T, E>` + `?` | `Drop` replaces `finally` |
| Laravel / Symfony | Axum / Actix-Web | Function-based handlers |
| Eloquent / PDO | `sqlx` / Diesel | Async-native, compile-time checked |
| Composer | Cargo | Same dependency resolution |

→ Full guide: `php-to-rust`

### C to Rust

| C | Rust | Quick Note |
|---|------|------------|
| `malloc` / `free` | `Box::new()` / auto-drop | No manual memory management |
| `T*` (pointer) | `&T` / `&mut T` / `Box<T>` | Lifetimes guarantee safety |
| `void*` | Generics `T` or `NonNull<T>` | Type-safe generic code |
| Header `.h` + `.c` | Module system (`mod` / `pub`) | No separate declaration |
| `#define` / `#ifdef` | `const` / `#[cfg(...)]` | Typed, namespaced constants |
| `errno` / `goto cleanup` | `Result<T, E>` + `Drop` (RAII) | Automatic cleanup; no goto |
| Makefile / CMake | `Cargo.toml` | Declarative build |
| `strcpy` / `strcat` | `String::push_str` / `format!` | No buffer overflow risk |

→ Full guide: `c-to-rust`

### C++ to Rust

| C++ | Rust | Quick Note |
|-----|------|------------|
| `std::unique_ptr<T>` | `Box<T>` | Single-owner heap allocation |
| `std::shared_ptr<T>` | `Arc<T>` | Thread-safe reference counting |
| `std::vector<T>` | `Vec<T>` | Same contiguous heap allocation |
| `virtual` dispatch | `dyn Trait` / enum dispatch | Prefer enum for closed sets |
| `template<typename T>` | Generics `<T: Trait>` | Monomorphization; trait bounds |
| Move semantics `std::move` | Implicit destructive move | No use-after-move |
| `std::optional<T>` | `Option<T>` | Same monadic interface |
| `std::exception` → `try/catch` | `Result<T, E>` + `?` | Errors as values |

→ Full guide: `cpp-to-rust`

### Zig to Rust

| Zig | Rust | Quick Note |
|-----|------|------------|
| `comptime` | Const generics + proc macros | Compile-time code generation |
| Allocator parameter | Ownership system (implicit) | No allocator to pass manually |
| `?T` (optional) | `Option<T>` | `null` → `None`; `.?` → `.unwrap()` |
| `!T` (error union) | `Result<T, E>` | `try` → `?` operator |
| `defer` | `Drop` trait + RAII | Scope-exit cleanup |
| `anytype` | Generics `<T: Trait>` | Trait-bounded, not duck-typed |
| `build.zig` / `build.zig.zon` | `Cargo.toml` | Declarative build |
| `std.ArrayList(T)` | `Vec<T>` | No allocator parameter needed |

→ Full guide: `zig-to-rust`

### Lua to Rust

| Lua | Rust | Quick Note |
|-----|------|------------|
| Lua VM (PUC/LuaJIT) | rustc + LLVM (AOT) | No interpreter overhead |
| `table` (array / hash) | `Vec<T>` / `HashMap<K,V>` / struct | Typed, not universal |
| `nil` | `Option<T>` | Explicit absence |
| `coroutine` | `async fn` + tokio | Async/await replaces yield/resume |
| metatable OOP | `trait` + `struct` | Compile-time polymorphism |
| `pcall` / `xpcall` | `Result<T, E>` + `?` | Error as value |
| OpenResty / `ngx.*` | `axum` + tower middleware | Function-based HTTP handlers |
| mlua embedding | `mlua` / `rlua` crate | Embed Lua in Rust incrementally |

→ Full guide: `lua-to-rust`

### R to Rust

| R | Rust | Quick Note |
|---|------|------------|
| `data.frame` / tibble | `polars::DataFrame` | Lazy + columnar |
| `dplyr::filter` / `mutate` / `group_by` | `polars::filter` / `with_column` / `group_by` | Nearly identical API |
| `lapply` / `sapply` / `apply` | `Iterator::map` / `ndarray::axis_iter` | Lazy iteration |
| `ggplot2` | `plotters` | Declarative plotting |
| `lm(y ~ x)` | `linregress::FormulaRegressionBuilder` | Formula-based regression |
| `NA` / `NULL` | `Option<T>` | Missing value as type |
| S3 method dispatch | Trait-based dispatch | Compile-time, not runtime |
| Shiny web apps | Leptos / Axum + HTMX | Reactive web apps |

→ Full guide: `r-to-rust`

### Julia to Rust

| Julia | Rust | Quick Note |
|-------|------|------------|
| JIT compilation | AOT compilation (rustc + LLVM) | No warmup, always fast |
| Multiple dispatch | Trait-based dispatch / enum | Compile-time or closed-set |
| `Array{T,N}` | `ndarray::Array<T, D>` | Same N-dim arrays |
| `@async` / `@sync` | `tokio::spawn` / `tokio::join!` | Async tasks |
| `DataFrames.jl` | `polars` | DataFrame operations |
| `Flux.jl` / deep learning | `burn` / `candle` / `tch-rs` | Deep learning |
| `Plots.jl` / `Makie.jl` | `plotters` (2D) / `kiss3d` (3D) | Visualization |
| `DifferentialEquations.jl` | `ode_solvers` / `diffsol` | ODE solving |

→ Full guide: `julia-to-rust`

### Kotlin to Rust

| Kotlin | Rust | Quick Note |
|---------|------|------------|
| Coroutine (`suspend`) | `async fn` + tokio | Structured concurrency in both |
| `data class` | `struct` + `#[derive(Debug, Clone)]` | Value types with destructuring |
| `sealed class` | `enum` with variant data | Exhaustive `when` → `match` |
| Extension function | Extension trait (`impl X for Y`) | Same pattern via traits |
| `T?` (nullable) | `Option<T>` | Identical semantics |
| Ktor / Spring Boot | `axum` / `actix-web` | Function-based handlers |
| `Flow<T>` (cold stream) | `futures::Stream` | Async stream processing |
| Gradle / `build.gradle.kts` | `Cargo.toml` | Declarative build |

→ Full guide: `kotlin-to-rust`

### Swift to Rust

| Swift | Rust | Quick Note |
|-------|------|------------|
| ARC | Ownership + borrowing | Compile-time, zero runtime overhead |
| `enum` with associated values | `enum` with variant data | Identical pattern |
| `protocol` | `trait` | Explicit conformance |
| `async` / `await` | `async fn` / `.await` | Same syntax model |
| `Task { }` | `tokio::spawn(async { })` | Async task creation |
| `actor` | `Arc<Mutex<T>>` + channel pattern | Isolated mutable state |
| `guard let` / `if let` | `let-else` / `?` / `match` | Early exit patterns |
| Vapor / Hummingbird | `axum` / `actix-web` | HTTP server framework |

→ Full guide: `swift-to-rust`

### Ruby to Rust

| Ruby | Rust | Quick Note |
|------|------|------------|
| GC (generational) | Ownership + RAII | Deterministic, zero pause |
| `class` / module mixin | `struct` + `impl` + `trait` | Composition over inheritance |
| Block / `yield` | `FnMut` closure / `for` | Typed, zero-cost |
| `rescue` / `ensure` | `Result<T, E>` + `?` / `Drop` | Errors as values |
| `nil` | `Option<T>` | Explicit absence |
| `Hash` / `Array` | `HashMap<K,V>` / `Vec<T>` | Typed, homogeneous |
| Rails / Sinatra | `axum` / `actix-web` | Function-based handlers |
| `Gemfile` + Bundler | `Cargo.toml` | Same dependency workflow |

→ Full guide: `ruby-to-rust`

### Vue to Rust (WASM)

| Vue | Rust WASM | Quick Note |
|-----|-----------|------------|
| Vue SFC (`.vue`) | Leptos component fn / Dioxus `rsx!` | Single-file → single-function |
| `ref()` / `reactive()` | `RwSignal::new()` / `create_signal()` | Fine-grained reactivity |
| `computed()` | `Memo::new()` / `create_memo()` | Derived signals |
| `v-if` / `v-for` | `<Show>` / `<For>` components | Conditional/loop rendering |
| Pinia / Vuex | Signal-based store module | Reactive state management |
| `vue-router` | `leptos_router` / `dioxus-router` | Client-side routing |
| `provide` / `inject` | `provide_context()` / `use_context()` | Dependency injection |
| Vite / webpack | Trunk / `wasm-pack` | WASM build pipeline |

→ Full guide: `vue-to-rust`

---

## Universal Common Mistakes

These mistakes apply across all source languages:

### 1. Over-Cloning (GC Mindset)
GC-trained developers clone everything to "keep a reference." In Rust, borrow first, clone only when you truly need independent ownership.

### 2. `unwrap()` / `expect()` as Null-Checks
Source-language habits of ignoring null/errors become `unwrap()` calls that panic. Use `?`, `match`, or `unwrap_or()` instead.

### 3. Holding Locks Across `.await`
Most source languages serialize via GIL or single-threaded event loops. In Rust's multi-threaded runtime, holding a `MutexGuard` across `.await` causes deadlocks.

### 4. Class Hierarchies → Trait Emulation
Don't recreate deep inheritance trees. Use `enum` for closed type sets, traits for open extension, and composition for code reuse.

### 5. `Box<dyn Error>` Everywhere
Type-erased errors lose information. Use `thiserror` to define structured error enums that callers can match on.

### 6. String Slicing by Character Index
Most source languages index strings by character. Rust strings are UTF-8 bytes. `&s[0..1]` panics on multi-byte characters. Use `s.chars()`.

### 7. Expecting Runtime Reflection
Rust has no `typeof`, `instanceof`, `getattr`, or `method_exists` for production logic. Use traits, enums, and generics instead.

---

## Migration Strategy (Language-Agnostic)

Regardless of source language, follow this migration order:

| Phase | What | How |
|-------|------|-----|
| 1 | **Data types** | Define Rust structs/enums matching source types. Share via JSON/Protobuf schema. |
| 2 | **Pure functions** | Port stateless logic first. These have no dependencies and are easiest to test. |
| 3 | **I/O boundary** | Replace HTTP handlers, DB queries, file I/O behind the same interfaces. |
| 4 | **Concurrency** | Convert threads/coroutines/async tasks to tokio tasks. Measure, don't assume. |
| 5 | **Full cutover** | Remove the source runtime. Keep FFI bridges for legacy compatibility. |

## Cross-Reference

This skill is the master index. Each per-language skill contains:
- Detailed architecture mapping with narrative explanations
- Complete type-system mapping tables (30-80+ entries)
- Framework-specific migration guides (e.g., Django→Axum, Spring→Actix)
- Canonical code patterns with before/after examples
- Language-specific common mistakes with fixes
- FFI and incremental migration strategies
- Reference implementations with real project examples

| Skill | Best For |
|-------|----------|
| `python-to-rust` | Scientific computing, web backends, PyO3 extensions |
| `php-to-rust` | Laravel/Symfony migration, dynamic-to-static typing |
| `nodejs-to-rust` | Express/Fastify backends, event-loop to tokio |
| `csharp-to-rust` | ASP.NET migration, LINQ to iterators |
| `cpp-to-rust` | Template-to-generics, class hierarchies |
| `zig-to-rust` | Comptime-to-proc-macro, allocator patterns |
| `java-to-rust` | Spring Boot services, enterprise patterns |
| `r-to-rust` | Statistical computing, tidyverse to polars |
| `go-to-rust` | Goroutine/channel patterns, interface migration |
| `julia-to-rust` | Numerical computing, multiple dispatch |
| `c-to-rust` | Systems programming, FFI, manual memory |
| `lua-to-rust` | OpenResty/Nginx, embedded scripting |
| `kotlin-to-rust` | Coroutine-based services, Android backends, Ktor to Axum |
| `swift-to-rust` | iOS/macOS services, ARC to ownership, SwiftUI to Leptos |
| `ruby-to-rust` | Rails/Sinatra migration, dynamic-to-static typing, block patterns |
| `vue-to-rust` | Frontend WASM, reactive components |
