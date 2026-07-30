---
name: go-to-rust
description: Use when migrating Go codebases to Rust — covers goroutine/async translation, channel-to-mpsc mapping, interface-to-trait conversion, ownership semantics for shared memory, FFI bridging with cgo replacement, and incremental migration strategy. Includes canonical signatures, common mistakes, and reference implementations.
---

# Go to Rust Migration

## Architecture Mapping

Go's runtime (goroutine scheduler, garbage collector, escape analyzer) maps to Rust's zero-cost abstractions and compile-time guarantees. Where Go relies on a runtime with green-thread multiplexing and tri-color GC, Rust compiles to a native binary with no runtime overhead, stack-allocated values by default, and deterministic cleanup via ownership. The Go SSA compiler and linker become rustc + LLVM with Cargo-driven dependency resolution. Go's `GOPATH` / module workspace becomes a Cargo workspace with crate-level visibility. The Go standard library's "batteries included" philosophy maps to Rust's ecosystem via crates.io -- you pull in exactly what you need rather than carrying the entire `$GOROOT`.

| Go Concept                | Rust Equivalent                              | Notes                                              |
|---------------------------|----------------------------------------------|----------------------------------------------------|
| goroutine                 | `tokio::spawn(async { ... })`                | M:N scheduling vs. work-stealing async runtime      |
| channel (unbuffered)      | `tokio::sync::oneshot`                       | Single-value rendezvous                             |
| channel (buffered)        | `tokio::sync::mpsc::channel(n)`              | Multi-producer, single-consumer                      |
| select                    | `tokio::select!` macro                       | Match on first-ready future or channel               |
| interface                 | `trait`                                      | Static dispatch by default, `dyn Trait` for dynamic  |
| struct embedding          | Field composition + `Deref`                  | No type promotion; use newtype or delegate methods   |
| defer                     | `Drop` trait + RAII                          | No function-scoped defer; scope-bound cleanup         |
| error return (value, err) | `Result<T, E>`                               | Exhaustive match with `?` operator                   |
| nil                       | `Option<T>`                                  | No universal zero value; explicit absence             |
| slice                     | `&[T]` or `Vec<T>`                           | Fat pointer (ptr + len) vs. owned growable buffer     |
| map                       | `HashMap<K, V>` / `BTreeMap<K, V>`           | No built-in map literal syntax                        |
| context.Context           | `CancellationToken` / `tokio::time::timeout` | Explicit propagation or scoped cancellation           |
| init()                    | `lazy_static!` / `once_cell::sync::Lazy`     | Deferred initialization with exactly-once semantics   |
| reflect                   | `std::any::Any` + `serde` (runtime data)     | Limited runtime reflection; prefer code generation    |
| go generate               | `build.rs` (build script)                    | Compile-time code generation                          |
| go:embed                  | `include_str!` / `include_bytes!`            | Compile-time asset embedding                          |
| sync.Mutex                | `std::sync::Mutex<T>`                        | Data is *inside* the mutex, unlocking yields access   |
| sync.WaitGroup            | `tokio::sync::Barrier` / `JoinSet`           | Task coordination patterns                            |

## Type System Mapping

| Go Type           | Rust Type                  | Notes                                              |
|-------------------|----------------------------|----------------------------------------------------|
| `bool`            | `bool`                     | Identical                                           |
| `int`, `int64`    | `i32`, `i64`               | Go int is platform-width; prefer sized in Rust       |
| `uint64`          | `u64`                      | Unsigned integer                                     |
| `float64`         | `f64`                      | IEEE 754 double                                      |
| `string`          | `String` / `&str`          | Owned vs. borrowed; Go strings are immutable []byte  |
| `[]byte`          | `Vec<u8>` / `&[u8]`        | Byte slice vs. byte buffer                           |
| `[]T`             | `Vec<T>` / `&[T]`          | Owned slice vs. borrowed slice                       |
| `map[K]V`         | `HashMap<K, V>`            | Requires `Hash` impl on K                            |
| `chan T`          | `mpsc::Sender<T>`          | Async channel sender                                  |
| `func(...) ...`   | `fn(...)` or closure trait | Function pointers vs. capturing closures             |
| `interface{}`     | `Box<dyn Any>`             | Empty interface; rarely needed in Rust               |
| `struct { ... }`  | `struct { ... }`           | Similar syntax, different memory layout              |
| `*T`              | `&T` / `Box<T>` / `Arc<T>` | Explicit pointer semantics with lifetime tracking    |
| `error`           | `dyn Error` / enum variant | Error as interface vs. algebraic error type          |

