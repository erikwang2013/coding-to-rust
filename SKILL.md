---
name: coding-to-rust
description: Use when migrating code from ANY source language to Rust — serves as the master index covering 13 languages: Python, JavaScript/TypeScript, Go, Java, C#, PHP, C, C++, Zig, Lua, R, Julia, and Vue. Provides quick-reference comparison tables, shared migration concepts (ownership, async, error handling, build system), language-specific cheat sheets, and pointers to in-depth per-language skills. Trigger when the user says "migrate X to Rust", "rewrite in Rust", "port from X to Rust", or asks about Rust equivalents for concepts from another language.
---

# Coding to Rust — Master Migration Index

## Overview

This is the entry point for migrating code from any source language to Rust. It covers 13 languages with quick-reference tables, shared concepts that apply across all migrations, and pointers to detailed per-language skills.

**How to use this skill:**
1. Find your source language in the [Quick Selector](#quick-selector) below
2. Scan the shared concepts section for universal Rust patterns
3. Read your language's quick-reference table for the most common mappings
4. Follow the link to the per-language skill for deep-dive patterns, code examples, and FFI strategies

## Quick Selector

| Source Language | Skill | Key Paradigm Shift |
|-----------------|-------|--------------------|
| **Python** | [python-to-rust](#python--rust) | GIL → true parallelism, duck typing → traits, asyncio → tokio |
| **JavaScript / TypeScript** | [nodejs-to-rust](#javascript--rust) | Event loop → tokio, V8 → AOT, npm → Cargo |
| **Go** | [go-to-rust](#go--rust) | Goroutines → tokio tasks, channels → mpsc, GC → ownership |
| **Java** | [java-to-rust](#java--rust) | JVM → native binary, Spring → Axum, JPA → sqlx |
| **C#** | [csharp-to-rust](#c--rust-1) | CLR → native, LINQ → iterators, EF Core → Diesel/sqlx |
| **PHP** | [php-to-rust](#php--rust) | Dynamic typing → static types, FPM → long-lived process, Composer → Cargo |
| **C** | [c-to-rust](#c--rust) | malloc/free → ownership, pointers → references, headers → modules |
| **C++** | [cpp-to-rust](#c--rust-2) | Templates → generics, virtual → trait objects, move semantics → ownership |
| **Zig** | [zig-to-rust](#zig--rust) | comptime → proc macros, allocator → ownership, error sets → Result |
| **Lua** | [lua-to-rust](#lua--rust) | Table → struct/enum, metatable OOP → traits, coroutine → async |
| **R** | [r-to-rust](#r--rust) | data.frame → polars, formula → builder pattern, apply → iterators |
| **Julia** | [julia-to-rust](#julia--rust) | Multiple dispatch → traits, JIT → AOT, Array → ndarray |
| **Vue** | [vue-to-rust](#vue--rust) | SFC → component functions, ref() → RwSignal, Vite → Trunk |

## Universal Migration Concepts

These concepts apply regardless of source language. Master them once.

### Ownership vs. Garbage Collection

Every source language except C, C++, and Zig has a garbage collector. Rust replaces the GC with compile-time ownership rules.

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
| npm / pip / Maven / NuGet / Composer / LuaRocks / CRAN / Pkg | `cargo add` + `Cargo.toml` |
| package.json / requirements.txt / pom.xml / .csproj | `Cargo.toml` |
| lockfile (all forms) | `Cargo.lock` |
| node_modules / vendor / .m2 / .venv | `~/.cargo/registry` (global cache) |
| eslint / pylint / checkstyle | `cargo clippy` |
| prettier / black / gofmt | `cargo fmt` |
| jest / pytest / JUnit / PHPUnit | `cargo test` |

---

## Language Quick-Reference Tables

### Python → Rust

| Python | Rust | Quick Note |
|--------|------|------------|
| `def fn():` / `async def` | `fn fn()` / `async fn` | Same syntax; `.await` postfix |
| `class` | `struct` + `impl` + `trait` | Data + behavior separated |
| `list` / `dict` / `set` | `Vec<T>` / `HashMap<K,V>` / `HashSet<T>` | Typed, homogeneous |
| `None` | `Option::None` | Type-level, not a standalone value |
| `try/except` | `Result<T, E>` + `?` | Errors are values |
| `with open()` | `File::open()` + RAII | Drop auto-closes at scope exit |
| `@dataclass` / pydantic | `#[derive(Debug, Clone, Serialize)]` | Derive macros |
| `asyncio.gather()` | `tokio::join!()` | Concurrent execution |
| `numpy` array | `ndarray::Array2` | Same N-dim arrays; SIMD auto |
| `pandas` DataFrame | `polars::DataFrame` | Lazy + eager; columnar |
| FastAPI / Flask / Django | `axum` / `actix-web` | Function-based handlers |
| PyO3 extension | `pyo3` + `maturin` | Bidirectional Rust↔Python FFI |

→ Full guide: `python-to-rust`

### JavaScript / TypeScript → Rust

| JS / TS | Rust | Quick Note |
|---------|------|------------|
| `async function` / `await` | `async fn` / `.await` | Nearly identical syntax |
| `Promise.all([a,b])` | `tokio::join!(a, b)` | Concurrent futures |
| `Promise.race([a,b])` | `tokio::select!` | First to complete |
| `Array` / `T[]` | `Vec<T>` / `&[T]` | Owned vs. borrowed |
| `object` / `Record<K,V>` | `HashMap<K,V>` / struct | Typed; prefer struct |
| `null` / `undefined` | `Option::None` | Single sentinel |
| `throw new Error()` | `Err(...)` / `?` | Values, not exceptions |
| Express / Fastify | `axum` | Tower middleware stack |
| `EventEmitter` | `tokio::sync::broadcast` | One-to-many channel |
| `setTimeout(fn, ms)` | `tokio::time::sleep(ms).await` | Async, non-blocking |
| `JSON.parse` / `JSON.stringify` | `serde_json::from_str` / `to_string` | Derive-based |
| Prisma / TypeORM | `sqlx` / `diesel` / `sea-orm` | Compile-time checked SQL |

→ Full guide: `nodejs-to-rust`

### Go → Rust

| Go | Rust | Quick Note |
|----|------|------------|
| goroutine | `tokio::spawn(async {})` | M:N scheduling |
| channel (buffered) | `tokio::sync::mpsc::channel(n)` | Multi-producer, single-consumer |
| `select` | `tokio::select!` | Match first-ready |
| interface | `trait` | Static dispatch by default |
| `defer` | `Drop` trait + RAII | Scope-bound, not function-scoped |
| error return `(val, err)` | `Result<T, E>` | `?` operator for propagation |
| `nil` | `Option<T>` | Explicit absence |
| struct embedding | Composition + `Deref` | No type promotion |
| `context.Context` | `CancellationToken` | Explicit propagation |
| `go mod` / `go build` | `Cargo.toml` / `cargo build` | Same declarative approach |

→ Full guide: `go-to-rust`

### Java → Rust

| Java | Rust | Quick Note |
|------|------|------------|
| JVM | Native binary (rustc + LLVM) | AOT compilation, no warmup |
| `class` | `struct` + `impl` | No inheritance; composition |
| `interface` | `trait` | Explicit impl; no default methods |
| `Optional<T>` | `Option<T>` | Exhaustive matching |
| Checked exception | `Result<T, E>` | `thiserror` for ergonomics |
| `Stream<T>` | `Iterator<Item = T>` | Lazy, zero-cost |
| Spring Boot | Axum / Actix-Web | No DI container; constructor injection |
| JPA / Hibernate | Diesel / sqlx | Explicit SQL or type-safe DSL |
| `@Transactional` | `pool.begin().await?` | Explicit transaction scope |
| Maven / Gradle | Cargo | Single build tool |
| `synchronized` | `Mutex<T>` / `RwLock<T>` | Data-inside-lock pattern |
| `ExecutorService` | `tokio::runtime::Runtime` | Work-stealing scheduler |

→ Full guide: `java-to-rust`

### C# → Rust

| C# | Rust | Quick Note |
|----|------|------------|
| CLR / CoreCLR | Native binary | AOT, no JIT, no GC pauses |
| `class` / `interface` | `struct` + `trait` | Composition over inheritance |
| `record` | `#[derive(Clone, Debug, PartialEq)]` | Value equality |
| `Nullable<T>` / `T?` | `Option<T>` | Explicit absence |
| `async` / `await` / `Task<T>` | `async fn` → `impl Future` | Stackless coroutines |
| LINQ | `Iterator` combinators | `map`, `filter`, `fold` chains |
| `IDisposable` / `using` | `Drop` trait + RAII | Deterministic cleanup |
| ASP.NET Core | Axum / Actix-Web | Tower middleware pipeline |
| EF Core | Diesel / sqlx | No change tracker; explicit SQL |
| `event` / delegate | `broadcast` channel / `Fn` traits | No multicast built-in |
| NuGet | crates.io + Cargo | Same declarative dependency model |
| `Parallel.ForEach` | `rayon::par_iter()` | Data parallelism |

→ Full guide: `csharp-to-rust`

### PHP → Rust

| PHP | Rust | Quick Note |
|-----|------|------------|
| Zend Engine / FPM | Native binary + tokio | Long-lived process, no per-request isolation |
| Dynamic typing | Static typing + generics | All types known at compile time |
| `array` (list / dict / mixed) | `Vec<T>` / `HashMap<K,V>` / struct | Typed containers |
| `null` / `mixed` | `Option<T>` / concrete types | No universal "any" type |
| `class` / `extends` / `interface` | `struct` + `trait` composition | No inheritance hierarchy |
| `try/catch/finally` | `Result<T, E>` + `?` | `Drop` replaces `finally` |
| Laravel / Symfony | Axum / Actix-Web | Function-based handlers, no service container |
| Eloquent / PDO | `sqlx` / Diesel | Async-native, compile-time checked |
| Composer | Cargo | Same dependency resolution |
| Blade / Twig | `askama` / `tera` / `minijinja` | Template engines |
| Middleware pipeline | `tower::Layer` stack | Composable middleware |

→ Full guide: `php-to-rust`

### C → Rust

| C | Rust | Quick Note |
|---|------|------------|
| `malloc` / `free` | `Box::new()` / auto-drop | No manual memory management |
| `T*` (pointer) | `&T` / `&mut T` / `Box<T>` | Lifetimes guarantee safety |
| `void*` | Generics `T` or `NonNull<T>` | Type-safe generic code |
| `struct { ... }` | `struct { ... }` | `#[repr(C)]` for FFI only |
| Header `.h` + `.c` | Module system (`mod` / `pub`) | No separate declaration |
| `#define` / `#ifdef` | `const` / `#[cfg(...)]` | Typed, namespaced constants |
| `errno` / `goto cleanup` | `Result<T, E>` + `Drop` (RAII) | Automatic cleanup; no goto |
| `printf` / `scanf` | `println!` / `s.parse::<T>()` | Compile-time format checking |
| Makefile / CMake | `Cargo.toml` | Declarative build |
| `pthread_create` | `std::thread::spawn` | Type-safe closure entry |
| `qsort` / `bsearch` | `slice.sort()` / `slice.binary_search()` | Type-safe, no void* |
| `strcpy` / `strcat` | `String::push_str` / `format!` | No buffer overflow risk |

→ Full guide: `c-to-rust`

### C++ → Rust

| C++ | Rust | Quick Note |
|-----|------|------------|
| `std::unique_ptr<T>` | `Box<T>` | Single-owner heap allocation |
| `std::shared_ptr<T>` | `Arc<T>` | Thread-safe reference counting |
| `std::vector<T>` | `Vec<T>` | Same contiguous heap allocation |
| `virtual` dispatch | `dyn Trait` / enum dispatch | Prefer enum for closed sets |
| `template<typename T>` | Generics `<T: Trait>` | Monomorphization; trait bounds |
| `concept` (C++20) | Trait bounds `T: Trait` | Compile-time interface constraint |
| Move semantics `std::move` | Implicit destructive move | No use-after-move |
| Copy constructor `T(const T&)` | `#[derive(Clone)]` | Explicit, not implicit |
| Destructor `~T()` | `impl Drop` | Identical semantics; cannot fail |
| `std::optional<T>` | `Option<T>` | Same monadic interface |
| `std::variant<A,B>` | `enum { A(A), B(B) }` | `match` replaces `std::visit` |
| `std::exception` → `try/catch` | `Result<T, E>` + `?` | Errors as values |

→ Full guide: `cpp-to-rust`

### Zig → Rust

| Zig | Rust | Quick Note |
|-----|------|------------|
| `comptime` | Const generics + proc macros | Compile-time code generation |
| Allocator parameter | Ownership system (implicit) | No allocator to pass manually |
| `?T` (optional) | `Option<T>` | `null` → `None`; `.?` → `.unwrap()` |
| `!T` (error union) | `Result<T, E>` | `try` → `?` operator |
| `defer` | `Drop` trait + RAII | Scope-exit cleanup |
| `errdefer` | `Drop` with state check | Conditional cleanup on error |
| `anytype` | Generics `<T: Trait>` | Trait-bounded, not duck-typed |
| `[]T` (slice) | `&[T]` | Fat pointer (ptr + len), identical |
| `build.zig` / `build.zig.zon` | `Cargo.toml` | Declarative build |
| `@import("std")` | `use std::...` | Standard library import |
| `std.ArrayList(T)` | `Vec<T>` | No allocator parameter needed |
| `packed struct` / `extern struct` | `#[repr(C)]` / `#[repr(packed)]` | ABI control |

→ Full guide: `zig-to-rust`

### Lua → Rust

| Lua | Rust | Quick Note |
|-----|------|------------|
| Lua VM (PUC/LuaJIT) | rustc + LLVM (AOT) | No interpreter overhead |
| `table` (array / hash / mixed) | `Vec<T>` / `HashMap<K,V>` / struct | Typed, not universal |
| `nil` | `Option<T>` | Explicit absence |
| `function` | `fn(...)` / closures | Typed, zero-cost |
| `coroutine` | `async fn` + tokio | Async/await replaces yield/resume |
| metatable OOP | `trait` + `struct` | Compile-time polymorphism |
| `require` / LuaRocks | `mod` / Cargo | Compile-time module resolution |
| `pcall` / `xpcall` | `Result<T, E>` + `?` | Error as value |
| `string.match` / `gsub` | `regex::Regex` | More powerful regex engine |
| OpenResty / `ngx.*` | `axum` + tower middleware | Function-based HTTP handlers |
| `lfs` (filesystem) | `std::fs` / `walkdir` | Standard library |
| `luasocket` | `tokio::net` / `reqwest` | Async networking |
| mlua embedding | `mlua` / `rlua` crate | Embed Lua in Rust incrementally |

→ Full guide: `lua-to-rust`

### R → Rust

| R | Rust | Quick Note |
|---|------|------------|
| `data.frame` / tibble | `polars::DataFrame` | Lazy + columnar |
| `c(1, 2, 3)` / `seq()` | `vec![1, 2, 3]` / ranges | Vector creation |
| `dplyr::filter` / `mutate` / `group_by` | `polars::filter` / `with_column` / `group_by` | Nearly identical API |
| `lapply` / `sapply` / `apply` | `Iterator::map` / `ndarray::axis_iter` | Lazy iteration |
| `ggplot2` | `plotters` | Declarative plotting |
| `lm(y ~ x)` | `linregress::FormulaRegressionBuilder` | Formula-based regression |
| `glm()` / `randomForest` | `smartcore` / `linfa` | ML in Rust |
| `NA` / `NULL` | `Option<T>` | Missing value as type |
| S3 method dispatch | Trait-based dispatch | Compile-time, not runtime |
| `install.packages("pkg")` | `cargo add` | Same workflow |
| `testthat` | `#[test]` + `cargo test` | Built-in test harness |
| Shiny web apps | Leptos / Axum + HTMX | Reactive web apps |
| `extendr` FFI | `extendr` crate | Call Rust from R packages |

→ Full guide: `r-to-rust`

### Julia → Rust

| Julia | Rust | Quick Note |
|-------|------|------------|
| JIT compilation | AOT compilation (rustc + LLVM) | No warmup, always fast |
| Multiple dispatch | Trait-based dispatch / enum | Compile-time or closed-set |
| `Array{T,N}` | `ndarray::Array<T, D>` | Same N-dim arrays |
| `Vector{T}` / `Matrix{T}` | `Vec<T>` / `ndarray::Array2` | Dynamic / fixed-dim |
| `@async` / `@sync` | `tokio::spawn` / `tokio::join!` | Async tasks |
| `@spawn` / `@threads for` | `rayon::spawn` / `rayon::par_iter` | CPU parallelism |
| `Channel{T}` | `tokio::sync::mpsc` | Async channels |
| `DataFrames.jl` | `polars` | DataFrame operations |
| `DifferentialEquations.jl` | `ode_solvers` / `diffsol` | ODE solving |
| `Flux.jl` / deep learning | `burn` / `candle` / `tch-rs` | Deep learning |
| `Plots.jl` / `Makie.jl` | `plotters` (2D) / `kiss3d` (3D) | Visualization |
| `Symbolics.jl` | `symbolica` crate | Computer algebra |

→ Full guide: `julia-to-rust`

### Vue → Rust (WASM)

| Vue | Rust WASM | Quick Note |
|-----|-----------|------------|
| Vue SFC (`.vue`) | Leptos component fn / Dioxus `rsx!` | Single-file → single-function |
| `ref()` / `reactive()` | `create_signal()` / `RwSignal::new()` | Fine-grained reactivity |
| `computed()` | `create_memo()` / `Memo::new()` | Derived signals |
| `watch()` / `watchEffect()` | `create_effect()` | Reactive side effects |
| `v-if` / `v-show` | `<Show>` component / `style:display` | Conditional rendering |
| `v-for` | `.iter().map()` / `<For>` component | Keyed list rendering |
| `v-model` | Controlled input with `on:input` | Two-way binding |
| Pinia / Vuex | Signal-based store module | Reactive state management |
| `vue-router` | `leptos_router` / `dioxus-router` | Client-side routing |
| `provide` / `inject` | `provide_context()` / `use_context()` | Dependency injection |
| `onMounted` / `onUnmounted` | `on_mount()` / `on_cleanup()` | Lifecycle hooks |
| Vite / webpack | Trunk / `wasm-pack` | WASM build pipeline |
| Nuxt (SSR) | Leptos SSR / Dioxus Fullstack | Isomorphic Rust rendering |
| `fetch()` / axios | `reqwest` (WASM) / `gloo-net` | HTTP from browser |
| TypeScript | Rust (always typed) | No distinction needed |

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

| Skill | Lines | Best For |
|-------|-------|----------|
| `python-to-rust` | 1012 | Scientific computing, web backends, PyO3 extensions |
| `php-to-rust` | 928 | Laravel/Symfony migration, dynamic-to-static typing |
| `nodejs-to-rust` | 859 | Express/Fastify backends, event-loop to tokio |
| `csharp-to-rust` | 817 | ASP.NET migration, LINQ to iterators |
| `cpp-to-rust` | 802 | Template-to-generics, class hierarchies |
| `zig-to-rust` | 777 | Comptime-to-proc-macro, allocator patterns |
| `java-to-rust` | 743 | Spring Boot services, enterprise patterns |
| `r-to-rust` | 674 | Statistical computing, tidyverse to polars |
| `go-to-rust` | 650 | Goroutine/channel patterns, interface migration |
| `julia-to-rust` | 644 | Numerical computing, multiple dispatch |
| `c-to-rust` | 641 | Systems programming, FFI, manual memory |
| `lua-to-rust` | 640 | OpenResty/Nginx, embedded scripting |
| `vue-to-rust` | 627 | Frontend WASM, reactive components |
