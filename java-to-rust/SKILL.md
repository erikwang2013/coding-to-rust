---
name: java-to-rust
description: Use when migrating Java codebases to Rust — covers JVM-to-native compilation, class/inheritance-to-trait composition, Spring Boot to Actix/Axum mapping, JPA/Hibernate to Diesel/sqlx translation, checked exception to Result conversion, and incremental migration strategy. Includes canonical signatures, common mistakes, and reference implementations.
---

# Java to Rust Migration

## Architecture Mapping

Java's JVM (Just-In-Time compilation, garbage collection, classloader hierarchy) maps to Rust's ahead-of-time compilation producing a single native binary. Where the JVM provides runtime reflection, dynamic class loading, and generational GC, Rust replaces all three with compile-time monomorphization, zero-cost abstractions, and ownership-based memory management. A Spring Boot fat JAR becomes a statically linked binary served by Actix-Web or Axum. Maven's POM dependency tree becomes Cargo's `Cargo.toml` with Cargo.lock ensuring reproducible builds. The Java module system (`module-info.java`) maps to crate-level visibility (`pub`, `pub(crate)`, `pub(super)`).

| Java Concept              | Rust Equivalent                         | Notes                                              |
|---------------------------|-----------------------------------------|----------------------------------------------------|
| JVM                       | Native binary (rustc + LLVM)            | AOT compilation, no warmup, no GC pauses            |
| Class                     | `struct` + `impl`                       | Data + behavior separated; no inheritance           |
| Abstract class            | Trait with default methods              | Partial implementation sharing                      |
| Interface                 | `trait`                                 | Explicit implementation needed                      |
| Inheritance               | Composition + trait delegation          | No class hierarchy; prefer enums for sealed sets    |
| Generics `<T>`            | Generics `<T: Trait>`                   | Monomorphization + trait bounds; no type erasure    |
| `Optional<T>`             | `Option<T>`                             | Exhaustive pattern matching; no `.get()` without check |
| `Stream<T>`               | `Iterator<Item = T>`                    | Lazy evaluation, same combinators, zero-cost        |
| `CompletableFuture<T>`    | `async fn` -> `impl Future`             | Stackless coroutines, poll-based                     |
| Checked exception         | `Result<T, E>`                          | No distinction between checked/unchecked             |
| `null`                    | `Option<T>`                             | Compiler-enforced null safety                        |
| `synchronized`            | `Mutex<T>` / `RwLock<T>`                | Data-inside-lock pattern                             |
| `Thread`                  | `std::thread::spawn` / `tokio::spawn`   | OS threads or async tasks                            |
| `ExecutorService`         | `tokio::runtime::Runtime`               | Thread pool + work-stealing scheduler                 |
| `volatile`                | `AtomicBool` / `Ordering::SeqCst`       | Explicit memory ordering                              |
| Annotation                 | `#[derive(...)]` / `#[attribute]`        | Compile-time code generation; no runtime reflection  |
| `enum` (Java, pre-sealed) | `enum` (Rust, algebraic)                | Rust enums carry data; exhaustive matching           |
| `record`                  | `struct` (plain data)                   | Both are immutable-by-convention data carriers       |
| `package`                 | `mod` / `pub mod`                       | Directory-organized modules with explicit visibility |
| `import`                  | `use`                                   | Path-qualified imports                                |
| ServiceLoader (SPI)       | `linkme` / manual registry              | No runtime service discovery; link-time alternatives |
| `final`                   | No direct equivalent                    | Rust variables are immutable by default              |
| `static` field            | `static` / `thread_local!`              | With const or lazy initialization                    |
| `instanceof`              | `match` / `Any::downcast_ref`           | Pattern matching preferred                            |

## Type System Mapping

