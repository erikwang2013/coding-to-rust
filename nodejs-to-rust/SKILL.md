---
name: nodejs-to-rust
description: Use when migrating Node.js/TypeScript codebases to Rust — covers event-loop to tokio, Express/Fastify to Axum, npm to Cargo, Promise to Future, and incremental replacement strategy. Includes canonical code patterns, common mistakes, and reference implementations.
updated: 2026-07-30
---

# Node.js / TypeScript to Rust Migration

## Architecture Mapping

Node.js applications run on a single-threaded event loop (libuv) with asynchronous, non-blocking I/O. The V8 engine interprets JavaScript and relies on worker threads for CPU-bound work. Rust's equivalent is the `tokio` async runtime, which provides a multi-threaded work-stealing scheduler with the same async/await syntax Node.js developers already know. The conceptual gap is smaller than most expect: both use `async`/`await`, both are non-blocking by default, and both offer process-level parallelism via separate OS threads.

A Node.js project:

```text
src/
  index.ts
  routes/
    users.ts
    products.ts
  middleware/
    auth.ts
  services/
    db.ts
package.json
tsconfig.json
```

becomes:

```text
src/
  main.rs
  routes/
    mod.rs
    users.rs
    products.rs
  middleware/
    mod.rs
    auth.rs
  services/
    mod.rs
    db.rs
Cargo.toml
```

The critical architectural difference: Node.js naturally allows shared mutable state (closures, module-level variables) because JavaScript's event loop guarantees no concurrent access. Rust requires explicit synchronization (`Arc<Mutex<T>>`, channels) when state is shared across concurrent tasks. This is the single biggest mindset shift for Node.js developers.

## Type System Mapping

| TypeScript Type | Rust Type | Notes |
|-----------------|-----------|-------|
| `string` | `String` / `&str` | `String` is owned (like JS string); `&str` is borrowed |
| `number` | `i32` / `u32` / `f64` (explicit) | No unified number type; choose width and signedness |
| `bigint` | `i64` / `u64` / `i128` | Fixed-width, not arbitrary precision (use `num-bigint` crate if needed) |
| `boolean` | `bool` | Direct mapping |
| `null` / `undefined` | `None` (via `Option<T>`) | No `undefined`; `Option::None` covers both |
| `void` | `()` (unit type) | `()` is a real value, not "nothing" |
| `never` | `!` (never type, unstable) | Use `std::convert::Infallible` or a custom empty enum |
| `unknown` | `Box<dyn Any>` (type-erased) | Prefer enum or trait objects over type erasure |
| `any` | Avoid: use generics or `serde_json::Value` | `any` disables type checking; Rust requires static typing |
| `Array<T>` / `T[]` | `Vec<T>` (owned) / `&[T]` (borrowed) | Vec is growable; slices are borrowed views |
| `ReadonlyArray<T>` | `&[T]` | Immutable borrowed slice |
| `[T, T]` (tuple) | `(T, T)` | Fixed-length heterogeneous tuple |
| `Record<K, V>` | `HashMap<K, V>` / `BTreeMap<K, V>` | Ordered or hashed map |
| `Map<K, V>` | `HashMap<K, V>` (or `BTreeMap`) | Same semantics |
| `Set<T>` | `HashSet<T>` (or `BTreeSet`) | Unique, unordered collection |
| `Partial<T>` | `Option` wrapping each field, or builder pattern | Rust has no utility type; use `derive_builder` crate |
| `Pick<T, K>` / `Omit<T, K>` | Manual struct composition; No direct equivalent | Use separate structs or derive macros |
| `A \| B` (union type) | `enum { A(A), B(B) }` | Rust enums are discriminated unions |
| `A & B` (intersection type) | Trait composition `T: TraitA + TraitB`, or newtype | Rare pattern in Rust; prefer single struct or trait |
| `enum` (TS literal union) | `enum` | Rust enums are more powerful; can carry data |
| `interface` / `type` | `trait` for behavior; `struct` for data | Traits define shared behavior; structs hold state |
| `class` | `struct` + `impl` block | No inheritance; use composition + trait delegation |
| `Promise<T>` | `Future<Output = T>` or `impl Future<Output = T>` | Same async concept; Rust futures are lazy (poll-based) |
| `AsyncIterable<T>` | `futures::Stream<Item = T>` or `tokio_stream::Stream` | Async iteration; use `while let Some(item) = stream.next().await` |
| `Buffer` | `Vec<u8>` / `bytes::Bytes` | `bytes` crate for zero-copy buffer operations |
| `ReadableStream` | `tokio::io::AsyncRead` / `AsyncBufRead` | Async byte stream consumption |
| `WritableStream` | `tokio::io::AsyncWrite` | Async byte stream production |
| `Error` (throw) | `Result<T, E>` + `anyhow::Error` for type-erased errors | No try/catch; errors are values via Result |
| `Symbol` | Enum variants or marker types | No runtime-unique symbol primitive |
| `JSON` (dynamic object) | `serde_json::Value` for dynamic; `serde::Deserialize` for typed | Parse once into typed structs for performance |
| `Date` | `chrono::DateTime<Utc>` / `time::OffsetDateTime` | Use `chrono` or `time` crate |
| `RegExp` | `regex::Regex` | Compile regex once; `Regex::new(...).unwrap()` |

