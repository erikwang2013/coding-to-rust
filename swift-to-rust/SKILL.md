---
name: swift-to-rust
description: Use when migrating Swift codebases to Rust — covers ARC to ownership, Swift Concurrency (async/await, Task, Actor) to tokio, protocol to trait, enum with associated values to Rust enum, struct/value type to Copy/Clone, SwiftUI to Leptos/Dioxus, Vapor to Axum, Swift Package Manager to Cargo, and incremental migration via C ABI. Includes canonical code patterns, common mistakes, and reference implementations.
updated: 2026-07-30
---

# Swift to Rust Migration

## Architecture Mapping

Swift's LLVM-based compilation (with Automatic Reference Counting for memory management, `DispatchQueue` for concurrency, and the Swift Concurrency model with `async`/`await`, `Task`, and `Actor`) maps to Rust's AOT compilation with ownership-based memory management, tokio for async I/O, and `Send + Sync` traits for thread safety.

Swift and Rust share deep architectural similarities: both are expression-oriented, both have algebraic enums with associated values, both favor value types over reference types, both use protocol/trait-based polymorphism, and both reject null in favor of `Optional`/`Option`. Swift's ARC (coarse-grained compile-time retain/release) becomes Rust's ownership system (fine-grained, zero-runtime-overhead). Swift's `actor` becomes Rust's `Arc<Mutex<T>>` pattern or actor model via channels.

| Swift Concept | Rust Equivalent | Notes |
|---------------|-----------------|-------|
| ARC (Automatic Reference Counting) | Ownership + borrowing | Compile-time, zero runtime overhead |
| `class` (reference type) | `struct` + `Arc<T>` for shared state | Prefer value types |
| `struct` (value type) | `struct` + `Copy` / `Clone` | Both are value types |
| `enum` with associated values | `enum` with variant data | Identical pattern |
| `protocol` | `trait` | Explicit conformance |
| `protocol extension` | Trait with default methods | Same pattern |
| `extension` | `impl ExtTrait for T` | Separate conformance blocks |
| `async` / `await` | `async fn` / `.await` | Same syntax model |
| `Task { }` / `Task.detached { }` | `tokio::spawn(async { })` | Async task creation |
| `actor` | Actor pattern via channel + `Arc<Mutex<T>>` | Isolated mutable state |
| `@MainActor` | Main-thread constraint via spawn_local or runtime | UI thread safety |
| `DispatchQueue` (serial/concurrent) | `tokio::task` / `rayon` | Task scheduling |
| `Combine` (Publisher/Subscriber) | `futures::Stream` / `tokio::sync::broadcast` | Reactive streams |
| `@Published` / `@State` | `RwSignal` / `create_signal` (Leptos) | Reactive state |
| `defer { }` | `Drop` trait | Scope-exit cleanup |
| `guard let x = opt else { return }` | `let x = opt?;` / `let Some(x) = opt else { return }` | Early exit |
| `if let` / `switch` | `if let` / `match` | Pattern matching |
| `Result<T, E>` | `Result<T, E>` | Same monadic error type |
| `throws` | `Result<T, E>` + `?` | Checked errors in both |
| `Optional<T>` / `T?` | `Option<T>` | Identical semantics |
| `typealias` | `type Alias = ...;` | Type alias |
| `associatedtype` | Associated type `type Item;` in trait | Generic protocol |
| `where T: P, T: Q` | `where T: P + Q` | Same constraint syntax |
| `KeyPath<T, V>` | Function pointer or closure | No first-class keypath |
| Property wrapper (`@Clamped`) | Proc macro or newtype pattern | Custom behavior |

## Type System Mapping