## Memory & Ownership Model

Go's garbage collector eliminates explicit memory management but introduces GC pauses and heap allocation unpredictability. Rust's ownership system replaces the GC entirely with compile-time rules:

**Go pointer-sharing pattern:**

```go
// Go: shared mutable state via pointer + GC
type Cache struct {
    mu    sync.RWMutex
    items map[string]Item
}

func (c *Cache) Get(key string) *Item {
    c.mu.RLock()
    defer c.mu.RUnlock()
    item := c.items[key]  // GC keeps this alive
    return &item
}
```

**Rust equivalent with safe sharing:**

```rust
// Rust: clear ownership, borrow checker guarantees thread safety
use std::collections::HashMap;
use std::sync::RwLock;

struct Cache {
    items: RwLock<HashMap<String, Item>>,
}

impl Cache {
    fn get(&self, key: &str) -> Option<Item>
    where
        Item: Clone, // explicit clone, no implicit sharing
    {
        self.items.read().unwrap()
            .get(key)
            .cloned() // explicit copy semantics
    }
}
```

### Ownership Rules for Go Developers

1. **No escape analysis to Rust.** Go's compiler decides stack vs. heap. In Rust, `Box<T>` explicitly allocates; everything else is stack.
2. **No shared mutable state without synchronization.** `Arc<Mutex<T>>` is the Go `sync.Mutex` equivalent -- but the data is inside the lock.
3. **No finalizers.** Go's `runtime.SetFinalizer` has no Rust equivalent. Use `Drop` for deterministic cleanup.
4. **Slices own their backing array in Rust.** Go slices share the underlying array; `Vec<T>` owns, `&[T]` borrows.

## Concurrency / Async Translation

Go's CSP model (goroutines + channels) maps to Rust's async/await + channel primitives via the tokio runtime.

### Goroutine -> Async Task

```go
// Go: goroutine — lightweight concurrency
go func() {
    result, err := doWork()
    if err != nil {
        log.Printf("error: %v", err)
        return
    }
    ch <- result
}()
```

```rust
// Rust: tokio task — same M:N scheduling, strongly-typed error handling
tokio::spawn(async move {
    match do_work().await {
        Ok(result) => { let _ = tx.send(result).await; }
        Err(e) => { tracing::error!("error: {e}"); }
    }
});
```

### Channel Patterns

| Go Pattern                       | Rust Equivalent                                      |
|----------------------------------|------------------------------------------------------|
| `ch := make(chan T)`             | `let (tx, rx) = mpsc::channel::<T>(cap);`            |
| `ch := make(chan T, n)`          | `let (tx, rx) = mpsc::channel::<T>(n);`              |
| `ch <- val`                      | `tx.send(val).await?`                                |
| `val := <-ch`                    | `let val = rx.recv().await?;`                        |
| `val, ok := <-ch`                | `match rx.recv().await { Some(v) => ..., None => ... }` |
| `close(ch)`                      | `drop(tx)` (all senders dropped = channel closed)    |
| `for val := range ch`            | `while let Some(val) = rx.recv().await { ... }`      |
| `select { case v := <-ch: ... }` | `tokio::select! { Some(v) = rx.recv() => { ... } }`  |

### Context Propagation

```go
// Go: context for timeout, cancellation, value propagation
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

select {
case result := <-doWithCtx(ctx):
    return result, nil
case <-ctx.Done():
    return nil, ctx.Err()
}
```

```rust
// Rust: tokio::time::timeout + CancellationToken combination
use tokio::time::{timeout, Duration};
use tokio_util::sync::CancellationToken;

let token = CancellationToken::new();
let child_token = token.child_token();

let task = tokio::spawn(async move {
    tokio::select! {
        result = do_work() => { /* handle result */ }
        _ = child_token.cancelled() => { /* cancellation received */ }
    }
});

// cancel after 5 seconds
tokio::spawn(async move {
    tokio::time::sleep(Duration::from_secs(5)).await;
    token.cancel();
});
```

## Build System & Dependencies

| Go                     | Rust / Cargo                          |
|------------------------|---------------------------------------|
| `go.mod`               | `Cargo.toml`                          |
| `go.sum`               | `Cargo.lock`                          |
| `go get github.com/x`  | `cargo add x`                         |
| `go build`             | `cargo build`                         |
| `go test`              | `cargo test`                          |
| `go run`               | `cargo run`                           |
| `go vet`               | `cargo clippy`                        |
| `go fmt`               | `cargo fmt`                           |
| `GOPATH`/workspace     | Cargo workspace                       |
| `go generate`          | `build.rs`                            |
| `go:embed`             | `include_str!` / `include_bytes!`     |
| internal/              | `pub(crate)` visibility              |
| `GOOS`/`GOARCH`        | `#[cfg(target_os = "...")]`          |