| Java Type            | Rust Type                          | Notes                                              |
|----------------------|------------------------------------|----------------------------------------------------|
| `boolean`            | `bool`                             | Identical                                           |
| `byte`               | `i8`                               | Signed 8-bit                                        |
| `short`              | `i16`                              | Signed 16-bit                                       |
| `int`                | `i32`                              | 32-bit signed                                       |
| `long`               | `i64`                              | 64-bit signed                                       |
| `float`              | `f32`                              | IEEE 754 single                                     |
| `double`             | `f64`                              | IEEE 754 double                                     |
| `char`               | `char`                             | Java char is UTF-16 code unit; Rust char is Unicode scalar |
| `String`             | `String` / `&str`                  | Owned vs. borrowed                                   |
| `BigDecimal`         | `rust_decimal::Decimal`            | Fixed-point decimal; use `rust_decimal` crate        |
| `BigInteger`         | `num_bigint::BigInt`               | Arbitrary precision integer                          |
| `List<T>`            | `Vec<T>`                           | Contiguous growable array                            |
| `Set<T>`             | `HashSet<T>` / `BTreeSet<T>`       | Hash set vs. ordered set                             |
| `Map<K,V>`           | `HashMap<K,V>` / `BTreeMap<K,V>`   | Hash map vs. ordered map                             |
| `Queue<T>`           | `VecDeque<T>`                      | Double-ended queue                                   |
| `Stack<T>`           | `Vec<T>` (push/pop)                | Vec can serve as stack                               |
| `Array T[]`          | `[T; N]` / `Vec<T>`                | Fixed-size array vs. dynamically sized               |
| `enum` (Java)        | `enum` (Rust, full ADT)            | Rust enums hold variant-specific data                |
| `record`             | `struct`                           | Both provide value semantics                         |
| `Exception`          | `dyn Error` / custom error enum    | No checked exceptions; use `thiserror` for ergonomics |
| `Runnable`           | `FnOnce()` / `dyn Fn()`            | Closure trait                                       |
| `Callable<T>`        | `FnOnce() -> T` / `async { ... }`  | Closure returning value or Future                    |
| `Supplier<T>`        | `Fn() -> T` / `dyn Fn() -> T`     | Zero-argument closure                                |
| `Consumer<T>`        | `Fn(T)` / `dyn Fn(T)`             | Single-argument closure                              |
| `Function<T,R>`      | `Fn(T) -> R` / `dyn Fn(T) -> R`   | Single-argument mapping closure                      |

## Memory & Ownership Model

Java's heap-centric memory model (everything is a reference except primitives) is fundamentally different from Rust's stack-default approach. The garbage collector's tri-color mark-and-sweep is replaced by deterministic RAII. This is the hardest mental shift for Java developers.

### The Stack/Heap Inversion

```java
// Java: 对象总是在堆上分配，变量是引用
class User {
    private String name;      // 堆分配
    private int age;          // 内联在 User 对象中
    private List<Role> roles; // 堆分配，引用
}

// 所有 new User() 都在堆上
User user = new User("Alice", 30, List.of(...));
```

```rust
// Rust: 默认栈分配，堆需要显式 Box
struct User {
    name: String,     // String 数据在堆上（自有），结构体本身可在栈上
    age: u32,         // 内联，栈上
    roles: Vec<Role>, // Vec 数据在堆上，结构体本身可在栈上
}

// User 默认在栈上分配；如需堆分配使用 Box::new
let user = User { name: "Alice".into(), age: 30, roles: vec![] };
```

### Ownership Rules for Java Developers

| Java Pattern                                | Rust Translation                                     |
|---------------------------------------------|------------------------------------------------------|
| Pass object reference, modify in place      | `&mut T` (exclusive mutable borrow)                  |
| Pass object reference, read only            | `&T` (shared immutable borrow)                       |
| Multiple threads sharing mutable state      | `Arc<Mutex<T>>` or `Arc<RwLock<T>>`                  |
| Garbage collector cleans up unreachable     | `Drop` trait, scope-based cleanup, no finalizers     |
| `synchronized` block                        | `Mutex::lock()` returns `MutexGuard`, auto-unlock on drop |
| Lazy singleton via `getInstance()`          | `OnceLock<T>`, `LazyLock<T>`, or `lazy_static!`     |
| `ThreadLocal<T>`                            | `thread_local!` macro with `RefCell`                 |
| `WeakReference<T>`                          | `Weak<T>` (from `Arc`)                               |

