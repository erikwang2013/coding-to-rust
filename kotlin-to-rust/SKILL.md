---
name: kotlin-to-rust
description: Use when migrating Kotlin codebases to Rust — covers JVM/Native to rustc AOT, coroutines to tokio, data class to struct, sealed class to enum, extension functions to trait extensions, Ktor/Spring Boot to Axum/Actix, Exposed to Diesel/sqlx, Gradle to Cargo, and incremental migration via JNI. Includes canonical code patterns, common mistakes, and reference implementations.
updated: 2026-07-30
---

# Kotlin to Rust Migration

## Architecture Mapping

Kotlin's multi-platform runtime (JVM bytecode with JIT compilation, or Kotlin/Native via LLVM) maps to Rust's AOT-compiled native binary. Where Kotlin relies on the JVM's garbage collector (or Kotlin/Native's cycle-aware reference counting), Rust replaces both with compile-time ownership and RAII. Kotlin's coroutine-based structured concurrency maps to tokio's async/await and `JoinSet` for structured task management.

Kotlin's emphasis on immutability, expression-oriented programming, null safety, and functional combinators makes it one of the most idiomatically aligned source languages for Rust migration. Kotlin's `val`-first convention matches Rust's immutable-by-default bindings; Kotlin's `?` nullable types map cleanly to `Option<T>`; and Kotlin's `when` expression mirrors Rust's `match`.

| Kotlin Concept | Rust Equivalent | Notes |
|---------------|-----------------|-------|
| JVM / Kotlin/Native | rustc + LLVM (AOT) | No runtime, no GC pauses |
| Coroutine (`suspend`) | `async fn` + tokio | Structured concurrency in both |
| `data class` | `struct` + `#[derive(Debug, Clone)]` | Value types with destructuring |
| `sealed class` / `sealed interface` | `enum` with variant data | Exhaustive `when` → `match` |
| `object` (singleton) | `static` / `OnceLock<T>` | Explicit lazy initialization |
| `companion object` | `impl` block without `self` | Associated functions |
| Extension function | Extension trait (`impl X for Y`) | Same pattern via traits |
| `inline fun` + `reified` | Generics `<T: Trait>` (monomorphized) | No type erasure in Rust |
| `typealias` | `type Alias = ...;` | Type alias |
| `when` expression | `match` expression | Exhaustive, expression-oriented |
| `?.` (safe call) | `Option::map` / `?` / `and_then` | Monadic chaining |
| `?:` (Elvis operator) | `unwrap_or()` / `unwrap_or_else()` | Default value |
| `!!` (force unwrap) | `.unwrap()` | Panics on `None` |
| `lateinit var` | `Option<T>` + `OnceLock<T>` | Deferred initialization |
| `by lazy` | `LazyLock<T>` / `once_cell::sync::Lazy<T>` | Thread-safe lazy init |
| `Flow<T>` (cold stream) | `futures::Stream` / `async_stream::stream!` | Async stream processing |
| `Channel<T>` | `tokio::sync::mpsc::channel(n)` | Buffered async channel |
| `sequence { yield() }` | `std::iter::from_fn` / `impl Iterator` | Lazy sequence generation |

## Type System Mapping

| Kotlin Type | Rust Type | Notes |
|-------------|-----------|-------|
| `Int` / `Long` | `i32` / `i64` | Fixed-width integers |
| `UInt` / `ULong` | `u32` / `u64` | Unsigned integers |
| `Float` / `Double` | `f32` / `f64` | IEEE 754 |
| `Boolean` | `bool` | Identical |
| `String` | `String` / `&str` | Owned vs. borrowed |
| `Char` | `char` | Unicode scalar (4 bytes) |
| `List<T>` / `MutableList<T>` | `&[T]` / `Vec<T>` | Immutable vs. mutable |
| `Set<T>` | `HashSet<T>` / `BTreeSet<T>` | |
| `Map<K,V>` / `MutableMap<K,V>` | `HashMap<K,V>` / `BTreeMap<K,V>` | |
| `Array<T>` | `[T; N]` / `Vec<T>` | Fixed at compile time or dynamic |
| `Pair<A,B>` / `Triple<A,B,C>` | `(A, B)` / `(A, B, C)` | Native tuples |
| `Result<T, E>` | `Result<T, E>` | Same monadic error handling |
| `T?` (nullable) | `Option<T>` | Identical semantics |
| `Unit` | `()` (unit type) | Zero-size type |
| `Nothing` | `!` (never type) | Diverging functions |
| `Any` | `dyn Any` / generics | Minimize type erasure |
| `enum class` (Kotlin) | `enum` (Rust, full ADT) | Rust enums carry variant data |
| `value class` / `inline class` | Newtype pattern `struct Id(i64)` | Zero-cost wrapper |
| `data class` | `struct` with `#[derive(...)]` | Value semantics |
| `vararg T` | `&[T]` | Slice parameter |
| `(A, B) -> R` (function type) | `Fn(A, B) -> R` / `Box<dyn Fn(A,B)->R>` | Closure traits |
| `suspend (A) -> R` | `async fn(A) -> R` / `BoxFuture<'_, R>` | Async function |

