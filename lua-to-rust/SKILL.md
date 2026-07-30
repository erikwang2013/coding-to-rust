---
name: lua-to-rust
description: Use when migrating Lua codebases to Rust — covers table to struct/enum, metatable OOP to traits, coroutine to async/await, OpenResty to Axum, LuaRocks to Cargo, and incremental migration via mlua embedding. Includes canonical code patterns, common mistakes, and reference implementations.
updated: 2026-07-30
---

# Lua to Rust Migration

## Architecture Mapping

Lua's register-based VM (PUC-Rio Lua or LuaJIT's tracing JIT) executes bytecode in a single-threaded event loop, with all state held in the global `_G` table and a C API for native extensions. Rust replaces the interpreter entirely — producing a statically linked native binary with no runtime overhead. Where Lua relies on garbage collection (incremental mark-sweep or generational in LuaJIT) and the `coroutine` module for cooperative multitasking, Rust provides ownership-based memory management with deterministic cleanup and a rich async runtime via `tokio`.

The critical architectural shift: Lua's "table as universal data structure" becomes Rust's typed `struct` and `enum`; Lua's metatable-based OOP becomes Rust's trait system; and Lua's C-module extension mechanism (`require "module"` loading `.so` files) becomes Cargo's compile-time dependency resolution. For large Lua codebases (OpenResty/Nginx, Redis scripts, game engine scripting), the recommended strategy embeds Lua in Rust via `mlua` and incrementally rewrites modules, preserving the scripting flexibility at the edges where it matters.

| Lua Concept | Rust Equivalent |
|---|---|
| Lua VM (lua/luajit) | rustc + LLVM (AOT compilation) |
| LuaRocks | Cargo + crates.io |
| C API / LuaJIT FFI | bindgen + cc crate / FFI declarations |
| require / package.path | mod / use / `Cargo.toml [dependencies]` |
| Global table `_G` | No global mutable state — prefer dependency injection or `once_cell::sync::Lazy` |
| Embedded scripting | mlua / rlua crate (embed Lua in Rust) or rewrite natively |
| Nginx + OpenResty | Custom Rust HTTP service (actix-web / axum) |

Lua excels at glue code and DSLs. When migrating to Rust, preserve the scripting flexibility either through mlua embedding or by defining a clean trait-based plugin system.

## Type System Mapping

Lua has exactly 8 types, all dynamic. Rust has a rich static type system. The migration requires deciding which concrete Rust type replaces each polymorphic Lua table.

| Lua Type | Rust Type | Notes |
|---|---|---|
| `nil` | `Option<T>` | Every nullable value becomes `Option` |
| `boolean` | `bool` | Direct mapping |
| `number` | `f64` or `i64` | Lua 5.3+ has integer subtype; LuaJIT uses f64. Choose based on domain. |
| `string` | `String` / `&str` | Lua strings are immutable byte buffers; Rust strings are UTF-8. Use `Vec<u8>` for binary data. |
| `table` (array) | `Vec<T>` | Numeric keys 1..n in Lua map to 0-indexed `Vec` in Rust |
| `table` (hash) | `HashMap<K, V>` / `BTreeMap<K, V>` | String-keyed tables; use `BTreeMap` when ordering matters |
| `table` (mixed) | `struct` with named fields | When keys are fixed and known at compile time |
| `table` (object) | `struct` + `impl` | Metatable-based OOP becomes trait implementations |
| `function` | `fn(...) -> ...` / `Fn` / `FnMut` / `FnOnce` | Function pointers or closure traits |
| `thread` (coroutine) | `async fn` / `Future` | Lua coroutines map to async/await |
| `userdata` | `Box<dyn Any>` / opaque struct | C-protected userdata becomes Rust struct with private fields |

### Dynamic Dispatch Pattern

Lua tables that hold mixed types require an enum for static typing:

```rust
// Lua: local mixed = { name = "alice", age = 30, tags = {"a","b"} }
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
enum LuaValue {
    Nil,
    Bool(bool),
    Number(f64),
    String(String),
    Table(HashMap<String, LuaValue>),
    Array(Vec<LuaValue>),
}

// or define a concrete struct for known shapes
#[derive(Debug, Clone)]
struct Player {
    name: String,
    age: u32,
    tags: Vec<String>,
}
```