## Memory & Ownership Model

Node.js's single-threaded event loop means that module-level variables, closures, and object properties are inherently safe -- only one function runs at a time. Rust's `tokio` is multi-threaded by default, so any state shared across `.await` points must be explicitly synchronized.

| Node.js Pattern | Rust Pattern | Notes |
|-----------------|-------------|-------|
| Module-level `let counter = 0;` | `Arc<AtomicU64>` for numeric counters; `Arc<Mutex<T>>` for complex state | Multi-threaded runtime requires thread-safe state |
| Closure capturing outer variables | Closure + `Arc` if shared; ownership transfer via `move` | Rust closures capture by reference or by move; explicit control |
| `let cache = {}` (object as cache) | `Arc<DashMap<K, V>>` for concurrent map; `moka::Cache` for TTL-based | Concurrency-safe collections from `dashmap` or `moka` crates |
| `new Map()` shared across requests | `Arc<tokio::sync::RwLock<HashMap<K, V>>>` if read-heavy; `Arc<Mutex<HashMap>>` otherwise | Choose lock based on read/write ratio |
| `class Service { field: Data }` instance state | Actor pattern via `tokio::spawn` + `mpsc` channels; or `Arc<Mutex<State>>` | Actor pattern avoids locks entirely for complex workflows |
| `process.env` variables | `std::env::var("KEY")` or `dotenv` crate at startup; `OnceLock` | Load once at startup; no hot-reload by default |
| `global` object | `lazy_static!` or `OnceLock` | Explicit global state registry; discouraged unless truly needed |

## Concurrency / Async Translation

| Node.js Pattern | Rust Pattern | Notes |
|-----------------|-------------|-------|
| `async function` | `async fn` | Identical syntax; Rust futures are lazy until `.await`ed |
| `await promise` | `.await` | Suspension point; postfix syntax |
| `Promise.all([a, b])` | `tokio::join!(a, b)` or `futures::future::join_all(iter)` | Concurrent execution of multiple futures |
| `Promise.race([a, b])` | `tokio::select! { result = a => ..., result = b => ... }` | First-to-complete wins |
| `Promise.any([a, b])` | `futures::future::select_ok(iter)` | First successful future wins |
| `new Promise((resolve, reject) => ...)` | `async { ... }` block or `tokio::sync::oneshot::channel()` | Manual future via channel |
| `setTimeout(fn, ms)` | `tokio::time::sleep(Duration::from_millis(ms)); fn()` | Async sleep + call |
| `setInterval(fn, ms)` | `tokio::time::interval(Duration::from_millis(ms))` into loop `tick().await` | Recurring timer as Stream |
| `process.nextTick(fn)` | `tokio::spawn(async { fn() })` or `tokio::task::yield_now().await` | Defer to next microtask |
| `EventEmitter` | `tokio::sync::broadcast` (fan-out) or `tokio::sync::mpsc` (point-to-point) | Broadcast for one-to-many; mpsc for one-to-one |
| `stream.Readable` | `tokio::io::AsyncRead` / `tokio_stream::StreamExt` | Async data source |
| `stream.Writable` | `tokio::io::AsyncWrite` | Async data sink |
| `stream.Transform` | Pipe via `tokio::io::copy` or custom adapter | Transform stream as async function |
| `stream.pipeline(src, transform, dst)` | `tokio::io::copy(&mut src, &mut dst)` or manual piping | Stream composition |
| `worker_threads` | `std::thread::spawn` or `tokio::task::spawn_blocking` for CPU work | Dedicated OS thread pool for blocking ops |
| `cluster.fork()` | Multiple processes managed by systemd/docker; `tokio::spawn` for task-level | No built-in cluster; use process-per-core + reverse proxy |
| `child_process.exec` | `std::process::Command` | Spawn and capture output |
| `child_process.spawn` | `std::process::Command::new("cmd").spawn()` | Streaming subprocess I/O |

## Build System & Dependencies

| Node.js Tool / Concept | Rust Equivalent | Notes |
|------------------------|-----------------|-------|
| `package.json` | `Cargo.toml` | Declarative manifest for deps, scripts, metadata |
| `npm install` / `yarn` / `pnpm` | `cargo build` (auto-fetches deps) | Cargo is both package manager and build system |
| `node_modules/` | `~/.cargo/registry/` (global cache) | No project-local copy; Cargo caches globally |
| `npm run build` | `cargo build` / `cargo build --release` | Compile debug or release artifact |
| `npm run start` | `cargo run` | Build + execute binary |
| `npm run test` | `cargo test` | Runs `#[test]` annotated functions |
| `npm run lint` | `cargo clippy` | Linter with 500+ rules |
| `npm run format` | `cargo fmt` | Automatic code formatting (opinionated, no config) |
| `import { foo } from './bar'` | `mod bar; use bar::foo;` | Module system; declaration in parent, definition in child file |
| `import foo from 'some-package'` | `use some_package::foo;` (auto-resolved by Cargo) | Cargo resolves crate names from Cargo.toml |
| `require('./foo')` | `mod foo;` (declared in lib.rs/main.rs) | Module must be declared at crate root |
| `module.exports = { ... }` | `pub fn` / `pub struct` / `pub use` | Visibility via `pub` keyword |
| `tsconfig.json` | `Cargo.toml` profiles + `.cargo/config.toml` | Compiler configuration |
| `.env` + `dotenv` | `dotenv` crate (same name) | `.env` file parsing |
| `nodemon --watch src` | `cargo watch -x run` (via `cargo-watch` crate) | Auto-reload on file change |
| `cross-env NODE_ENV=production` | Environment variable defaults in `build.rs` or `cfg` attributes | No cross-platform env wrapper needed |
| `node-gyp` (native addons) | `cc` crate in `build.rs` | Compile C/C++ dependencies |
| `nvm` (Node version manager) | `rustup` | Toolchain version manager |
| `jest` / `mocha` / `vitest` | Built-in `#[test]` + `cargo test` | Standard test runner; `tokio::test` for async tests |
| `chai` / `expect` assertions | Built-in `assert!` / `assert_eq!` / `assert_ne!` | Macro-based assertions; no chaining API |
| `nyc` / `istanbul` coverage | `cargo-tarpaulin` / `grcov` | Code coverage with source-based instrumentation |