## Memory & Ownership Model

Kotlin/JVM uses generational GC; Kotlin/Native uses automatic reference counting with cycle collection. Rust replaces both with compile-time ownership. Kotlin's `val` (immutable reference) and `var` (mutable) conventions map cleanly to Rust's `let` (immutable) and `let mut`.

| Kotlin Pattern | Rust Translation |
|---------------|------------------|
| `val x = obj` (read-only reference) | `let x = &obj` (shared borrow) |
| `var x = obj; x = newObj` (reassign) | `let mut x = obj; x = newObj` |
| `data class` with `.copy()` | `#[derive(Clone)]` + `.clone()` |
| GC cleans up unreachable objects | `Drop` trait, scope-based |
| `by lazy { heavyInit() }` | `LazyLock::new(|| heavy_init())` |
| `lateinit var` + `::x.isInitialized` | `OnceLock<T>` or `Option<T>` |
| Shared mutable state across threads | `Arc<Mutex<T>>` |
| `WeakReference<T>` | `Weak<T>` (from `Arc`) |

### The GC-Free Mindset

```kotlin
// Kotlin: trust the GC, freely create objects
fun processUsers(users: List<User>): List<UserDto> {
    return users
        .filter { it.isActive }
        .map { it.toDto() }  // new list created, GC handles the old
}
```

```rust
// Rust: explicit ownership — consume or borrow?
fn process_users(users: Vec<User>) -> Vec<UserDto> {
    users
        .into_iter()  // consume original
        .filter(|u| u.is_active)
        .map(|u| u.to_dto())
        .collect()
}

// To keep the original, borrow:
fn process_users_ref(users: &[User]) -> Vec<UserDto> {
    users
        .iter()  // borrow, does not consume
        .filter(|u| u.is_active)
        .map(|u| u.to_dto())
        .collect()
}
```

## Concurrency / Async Translation

Kotlin coroutines (`kotlinx.coroutines`) provide structured concurrency with `async`/`await`, `launch`, `supervisorScope`, and `Flow`. Rust's tokio provides the same patterns with `tokio::spawn`, `JoinSet`, and `futures::Stream`.

| Kotlin Coroutines | Rust / Tokio |
|-------------------|--------------|
| `suspend fun fn()` | `async fn fn()` |
| `launch { ... }` | `tokio::spawn(async { ... })` |
| `async { ... }.await()` | `tokio::spawn(async { ... }).await?` |
| `coroutineScope { }` | `JoinSet` + `join_all` |
| `supervisorScope { }` | Individual `tokio::spawn` per task |
| `withContext(Dispatchers.IO) { }` | `tokio::task::spawn_blocking(|| {})` |
| `delay(ms)` | `tokio::time::sleep(Duration::from_millis(ms))` |
| `withTimeout(ms) { }` | `tokio::time::timeout(dur, future)` |
| `Flow<T>` (cold) | `futures::stream::Stream` / `async_stream::stream!` |
| `StateFlow<T>` / `SharedFlow<T>` | `tokio::sync::broadcast` / `watch` |
| `Mutex` (kotlinx) | `tokio::sync::Mutex<T>` |
| `Semaphore` (kotlinx) | `tokio::sync::Semaphore` |
| `Channel<T>` | `tokio::sync::mpsc::channel(n)` |
| `select { }` | `tokio::select!` macro |
| `actor { }` | Actor pattern via `tokio::spawn` + mpsc channel |
| `Dispatchers.Default` | `tokio::runtime::Runtime` (multi-thread) |
| `Dispatchers.IO` | `tokio::task::spawn_blocking` pool |

### Structured Concurrency Example

```kotlin
// Kotlin: structured concurrency with coroutineScope
suspend fun fetchAll(urls: List<String>): List<Response> = coroutineScope {
    urls.map { url ->
        async { httpClient.get(url) }
    }.awaitAll()
}
```