### The GC-Free Mindset

```java
// Java: 信任 GC，随便分配
public List<Product> filterByCategory(List<Product> products, String category) {
    return products.stream()
        .filter(p -> p.getCategory().equals(category))
        .collect(Collectors.toList()); // 新列表，GC 会处理旧列表
}
```

```rust
// Rust: 明确所有权 —— 消耗原列表还是创建新列表？
fn filter_by_category(products: Vec<Product>, category: &str) -> Vec<Product> {
    // 消耗原 Vec，过滤后返回新 Vec（旧 Vec 被 drop）
    products
        .into_iter()  // 消耗原集合
        .filter(|p| p.category == category)
        .collect()    // 新集合
}

// 如果需要保留原列表，借用一个迭代器
fn filter_ref<'a>(products: &'a [Product], category: &str) -> Vec<&'a Product> {
    products
        .iter()       // 借用，不消耗
        .filter(|p| p.category == category)
        .collect()    // 返回引用集合
}
```

## Concurrency / Async Translation

Java's threading model (platform threads, virtual threads in Java 21+) maps to Rust's `tokio` async runtime. Virtual threads and goroutines share design philosophy; tokio provides work-stealing M:N scheduling.

### Thread / CompletableFuture -> Async/Await

```java
// Java: CompletableFuture 组合式异步
CompletableFuture<User> userFuture =
    CompletableFuture.supplyAsync(() -> fetchUser(id));

CompletableFuture<List<Order>> ordersFuture =
    CompletableFuture.supplyAsync(() -> fetchOrders(id));

String result = userFuture
    .thenCombine(ordersFuture, (user, orders) -> {
        return formatSummary(user, orders);
    })
    .exceptionally(ex -> "Error: " + ex.getMessage())
    .get(5, TimeUnit.SECONDS);
```

```rust
// Rust: async/await — 顺序看起来像同步代码
use tokio::time::{timeout, Duration};

let result = timeout(Duration::from_secs(5), async {
    let (user, orders) = tokio::join!(
        fetch_user(id),   // 并发执行
        fetch_orders(id), // 并发执行
    );
    format_summary(&user?, &orders?)
}).await
    .unwrap_or_else(|_| Err(anyhow::anyhow!("timeout")));
```

### Thread Pool -> Tokio Runtime

| Java                             | Rust / Tokio                       |
|----------------------------------|------------------------------------|
| `Executors.newFixedThreadPool(n)` | `tokio::runtime::Builder::new_multi_thread().worker_threads(n).build()` |
| `Executors.newCachedThreadPool()` | `tokio::runtime::Runtime::new()` (default multi-thread) |
| `Executors.newVirtualThreadPerTaskExecutor()` | `tokio::spawn` (lightweight tasks, M:N scheduling) |
| `Thread.sleep(ms)`               | `tokio::time::sleep(Duration::from_millis(ms)).await` |
| `synchronized(obj) { ... }`      | `let guard = mutex.lock().unwrap();` (data inside lock) |
| `CountDownLatch`                 | `tokio::sync::Barrier`             |
| `Semaphore`                      | `tokio::sync::Semaphore`           |
| `BlockingQueue`                  | `tokio::sync::mpsc::channel(n)`    |
| `Executors.newScheduledThreadPool` | `tokio::time::interval`           |

### Virtual Threads vs. Async Tasks

```java
// Java 21+: 虚拟线程 — 阻塞代码自动让出平台线程
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        var data = blockingApiCall(); // JVM 在阻塞时让出平台线程
        process(data);
    });
}
```