| Swift Type | Rust Type | Notes |
|------------|-----------|-------|
| `Int` / `Int64` | `i64` | Platform-width in Swift; explicit in Rust |
| `UInt` / `UInt64` | `u64` | Unsigned |
| `Float` / `Double` | `f32` / `f64` | IEEE 754 |
| `Bool` | `bool` | Identical |
| `String` | `String` / `&str` | Swift String is value-type; Rust String is owned |
| `Character` | `char` | Unicode extended grapheme cluster vs. scalar |
| `Array<T>` / `[T]` | `Vec<T>` / `&[T]` | Value type in Swift, owned/borrowed in Rust |
| `Dictionary<K,V>` / `[K:V]` | `HashMap<K,V>` / `BTreeMap<K,V>` | |
| `Set<T>` | `HashSet<T>` / `BTreeSet<T>` | |
| `Tuple (A, B, C)` | `(A, B, C)` | Native tuples in both |
| `Range<Int>` / `ClosedRange<Int>` | `start..end` / `start..=end` | Range expressions |
| `Optional<T>` / `T?` | `Option<T>` | `nil` → `None` |
| `Data` | `Vec<u8>` / `bytes::Bytes` | Byte buffer |
| `URL` | `url::Url` (url crate) / `&str` | |
| `UUID` | `uuid::Uuid` | |
| `Date` / `DateComponents` | `chrono::DateTime<Utc>` / `time::Date` | |
| `Decimal` | `rust_decimal::Decimal` | Fixed-point decimal |
| `Codable` (Encodable/Decodable) | `serde::Serialize` / `serde::Deserialize` | Derive-based |
| `Error` (protocol) | `std::error::Error` / `thiserror::Error` | Custom error types |
| `Any` / `AnyObject` | `dyn Any` / `&dyn Trait` | Type erasure |

## Memory & Ownership Model

Swift's ARC (Automatic Reference Counting) inserts retain/release calls at compile time, with the optimizer eliding redundant operations. Rust's ownership system is zero-runtime-overhead — no reference counting for typical borrows, only for `Rc`/`Arc`.

| Swift Pattern | Rust Translation |
|---------------|------------------|
| `let x = obj` (strong reference) | `let x = obj` (ownership transfer) or `let x = &obj` |
| `weak var` (weak reference) | `Weak<T>` (from `Rc`/`Arc`) |
| `unowned` (unowned reference) | `&T` borrow (lifetime-checked) |
| `struct` copy-on-write (CoW) | `Clone` (explicit copy) |
| `class` reference semantics | `Arc<T>` for shared ownership |
| `deinit { }` (deinitializer) | `Drop::drop(&mut self)` |
| Capturing `[weak self]` in closures | `Weak::upgrade()` pattern |
| Capturing `self` in `@Sendable` closure | `Arc::clone(&self)` + `move` closure |

### Value Type vs. Reference Type

```swift
// Swift: struct = value type, class = reference type
struct Point { var x: Double; var y: Double }  // copied on assignment
class User  { var name: String }               // shared reference (ARC)
```

```rust
// Rust: everything is a value type by default
#[derive(Copy, Clone)]  // implicit copy for small types
struct Point { x: f64, y: f64 }  // copied on assignment if Copy

struct User { name: String }     // moved on assignment (String is not Copy)
// For shared ownership: Arc<User>
```

## Concurrency / Async Translation

Swift Concurrency (WWDC 2021) and Rust's tokio share the same `async`/`await` model with stackless coroutines. Swift's `Task` and structured concurrency map directly to tokio's `JoinSet`.

| Swift Concurrency | Rust / Tokio |
|-------------------|--------------|
| `async func fn() -> T` | `async fn fn() -> T` |
| `await fn()` | `fn().await` |
| `Task { await work() }` | `tokio::spawn(async { work().await })` |
| `async let x = work()` | `let handle = tokio::spawn(work())` |
| `withTaskGroup(of: T.self) { }` | `JoinSet<T>` |
| `withThrowingTaskGroup { }` | `JoinSet` + `try_join_next` |
| `Task.sleep(nanoseconds:)` | `tokio::time::sleep(dur)` |
| `withTimeout(.seconds(n)) { }` | `tokio::time::timeout(dur, future)` |
| `actor MyActor { }` | `struct MyActor { state: Arc<Mutex<State>> }` |
| `@MainActor func updateUI()` | `spawn_local` on main thread |
| `AsyncSequence` | `futures::Stream` / `async_stream::stream!` |
| `AsyncStream` (continuation) | `async_channel` / `tokio::sync::mpsc` |
| `CheckedContinuation` | `oneshot` channel for callbacks |
| `Sendable` protocol | `Send` trait (auto-derived) |
| `nonisolated` | Regular `&self` method (no lock needed) |
| `TaskPriority` | `tokio::task::Builder::priority` |
| `Task.yield()` | `tokio::task::yield_now()` |