```rust
// Rust: structured concurrency with JoinSet
use tokio::task::JoinSet;

async fn fetch_all(urls: &[String]) -> Result<Vec<Response>, reqwest::Error> {
    let client = reqwest::Client::new();
    let mut tasks = JoinSet::new();

    for url in urls {
        let client = client.clone();
        let url = url.clone();
        tasks.spawn(async move { client.get(&url).send().await });
    }

    let mut results = Vec::new();
    while let Some(result) = tasks.join_next().await {
        results.push(result??);
    }
    Ok(results)
}
```

## Build System & Dependencies

| Kotlin Tool | Rust Equivalent |
|-------------|-----------------|
| Gradle / `build.gradle.kts` | `Cargo.toml` |
| Maven Central / `mavenCentral()` | crates.io |
| `gradle.properties` / `libs.versions.toml` | `Cargo.toml [dependencies]` |
| `./gradlew build` | `cargo build` |
| `./gradlew test` | `cargo test` |
| `./gradlew run` | `cargo run` |
| `detekt` / `ktlint` | `cargo clippy` / `cargo fmt` |
| Kover / JaCoCo | `cargo tarpaulin` / `cargo-llvm-cov` |
| Kotlin DSL for Gradle | `Cargo.toml` (TOML) |
| `versionCatalogs` (Gradle) | Cargo workspace `[workspace.dependencies]` |

**Cargo.toml for a migrated Ktor service:**

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
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "chrono", "uuid"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
anyhow = "1"
thiserror = "2"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json", "env-filter"] }
reqwest = { version = "0.12", features = ["json"] }
```

## Standard Library & Ecosystem Mapping

### Ktor / Spring Boot → Axum / Actix-Web

| Kotlin Framework | Rust Equivalent | Notes |
|-----------------|-----------------|-------|
| Ktor `routing { get("/path") { } }` | `Router::route("/path", get(handler))` | DSL routing |
| Ktor `call.respond(obj)` | `Json(obj)` response | Serde-based |
| Ktor `call.receive<T>()` | `Json<T>` extractor | Type-safe deserialization |
| Ktor `ApplicationCall` | `axum::extract` (Request, State, Query, Path) | Typed extractors |
| Spring Boot `@RestController` | Axum handler fn + Router | Function-based |
| Spring Boot `@Service` | Plain `struct` + `impl` | No DI container needed |
| Spring Boot `@Autowired` | Constructor injection via `AppState` | Explicit wiring |
| Spring Boot `@Transactional` | `sqlx::Transaction` / `pool.begin().await?` | Explicit scope |
| Spring Boot `@Valid` | `validator` crate / `garde` | Compile-time or extractor validation |
| Exposed ORM | `sqlx` / `diesel` / `sea-orm` | Async or type-safe DSL |
| Room (Android) | `sqlx` compile-time checked SQL | Cross-platform |
| `kotlinx.serialization` | `serde` + `serde_json` | Derive macros |
| `kotlinx-datetime` | `chrono` / `time` | |
| `kotlin-logging` / SLF4J | `tracing` + `tracing-subscriber` | Structured logging |
| MockK / Mockito | `mockall` / test doubles | |

### Common Kotlin stdlib → Rust

| Kotlin stdlib | Rust |
|---------------|------|
| `list.filter { }.map { }` | `.iter().filter().map().collect()` |
| `list.find { }` | `.iter().find(|x| ...)` returns `Option` |
| `list.groupBy { it.key }` | `.iter().fold(HashMap::new(), ...)` or `itertools::group_by` |
| `list.associateBy { it.id }` | `.iter().map(|x| (x.id, x)).collect::<HashMap<_,_>>()` |
| `list.take(n)` / `list.drop(n)` | `.iter().take(n)` / `.iter().skip(n)` |
| `list.firstOrNull()` | `.first().ok()` or `.get(0)` |
| `list.chunked(n)` | `.chunks(n)` (slice) |
| `String.trim()` | `s.trim()` |
| `String.split(delim)` | `s.split(delim)` returns iterator |
| `"${variable}"` (string template) | `format!("{variable}")` |
| `buildString { append(...) }` | `String::push_str` / `format!` |
| `require(cond) { msg }` | `assert!(cond, "{msg}")` |
| `checkNotNull(x)` | `x.expect("must not be null")` |
| `TODO()` / `error("msg")` | `todo!()` / `panic!("msg")` |
| `runCatching { }.getOrElse { }` | `fallible_fn().unwrap_or_else(|e| ...)` |
| `list.sortedBy { it.name }` | `vec.iter().sorted_by(|a,b| a.name.cmp(&b.name))` (itertools) |
| `list.distinct()` / `list.distinctBy { }` | `vec.iter().unique()` (itertools) / `unique_by` |
| `list.zip(other) { a,b -> }` | `a.iter().zip(b.iter()).map(|(x,y)| ...)` |
| `list.joinToString(", ")` | `vec.iter().join(", ")` (itertools) / `itertools::join` |
| `list.windowed(size, step)` | `vec.windows(size)` / `itertools::tuple_windows` |
| `let { }` / `also { }` | `let x = expr; f(&x); x` (no inline scope function) |
| `apply { }` (builder scope) | Method chaining / `typed_builder` crate |
| `String?.isNullOrBlank()` | `s.map(|s| s.is_empty()).unwrap_or(true)` |
| `"123".toIntOrNull()` | `"123".parse::<i32>().ok()` |
| `list.sumOf { it.price }` / `maxOf { }` | `vec.iter().map(|x| x.price).sum()` / `max_by` |
| `repeat(times) { }` | `(0..n).for_each(|_| { ... })` |

## Canonical Patterns

### 1. Data Class → Struct with Derive

```kotlin
// Kotlin: data class with validation
data class CreateUserRequest(
    val name: String,
    val email: String,
    val age: Int,
) {
    init {
        require(name.isNotBlank()) { "name required" }
        require(email.contains('@')) { "invalid email" }
        require(age > 0) { "age must be positive" }
    }
}
```

```rust
// Rust: struct with validation constructor
#[derive(Debug, Clone, serde::Deserialize)]
pub struct CreateUserRequest {
    pub name: String,
    pub email: String,
    pub age: i32,
}