## Standard Library & Ecosystem Mapping

### Core APIs

| Node.js API | Rust Equivalent | Notes |
|-------------|-----------------|-------|
| `console.log(x)` | `println!("{x}")` | Format string with `{}` |
| `console.error(x)` | `eprintln!("{x}")` | Standard error output |
| `console.table(arr)` | Custom: iterate + print with padding | No built-in table printer |
| `JSON.stringify(obj)` | `serde_json::to_string(&obj)?` | Returns `Result<String>` |
| `JSON.parse(str)` | `serde_json::from_str::<T>(&str)?` | Type must be known at compile time |
| `JSON.parse(str)` (dynamic) | `serde_json::from_str::<Value>(&str)?` | `Value` is the dynamic JSON enum |
| `typeof x` | Compile-time: `std::any::type_name::<T>()` | Only for debugging; no runtime type reflection |
| `instanceof` | `std::any::Any::downcast_ref::<T>()` | Limited to `dyn Any` trait objects |
| `Array.isArray(x)` | `x.is_empty()` on slices; always known at compile time | Type system makes runtime checks unnecessary |
| `Number.parseInt` / `parseFloat` | `s.parse::<i32>()` / `s.parse::<f64>()` | Returns `Result`, not `NaN` |
| `Number.isNaN(x)` | `x.is_nan()` (for `f32`/`f64` only) | NaN is a float concept |
| `Math.random()` | `rand::random::<f64>()` | From the `rand` crate |
| `Math.floor` / `Math.ceil` / `Math.round` | `x.floor()` / `x.ceil()` / `x.round()` | Methods on `f32`/`f64` |
| `Math.max(a, b)` | `a.max(b)` | Method, not static; works on all `Ord` types |
| `Math.min(a, b)` | `a.min(b)` | Same |
| `String.prototype.length` | `s.len()` (bytes for `str`; chars count via `s.chars().count()`) | Rust String is UTF-8; `.len()` gives byte count |
| `String.prototype.includes` | `s.contains(needle)` | Same semantics |
| `String.prototype.startsWith` | `s.starts_with(prefix)` | Same |
| `String.prototype.endsWith` | `s.ends_with(suffix)` | Same |
| `String.prototype.split` | `s.split(delim).collect::<Vec<_>>()` | Returns iterator; `.collect()` to materialize |
| `String.prototype.trim` | `s.trim()` / `s.trim_start()` / `s.trim_end()` | Same |
| `String.prototype.toUpperCase` | `s.to_uppercase()` | Unicode-aware |
| `String.prototype.toLowerCase` | `s.to_lowercase()` | Unicode-aware |
| `String.prototype.replace` | `s.replace(from, to)` or `regex::Regex::replace_all` | Simple or regex replacement |
| `String.prototype.slice` | `&s[0..5]` (byte-indexed; use `char_indices` for char boundaries) | Byte-indexed, not char-indexed; respect UTF-8 boundaries |
| `Array.prototype.push` | `vec.push(item)` | Same |
| `Array.prototype.pop` | `vec.pop()` returns `Option<T>` | No `undefined`; `None` on empty |
| `Array.prototype.map(fn)` | `iter.map(|x| f(x)).collect()` | Lazy + collect |
| `Array.prototype.filter(fn)` | `iter.filter(|x| pred(x)).collect()` | Lazy + collect |
| `Array.prototype.reduce(fn, init)` | `iter.fold(init, \|acc, x\| f(acc, x))` | Same semantics |
| `Array.prototype.find(fn)` | `iter.find(\|x\| pred(x))` returns `Option<&T>` | No `undefined` on failure |
| `Array.prototype.some(fn)` | `iter.any(\|x\| pred(x))` | Same |
| `Array.prototype.every(fn)` | `iter.all(\|x\| pred(x))` | Same |
| `Array.prototype.join(sep)` | `vec.join(sep)` (for `Vec<String>`) or `itertools::Itertools::join` | `itertools` crate for iterator-level join |
| `Array.prototype.sort(fn?)` | `vec.sort()` / `sort_by(\|a, b\| a.cmp(b))` | In-place stable sort |
| `Array.prototype.slice` | `&vec[1..3]` | Returns `&[T]` slice |
| `Array.prototype.concat` | `[a, b].concat()` or `vec.extend(other)` | Extend mutable; concat for new |