## Memory & Ownership Model

Lua uses a tracing garbage collector (incremental mark-sweep in PUC Lua, generational in LuaJIT). Rust ownership eliminates GC entirely.

| Lua Pattern | Rust Translation |
|---|---|
| GC-managed objects | Ownership system — values dropped at end of scope |
| Shared references (table aliases) | `Rc<T>` for shared ownership, `Arc<T>` for thread-safe sharing |
| Mutable shared state | `RefCell<T>` (single-threaded) or `RwLock<T>` / `Mutex<T>` (multi-threaded) |
| Weak references | `Weak<T>` — prevents reference cycles |
| `__gc` metamethod | `Drop` trait implementation |
| Circular references | `Weak<T>` or arena-based allocation with indices instead of references |

### Reference Cycle Example

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

// Lua:
// local a = {}
// local b = { parent = a }
// a.child = b
// -- GC handles the cycle

// Rust:
struct Node {
    parent: RefCell<Weak<Node>>,   // Weak breaks reference cycles
    child: RefCell<Option<Rc<Node>>>,
}

impl Drop for Node {
    fn drop(&mut self) {
        // Rust equivalent of __gc metamethod
        tracing::debug!("Node dropped");
    }
}
```

## Concurrency / Async Translation

Lua has no native threading (only coroutines). LuaJIT has basic FFI-based threads but no memory safety guarantees. Rust provides both sync and async concurrency with compile-time safety.

| Lua Concurrency | Rust Equivalent |
|---|---|
| `coroutine.create(f)` | `tokio::spawn(async move { ... })` |
| `coroutine.resume(co)` | `.await` on a `Future` |
| `coroutine.yield(val)` | `pending!()` / async streams / `yield_now().await` |
| `coroutine.status(co)` | `Future` state is implicit; use `futures::future::poll_fn` |
| `coroutine.wrap(f)` | Closure returning a `Future` |
| No parallelism (single OS thread) | `rayon::spawn` / `tokio::task::spawn_blocking` |
| `lua_lock` (GIL) | No GIL — concurrent access via `Arc<RwLock<T>>` |

### Coroutine to Async Stream

```rust
use futures::stream::{self, Stream};

// Lua:
// function range_gen(n)
//   for i = 1, n do coroutine.yield(i) end
// end

// Rust:
fn range_gen(n: u64) -> impl Stream<Item = u64> {
    stream::iter(1..=n)
}
// or use async-generator syntax (nightly or genawaiter crate)
```

### Error-Safe Coroutine Resume

```rust
// Lua:
// local ok, err = pcall(some_function, arg1, arg2)
// if not ok then print(err) end

// Rust:
fn safe_call<F, T, E>(f: F) -> Result<T, Box<dyn std::error::Error>>
where
    F: FnOnce() -> Result<T, E>,
    E: std::error::Error + 'static,
{
    f().map_err(|e| Box::new(e) as Box<dyn std::error::Error>)
}

// or catch panics:
fn safe_panic<F, T>(f: F) -> Result<T, Box<dyn std::any::Any + Send>>
where
    F: FnOnce() -> T + std::panic::UnwindSafe,
{
    std::panic::catch_unwind(f)
}
```

## Build System & Dependencies

| Lua | Rust |
|---|---|
| LuaRocks (.rockspec) | Cargo.toml |
| `require "module"` | `mod module;` / `use crate::module;` |
| `package.path` | Module search via filesystem hierarchy under `src/` |
| `package.cpath` (C modules) | `[dependencies]` with `-sys` crates or build.rs + cc crate |
| `luarocks install` | `cargo add <crate>` |
| luarocks tree (local) | `target/` directory |
| Lua 5.1 / 5.2 / 5.3 / 5.4 / LuaJIT | rustc edition 2018/2021/2024 + target triple |

### Cargo.toml for a Migrated Lua Project

```toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
regex = "1"
mlua = { version = "0.10", features = ["lua54", "vendored"] }  # 渐进式迁移时嵌入 Lua
once_cell = "1"
tracing = "0.1"
thiserror = "2"