impl CreateUserRequest {
    pub fn validate(self) -> Result<ValidatedUser, ValidationError> {
        if self.name.trim().is_empty() {
            return Err(ValidationError("name is required".into()));
        }
        if !self.email.contains('@') {
            return Err(ValidationError("invalid email".into()));
        }
        if self.age <= 0 {
            return Err(ValidationError("age must be positive".into()));
        }
        Ok(ValidatedUser { name: self.name, email: self.email, age: self.age })
    }
}
```

### 2. Sealed Class → Enum with Variant Data

```kotlin
// Kotlin: sealed class hierarchy
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}

fun render(state: UiState<User>) = when (state) {
    is UiState.Loading -> showSpinner()
    is UiState.Success -> showUser(state.data)
    is UiState.Error -> showError(state.message)
}
```

```rust
// Rust: enum with variant data — same exhaustive matching
enum UiState<T> {
    Loading,
    Success(T),
    Error(String),
}

fn render(state: &UiState<User>) {
    match state {
        UiState::Loading => show_spinner(),
        UiState::Success(user) => show_user(user),
        UiState::Error(msg) => show_error(msg),
    }
}
```

### 3. Extension Function → Extension Trait

```kotlin
// Kotlin: extension function on String
fun String.isValidEmail(): Boolean =
    this.contains('@') && this.contains('.')
```

```rust
// Rust: extension trait pattern
pub trait StringExt {
    fn is_valid_email(&self) -> bool;
}

impl StringExt for str {
    fn is_valid_email(&self) -> bool {
        self.contains('@') && self.contains('.')
    }
}
```

### 4. Coroutine Scope → Async Block

```kotlin
// Kotlin: coroutineScope for parallel work
suspend fun loadDashboard(userId: String): Dashboard = coroutineScope {
    val user = async { userRepo.findById(userId) }
    val orders = async { orderRepo.findRecent(userId) }
    val notifications = async { notificationRepo.unread(userId) }
    Dashboard(user.await(), orders.await(), notifications.await())
}
```

```rust
// Rust: tokio::try_join! for concurrent work
async fn load_dashboard(user_id: &str) -> Result<Dashboard, AppError> {
    let (user, orders, notifications) = tokio::try_join!(
        user_repo.find_by_id(user_id),
        order_repo.find_recent(user_id),
        notification_repo.unread(user_id),
    )?;
    Ok(Dashboard { user, orders, notifications })
}
```

### 5. Builder Pattern (Kotlin DSL → typed-builder)

```kotlin
// Kotlin: builder DSL
data class QueryRequest(
    val index: String,
    val query: String,
    val page: Int = 1,
    val size: Int = 20,
)