### File System

| Node.js `fs` API | Rust Equivalent | Notes |
|------------------|-----------------|-------|
| `fs.readFileSync(path)` | `std::fs::read(path)?` returns `Vec<u8>`; `read_to_string` for UTF-8 | Synchronous by default; use `tokio::fs::read` for async |
| `fs.readFile(path, cb)` | `tokio::fs::read(path).await?` | Async read via tokio |
| `fs.writeFileSync(path, data)` | `std::fs::write(path, data)?` | Synchronous write |
| `fs.writeFile(path, data, cb)` | `tokio::fs::write(path, data).await?` | Async write |
| `fs.existsSync(path)` | `path.try_exists()?` | Returns `Result<bool>` |
| `fs.mkdirSync(path, {recursive: true})` | `std::fs::create_dir_all(path)?` | Recursive directory creation |
| `fs.readdirSync(path)` | `std::fs::read_dir(path)?` returns iterator of `DirEntry` | Iterator yields `Result<DirEntry>` |
| `fs.statSync(path)` | `std::fs::metadata(path)?` | File metadata (size, permissions, type) |
| `fs.unlinkSync(path)` | `std::fs::remove_file(path)?` | Delete a file |
| `fs.rmSync(path, {recursive: true})` | `std::fs::remove_dir_all(path)?` | Recursive directory deletion |
| `fs.renameSync(old, new)` | `std::fs::rename(old, new)?` | Same |
| `fs.copyFileSync(src, dst)` | `std::fs::copy(src, dst)?` | Same |
| `fs.watch(path, cb)` | `notify` crate with `Watcher` | Cross-platform filesystem events |
| `fs.createReadStream(path)` | `tokio::fs::File::open(path).await?` | Async file handle with `AsyncRead` |
| `fs.createWriteStream(path)` | `tokio::fs::File::create(path).await?` | Async file handle with `AsyncWrite` |

### HTTP / Networking

| Node.js HTTP | Rust Equivalent | Notes |
|-------------|-----------------|-------|
| `http.createServer((req, res) => ...)` | `axum::Router` or `actix_web::App` | High-level routing framework |
| `app.get('/path', handler)` (Express) | `Router::new().route("/path", get(handler))` (axum) | Route definition |
| `app.use(middleware)` | `Router::layer(middleware)` or `Router::route_layer(...)` | Middleware composition |
| `req.params` (URL params) | `Path(params): Path<ParamsStruct>` (axum extractor) | Type-safe path parameter extraction |
| `req.query` | `Query(params): Query<ParamsStruct>` (axum extractor) | Type-safe query string |
| `req.body` (JSON body) | `Json(body): Json<BodyStruct>` (axum extractor) | Auto-deserialized JSON body |
| `req.headers` | `HeaderMap` extractor or `TypedHeader` | Header extraction |
| `res.json(obj)` | `Json(response)` return type | Auto-serialized JSON response |
| `res.status(code)` | `(StatusCode::CREATED, Json(response))` tuple return | Status + body combination |
| `res.sendStatus(code)` | `StatusCode::OK` return | Bare status code |
| `res.setHeader(key, val)` | `([(header::SET_COOKIE, "val")], body)` tuple response | Header injection via tuple |
| `next()` (middleware chain) | `axum::middleware::from_fn` with `next.run(req).await` | Middleware continuation |
| `http.request(url, options, cb)` | `reqwest::Client` or `hyper::Client` | HTTP client |
| `fetch(url)` | `reqwest::get(url).await?.text().await?` | Standard async HTTP |
| `new WebSocket(url)` | `tokio_tungstenite` or `axum::extract::ws` | WebSocket client/server |
| `net.createServer(socket => ...)` | `tokio::net::TcpListener` | Raw TCP server |
| `dgram.createSocket('udp4')` | `tokio::net::UdpSocket` | UDP socket |
| `tls.createServer(options, cb)` | `tokio_rustls::TlsAcceptor` | TLS via rustls |
| `https.request(url, cb)` | `reqwest::Client` (built-in TLS via `rustls` or `native-tls`) | HTTPS client |

### Process & OS

| Node.js API | Rust Equivalent | Notes |
|-------------|-----------------|-------|
| `process.env.PATH` | `std::env::var("PATH")?` | Returns `Result<String, VarError>` |
| `process.argv` | `std::env::args()` (iterator) or `clap` for CLI parsing | Use `clap` crate for declarative CLI definition |
| `process.cwd()` | `std::env::current_dir()?` | Returns `Result<PathBuf>` |
| `process.chdir(path)` | `std::env::set_current_dir(path)?` | Changes process cwd |
| `process.exit(code)` | `std::process::exit(code)` | Immediate process termination |
| `process.pid` | `std::process::id()` | Process ID |
| `process.platform` | `std::env::consts::OS` | Compile-time OS constant |
| `process.arch` | `std::env::consts::ARCH` | Compile-time architecture |
| `process.memoryUsage()` | `jemalloc_ctl` crate or `/proc/self/status` (Linux) | Memory stats not in std |
| `process.uptime()` | `std::time::Instant::now()` relative to program start | Monotonic timer |
| `os.homedir()` | `dirs::home_dir()` or `std::env::var("HOME")` | Home directory path |
| `os.tmpdir()` | `std::env::temp_dir()` | Temporary directory |
| `os.cpus()` | `num_cpus::get()` crate | CPU count |
| `os.freemem()` | `sysinfo` crate | System memory information |