```rust
// Rust: 显式 async/await — 不阻塞系统线程
tokio::spawn(async move {
    let data = reqwest::get(url).await?.json().await?;
    // .await 点自动让出运行时线程
    process(data);
});

// 对不可避免的阻塞调用，使用 spawn_blocking
let data = tokio::task::spawn_blocking(|| {
    blocking_api_call() // 在专用阻塞线程池中运行
}).await?;
```

## Build System & Dependencies

| Java / Maven                     | Rust / Cargo                          |
|----------------------------------|---------------------------------------|
| `pom.xml`                        | `Cargo.toml`                          |
| `~/.m2/repository`               | `~/.cargo/registry/cache`             |
| Maven Central / `mvn install`    | crates.io / `cargo build`             |
| `mvn compile`                    | `cargo build`                         |
| `mvn test`                       | `cargo test`                          |
| `mvn package`                    | `cargo build --release`               |
| `mvn dependency:tree`            | `cargo tree`                          |
| `mvn verify`                     | `cargo test && cargo clippy`          |
| Maven multi-module               | Cargo workspace                       |
| `mvn spring-boot:run`            | `cargo run`                           |
| `mvn javadoc:javadoc`            | `cargo doc --open`                    |
| Checkstyle / SpotBugs            | `cargo clippy`                        |
| JaCoCo                           | `cargo tarpaulin`                     |
| JUnit 5                          | `#[test]` + `assert_eq!`             |
| Mockito                          | `mockall` / test doubles               |
| Testcontainers                   | `testcontainers` crate (same pattern) |
| Lombok                           | `#[derive(Debug, Clone, Serialize)]`  |
| MapStruct                        | `From` trait / `impl From<X> for Y`  |
| `spring-boot-starter-*`          | Feature-gated dependencies in `[features]` |

**Cargo.toml for a migrated Spring Boot service:**

```toml
[package]
name = "order-service"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "chrono"] }
diesel = { version = "2", features = ["postgres", "chrono"] }
anyhow = "1"
thiserror = "2"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json", "env-filter"] }
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1", features = ["v4", "serde"] }
validator = { version = "0.18", features = ["derive"] }
```

## Standard Library & Framework Mapping

### Spring Boot -> Axum / Actix-Web

| Spring Boot Component          | Rust Equivalent                          | Notes                                   |
|--------------------------------|------------------------------------------|-----------------------------------------|
| `@RestController`              | `axum::Router` + handler fn              | Function-based, not annotation-driven    |
| `@GetMapping("/path")`         | `Router::route("/path", get(handler))`   | Method + path in routing definition      |
| `@PostMapping`                 | `Router::route("/path", post(handler))`  | Type-safe extractors                     |
| `@RequestBody`                 | `Json<T>` extractor                      | Serde deserialization                    |
| `@PathVariable`                | `Path<T>` extractor                      | URL path parameter extraction            |
| `@RequestParam`                | `Query<T>` extractor                     | Query string deserialization             |
| `@Service`                     | Plain `struct` + `impl`                  | No framework annotation needed           |
| `@Repository`                  | `sqlx` query functions / Diesel schema   | Database abstraction                     |
| `@Autowired` / `@Inject`       | Manual DI / constructor injection        | No runtime DI container                  |
| `@Transactional`               | `sqlx::Transaction` / explicit tx mgmt   | Explicit, not annotation-driven          |
| `@Valid`                       | `validator` crate / `axum::extract` validation | Compile-time or extractor-based    |
| `application.properties`       | `.env` / config crate / `figment`        | Configuration management                 |
| `@Scheduled`                   | `tokio::time::interval` + `tokio::spawn` | Explicit scheduling loop                 |
| `@Async`                       | `tokio::spawn(async { ... })`            | Explicit async task creation             |
| `@ExceptionHandler`            | `IntoResponse` impl for error types      | Error type -> HTTP response mapping      |
| Spring Security                | `tower::ServiceBuilder::layer` + middleware | Layered, composable security middleware |
| `@EventListener`               | `tokio::sync::broadcast` channel         | In-process event bus                     |
| `@Value("${prop}")`            | `std::env::var("PROP")` / `dotenvy`      | Environment variable injection           |
| Actuator `/health`             | Custom health-check handler              | No built-in, easy to implement           |