**Cargo.toml for a migrated Go service:**

```toml
[package]
name = "my-service"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
axum = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres"] }
tracing = "0.1"
tracing-subscriber = "0.3"
anyhow = "1"
thiserror = "2"
```

## Standard Library Mapping

| Go stdlib                          | Rust crate / std                        | Notes                                   |
|------------------------------------|-----------------------------------------|-----------------------------------------|
| `fmt.Println` / `fmt.Sprintf`      | `println!` / `format!`                  | Macros, not functions                    |
| `encoding/json`                    | `serde_json`                            | Derive-based serialization               |
| `net/http`                         | `axum` / `actix-web` / `reqwest`        | Split client/server crates               |
| `database/sql`                     | `sqlx` / `diesel`                       | Async-native, compile-time checked       |
| `time.Now` / `time.Since`          | `std::time::Instant` / `chrono`         | System time vs. monotonic                |
| `os.Open` / `os.ReadFile`          | `std::fs::File` / `std::fs::read_to_string` | Nearly identical API                  |
| `log`                              | `tracing` / `log`                       | Structured, span-based logging           |
| `flag`                             | `clap`                                  | Derive-based argument parsing            |
| `crypto/sha256`                    | `sha2` crate                            | Drop-in replacement                      |
| `sync.Mutex`                       | `std::sync::Mutex<T>`                   | Data-inside-lock pattern                 |
| `sync.RWMutex`                     | `std::sync::RwLock<T>`                  | Multiple readers, single writer          |
| `sync.Once`                        | `std::sync::Once` / `OnceLock<T>`       | One-time initialization                  |
| `sort.Slice`                       | `slice::sort` / `slice::sort_by`        | In-place sorting                         |
| `regexp`                           | `regex`                                 | Similar API, different syntax nuances    |
| `strings.Split`                    | `str::split`                            | Iterator-based, not slice return         |
| `strconv.Itoa`                     | `i.to_string()`                         | `ToString` trait                         |
| `context.Context`                  | `CancellationToken`                     | tokio-util or manual future cancellation |
| `testing.T`                        | `#[test]` + `assert_eq!`               | Built-in test harness                    |
| `net.Listen`                       | `tokio::net::TcpListener`               | Async-native networking                  |

## Canonical Patterns

### 1. Interface -> Trait + Generics

```go
// Go: implicit interface implementation
type Reader interface {
    Read(p []byte) (n int, err error)
}

type FileReader struct{ path string }

func (f *FileReader) Read(p []byte) (n int, err error) {
    // ...
}

func process(r Reader) error {
    buf := make([]byte, 1024)
    _, err := r.Read(buf)
    return err
}
```

```rust
// Rust: explicit trait impl + generics
trait Reader {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize, std::io::Error>;
}

struct FileReader {
    path: String,
}

impl Reader for FileReader {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize, std::io::Error> {
        // ...
    }
}

// static dispatch: compile-time monomorphization
fn process<R: Reader>(reader: &mut R) -> Result<(), std::io::Error> {
    let mut buf = [0u8; 1024];
    reader.read(&mut buf)?;
    Ok(())
}

// dynamic dispatch: when heterogeneous collections are needed
fn process_dyn(reader: &mut dyn Reader) -> Result<(), std::io::Error> {
    let mut buf = [0u8; 1024];
    reader.read(&mut buf)?;
    Ok(())
}
```

### 2. defer -> Drop Trait

```go
// Go: defer guarantees cleanup on function exit
func copyFile(src, dst string) error {
    f, err := os.Open(src)
    if err != nil {
        return err
    }
    defer f.Close()

    out, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer out.Close()

    _, err = io.Copy(out, f)
    return err
}
```

```rust
// Rust: RAII — Drop runs automatically at scope exit
use std::fs::File;
use std::io;

fn copy_file(src: &str, dst: &str) -> io::Result<()> {
    let mut f = File::open(src)?;        // auto-closes at scope exit
    let mut out = File::create(dst)?;    // same as above
    io::copy(&mut f, &mut out)?;
    Ok(())
}
```

### 3. Multiple Return Values -> Result with Tuple