### Ecosystem Crate Mapping

| Node.js Library | Rust Crate | Notes |
|---------------|-----------|-------|
| Express / Koa / Fastify | `axum` | Route-first, async handlers, tower middleware |
| NestJS | `actix-web` (full-featured) | Decorator pattern becomes derive macros |
| `node-fetch` / `axios` / `undici` | `reqwest` | Async HTTP client with TLS |
| `ws` / `socket.io` | `tokio-tungstenite` / `axum::extract::ws` | WebSocket client/server |
| `jsonwebtoken` | `jsonwebtoken` crate | Same name, same API shape |
| `bcrypt` / `argon2` | `bcrypt` / `argon2` crate | Password hashing |
| `dotenv` | `dotenvy` crate | `.env` file parsing |
| `winston` / `pino` | `tracing` + `tracing-subscriber` | Structured, span-based logging |
| `pg` / `mysql2` | `sqlx` (async, compile-time SQL verification) | Type-safe SQL |
| `better-sqlite3` | `rusqlite` (sync) / `sqlx` with sqlite feature | SQLite bindings |
| `prisma` | `diesel` (sync) / `sea-orm` (async) | Schema-first ORM |
| `mongoose` (MongoDB) | `mongodb` crate | Official driver |
| `ioredis` / `redis` | `redis` crate | Async Redis client |
| `graphql-yoga` / `apollo-server` | `async-graphql` | Code-first GraphQL |
| `jest` / `vitest` / `mocha` | `#[test]` (built-in) + `mockall` crate | Test framework + mocking |
| `multer` (file upload) | `axum::extract::Multipart` | Multipart form parsing |
| `passport` / `next-auth` | `tower-http` auth + `oauth2` crate | Authentication middleware |
| `bull` / `bullmq` (job queue) | Background `tokio::spawn` + Redis or `apalis` crate | Job processing |
| `commander` / `yargs` | `clap` crate (derive mode) | CLI argument parsing |
| `sharp` (image processing) | `image` crate | Image decode, resize, encode |
| `zod` / `yup` (validation) | `validator` crate | Derive-based struct validation |
| `puppeteer` / `playwright` | `headless_chrome` / `chromiumoxide` | Browser automation |
| `cron` / `node-cron` | `tokio-cron-scheduler` | Scheduled task execution |
| `nodemailer` | `lettre` crate | SMTP email sending |
| `compression` (Express) | `tower-http::compression` | Response compression middleware |
| `cors` (Express) | `tower-http::cors` | CORS middleware |
| `helmet` | `tower-http` security headers + custom middleware | Security headers |
| `rate-limiter-flexible` | `tower::limit::RateLimitLayer` or `governor` crate | Rate limiting |
| `express-session` | `tower-sessions` crate (for axum) | Session management |
| `swagger-jsdoc` | `utoipa` crate (code-first OpenAPI) | API documentation |
| `json-schema-to-typescript` | `schemars` crate | JSON Schema generation |

## Canonical Patterns

### Pattern 1: Express Route → Axum Handler

**Node.js (Express):**
```typescript
// Express route: middleware chain + request handler
import express from 'express';

const app = express();
app.use(express.json());

app.get('/api/users/:id', async (req, res) => {
  const { id } = req.params;
  const user = await db.findUser(parseInt(id));
  if (!user) {
    return res.status(404).json({ error: 'not found' });
  }
  res.json({ data: user });
});

app.listen(3000);
```

**Rust (Axum):**
```rust
// Axum route: type-safe extractors + async handler
use axum::{
    extract::{Path, State},
    http::StatusCode,
    routing::get,
    Json, Router,
};
use serde::Serialize;

#[derive(Serialize)]
struct UserResponse {
    data: UserDto,
}

async fn get_user(
    Path(id): Path<u32>,                    // type-safe path parameter extraction
    State(db): State<Arc<DbPool>>,          // shared state extraction
) -> Result<Json<UserResponse>, StatusCode> {
    let user = db.find_user(id).await.ok_or(StatusCode::NOT_FOUND)?;
    Ok(Json(UserResponse { data: user.into() }))
}

#[tokio::main]
async fn main() {
    let db = Arc::new(DbPool::connect().await.unwrap());

    let app = Router::new()
        .route("/api/users/:id", get(get_user))
        .with_state(db);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### Pattern 2: Promise.all → tokio::join!

**Node.js:**
```typescript
// Promise.all: concurrently await multiple async operations
const [user, posts, settings] = await Promise.all([
    db.findUser(id),
    db.findPosts(id),
    db.getSettings(id),
]);
```

**Rust:**
```rust
// tokio::join!: execute multiple futures concurrently
let user_fut = db.find_user(id);
let posts_fut = db.find_posts(id);
let settings_fut = db.get_settings(id);

let (user, posts, settings) = tokio::join!(user_fut, posts_fut, settings_fut);
let user = user?;
let posts = posts?;
let settings = settings?;