### Actor Model

```swift
// Swift: actor isolates mutable state
actor Counter {
    var value = 0
    func increment() -> Int {
        value += 1
        return value
    }
}
```

```rust
// Rust: Arc<Mutex<T>> provides equivalent isolation
use std::sync::{Arc, Mutex};

#[derive(Clone)]
struct Counter {
    inner: Arc<Mutex<i32>>,
}

impl Counter {
    fn new() -> Self {
        Self { inner: Arc::new(Mutex::new(0)) }
    }

    fn increment(&self) -> i32 {
        let mut guard = self.inner.lock().unwrap();
        *guard += 1;
        *guard
    }
}
```

## Build System & Dependencies

| Swift Tool | Rust Equivalent |
|------------|-----------------|
| Swift Package Manager (`Package.swift`) | `Cargo.toml` |
| `Package.resolved` | `Cargo.lock` |
| `swift build` | `cargo build` |
| `swift test` | `cargo test` |
| `swift run` | `cargo run` |
| `swift package init` | `cargo init` |
| Xcode project (`.xcodeproj`) | IDE-independent (VS Code + rust-analyzer) |
| `swiftlint` / SwiftFormat | `cargo clippy` / `cargo fmt` |
| Carthage / CocoaPods | crates.io (Cargo native) |
| XCTest | `#[test]` + `cargo test` |

**Cargo.toml for a migrated Vapor service:**

```toml
[package]
name = "api-service"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "uuid", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
anyhow = "1"
thiserror = "2"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json", "env-filter"] }
reqwest = { version = "0.12", features = ["json"] }
jsonwebtoken = "9"
argon2 = "0.5"
```

## Standard Library & Framework Mapping

### Vapor / Hummingbird → Axum / Actix-Web

| Swift Framework | Rust Equivalent | Notes |
|-----------------|-----------------|-------|
| Vapor `app.get("path") { req in }` | `Router::route("/path", get(handler))` | Route handler |
| Vapor `req.content.decode(T.self)` | `Json<T>` extractor | Codable → Serde |
| Vapor `req.query.decode(T.self)` | `Query<T>` extractor | Query parameters |
| Vapor `req.parameters.get("id")` | `Path<T>` extractor | URL params |
| Vapor `Model` (Fluent) | `sqlx::FromRow` / Diesel schema | ORM mapping |
| Vapor `req.db` (Fluent) | `State<sqlx::PgPool>` + `Extension` | DB pool injection |
| Vapor `Migration` (Fluent) | `sqlx migrate` / Diesel migrations | Schema versioning |
| Vapor `Middleware` | `tower::Layer` + `tower::Service` | Composable middleware |
| Vapor `JWTKit` | `jsonwebtoken` crate | JWT handling |
| Hummingbird `HBRequest` | `axum::extract::Request` | Typed request |
| APNS / Push notifications | `a2` crate | Apple Push Notification service |
| `NIO` (non-blocking I/O) | `tokio` (M:N scheduler) | Event loop vs. work-stealing |
| `AsyncHTTPClient` | `reqwest` | Async HTTP client |

### Swift Standard Library → Rust

| Swift stdlib | Rust |
|--------------|------|
| `array.map { $0.x }` | `vec.iter().map(|x| x.x).collect()` |
| `array.filter { $0 > 0 }` | `vec.iter().filter(|&x| x > 0).collect()` |
| `array.compactMap { $0 }` | `vec.iter().filter_map(|x| *x).collect()` |
| `array.flatMap { $0.children }` | `vec.iter().flat_map(|x| &x.children).collect()` |
| `array.reduce(0, +)` | `vec.iter().sum()` or `vec.iter().fold(0, |a,b| a+b)` |
| `array.prefix(n)` | `vec.iter().take(n)` |
| `array.dropFirst(n)` | `vec.iter().skip(n)` |
| `array.first(where: { })` | `vec.iter().find(|x| ...)` |
| `str.hasPrefix("...")` / `hasSuffix` | `s.starts_with("...")` / `s.ends_with("...")` |
| `str.components(separatedBy: ",")` | `s.split(',')` |
| `str.trimmingCharacters(in: .whitespaces)` | `s.trim()` |
| `"\(variable)"` (interpolation) | `format!("{variable}")` |
| `Data()` / `[UInt8]` | `Vec<u8>` / `bytes::Bytes` |
| `JSONEncoder` / `JSONDecoder` | `serde_json::to_string` / `from_str` |
| `FileManager.default` | `std::fs` (read, write, create_dir_all) |
| `UserDefaults` | `dirs` crate + serde JSON |
| `NotificationCenter.default` | `tokio::sync::broadcast` channel |