```go
// Go: multiple return value pattern
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

result, err := divide(10, 0)
if err != nil {
    // handle error
}
```

```rust
// Rust: Result type — enforces error path handling
fn divide(a: f64, b: f64) -> Result<f64, &'static str> {
    if b == 0.0 {
        Err("division by zero")
    } else {
        Ok(a / b)
    }
}

// propagate errors with ? operator
let result = divide(10.0, 0.0)?; // error automatically propagates upward
```

### 4. Struct Embedding -> Composition + Deref

```go
// Go: struct embedding — method promotion
type Base struct{ Name string }

func (b Base) Greet() string {
    return "Hello, " + b.Name
}

type Derived struct {
    Base                    // embedded: Greet method is promoted
    Age int
}
```

```rust
// Rust: composition + Deref for similar effect
use std::ops::Deref;

struct Base {
    name: String,
}

impl Base {
    fn greet(&self) -> String {
        format!("Hello, {}", self.name)
    }
}

struct Derived {
    base: Base, // explicit composition
    age: u32,
}

// Deref lets Derived call Base methods directly
impl Deref for Derived {
    type Target = Base;
    fn deref(&self) -> &Self::Target {
        &self.base
    }
}
```

### 5. Select Statement -> tokio::select!

```go
// Go: select multiplexing
select {
case msg := <-ch1:
    handle(msg)
case msg := <-ch2:
    handle(msg)
case <-time.After(time.Second):
    fmt.Println("timeout")
case <-ctx.Done():
    return ctx.Err()
}
```

```rust
// Rust: tokio::select! macro
tokio::select! {
    Some(msg) = rx1.recv() => {
        handle(msg);
    }
    Some(msg) = rx2.recv() => {
        handle(msg);
    }
    _ = tokio::time::sleep(Duration::from_secs(1)) => {
        tracing::warn!("timeout");
    }
    _ = cancellation_token.cancelled() => {
        return Err(anyhow::anyhow!("cancelled"));
    }
}
```

### 6. Newtype Validation Pattern

```go
// Go: weak typing — any string can be an Email
type Email string  // just an alias, no compile-time guarantee

func SendEmail(to Email, body string) error {
    // cannot guarantee to is really a valid email
}
```

```rust
// Rust: newtype wrapper + constructor validation
#[derive(Debug, Clone)]
pub struct Email(String);

impl Email {
    pub fn new(raw: impl Into<String>) -> Result<Self, &'static str> {
        let s: String = raw.into();
        if s.contains('@') && !s.is_empty() {
            Ok(Email(s))
        } else {
            Err("invalid email address")
        }
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

fn send_email(to: &Email, body: &str) {
    // compile-time guarantee: to was validated by Email::new
}
```

## FFI & Incremental Migration

Go's cgo is notoriously slow (per-call overhead). Rust's FFI is zero-cost. Migration can happen one package at a time via C-ABI bridging.

### Strategy: Package-by-Package Replacement

1. Define a C-compatible ABI boundary in Rust, exposing functions via `#[no_mangle] extern "C"`.
2. Build as `cdylib` -- Go loads via `syscall` or cgo (minimal shim).
3. Replace Go packages incrementally: models first, then business logic, then I/O layer.

**Rust side (exporting):**

```rust
// src/lib.rs — export as C dynamic library
#[no_mangle]
pub extern "C" fn process_order(json_ptr: *const c_char) -> *mut c_char {
    let json = unsafe { CStr::from_ptr(json_ptr) }.to_str().unwrap();
    let result = match serde_json::from_str::<Order>(json) {
        Ok(order) => {
            // business logic
            format!("{{ \"status\": \"ok\", \"id\": {} }}", order.id)
        }
        Err(e) => {
            format!("{{ \"error\": \"{}\" }}", e)
        }
    };
    CString::new(result).unwrap().into_raw()
}

#[no_mangle]
pub extern "C" fn free_string(ptr: *mut c_char) {
    unsafe { let _ = CString::from_raw(ptr); }
}
```

**Go side (importing):**

```go
// #cgo LDFLAGS: -L./target/release -lmy_service
// #include <stdlib.h>
// extern char* process_order(const char* json);
// extern void free_string(char* ptr);
import "C"
import "unsafe"

func ProcessOrderInRust(order Order) (*Result, error) {
    json, _ := json.Marshal(order)
    cJson := C.CString(string(json))
    defer C.free(unsafe.Pointer(cJson))

    cResult := C.process_order(cJson)
    defer C.free_string(cResult)

    // parse JSON returned by Rust
    var result Result
    err := json.Unmarshal([]byte(C.GoString(cResult)), &result)
    return &result, err
}
```