### JPA / Hibernate -> Diesel / sqlx

| JPA / Hibernate                | Diesel                                   | sqlx                                    |
|--------------------------------|------------------------------------------|-----------------------------------------|
| `@Entity`                      | `#[derive(Queryable, Insertable)]`       | `#[derive(sqlx::FromRow)]`              |
| `@Id` / `@GeneratedValue`      | `#[diesel(id)]` / SERIAL column          | Schema-defined, query-driven            |
| `@Column`                      | `#[diesel(column_name = "...")]`         | Field name matches column by default    |
| `@OneToMany`                   | Join query / `belongs_to` association    | Manual JOIN query with `sqlx::query_as` |
| `@ManyToOne`                   | `#[diesel(belongs_to(Parent))]`         | Foreign key field + JOIN                |
| `@ManyToMany`                  | Join table query                         | Manual join table query                 |
| `EntityManager.find()`         | `table.find(id).first(&mut conn)?`       | `sqlx::query_as("SELECT ... WHERE id = $1").fetch_one(&pool)` |
| JPQL / HQL                     | Raw SQL or `diesel::dsl` builder         | Typed SQL with `sqlx::query!` macro      |
| `@Transactional`               | `conn.transaction(|| { ... })`           | `pool.begin().await?` / explicit scope  |
| Lazy loading                   | N+1 prevention via `.load()`             | Manual eager loading with JOINs          |
| Migrations (Flyway/Liquibase)  | `diesel_migrations`                      | `sqlx migrate` CLI                      |

### Java Standard Library -> Rust

| Java stdlib                     | Rust std / crate                        |
|---------------------------------|-----------------------------------------|
| `java.util.Optional`            | `std::option::Option<T>`                |
| `java.util.stream.Stream`       | `std::iter::Iterator`                   |
| `java.util.Collections`         | `std::collections` / `itertools`        |
| `java.time.LocalDateTime`       | `chrono::NaiveDateTime`                 |
| `java.time.ZonedDateTime`       | `chrono::DateTime<Utc>`                 |
| `java.util.UUID`                | `uuid::Uuid`                            |
| `java.math.BigDecimal`          | `rust_decimal::Decimal`                 |
| `java.util.concurrent.locks`    | `std::sync` (Mutex, RwLock, Condvar)     |
| `java.util.regex.Pattern`       | `regex::Regex`                          |
| `java.util.Base64`              | `base64` crate                          |
| `java.security.MessageDigest`   | `sha2` / `md-5` / `ring`                |
| `java.util.logging` / SLF4J     | `tracing` / `log`                       |
| `java.util.Properties`          | `std::env` / `dotenvy`                  |
| `java.nio.file.Files`           | `std::fs`                               |
| `java.net.http.HttpClient`      | `reqwest`                               |

## Canonical Patterns

### 1. Interface -> Trait

```java
// Java: 接口定义契约
public interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
    boolean supports(PaymentMethod method);
}

public class StripeProcessor implements PaymentProcessor {
    @Override
    public PaymentResult process(PaymentRequest request) {
        // Stripe 实现
    }

    @Override
    public boolean supports(PaymentMethod method) {
        return method == PaymentMethod.CREDIT_CARD;
    }
}
```