## Canonical Patterns

### 1. Protocol → Trait

```swift
// Swift: protocol with associated type
protocol Repository {
    associatedtype Item
    func find(by id: String) async throws -> Item?
    func save(_ item: Item) async throws
}

struct UserRepository: Repository {
    typealias Item = User
    func find(by id: String) async throws -> User? { ... }
    func save(_ item: User) async throws { ... }
}
```

```rust
// Rust: trait with associated type
use async_trait::async_trait;

#[async_trait]
pub trait Repository {
    type Item;
    async fn find_by_id(&self, id: &str) -> Result<Option<Self::Item>, AppError>;
    async fn save(&self, item: Self::Item) -> Result<(), AppError>;
}

pub struct UserRepository { pool: PgPool }

#[async_trait]
impl Repository for UserRepository {
    type Item = User;
    async fn find_by_id(&self, id: &str) -> Result<Option<User>, AppError> { ... }
    async fn save(&self, item: User) -> Result<(), AppError> { ... }
}
```

### 2. Enum with Associated Values

```swift
// Swift: enum with associated values
enum ApiResponse<T: Decodable> {
    case success(data: T)
    case error(code: Int, message: String)
    case loading
}

switch response {
case .success(let data): handle(data)
case .error(let code, let message): showError(code, message)
case .loading: showSpinner()
}
```

```rust
// Rust: identical pattern
enum ApiResponse<T> {
    Success(T),
    Error { code: i32, message: String },
    Loading,
}

match &response {
    ApiResponse::Success(data) => handle(data),
    ApiResponse::Error { code, message } => show_error(*code, message),
    ApiResponse::Loading => show_spinner(),
}
```

### 3. guard let → let-else / ?

```swift
// Swift: guard let for early exit
func process(userId: String?) throws -> User {
    guard let userId = userId else { throw AppError.missingId }
    guard let user = database.find(userId) else { throw AppError.notFound }
    return user
}
```

```rust
// Rust: ? operator for Option/Result propagation
fn process(user_id: Option<&str>) -> Result<User, AppError> {
    let user_id = user_id.ok_or(AppError::MissingId)?;
    let user = database.find(user_id)?.ok_or(AppError::NotFound)?;
    Ok(user)
}

// Or with let-else (Rust 1.65+):
fn process(user_id: Option<&str>) -> Result<User, AppError> {
    let Some(user_id) = user_id else { return Err(AppError::MissingId) };
    let Some(user) = database.find(user_id)? else { return Err(AppError::NotFound) };
    Ok(user)
}
```

### 4. Result Type → Result Type

```swift
// Swift: throwing function with Result
func divide(_ a: Double, by b: Double) -> Result<Double, MathError> {
    guard b != 0 else { return .failure(.divisionByZero) }
    return .success(a / b)
}

let result = divide(10, by: 0)
    .map { $0 * 2 }
    .flatMap { process($0) }

switch result {
case .success(let value): print(value)
case .failure(let error): print(error)
}
```

```rust
// Rust: same Result type, same monadic combinators
fn divide(a: f64, b: f64) -> Result<f64, MathError> {
    if b == 0.0 { return Err(MathError::DivisionByZero); }
    Ok(a / b)
}

let result = divide(10.0, 0.0)
    .map(|x| x * 2.0)
    .and_then(|x| process(x));

match result {
    Ok(value) => println!("{value}"),
    Err(e) => eprintln!("{e}"),
}
```

### 5. SwiftUI View → Leptos Component