### Migration Order

| Phase | Scope          | Strategy                       |
|-------|----------------|--------------------------------|
| 1     | Data models    | Rust structs, shared via JSON/Protobuf |
| 2     | Pure logic     | Stateless functions ported first |
| 3     | Concurrency    | goroutine pools -> tokio tasks |
| 4     | HTTP handlers  | net/http -> axum handlers      |
| 5     | Database layer | database/sql -> sqlx           |
| 6     | Full service   | Remove Go binary, keep lib for legacy |

## Common Mistakes

### Mistake 1: Cloning Everything

```rust
// MISTAKE: Go devs pass by value (Go copies implicitly), causing excessive .clone() in Rust
fn handle_user(user: User) {  // user is moved into the function
    save(&user);
    notify(&user);  // compile error: user already moved
}

// CORRECT: pass references, be explicit about ownership
fn handle_user(user: &User) {  // borrow, do not take ownership
    save(user);
    notify(user);  // can be borrowed multiple times
}
```

### Mistake 2: Over-using Arc<Mutex<T>>

```rust
// WARNING: Go shares mutable state via GC + Mutex; Rust newcomers over-use Arc<Mutex<>>
// Only use Arc when multiple owners are truly needed. Single owner: &Mutex<T> is sufficient.

// In most cases Arc is unnecessary
use std::sync::Mutex;

struct AppState {
    db: sqlx::PgPool,         // Pool is already Arc internally
    cache: Mutex<LruCache>,   // AppState has a single owner, no Arc needed
}
```

### Mistake 3: Ignoring Result

```rust
// MISTAKE: Go habit of ignoring error return values
// Rust #[must_use] Result triggers compile warning if unhandled
let _ = file.write_all(b"data"); // let _ = to suppress warnings is an anti-pattern

// CORRECT: handle explicitly
file.write_all(b"data").context("failed to write data")?;
```

### Mistake 4: Channel as Primary Communication

```go
// Go mantra: "share memory by communicating, not communicate by sharing memory"
// this mantra is overused in many scenarios
ch := make(chan Item, 100)
go producer(ch)
go consumer(ch)
```

```rust
// Rust: channels are effective, but simpler patterns often work inside async/await
// Consider: Arc<Mutex<T>> or specialized primitives (e.g. tokio::sync::watch)
use tokio::sync::watch;

let (tx, mut rx) = watch::channel(initial_state);
// producer updates state
tx.send(new_state)?;
// multiple consumers receive notifications
let mut rx2 = rx.clone(); // watch channel supports multiple consumers
```

### Mistake 5: String vs &str Confusion

```rust
// MISTAKE: Go devs using &str as string
struct Config {
    name: &str,  // compile error: missing lifetime annotation
}

// CORRECT: Use String when owning, &str for borrowed contexts
struct Config<'a> {
    name: &'a str,  // requires lifetime annotation
}

// Or: own directly for simple cases
struct Config {
    name: String,  // owns data, simpler
}
```

## Reference Implementations

| Project                          | Description                                        | Go LOC | Rust LOC |
|----------------------------------|----------------------------------------------------|--------|----------|
| Zed editor (parsers)             | Tree-sitter grammars migrated from Go to Rust      | ~8k    | ~6k      |
| Ruff (Python linter)             | Python tooling written in Rust; benchmarked 10-100x faster than Go equivalents | N/A | ~40k |
| Meilisearch                      | Originally Go search engine; later core rewritten  | ~30k   | ~25k     |
| Tantivy (full-text search)       | Rust-native Lucene-equivalent; inspired by Go FT libs but 5x faster | N/A | ~60k |
| Glommio (async runtime)          | Purpose-built Rust i/o-uring; often replaces Go services for I/O-heavy workloads | N/A | ~30k |
| ripgrep                          | Code search replacing Go-equivalent tools; benchmarked 3-5x faster | N/A | ~40k |
| Oso (authz engine)               | Core engine migration from Go to Rust for perf     | ~15k   | ~12k     |
| TiKV (distributed KV)            | FoundationDB-inspired; CNCF project, Rust core     | ~50k   | ~80k     |

## Cross-Reference

- **c-to-rust**: Systems-level FFI patterns for incremental Go service migration
- **nodejs-to-rust**: Async runtime and web framework patterns shared with Go migration
- **java-to-rust**: Enterprise service migration patterns; comparable to Go gRPC/HTTP service refactoring