[dev-dependencies]
rstest = "0.22"  # 表驱动测试，类似 busted
```

## Standard Library & Ecosystem Mapping

| Lua Library / Function | Rust Equivalent |
|---|---|
| `table.insert(t, v)` | `vec.push(v)` |
| `table.remove(t, pos)` | `vec.remove(pos)` |
| `table.concat(t, sep)` | `vec.join(sep)` (via itertools) 或 `vec.iter().join(sep)` |
| `table.sort(t, cmp)` | `vec.sort_by(cmp)` / `vec.sort()` |
| `table.pack(...)` | 元组 `(a, b, c)` 或 `vec![a, b, c]` |
| `table.unpack(t)` | 解构语法或索引访问 |
| `string.sub(s, i, j)` | `&s[i-1..j]` (注意边界检查) |
| `string.find(s, pat)` | `s.find(pat)` / `regex::Regex::find` |
| `string.gsub(s, pat, repl)` | `regex::Regex::replace_all` |
| `string.format(...)` | `format!(...)` |
| `string.len(s)` | `s.len()` (字节长度) 或 `s.chars().count()` (字符数) |
| `string.match(s, pat)` | `regex::Regex::captures` |
| `string.gmatch(s, pat)` | `regex::Regex::captures_iter` |
| `string.byte(s, i)` | `s.as_bytes()[i-1]` |
| `string.char(...)` | `char::from_u32(...)` / `std::str::from_utf8` |
| `io.open(path, mode)` | `std::fs::File::open(path)` / `std::fs::File::create(path)` |
| `io.read("*all")` | `std::fs::read_to_string(path)` |
| `io.write(...)` | `std::fs::write(path, content)` |
| `io.lines(path)` | `std::io::BufRead::lines()` |
| `os.time()` | `std::time::SystemTime::now()` |
| `os.date(fmt)` | `chrono::Local::now().format(fmt)` |
| `os.execute(cmd)` | `std::process::Command::new(cmd).status()` |
| `math.random()` | `rand::random()` / `rand::thread_rng()` |
| `math.floor(x)` | `x.floor()` (f64) |
| `bit32 / bit` | 位操作直接使用 `\| & ^ << >>` |
| `debug.traceback()` | `std::backtrace::Backtrace::capture()` |
| `lpeg` (LPEG parsing) | `nom` / `pest` / `chumsky` |
| `cjson` | `serde_json` |
| `lfs` (LuaFileSystem) | `std::fs` / `walkdir` crate |
| `luasocket` | `tokio::net` (TCP/UDP) / `reqwest` (HTTP) |
| `lua-http-parser` | `hyper` / `http` crate |
| `busted` (testing) | `rstest` + `tokio::test` / `cargo test` |
| `luacov` | `cargo tarpaulin` / `cargo-llvm-cov` |

### OpenResty / Nginx → Rust HTTP Services

OpenResty (Nginx + LuaJIT) is one of the most common Lua deployment targets. Its non-blocking I/O model maps naturally to Rust's async ecosystem.

| OpenResty / Nginx Concept | Rust Equivalent | Notes |
|---|---|---|
| `nginx.conf` `location / {}` | `axum::Router::route()` | Route definitions in Rust, not config files |
| `ngx.req.get_uri_args()` | `axum::extract::Query<T>` | Type-safe query extraction via serde |
| `ngx.req.get_post_args()` | `axum::extract::Form<T>` / `Json<T>` | Typed body extraction |
| `ngx.req.get_headers()` | `axum::http::HeaderMap` | Header extraction |
| `ngx.say()` / `ngx.print()` | `axum::response::Html` / `Json` | Typed response types |
| `ngx.exit(code)` | Return `StatusCode` enum variant | Map HTTP status directly |
| `ngx.sleep(seconds)` | `tokio::time::sleep(Duration::from_secs(n))` | Async sleep, no blocking |
| `ngx.timer.at(delay, callback)` | `tokio::time::interval` + `tokio::spawn` | Scheduled async tasks |
| `ngx.location.capture()` | `reqwest::Client` internal subrequest | HTTP client for internal calls |
| `ngx.shared.DICT` (shared memory) | `dashmap::DashMap` / `moka::Cache` | In-process concurrent cache |
| `lua-resty-core` | Axum + tower ecosystem | Standard library of middleware |
| `lua-resty-redis` / `redis-lua` | `redis` crate (async) | Async Redis with connection pooling |
| `lua-resty-mysql` / `pgmoon` | `sqlx` | Async, compile-time checked SQL |
| `lua-resty-jwt` | `jsonwebtoken` crate | JWT encode/decode |
| `lua-resty-openidc` | `openidconnect` crate | OIDC client |
| `lua-resty-http` | `reqwest` crate | Async HTTP client |
| `lua-resty-template` | `askama` / `tera` / `minijinja` | Template rendering |
| `lua-resty-limit-traffic` | `tower::limit::RateLimitLayer` / `governor` | Rate limiting middleware |
| Kong API Gateway (Lua plugins) | Custom Axum middleware / tower Layer | Plugin architecture via middleware |

**Nginx config → Cargo.toml + main.rs:**

```nginx
# Lua/OpenResty: routing in nginx.conf
location /api/v1/users {
    content_by_lua_block {
        local users = require("handlers.users")
        users.handle_get()
    }
}
```

```rust
// Rust/Axum: routing in code
use axum::{routing::get, Router};