// dynamic number of concurrent futures (like Promise.all with arrays):
let futures: Vec<_> = ids.iter()
    .map(|id| db.find_user(*id))
    .collect();
let results: Vec<Result<User, _>> = futures::future::join_all(futures).await;
```

### Pattern 3: EventEmitter → Channel / Broadcast

**Node.js:**
```typescript
// EventEmitter: one-to-many event notification
import { EventEmitter } from 'events';

const bus = new EventEmitter();

// subscribe
bus.on('order:created', (order) => {
  sendEmail(order);
});
bus.on('order:created', (order) => {
  updateInventory(order);
});

// publish
bus.emit('order:created', { id: 42, total: 99.99 });
```

**Rust:**
```rust
// tokio::sync::broadcast (one-to-many, equivalent to EventEmitter)
use tokio::sync::broadcast;

let (tx, _) = broadcast::channel::<Order>(32);

// subscriber 1
let mut rx1 = tx.subscribe();
tokio::spawn(async move {
    while let Ok(order) = rx1.recv().await {
        send_email(&order).await;
    }
});

// subscriber 2
let mut rx2 = tx.subscribe();
tokio::spawn(async move {
    while let Ok(order) = rx2.recv().await {
        update_inventory(&order).await;
    }
});

// publish
tx.send(Order { id: 42, total: 99.99 }).unwrap();
```

### Pattern 4: Middleware → Tower Layer

**Node.js (Express middleware):**
```typescript
// Express middleware: auth + logging
function authMiddleware(req, res, next) {
  const token = req.headers.authorization;
  if (!token) return res.status(401).json({ error: 'unauthorized' });
  req.user = verifyToken(token);
  next();
}

app.use(authMiddleware);
app.use(logger);
```

**Rust (Axum / Tower):**
```rust
// Tower middleware: layered composition via Service trait
use axum::{
    middleware,
    http::{Request, StatusCode},
    response::Response,
};

// auth middleware
async fn auth_middleware<B>(
    mut req: Request<B>,
    next: middleware::Next<B>,
) -> Result<Response, StatusCode> {
    let token = req.headers()
        .get("authorization")
        .ok_or(StatusCode::UNAUTHORIZED)?;
    let user = verify_token(token).map_err(|_| StatusCode::UNAUTHORIZED)?;
    req.extensions_mut().insert(user);  // inject user info into request extensions
    Ok(next.run(req).await)
}

let app = Router::new()
    .route("/api/*", get(handler))
    .layer(middleware::from_fn(auth_middleware))
    // Tower Trace layer for request/response logging:
    .layer(tower_http::trace::TraceLayer::new_for_http());
```

### Pattern 5: Class + Dependency Injection → Struct + Trait

**Node.js (class-based service):**
```typescript
class UserService {
  constructor(private db: Database, private email: EmailService) {}

  async register(data: RegisterDto): Promise<User> {
    const user = await this.db.users.create(data);
    await this.email.sendWelcome(user.email);
    return user;
  }
}
```

**Rust:**
```rust
// trait defines dependency interface (struct holds concrete impl or Arc<dyn Trait>)
#[async_trait]
pub trait EmailService: Send + Sync {
    async fn send_welcome(&self, email: &str) -> Result<(), EmailError>;
}

pub struct UserService {
    db: Arc<DbPool>,
    email: Arc<dyn EmailService>,  // dependency injection
}

impl UserService {
    pub async fn register(&self, data: RegisterDto) -> Result<User, AppError> {
        let user = self.db.create_user(data).await?;
        self.email.send_welcome(&user.email).await?;
        Ok(user)
    }
}
```

### Pattern 6: JSON Handling → Serde

**Node.js:**
```typescript
// dynamic JSON + manual type guard
const data = JSON.parse(rawString);
if (typeof data.name === 'string' && typeof data.age === 'number') {
  const user: User = data as User;
}
```

**Rust:**
```rust
use serde::{Deserialize, Serialize};

// Option A: compile-time type-safe deserialization (recommended)
#[derive(Deserialize, Serialize)]
struct User {
    name: String,
    age: u32,
    #[serde(default)]
    email: Option<String>,
}

let user: User = serde_json::from_str(&raw_string)?;
// types verified at compile time, no runtime guards needed

// Option B: dynamic JSON (like JS arbitrary object access)
let value: serde_json::Value = serde_json::from_str(&raw_string)?;
if let (Some(name), Some(age)) = (
    value["name"].as_str(),
    value["age"].as_u64(),
) {
    println!("{name} is {age} years old");
}
```

### Pattern 7: Stream Processing → AsyncRead/AsyncWrite

**Node.js:**
```typescript
// Transform stream: convert lines to uppercase
import { createReadStream, createWriteStream } from 'fs';
import { Transform } from 'stream';

const upperCase = new Transform({
  transform(chunk, encoding, callback) {
    callback(null, chunk.toString().toUpperCase());
  }
});

createReadStream('input.txt')
  .pipe(upperCase)
  .pipe(createWriteStream('output.txt'));
```

**Rust:**
```rust
use tokio::fs::File;
use tokio::io::{self, AsyncBufReadExt, AsyncWriteExt, BufReader, BufWriter};