```rust
// Rust: trait 定义行为契约，impl 显式实现
pub trait PaymentProcessor {
    fn process(&self, request: &PaymentRequest) -> Result<PaymentResult, PaymentError>;
    fn supports(&self, method: PaymentMethod) -> bool;
}

pub struct StripeProcessor {
    api_key: String,
    client: reqwest::Client,
}

impl PaymentProcessor for StripeProcessor {
    fn process(&self, request: &PaymentRequest) -> Result<PaymentResult, PaymentError> {
        // Stripe 实现——self 方法自动借用
    }

    fn supports(&self, method: PaymentMethod) -> bool {
        matches!(method, PaymentMethod::CreditCard)
    }
}

// 静态分发：编译期单态化，零运行时开销
fn process_payment(processor: &impl PaymentProcessor, req: &PaymentRequest) {
    processor.process(req);
}

// 动态分发：需要异构集合
fn process_batch(processors: &[&dyn PaymentProcessor], req: &PaymentRequest) {
    for p in processors {
        p.process(req);
    }
}
```

### 2. Checked Exception -> Result

```java
// Java: checked exception 强制声明
public Order placeOrder(OrderRequest req)
    throws InsufficientStockException,
           PaymentFailedException,
           InvalidAddressException {

    if (!inventory.checkStock(req.getItemId(), req.getQuantity())) {
        throw new InsufficientStockException(req.getItemId());
    }

    PaymentResult payment = paymentGateway.charge(req.getPayment());
    if (!payment.isSuccess()) {
        throw new PaymentFailedException(payment.getError());
    }

    return orderRepository.save(new Order(req, payment));
}
```

```rust
// Rust: Result 携带成功或错误，? 运算符传播
use thiserror::Error;

#[derive(Error, Debug)]
pub enum OrderError {
    #[error("insufficient stock for item {0}")]
    InsufficientStock(String),

    #[error("payment failed: {0}")]
    PaymentFailed(String),

    #[error("invalid shipping address")]
    InvalidAddress,

    #[error("database error: {0}")]
    DatabaseError(#[from] sqlx::Error),
}

fn place_order(req: &OrderRequest) -> Result<Order, OrderError> {
    inventory.check_stock(&req.item_id, req.quantity)
        .map_err(|_| OrderError::InsufficientStock(req.item_id.clone()))?;

    let payment = payment_gateway
        .charge(&req.payment)
        .map_err(|e| OrderError::PaymentFailed(e.to_string()))?;

    let order = order_repo.save(Order::new(req, payment))?;

    Ok(order)
}
```

### 3. Stream -> Iterator

```java
// Java: Stream API — 声明式集合处理
List<Invoice> overdueInvoices = invoices.stream()
    .filter(inv -> inv.getDueDate().isBefore(LocalDate.now()))
    .filter(inv -> inv.getStatus() == Status.SENT)
    .sorted(Comparator.comparing(Invoice::getDueDate))
    .limit(50)
    .collect(Collectors.toList());
```

```rust
// Rust: Iterator — 同样的组合模式，延迟计算
use itertools::Itertools;

let overdue_invoices: Vec<&Invoice> = invoices
    .iter()                                    // 借用迭代
    .filter(|inv| inv.due_date < today)
    .filter(|inv| inv.status == Status::Sent)
    .sorted_by_key(|inv| inv.due_date)         // itertools 提供
    .take(50)
    .collect();
```

### 4. Spring Service -> Feature-Gated Module

```java
// Java: Spring 服务 —— DI 容器管理
@Service
@Transactional
public class OrderService {
    @Autowired
    private OrderRepository orderRepo;

    @Autowired
    private PaymentGateway paymentGateway;

    @Autowired
    private NotificationService notificationService;

    public OrderDTO createOrder(CreateOrderRequest req) {
        // 业务逻辑
    }
}
```