```swift
// SwiftUI: declarative view
struct UserRow: View {
    @State var user: User

    var body: some View {
        HStack {
            Text(user.name).font(.headline)
            Text(user.email).foregroundColor(.secondary)
        }
    }
}
```

```rust
// Leptos: declarative component
#[component]
fn UserRow(user: User) -> impl IntoView {
    view! {
        <div class="flex gap-4">
            <span class="font-bold">{user.name}</span>
            <span class="text-gray-500">{user.email}</span>
        </div>
    }
}
```

## FFI & Incremental Migration

Swift's C interoperability (via module maps, bridging headers, and `@_cdecl`) enables Rust-Swift FFI through the C ABI.

| Strategy | Tool | When |
|----------|------|------|
| C ABI bridging | `extern "C"` + `cbindgen` on Rust side | Incremental library replacement |
| `uniffi` crate | Auto-generate Swift bindings from Rust | Feature-level Rust integration |
| HTTP/gRPC boundary | Reverse proxy split | Service-level migration |
| Shared database | Schema-compatible writes | Read/write during transition |
| Embedded Rust in iOS | `cargo lipo` / `cargo-xcode` | Rust static lib in Xcode |

### Migration Order

1. **Models & Codable**: Define shared types via JSON/Protobuf; auto-generate both sides
2. **Pure logic**: Port network-agnostic business rules to Rust
3. **Networking**: Move URLSession operations to reqwest + tokio
4. **Database**: Replace CoreData/GRDB with sqlx or Diesel
5. **Server-side**: Migrate Vapor/Hummingbird services to Axum/Actix
6. **Full cutover**: Remove Swift dependency; keep C ABI bridges for legacy

## Common Mistakes

### Mistake 1: Overusing `Arc` (ARC Mindset)

```rust
// WRONG: wrapping everything in Arc like Swift's ARC
struct Service {
    db: Arc<PgPool>,     // unnecessary
    cache: Arc<Redis>,   // unnecessary
}

// CORRECT: PgPool already uses Arc internally
struct Service {
    db: PgPool,          // PgPool is already Clone and cheap
    cache: redis::Client,
}
```

### Mistake 2: Nested Optional Chains Instead of `?`

```rust
// WRONG: Swift-style nested flat_map chains
fn get_street(user: Option<&User>) -> Option<&str> {
    user.and_then(|u| u.address.as_ref())
        .and_then(|a| a.street.as_deref())
}

// CORRECT: use ? operator for Option propagation
fn get_street(user: Option<&User>) -> Option<&str> {
    Some(user?.address.as_ref()?.street.as_deref()?)
}
```

### Mistake 3: Treating `for` Loops Like Swift's Mutable Iteration

```swift
// Swift: for-in with mutable array is fine
for i in 0..<items.count { items[i].value *= 2 }
```

```rust
// Rust: prefer iter_mut over index-based mutation
for item in &mut items { item.value *= 2; }
```

### Mistake 4: Closing Over Self Without Arc

```swift
// Swift: closures capture self strongly by default
Task { await self.updateUI() }
```

```rust
// Rust: need explicit Arc::clone for 'static async tasks
let self_arc = Arc::new(self);
let self_clone = self_arc.clone();
tokio::spawn(async move { self_clone.update().await });
```

## Reference Implementations

| Project | Description | Pattern |
|---------|-------------|---------|
| Tauri | Desktop app framework; Swift → Rust backend | Rust core with native UI shells |
| WezTerm | GPU-accelerated terminal; Swift+Metal → Rust+wgpu | GPU rendering |
| Meilisearch | Search engine; Swift SDK, Rust core | Shared wire protocol |
| Polars | DataFrame; used from Swift via Rust C bridge | C ABI interop |
| swift-bridge | Auto-generate Swift↔Rust bindings | Direct FFI crate |

## Cross-Reference

- **csharp-to-rust**: Similar ARC/GC to ownership paradigm; shared enterprise patterns
- **vue-to-rust**: SwiftUI to Leptos/Dioxus component mapping; reactive UI patterns
- **go-to-rust**: Concurrency model translation; shared structured concurrency concepts
- **kotlin-to-rust**: Mobile ecosystem (iOS/Android); shared JVM/Kotlin migration parallels