async fn handle_get_users(
    State(state): State<Arc<AppState>>,
    Query(params): Query<UserParams>,
) -> Result<Json<Vec<User>>, AppError> {
    let users = state.db.get_users(&params).await?;
    Ok(Json(users))
}

let app = Router::new()
    .route("/api/v1/users", get(handle_get_users))
    .with_state(app_state);
```

### String Pattern to Regex Translation

```rust
use regex::Regex;

// Lua: string.match("hello 123", "(%d+)")
// Rust:
fn lua_style_match(input: &str) -> Option<String> {
    let re = Regex::new(r"(\d+)").unwrap();
    re.captures(input)
        .and_then(|caps| caps.get(1))
        .map(|m| m.as_str().to_string())
}

// Lua: string.gsub("a-b-c", "%-", "_")  => "a_b_c"
// Rust:
fn lua_style_gsub(input: &str) -> String {
    let re = Regex::new(r"-").unwrap();
    re.replace_all(input, "_").to_string()
}
```

## Canonical Patterns

### Pattern 1: Module Definition

```rust
// Lua:
// local M = {}
// function M.hello(name)
//   return "Hello " .. name
// end
// return M

// Rust — in src/greeting.rs:
pub fn hello(name: &str) -> String {
    format!("Hello {name}")
}

// in src/lib.rs or src/main.rs:
// mod greeting;
// use greeting::hello;
```

### Pattern 2: Multi-Return Values

```rust
// Lua:
// local found, index = find_item(items, "target")

// Rust:
fn find_item(items: &[String], target: &str) -> (bool, Option<usize>) {
    items.iter()
        .position(|s| s == target)
        .map_or((false, None), |i| (true, Some(i)))
}
// let (found, index) = find_item(&items, "target");
```

### Pattern 3: Varargs to Generic Slices

```rust
// Lua:
// function sum(...)
//   local total = 0
//   for _, v in ipairs({...}) do total = total + v end
//   return total
// end

// Rust — using slices:
fn sum(args: &[i64]) -> i64 {
    args.iter().sum()
}

// variadic macro (true variadic):
macro_rules! sum {
    ($($x:expr),*) => {
        {
            let mut total = 0i64;
            $(total += $x;)*
            total
        }
    };
}
// let result = sum!(1, 2, 3, 4);
```

### Pattern 4: Metatable-Based OOP

```rust
// Lua:
// local Animal = {}
// function Animal:new(name)
//   local obj = { name = name }
//   setmetatable(obj, { __index = Animal })
//   return obj
// end
// function Animal:speak() return self.name .. " makes a sound" end