```rust
// Rust: 无 DI 容器 —— 构造函数注入 + 接口 trait
pub struct OrderService<P: PaymentProcessor, N: Notifier> {
    order_repo: OrderRepository,
    payment_gateway: P,
    notifier: N,
}

impl<P: PaymentProcessor, N: Notifier> OrderService<P, N> {
    pub fn new(repo: OrderRepository, payment: P, notifier: N) -> Self {
        Self { order_repo: repo, payment_gateway: payment, notifier }
    }

    pub async fn create_order(
        &self,
        req: CreateOrderRequest,
    ) -> Result<OrderDto, OrderError> {
        // 显式事务管理
        let mut tx = self.order_repo.begin().await?;
        let order = tx.insert_order(&req).await?;
        self.payment_gateway.charge(&req.payment).await?;
        tx.commit().await?;
        self.notifier.notify(OrderCreated { id: order.id }).await?;
        Ok(order.into())
    }
}
```

### 5. Builder Pattern

```java
// Java: Lombok @Builder 或手动 Builder
@Builder
@Data
public class QueryRequest {
    private String index;
    private String query;
    private int page;
    private int size;
    private List<String> filters;
}

// 使用
QueryRequest req = QueryRequest.builder()
    .index("products")
    .query("laptop")
    .page(1)
    .size(20)
    .build();
```

```rust
// Rust: typed-builder crate 或 derive_builder
use typed_builder::TypedBuilder;

#[derive(TypedBuilder)]
pub struct QueryRequest {
    index: String,
    query: String,
    #[builder(default = 1)]
    page: u32,
    #[builder(default = 20)]
    size: u32,
    #[builder(default)]
    filters: Vec<String>,
}

// 使用——编译期保证所有必填字段已设置
let req = QueryRequest::builder()
    .index("products".into())
    .query("laptop".into())
    .page(1)
    .build(); // 忘记 .index() 会导致编译错误
```

## FFI & Incremental Migration

Java-to-Rust migration typically happens service-by-service or via JNI for incremental replacement. For HTTP/REST services, the recommended path is to migrate at the network boundary.

### Migration Strategies

| Strategy                | When to Use                                | Risk      |
|-------------------------|--------------------------------------------|-----------|
| Service split           | Microservices already separated            | Low       |
| JNI bridge              | Performance-critical library in monolith   | Medium    |
| gRPC boundary           | Services communicate via gRPC              | Low       |
| Database-level          | Both read/write same DB during transition  | Medium    |
| Wire-compatible reimpl  | Rebuild HTTP API with identical contract   | Low-Medium |

### JNI Bridge (Rust library loaded by JVM)

```rust
// Rust 侧：导出为 cdylib
use jni::JNIEnv;
use jni::objects::JClass;
use jni::sys::jstring;

#[no_mangle]
pub extern "system" fn Java_com_example_RustBridge_compute(
    mut env: JNIEnv,
    _class: JClass,
    input: jstring,
) -> jstring {
    let input: String = env.get_string(&input.into()).unwrap().into();
    let result = compute_heavy(input); // Rust 高性能计算
    env.new_string(result).unwrap().into_raw()
}
```

```java
// Java 侧：加载 native library
public class RustBridge {
    static { System.loadLibrary("rust_core"); }

    public static native String compute(String input);
}
```

### Service-by-Service Migration Path

1. **Models & DTOs**: Define shared protobuf / OpenAPI schema. Both Java and Rust consume.
2. **Read endpoints first**: Build Rust service that reads from the same database. Route read traffic.
3. **Write endpoints**: Migrate mutation handlers, ensuring transactional consistency.
4. **Event consumers**: Move Kafka/RabbitMQ consumers to Rust for throughput-sensitive workloads.
5. **Tear down Java**: Once all traffic hits Rust, decommission the Java service.

## Common Mistakes

### Mistake 1: Over-Engineering the DI Container

```rust
// 错误：Java 开发者手动实现 DI 容器
struct Container {
    user_service: Arc<UserService>,
    order_service: Arc<OrderService>,
    // ... 50 more fields
}

// 正确：简单的构造函数注入或启动时组装
fn main() {
    let db_pool = PgPoolOptions::new().connect(&db_url).await.unwrap();
    let payment = StripeGateway::new(&config);
    let notifier = EmailNotifier::new(&config);

    let order_service = OrderService::new(
        OrderRepository::new(db_pool.clone()),
        payment,
        notifier,
    );

    let app_state = AppState {
        order_service: Arc::new(order_service),
        db: db_pool,
    };
    // 传给 axum Router
}
```