async fn transform_file(input: &str, output: &str) -> io::Result<()> {
    let file = File::open(input).await?;
    let reader = BufReader::new(file);
    let mut lines = reader.lines();

    let out = File::create(output).await?;
    let mut writer = BufWriter::new(out);

    while let Some(line) = lines.next_line().await? {
        writer.write_all(line.to_uppercase().as_bytes()).await?;
        writer.write_all(b"\n").await?;
    }
    writer.flush().await?;
    Ok(())
}
```

## FFI & Incremental Migration

### Migration Strategy: Strangler Fig (Route-by-Route)

For a running Node.js service, the most practical approach is the **strangler fig pattern**: deploy a reverse proxy (Nginx/Envoy) in front of both Node.js and Rust servers, then route individual endpoints from Node.js to Rust over time.

| Stage | Node.js | Rust | Proxy Rules |
|-------|---------|------|-------------|
| 1 | All routes | None | All traffic to Node.js |
| 2 | Most routes | `/api/health`, `/api/metrics` | New endpoints on Rust |
| 3 | Core routes | `/api/users`, `/api/products` | Business logic migrating |
| 4 | Auth middleware, payment | All data endpoints | Performance-critical routes on Rust |
| 5 | Static file serving only | Entire API surface | Node.js retired or kept for admin UI |
| 6 | None | Full monolith in Rust | Direct to Rust |

### Calling Rust from Node.js (via NAPI-RS)

```rust
// use napi-rs to create Node.js native addon
use napi_derive::napi;
use napi::bindgen_prelude::*;

#[napi(object)]
pub struct ParseResult {
    pub valid: bool,
    pub tokens: Vec<String>,
}

#[napi]
pub fn parse_markdown(input: String) -> Result<ParseResult> {
    let tokens = markdown_parser::tokenize(&input)
        .map_err(|e| napi::Error::from_reason(e.to_string()))?;
    Ok(ParseResult { valid: true, tokens })
}
```

```typescript
// Node.js side calling Rust native addon
import { parseMarkdown } from 'my-rust-parser';
const result = parseMarkdown('# Hello\nWorld');
console.log(result.tokens);  // ["#", " ", "Hello", "\n", "World"]
```

### Sidecar Pattern (JSON-over-stdin/stdout)

```rust
// Rust side: read JSON from stdin, process, write to stdout
use std::io::{self, BufRead, Write};

fn main() -> io::Result<()> {
    let stdin = io::stdin();
    for line in stdin.lock().lines() {
        let input: serde_json::Value = serde_json::from_str(&line?)?;
        let output = process(&input);
        println!("{}", serde_json::to_string(&output).unwrap());
        io::stdout().flush()?;
    }
    Ok(())
}
```

```typescript
// Node.js side: spawn Rust binary, communicate via JSON lines
const { spawn } = require('child_process');
const child = spawn('./rust-processor');

child.stdin.write(JSON.stringify({ action: 'transform', data: 'hello' }) + '\n');
child.stdout.on('data', (data) => {
  const result = JSON.parse(data.toString());
  console.log(result);
});
```

## Common Mistakes

### Mistake 1: Holding a `MutexGuard` Across an `.await` Point

**Wrong:**
```rust
// WRONG: holding a lock across .await -- std Mutex deadlocks, tokio Mutex panics
async fn checkout(pool: &PgPool, cart_id: i32) -> Result<Order, Error> {
    let mut cart = CART_CACHE.lock().unwrap();
    let items = cart.remove(&cart_id);

    let order = create_order(pool, items).await?;  // holding lock during .await!
    Ok(order)
}
```

**Right:**
```rust
// CORRECT: scope the lock to sync code only, no .await across it
async fn checkout(pool: &PgPool, cart_id: i32) -> Result<Order, Error> {
    let items = {
        let mut cart = CART_CACHE.lock().unwrap();
        cart.remove(&cart_id)
    };  // lock released here

    let order = create_order(pool, items).await?;  // no lock during .await
    Ok(order)
}
```

### Mistake 2: Returning `anyhow::Error` from Library Code

**Wrong:**
```rust
// WRONG: type-erased errors in library code
pub fn parse_config(path: &str) -> anyhow::Result<Config> {
    let content = std::fs::read_to_string(path)?;
    let config: Config = serde_json::from_str(&content)?;
    Ok(config)
}
// caller cannot distinguish file error from parse error
```

**Right:**
```rust
// CORRECT: structured error enum for library code (thiserror)
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ConfigError {
    #[error("failed to read config file '{path}': {source}")]
    Io { path: String, #[source] source: std::io::Error },
    #[error("invalid config format: {source}")]
    Parse { #[source] source: serde_json::Error },
}

pub fn parse_config(path: &str) -> Result<Config, ConfigError> {
    let content = std::fs::read_to_string(path)
        .map_err(|e| ConfigError::Io { path: path.to_string(), source: e })?;
    let config = serde_json::from_str(&content)
        .map_err(|e| ConfigError::Parse { source: e })?;
    Ok(config)
}
```

### Mistake 3: Using `panic!` / `unwrap()` for Expected Errors

**Wrong:**
```rust
// WRONG: using panic for business errors (no equivalent in JS)
fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("division by zero");  // Node.js returns Infinity; panic is unrecoverable
    }
    a / b
}
```

**Right:**
```rust
// CORRECT: return Result, let caller decide how to handle the error
fn divide(a: i32, b: i32) -> Result<i32, &'static str> {
    if b == 0 { Err("division by zero") } else { Ok(a / b) }
}