// Rust:
struct Animal {
    name: String,
}

impl Animal {
    fn new(name: impl Into<String>) -> Self {
        Self { name: name.into() }
    }

    fn speak(&self) -> String {
        format!("{} makes a sound", self.name)
    }
}

// inheritance via trait — Lua metatable __index delegation:
// Dog extends Animal
struct Dog {
    animal: Animal,
    breed: String,
}

impl std::ops::Deref for Dog {
    type Target = Animal;
    fn deref(&self) -> &Animal { &self.animal }
}
```

### Pattern 5: pcall / xpcall Error Handling

```rust
// Lua:
// local ok, result = pcall(risky_function)
// if not ok then log_error(result) end

// Rust:
fn risky_function() -> Result<String, MyError> {
    // ...
    Ok("result".into())
}

match risky_function() {
    Ok(result) => { /* 使用 result */ }
    Err(e) => tracing::error!("Operation failed: {e}"),
}

// Result combinators are equivalent to pcall chains:
let outcome = risky_function()
    .and_then(|val| another_op(&val))
    .map(|final_val| format!("processed: {final_val}"));
```

### Pattern 6: Closures Over Upvalues

```rust
// Lua:
// function counter()
//   local count = 0
//   return function() count = count + 1; return count end
// end

// Rust:
fn counter() -> impl FnMut() -> i32 {
    let mut count = 0;
    move || {           // move keyword captures ownership
        count += 1;
        count
    }
}
```

### Pattern 7: Iterator Generators

```rust
// Lua — stateless iterator:
// function range_iter(state, n)
//   if state > n then return nil end
//   return state + 1, state
// end
// for i, v in range_iter, {1, 10} do ... end

// Rust:
struct RangeIter {
    current: i32,
    end: i32,
}

impl Iterator for RangeIter {
    type Item = i32;
    fn next(&mut self) -> Option<Self::Item> {
        if self.current > self.end {
            None
        } else {
            let val = self.current;
            self.current += 1;
            Some(val)
        }
    }
}
```

## FFI & Incremental Migration

The most practical approach for migrating a large Lua codebase is to embed Lua in Rust via mlua, then incrementally rewrite modules to pure Rust.

| Strategy | Tool | When to Use |
|---|---|---|
| Embed Lua in Rust | `mlua` / `rlua` | Large existing Lua code; call Lua from Rust |
| Call Rust from Lua | mlua `create_function` / `UserData` | Replace hot paths first; keep glue code in Lua |
| Replace Lua C modules | `cc` crate + `bindgen` | Lua C library dependencies; wrap with safe Rust |
| Full rewrite | Pure Rust | Small codebase or clear performance motivation |
| LuaJIT FFI replacement | Rust FFI (`extern "C"`) | Direct C library calls; safer bindings |

### mlua Embedding Example

```rust
use mlua::{Lua, Function, Table, UserData, UserDataMethods};

struct Config { max_retries: u32, timeout_ms: u64 }

impl UserData for Config {
    fn add_methods<'lua, M: UserDataMethods<'lua, Self>>(methods: &mut M) {
        methods.add_method("get_timeout", |_, cfg, ()| Ok(cfg.timeout_ms));
    }
}