### Mistake 2: Catch-All Error Handling

```rust
// 错误：捕获所有错误为字符串（类似 catch(Exception e)）
fn process() -> Result<(), String> {
    let data = read_file().map_err(|e| e.to_string())?;
    let parsed = parse(&data).map_err(|e| e.to_string())?;
    save(parsed).map_err(|e| e.to_string())?;
    Ok(())
}

// 正确：定义有意义的错误类型
#[derive(Error, Debug)]
enum ProcessError {
    #[error("file read failed: {0}")]
    Io(#[from] std::io::Error),
    #[error("parse failed: {0}")]
    Parse(String),
    #[error("save failed: {0}")]
    Database(#[from] sqlx::Error),
}

fn process() -> Result<(), ProcessError> {
    let data = read_file()?;    // Io 错误自动转换
    let parsed = parse(&data)?; // Parse 错误手动映射
    save(parsed)?;              // DB 错误自动转换
    Ok(())
}
```

### Mistake 3: Forgetting Rust's Move Semantics

```rust
// 错误：Java 开发者在循环中无意移动所有权
let users: Vec<User> = load_users();
for user in users {
    process(user);       // user 被移动到 process()
}
// process_again(&users); // 编译错误：users 已经被消耗

// 正确：明确你要借用还是消耗
let users: Vec<User> = load_users();
for user in &users {    // 借用迭代
    process_ref(user);   // 传递引用
}
// users 仍然可用
```

### Mistake 4: Over-Using Box<dyn Error>

```rust
// 错误：类似 Java 的 throws Exception，丢失类型信息
fn handle_request(req: Request) -> Result<Response, Box<dyn std::error::Error>> {
    // ...
}

// 正确：使用 thiserror 定义具体错误类型，保留类型信息
#[derive(Error, Debug)]
enum ApiError {
    #[error("validation failed: {0}")]
    Validation(String),
    #[error("not found: {0}")]
    NotFound(String),
    #[error("internal: {0}")]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, msg) = match &self {
            ApiError::Validation(_) => (StatusCode::BAD_REQUEST, self.to_string()),
            ApiError::NotFound(_) => (StatusCode::NOT_FOUND, self.to_string()),
            ApiError::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "internal error".into()),
        };
        (status, Json(json!({ "error": msg }))).into_response()
    }
}
```

## Reference Implementations

| Project                         | Description                                             | Java LOC | Rust LOC |
|---------------------------------|---------------------------------------------------------|----------|----------|
| Apache DataFusion               | SQL query engine; compare to Apache Spark's Catalyst    | ~200k    | ~150k    |
| Oso (authorization engine)      | Polar language core rewritten from Java to Rust         | ~12k     | ~10k     |
| Sled (embedded DB)              | SQLite-style embedded database; Java developers often port RocksDB wrappers | N/A | ~20k |
| Meilisearch                     | Search engine; Java consumers replaced with Rust core   | ~30k     | ~25k     |
| Tantivy                         | Full-text search; Lucene-equivalent in Rust             | N/A      | ~60k     |
| RisingWave                      | Streaming database; replacement for Kafka Streams/Flink | N/A      | ~200k    |
| Polars                          | DataFrame library; pandas/Spark DataFrame in Rust       | N/A      | ~150k    |
| Turbopack (Vercel)              | Rust-based successor to webpack (JS, but Java monoliths similar) | N/A | ~80k |

## Cross-Reference

- **go-to-rust**: Goroutine/async patterns; shared M:N scheduling concepts with virtual threads
- **csharp-to-rust**: .NET runtime mapping; shared enterprise service patterns with Spring Boot migration
- **cpp-to-rust**: Systems-level JNI/native interop strategies