val req = QueryRequest(index = "products", query = "laptop", page = 1)
```

```rust
// Rust: typed-builder crate
use typed_builder::TypedBuilder;

#[derive(TypedBuilder)]
pub struct QueryRequest {
    index: String,
    query: String,
    #[builder(default = 1)]
    page: u32,
    #[builder(default = 20)]
    size: u32,
}

let req = QueryRequest::builder()
    .index("products".into())
    .query("laptop".into())
    .build(); // compile error if required fields are missing
```

### 6. Flow<T> → Stream Processing

```kotlin
// Kotlin: Flow for cold async streams
fun searchUsers(query: String): Flow<User> = flow {
    emitAll(cache.search(query))
    if (query.length >= 3) {
        emitAll(api.search(query))
    }
}.flowOn(Dispatchers.IO)
```

```rust
// Rust: async_stream or futures::Stream
use async_stream::stream;
use futures::StreamExt;

fn search_users(query: &str, cache: &Cache, api: &Api) -> impl Stream<Item = Result<User, Error>> {
    let query = query.to_string();
    stream! {
        for user in cache.search(&query).await? {
            yield Ok(user);
        }
        if query.len() >= 3 {
            for user in api.search(&query).await? {
                yield Ok(user);
            }
        }
    }
}
```

### 7. Spring Boot @Service → Axum State Handler

```kotlin
// Kotlin: Spring Boot service with DI
@Service
class OrderService(
    private val orderRepo: OrderRepository,
    private val paymentGateway: PaymentGateway,
) {
    @Transactional
    suspend fun placeOrder(req: OrderRequest): Order {
        val order = orderRepo.save(req.toOrder())
        paymentGateway.charge(req.amount, req.token)
        return orderRepo.confirm(order.id)
    }
}

@RestController
class OrderController(private val service: OrderService) {
    @PostMapping("/orders")
    suspend fun create(@RequestBody req: OrderRequest) = service.placeOrder(req)
}
```

```rust
// Rust: explicit state with Axum
use axum::{Router, routing::post, extract::State, Json};
use sqlx::PgPool;

pub struct AppState {
    db: PgPool,
    payment: PaymentGateway,
}

async fn create_order(
    State(state): State<Arc<AppState>>,
    Json(req): Json<OrderRequest>,
) -> Result<Json<Order>, AppError> {
    let mut tx = state.db.begin().await?;
    let order = sqlx::query_as::<_, Order>(
        "INSERT INTO orders (...) VALUES (...) RETURNING *"
    )
    .bind(&req.customer_id)
    .fetch_one(&mut *tx)
    .await?;
    state.payment.charge(req.amount, &req.token).await?;
    sqlx::query("UPDATE orders SET confirmed = true WHERE id = $1")
        .bind(&order.id)
        .execute(&mut *tx)
        .await?;
    tx.commit().await?;
    Ok(Json(order))
}

// Router setup — no DI container needed
let state = Arc::new(AppState { db: pool, payment: gateway });
let app = Router::new()
    .route("/orders", post(create_order))
    .with_state(state);
```

## FFI & Incremental Migration

Kotlin/JVM-to-Rust migration typically proceeds service-by-service or via JNI for incremental replacement within the same JVM process.

| Strategy | Tool | When |
|----------|------|------|
| Service split at API boundary | HTTP/gRPC reverse proxy | Microservices already separated |
| JNI bridge | `jni` crate + `#[no_mangle] extern "C"` | Performance-critical library in monolith |
| Shared database migration | Schema-compatible read/write | Both read/write same DB during transition |
| Kotlin/Native interop | C ABI via `cinterop` | Kotlin/Native projects (KMM) |

### Migration Order

1. **Models & DTOs**: Define shared types via OpenAPI/Protobuf schema
2. **Pure functions**: Port stateless business logic first
3. **Read endpoints**: Rust service reads same database, route read traffic
4. **Write endpoints**: Migrate mutation handlers incrementally
5. **Event consumers**: Move Kafka consumers to Rust for throughput
6. **Tear down Kotlin**: Decommission JVM when all traffic hits Rust

## Common Mistakes

### Mistake 1: Overusing `clone()` Instead of Borrowing

```rust
// WRONG: Kotlin developers .clone() everything (GC mindset)
fn process(users: Vec<User>) -> Vec<Dto> {
    let mut results = Vec::new();
    for user in &users {
        let u = user.clone();  // unnecessary
        results.push(transform(u));
    }
    results
}

// CORRECT: borrow what you can
fn process(users: &[User]) -> Vec<Dto> {
    users.iter().map(|user| transform(user)).collect()
}
```