fn run_legacy_script() -> mlua::Result<()> {
    let lua = Lua::new();

    // expose Rust config to Lua
    let config = Config { max_retries: 3, timeout_ms: 5000 };
    lua.globals().set("config", config)?;

    // register Rust function for Lua to call
    let log_fn = lua.create_function(|_, msg: String| {
        tracing::info!("[lua] {msg}");
        Ok(())
    })?;
    lua.globals().set("log_info", log_fn)?;

    // execute legacy Lua script
    lua.load(r#"
        log_info("Starting with timeout: " .. config:get_timeout() .. "ms")
    "#).exec()?;

    Ok(())
}
```

### Building an Incremental Replacement Pipeline

1. Wrap the entire Lua application in mlua with Rust `main()` as entry point.
2. Profile and identify hot Lua functions. Rewrite them as Rust `create_function` callbacks.
3. Move business logic from Lua tables into Rust `UserData` structs.
4. Gradually replace `require` calls with Rust module imports.
5. Once all logic is in Rust, remove the mlua dependency (or keep it for plugin scripting).

## Common Mistakes

### Mistake 1: 1-Based to 0-Based Index Confusion

```rust
// WRONG — Lua programmers common 1-index habit:
fn get_first<T>(items: &[T]) -> &T {
    &items[1]  // runtime panic or wrong element
}

// CORRECT:
fn get_first<T>(items: &[T]) -> Option<&T> {
    items.first()  // Rust arrays and slices start at 0
}
```

### Mistake 2: Overusing Rc<RefCell<T>> — Lua GC Emulation

```rust
// WRONG — trying to emulate Lua global mutable table with Rc<RefCell<>>:
type Globals = Rc<RefCell<HashMap<String, Rc<RefCell<LuaValue>>>>>;

// CORRECT — use structured state management:
#[derive(Debug)]
struct AppState {
    config: Config,
    players: HashMap<String, Player>,
    // each module owns its own data
}
// use Arc<RwLock<T>> only where sharing is needed, not globally
```

### Mistake 3: String Concatenation in Hot Loops

```rust
// WRONG — imitating Lua .. operator:
let mut result = String::new();
for item in &items {
    result = result + &format!("{item},"); // allocates new String each time
}

// CORRECT — use writeln! or join:
let result = items.iter()
    .map(|s| s.as_str())
    .collect::<Vec<_>>()
    .join(",");
```

### Mistake 4: Using panic! Instead of Result for Recoverable Errors

```rust
// WRONG — imitating Lua error() pattern:
fn load_config(path: &str) -> Config {
    let content = std::fs::read_to_string(path).unwrap(); // panics if file missing
    serde_json::from_str(&content).unwrap()               // panics if parse fails
}

// CORRECT — return Result, like pcall:
fn load_config(path: &str) -> Result<Config, Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string(path)?;
    let config = serde_json::from_str(&content)?;
    Ok(config)
}
```

### Mistake 5: Assuming Nil/None Semantics in Collections

```rust
// WRONG — Lua tables allow gaps:
let mut items: Vec<Option<String>> = vec![None, None];
items[5] = Some("hello".into()); // index out of bounds panic

// CORRECT — use HashMap for sparse collections:
let mut items: HashMap<usize, String> = HashMap::new();
items.insert(5, "hello".into()); // safe and semantically correct
```

## Reference Implementations

| Project | Description | Migration Strategy |
|---|---|---|
| [StyLua](https://github.com/JohnnyMorganz/StyLua) | Lua formatter written in Rust | Full Rust implementation of a Lua tool; uses full-moon for parsing |
| [mlua](https://github.com/mlua-rs/mlua) | High-level Lua bindings for Rust | Embedding pattern — call Lua from Rust; `UserData` trait maps to Lua objects |
| [rlua](https://github.com/mlua-rs/rlua) | Safe high-level Lua bindings (predecessor to mlua) | Similar embedding approach |
| [full-moon](https://github.com/Kampfkarren/full-moon) | Lossless Lua parser in Rust | Parse Lua AST; useful for migrating Lua DSLs to Rust-native parsers |
| [LuaJIT-remake](https://github.com/bombela/luajit-remake) | LuaJIT concepts implemented in Rust | Reference for understanding LuaJIT internals in Rust |
| [Roblox-ts](https://github.com/roblox-ts/roblox-ts) | TypeScript-to-Luau compiler | Type-first approach to Lua; similar migration philosophy |

## Cross-Reference

- **c-to-rust** — For migrating Lua C modules and FFI patterns
- **nodejs-to-rust** — For event-loop and async patterns common to both Lua and JS
- **go-to-rust** — For concurrent goroutine-to-tokio patterns similar to coroutine migration
- **zig-to-rust** — For low-level memory patterns relevant to LuaJIT FFI replacement