// caller can choose how to handle:
match divide(10, 0) {
    Ok(result) => println!("{result}"),
    Err(e) => eprintln!("error: {e}"),
}
```

### Mistake 4: Forgetting That `.len()` on Strings Returns Byte Count, Not Character Count

**Wrong:**
```rust
// WRONG: assuming .len() returns character count (JS habit)
let s = "你好";
assert_eq!(s.len(), 2);  // actually 6 (each CJK char is 3 bytes in UTF-8)
```

**Right:**
```rust
// CORRECT: distinguish byte count from char count
let s = "你好";
assert_eq!(s.len(), 6);           // byte count
assert_eq!(s.chars().count(), 2); // char count

// beware UTF-8 boundaries when slicing:
let slice = &s[..3];  // take first 3 bytes -> captures first char completely
// &s[..2] would panic, cutting into a multi-byte char
```

### Mistake 5: Using `serde_json::Value` as a Universal Type

**Wrong:**
```rust
// WRONG: using Value everywhere loses compile-time type checking
fn process(data: serde_json::Value) -> serde_json::Value {
    json!({ "result": data["name"].as_str().unwrap_or("") })
}
// runtime field name typos cannot be caught by the compiler
```

**Right:**
```rust
// CORRECT: define structured types, serialize/deserialize at boundaries
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct Input { name: String, age: u32 }

#[derive(Serialize)]
struct Output { greeting: String }

fn process(input: Input) -> Output {
    Output { greeting: format!("Hello, {}!", input.name) }
}
// compiler verifies all field access; typos caught at compile time
```

### Mistake 6: Cloning Everything for "Safety" (Node.js GC Mindset)

**Wrong:**
```rust
// WRONG: cloning everywhere as if protecting data from GC or async callbacks
fn handle(data: Data) {
    let data2 = data.clone();  // unnecessary clone
    tokio::spawn(async move {
        process(data2).await;

    });
    let data3 = data.clone();  // cloned again
    save(data3).await;
}
```

**Right:**
```rust
// CORRECT: share immutable data via Arc; or transfer ownership instead of copying
fn handle(data: Arc<Data>) {
    let data_clone = data.clone();  // Arc::clone only increments ref count
    tokio::spawn(async move {
        process(&data_clone).await;
    });
    save(&data).await;  // borrow, no clone needed
}

// if data doesn't need sharing: transfer ownership directly
fn handle(data: Data) {
    tokio::spawn(async move { process(data).await });
    // data ownership moved into spawned task
}
```

## Reference Implementations

| Project | Description | Relevant Patterns |
|---------|-------------|-------------------|
| [swc](https://github.com/swc-project/swc) | JS/TS compiler (replaced Babel, 10-20x faster) | AST visitor patterns, plugin system, npm integration |
| [rspack](https://github.com/web-infra-dev/rspack) | Web bundler (replaced Webpack, 5-10x faster) | Module graph, loader plugin system, HMR |
| [deno](https://github.com/denoland/deno) | JS/TS runtime (replaced Node.js) | Async I/O, HTTP server, npm compat, Wasm |
| [rolldown](https://github.com/rolldown/rolldown) | Bundler (replaced Rollup) | Plugin system, module resolution, tree-shaking |
| [oxc](https://github.com/oxc-project/oxc) | JS tooling suite (parser, linter, minifier) | High-performance parsing, AST, parallel linting |
| [napi-rs](https://github.com/napi-rs/napi-rs) | Node.js native addon framework | Rust-to-Node FFI, N-API bindings |
| [prisma-client-rust](https://github.com/Brendonovich/prisma-client-rust) | ORM (replaced Prisma Client JS) | Query builder, type-safe database access |
| [binsider](https://github.com/orhun/binsider) | Binary analyzer | CLI tool patterns, TUI, file parsing |

## Cross-Reference

- **c-to-rust**: For C dependencies that your Node.js native addons depend on
- **cpp-to-rust**: For C++ addon patterns and FFI migration
- **python-to-rust**: For shared dynamic-to-static typing patterns; npm/pip to Cargo dependency mapping
- **go-to-rust**: For event-loop to tokio concurrency model translation; similar web framework migration
- **java-to-rust**: For enterprise TypeORM/Prisma to Diesel/sqlx ORM migration patterns
- **php-to-rust**: For Express/Fastify to Axum web framework migration; shared middleware patterns
- For HTTP frameworks: `axum` (ergonomic, tokio-native), `actix-web` (battle-tested, actor-based), `poem` (ergonomic, OpenAPI-native)
- For database: `sqlx` (async, compile-time SQL checking), `diesel` (sync, strong type guarantees), `sea-orm` (async ORM)
- For gRPC: `tonic` (async, tokio-native gRPC server/client)
- For GraphQL: `async-graphql` (framework-agnostic, async)
- For WebSocket: `axum::extract::ws` (built-in), `tokio-tungstenite` (low-level)
- For CLI tools: `clap` (derive-based argument parsing), `indicatif` (progress bars)
- For NAPI bindings: `napi-rs` (Rust-to-Node.js FFI), `neon` (alternative binding framework)