### Mistake 2: Treating `String` Like Kotlin's Nullable `String?`

```rust
// WRONG: wrapping everything in Option when not needed
fn get_name(user: &User) -> Option<String> {
    Some(user.name.clone()) // user.name is never null!
}

// CORRECT: Option only for truly optional values
fn get_name(user: &User) -> &str { &user.name }
fn find_user(id: &str) -> Option<User> { /* may not exist */ }
```

### Mistake 3: `Box<dyn Error>` as Catch-All

```rust
// WRONG: like Kotlin's `catch (e: Exception)`
fn handle(req: Request) -> Result<Response, Box<dyn std::error::Error>> { ... }

// CORRECT: structured errors with thiserror
#[derive(Error, Debug)]
enum ApiError {
    #[error("validation: {0}")]
    Validation(String),
    #[error("not found: {0}")]
    NotFound(String),
    #[error("internal error")]
    Internal(#[from] anyhow::Error),
}
```

### Mistake 4: Holding Locks Across `.await`

```kotlin
// Kotlin: Mutex.withLock { } is safe across suspend points
mutex.withLock { sharedState.update() } // coroutine-safe
```

```rust
// WRONG: holding std::sync::Mutex across .await — deadlock risk
// CORRECT: use tokio::sync::Mutex for async contexts
let guard = async_mutex.lock().await;
guard.update();
drop(guard); // explicit drop before next .await
```

### Mistake 5: Overusing `object` / Singleton Pattern

```kotlin
// Kotlin: object declaration — thread-safe lazy singleton
object AppConfig {
    val apiUrl: String by lazy { System.getenv("API_URL") ?: "http://localhost" }
    val maxRetries: Int = 3
}
```

```rust
// WRONG: trying to mirror Kotlin's object with global static mut
// static mut CONFIG: Option<Config> = None; // unsafe, not thread-safe!

// CORRECT: use OnceLock for lazy, thread-safe singletons
use std::sync::OnceLock;

static CONFIG: OnceLock<AppConfig> = OnceLock::new();

pub fn config() -> &'static AppConfig {
    CONFIG.get_or_init(|| AppConfig::from_env())
}

// For complex initialization: LazyLock (Rust 1.80+)
use std::sync::LazyLock;
static CONFIG: LazyLock<AppConfig> = LazyLock::new(|| AppConfig::from_env());
```

### Mistake 6: Expecting Reified Generics (Runtime Type Info)

```kotlin
// Kotlin: reified generics preserve type at runtime
inline fun <reified T> decode(json: String): T = Json.decodeFromString(json)

val user = decode<User>("""{"name":"Alice"}""") // T known at runtime
```

```rust
// Rust: generics are monomorphized at compile time — no runtime type info
// Use trait dispatch or serde's DeserializeOwned instead

// CORRECT: trait bounds for compile-time dispatch
fn decode<T: DeserializeOwned>(json: &str) -> Result<T, serde_json::Error> {
    serde_json::from_str(json)
}

// For truly dynamic types, use serde_json::Value or enum dispatch
fn decode_dynamic(json: &str) -> Result<Value, serde_json::Error> {
    serde_json::from_str(json)
}
```

## Reference Implementations

| Project | Description | Pattern |
|---------|-------------|---------|
| Dagger (Dagger SDK) | CI/CD engine; Kotlin SDK with Rust core | Kotlin → Rust at API boundary |
| Apollo GraphQL Router | GraphQL runtime; Kotlin gateway → Rust | Service-level migration |
| RisingWave | Streaming DB; replaces Kafka Streams (often Kotlin) | Full Rust rewrite pattern |
| Polars | DataFrame library; Kotlin DataFrame → Rust core | Rust core with language bindings |
| Meilisearch | Search engine; Kotlin clients, Rust core | Shared wire protocol |

## Cross-Reference

- **java-to-rust**: Shared JVM ecosystem patterns; Spring Boot migration parallels with Ktor
- **go-to-rust**: Goroutine/coroutine async patterns; shared M:N scheduling concepts
- **csharp-to-rust**: .NET/Kotlin enterprise patterns; async/await translation
- **swift-to-rust**: Mobile ecosystem counterpart (Android/Java ↔ iOS/Swift); shared modern language features
- **cpp-to-rust**: Kotlin/Native interop via C ABI; systems-level FFI strategies
