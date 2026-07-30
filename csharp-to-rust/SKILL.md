---
name: csharp-to-rust
description: Use when migrating C# (.NET) codebases to Rust — covers CLR to native binary, async/Task to tokio, LINQ to Iterator combinators, ASP.NET to Axum/Actix, EF Core to Diesel/sqlx, and incremental replacement via platform invoke. Includes canonical code patterns, common mistakes, and reference implementations.
updated: 2026-07-30
---

# C# to Rust Migration

## Architecture Mapping

.NET's CLR (JIT compilation, generational GC, assembly loading) maps to Rust's AOT-compiled native binary with no runtime. The .NET garbage collector is replaced by ownership and RAII. C# `async`/`await` (which compiles to state-machine structs on the heap via `ValueTask`/`Task`) maps to Rust's stackless coroutines via `tokio` or `async-std`. ASP.NET Core's middleware pipeline becomes Axum's `tower::Layer` stack or Actix-Web's middleware. The NuGet package ecosystem maps to crates.io with Cargo. .NET solution files (`.sln`) become Cargo workspace members.

| C# Concept                 | Rust Equivalent                         | Notes                                              |
|----------------------------|-----------------------------------------|----------------------------------------------------|
| CLR / CoreCLR              | Native binary (rustc)                   | AOT compilation, no JIT warmup                     |
| GC                         | Ownership + RAII                        | Deterministic cleanup, no GC pauses                |
| `class`                    | `struct` + `impl`                       | No inheritance; composition preferred              |
| `interface`                | `trait`                                 | Explicit implementation, no default interface methods (stabilizing in future editions) |
| `abstract class`           | Trait with default methods              | Partial implementation sharing                     |
| `struct` (value type)      | `struct` + `Copy`                       | C# structs are value types; Rust structs can be too |
| `record` / `record struct` | `#[derive(Clone, Debug, PartialEq)]`    | Immutable data with value equality                 |
| `enum` (C# simple)         | `enum` (Rust, tagged union)             | Rust enums carry variant-specific data             |
| Generic `<T>`              | Generic `<T: Trait>`                    | Monomorphized, no type erasure                     |
| `Nullable<T>` / `T?`       | `Option<T>`                             | Explicit absence, no null reference types          |
| `null` (reference types)   | `Option<T>` / `&T` (non-null by default)| All references are non-null by default             |
| `async` / `await` / `Task` | `async fn` -> `impl Future`             | Stackless coroutines, poll-based                   |
| `Task<T>`                  | `impl Future<Output = T>`               | No `Task` wrapping; return value directly          |
| `CancellationToken`        | `tokio_util::sync::CancellationToken`   | Cooperative cancellation                           |
| `LINQ`                     | `Iterator` combinators                  | Lazy chains; `map`, `filter`, `flat_map`, `fold`   |
| Property (`{ get; set; }`) | Getter/setter methods or pub field      | No language-level property syntax                  |
| `event` / delegate         | Channel / callback / `Fn` traits        | Multiple patterns; no multicast built-in           |
| `IDisposable`              | `Drop` trait                            | Deterministic cleanup at scope exit                |
| `using` statement          | Scope guard + RAII                      | Automatic Drop at end of scope                     |
| `IAsyncDisposable`         | Async cleanup in explicit method        | Handle async cleanup in `Drop` or explicit methods |
| `lock` statement           | `Mutex::lock()` or `RwLock::write()`    | Data-inside-lock pattern                           |
| `Thread`                   | `std::thread::spawn`                    | OS threads                                          |
| `Task.Run`                 | `tokio::spawn`                          | Async task on runtime pool                         |
| `Parallel.ForEach`         | `rayon::par_iter`                       | Data parallelism via work-stealing                  |
| `[Attribute]`              | `#[derive(...)]` / proc macros          | Compile-time code generation                       |
| `namespace`                | `mod` / `pub mod`                       | Module system; no namespace nesting by convention   |
| `partial class`            | Multiple `impl` blocks                  | Split implementations across files                 |
| `extension method`         | Extension trait pattern                 | `impl MyExt for T { fn my_method(&self) { ... } }` |
| `ref` / `out` parameters   | `&mut T` returns via tuple              | Prefer return values over out parameters           |
| `dynamic`                  | `serde_json::Value` / `Any`             | Typed code preferred; JSON for schemaless data      |
| `System.Reflection`        | `std::any::Any` / `typetag`             | Minimal runtime type info; prefer generics         |
| `string`                   | `String` / `&str`                       | Owned vs. borrowed; C# strings are heap-allocated   |
| `StringBuilder`            | `String::push_str` / `format!`          | Mutable string building                            |

## Type System Mapping

| C# Type                   | Rust Type                          | Notes                                              |
|---------------------------|------------------------------------|----------------------------------------------------|
| `bool`                    | `bool`                             | Identical                                           |
| `byte` / `sbyte`          | `u8` / `i8`                        | Unsigned / signed byte                              |
| `short` / `ushort`        | `i16` / `u16`                      | 16-bit integers                                     |
| `int` / `uint`            | `i32` / `u32`                      | 32-bit integers                                     |
| `long` / `ulong`          | `i64` / `u64`                      | 64-bit integers                                     |
| `float`                   | `f32`                              | IEEE 754 single                                     |
| `double`                  | `f64`                              | IEEE 754 double                                     |
| `decimal`                 | `rust_decimal::Decimal`            | 128-bit fixed-point from `rust_decimal` crate       |
| `char`                    | `char`                             | Unicode scalar value (4 bytes in Rust)              |
| `string`                  | `String` / `&str`                  | Owned (heap) vs. borrowed reference                 |
| `object`                  | `Box<dyn Any>`                     | Dynamically typed; rarely needed in Rust            |
| `DateTime`                | `chrono::DateTime<Utc>`            | Timezone-aware; `chrono::NaiveDateTime` for local   |
| `DateTimeOffset`          | `chrono::DateTime<FixedOffset>`    | Offset-aware datetime                               |
| `TimeSpan`                | `std::time::Duration`              | Time interval                                       |
| `Guid`                    | `uuid::Uuid`                       | 128-bit unique identifier                           |
| `int[]`                   | `Vec<i32>` / `[i32; N]`            | Dynamic vs. fixed-size                              |
| `List<T>`                 | `Vec<T>`                           | Contiguous growable array                            |
| `Dictionary<K,V>`         | `HashMap<K,V>` / `BTreeMap<K,V>`   | Hash vs. ordered map                                |
| `HashSet<T>`              | `HashSet<T>`                       | Unordered unique set                                |
| `SortedSet<T>`            | `BTreeSet<T>`                      | Ordered unique set                                  |
| `Queue<T>`                | `VecDeque<T>`                      | Double-ended queue                                  |
| `Stack<T>`                | `Vec<T>` (used as stack)           | Push/pop operations                                 |
| `LinkedList<T>`           | `std::collections::LinkedList<T>`  | Doubly-linked; rarely used                          |
| `ConcurrentDictionary`    | `dashmap::DashMap`                 | Concurrent hash map from `dashmap` crate            |
| `ConcurrentQueue`         | `crossbeam::queue::SegQueue`       | Lock-free MPMC queue                                |
| `Channel<T>`              | `tokio::sync::mpsc::channel`       | Async bounded/unbounded channel                     |
| `Tuple<...>`              | `(T1, T2, ...)`                    | Anonymous struct; native syntax                     |
| `ValueTuple`              | `(T1, T2, ...)`                    | Native tuple (always value-like)                    |
| `Action` / `Action<T>`    | `FnOnce()` / `dyn Fn(T)`           | Closure trait object                                |
| `Func<T,R>`               | `Fn(T) -> R` / `dyn Fn(T) -> R`    | Mapping closure                                      |
| `Predicate<T>`            | `Fn(&T) -> bool`                   | Predicate closure                                   |

## Memory & Ownership Model

C# splits types into reference types (heap-allocated, garbage collected) and value types (stack-allocated or inline). Rust collapses this to a single model: everything is a value, and references are explicit borrows tracked by the compiler.

### Reference Types -> Ownership

```csharp
// C#: class 是引用类型，默认在堆上分配
public class Order
{
    public string Id { get; set; }          // string 是引用类型
    public List<LineItem> Items { get; set; } // List 是引用类型
    public decimal Total { get; set; }      // decimal 是值类型
}

// var order = new Order(); // 堆分配
// 变量 order 是指向堆对象的引用
```

```rust
// Rust: struct 默认栈分配；堆数据通过 String/Vec 间接引用
pub struct Order {
    pub id: String,            // String 的数据在堆上，结构体字段在栈上
    pub items: Vec<LineItem>,  // Vec 的数据在堆上
    pub total: rust_decimal::Decimal,
}

// let order = Order { ... }; // 栈分配
// 如需堆分配：let order = Box::new(Order { ... });
```

### Nullable Reference Types -> Option

```csharp
// C# 8+: 可空引用类型
#nullable enable

public class User
{
    public string Name { get; set; }       // 不可空
    public string? MiddleName { get; set; } // 可空
    public Address? Address { get; set; }  // 可为 null
}

// 使用时需检查 null
if (user.Address is not null)
{
    Console.WriteLine(user.Address.City);
}
```

```rust
// Rust: 所有引用类型都是不可空的，用 Option<T> 替代可空语义
pub struct User {
    pub name: String,                  // 始终有效
    pub middle_name: Option<String>,   // 可能不存在
    pub address: Option<Address>,      // 可能不存在
}

// 模式匹配确保安全访问
if let Some(ref addr) = user.address {
    println!("{}", addr.city);
}
```

### IDisposable -> Drop

```csharp
// C#: IDisposable + using 语句
public class DatabaseConnection : IDisposable
{
    private SqlConnection _conn;

    public void Dispose()
    {
        _conn?.Dispose();
        Console.WriteLine("connection closed");
    }
}

// 自动调用 Dispose
using var db = new DatabaseConnection();
db.Query("SELECT ...");
// Dispose 在此调用
```

```rust
// Rust: Drop trait —— 确定性析构
pub struct DatabaseConnection {
    conn: sqlx::PgConnection,
}

impl Drop for DatabaseConnection {
    fn drop(&mut self) {
        // conn 在离开作用域时自动关闭
        tracing::info!("connection closed");
    }
}

{
    let db = DatabaseConnection::new();
    db.query("SELECT ...");
} // Drop::drop 在此自动调用
```

## Concurrency / Async Translation

C# `async`/`await` with `Task<T>` and `ValueTask<T>` is conceptually close to Rust's `async`/`await`. Both are stackless coroutines compiled to state machines. The difference: C# tasks are reference types (heap-allocated) by default; Rust futures are value types (stack-allocated or on-stack until `.await`ed).

### Task -> Future

```csharp
// C#: async/await with Task<T>
public async Task<Order> GetOrderAsync(int orderId)
{
    var order = await _db.Orders.FindAsync(orderId);
    var items = await _db.Items
        .Where(i => i.OrderId == orderId)
        .ToListAsync();
    order.Items = items;
    return order;
}

// 并发执行多个任务
var (user, orders) = await (
    GetUserAsync(id),
    GetOrdersAsync(id)
);
```

```rust
// Rust: async/await —— 语法几乎相同
async fn get_order(pool: &PgPool, order_id: i32) -> Result<Order, sqlx::Error> {
    let order = sqlx::query_as::<_, Order>("SELECT * FROM orders WHERE id = $1")
        .bind(order_id)
        .fetch_one(pool)
        .await?;

    let items = sqlx::query_as::<_, LineItem>("SELECT * FROM items WHERE order_id = $1")
        .bind(order_id)
        .fetch_all(pool)
        .await?;

    Ok(Order { items, ..order })
}

// 并发执行 —— tokio::join! 同样支持 tuple 模式
let (user, orders) = tokio::join!(
    get_user(&pool, id),
    get_orders(&pool, id),
);
```

### Task Parallel Library -> Rayon / Tokio

| C# (TPL)                               | Rust                                       |
|----------------------------------------|--------------------------------------------|
| `Parallel.For(0, N, i => { ... })`     | `(0..N).into_par_iter().for_each(|i| {})`  |
| `Parallel.ForEach(items, item => {})`  | `items.par_iter().for_each(|item| {})`     |
| `Task.WhenAll(tasks)`                  | `futures::future::join_all(tasks)`         |
| `Task.WhenAny(tasks)`                  | `tokio::select!` / `futures::select_ok`    |
| `Task.Delay(ms)`                       | `tokio::time::sleep(Duration::from_millis(ms))` |
| `Task.FromResult(val)`                 | `async { val }` / `std::future::ready(val)` |
| `Task.CompletedTask`                   | `std::future::ready(())`                    |
| `SemaphoreSlim`                        | `tokio::sync::Semaphore`                    |
| `ManualResetEvent`                     | `tokio::sync::Notify`                       |
| `ConcurrentBag<T>`                     | `crossbeam::queue::SegQueue`               |
| `BlockingCollection<T>`                | `tokio::sync::mpsc::channel`               |

### Thread vs. Async Task

```csharp
// C#: Task.Run 将工作放入线程池
var result = await Task.Run(() => HeavyComputation(data));

// C#: 后台线程
var thread = new Thread(() => { ... });
thread.Start();
```

```rust
// Rust: 对于 CPU 密集工作，用 spawn_blocking 避免阻塞异步运行时
let result = tokio::task::spawn_blocking(move || {
    heavy_computation(data)
}).await?;

// Rust: 专用 OS 线程
std::thread::spawn(move || {
    // 长期运行的后台工作
});
```

## Build System & Dependencies

| .NET / NuGet                         | Rust / Cargo                          |
|--------------------------------------|---------------------------------------|
| `.csproj` / `.sln`                   | `Cargo.toml` / workspace              |
| `dotnet build`                       | `cargo build`                         |
| `dotnet test`                        | `cargo test`                          |
| `dotnet run`                         | `cargo run`                           |
| `dotnet publish`                     | `cargo build --release`               |
| `dotnet restore`                     | `cargo fetch`                         |
| `dotnet watch`                       | `cargo watch`                         |
| `dotnet format`                      | `cargo fmt`                           |
| `dotnet new`                         | `cargo new / cargo init`              |
| NuGet (`<PackageReference>`)         | crates.io (`[dependencies]`)          |
| `nuget.config`                       | `.cargo/config.toml`                  |
| `global.json`                        | `rust-toolchain.toml`                 |
| `packages.lock.json`                 | `Cargo.lock`                          |
| `<ProjectReference>`                 | Cargo workspace member dependency     |
| `dotnet user-secrets`                | `.env` / `dotenvy`                    |
| `appsettings.json`                   | `config.toml` / `serde` deserialization |
| `dotnet ef migrations`               | `diesel migration run` / `sqlx migrate run` |

**Cargo.toml for a migrated ASP.NET Core Web API:**

```toml
[package]
name = "web-api"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.7", features = ["macros"] }
tokio = { version = "1", features = ["full"] }
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace", "compression-gzip"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "chrono", "uuid"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
anyhow = "1"
thiserror = "2"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json", "env-filter"] }
dotenvy = "0.15"
```

## Standard Library & Framework Mapping

### ASP.NET Core -> Axum / Actix-Web

| ASP.NET Core                          | Rust / Axum                           | Notes                                   |
|---------------------------------------|---------------------------------------|-----------------------------------------|
| `ControllerBase`                      | `async fn handler(...) -> impl IntoResponse` | Function-based handlers               |
| `[HttpGet]` / `[HttpPost]`            | `Router::route("/", get(handler)).route("/", post(handler))` | Method chaining on Router |
| `[FromBody]`                          | `Json<T>` extractor                   | Serde-based deserialization             |
| `[FromRoute]`                         | `Path<T>` extractor                   | Path parameter extraction               |
| `[FromQuery]`                         | `Query<T>` extractor                  | Query string deserialization            |
| `[FromHeader]`                        | `HeaderMap` extractor                 | Custom header access                    |
| `IActionResult` / `ActionResult<T>`   | `impl IntoResponse`                   | Trait-based response conversion         |
| `ProblemDetails`                      | `(StatusCode, Json<ErrorBody>)`       | Custom error response; no built-in RFC 7807 |
| Middleware                            | `tower::Layer` / `axum::middleware`   | Layered, composable middleware           |
| `IAuthorizationFilter`                | `tower::ServiceBuilder::layer`        | Middleware-based auth                    |
| `IExceptionFilter`                    | Error type implementing `IntoResponse` | Centralized error mapping               |
| `appsettings.json` + `IOptions<T>`    | `serde` + `figment` / config crate    | Type-safe configuration                  |
| `ILogger<T>`                          | `tracing::instrument` + `tracing::Span` | Structured, span-based logging         |
| `IHostedService` / `BackgroundService` | `tokio::spawn` in main               | Async background task at startup        |
| gRPC service                          | `tonic`                               | Protobuf-based RPC                      |
| SignalR                               | WebSocket via `axum::extract::ws`     | Raw WebSocket, no high-level framework  |
| Swashbuckle / Swagger                 | `utoipa` / manual OpenAPI             | `utoipa` provides derive macros         |
| Health Checks                         | Custom health endpoint                | Simple to implement manually            |
| `System.Text.Json`                    | `serde_json`                          | Serialization / deserialization         |
| `System.Text.Json.Serialization`      | `#[serde(rename_all = "camelCase")]`  | Attribute-based customization           |

### Entity Framework Core -> Diesel / sqlx

| EF Core                            | Diesel / sqlx                             | Notes                                   |
|------------------------------------|-------------------------------------------|-----------------------------------------|
| `DbContext`                        | `PgPool` (sqlx) / `PgConnection` (diesel) | Connection pool vs. single connection   |
| `DbSet<T>`                         | Query functions on schema / `query_as::<T>` | No `DbSet` abstraction                |
| `modelBuilder.Entity<T>()`         | `table!` macro (diesel) / query structs   | Schema definition                        |
| `.Include(e => e.Nav)`             | `.load()` with JOIN manually (diesel)      | No automatic eager loading; manual JOIN |
| `.Where(e => ...)`                 | `.filter(...)`                             | Identical concept                       |
| `.Select(e => new { ... })`        | `.select(...)`                             | Projection                              |
| `.FirstOrDefaultAsync()`           | `.fetch_optional(&pool).await`             | Returns `Option<T>`                     |
| `.ToListAsync()`                   | `.fetch_all(&pool).await`                  | Returns `Vec<T>`                        |
| `.SaveChangesAsync()`              | `tx.commit().await`                        | Explicit transaction commit             |
| Migrations                         | `diesel migration generate` / `sqlx migrate add` | Schema version management            |
| LINQ query expression              | Raw SQL with `sqlx::query!`                | No query provider; use SQL              |
| `.FromSqlRaw()`                    | `sqlx::query_as(...)`                     | Direct SQL execution                    |
| Change tracking                    | Not applicable (no change tracker)        | Explicit UPDATE/INSERT statements       |
| `DatabaseFacade`                   | `pool.begin().await?`                     | Direct transaction management           |

## Canonical Patterns

### 1. LINQ -> Iterator Combinators

```csharp
// C#: LINQ —— 声明式集合转换
var vipEmails = users
    .Where(u => u.IsVip)
    .OrderByDescending(u => u.TotalSpent)
    .Select(u => new { u.Name, u.Email })
    .Take(10)
    .ToList();
```

```rust
// Rust: Iterator —— 零成本组合器，惰性求值
use itertools::Itertools;

let vip_emails: Vec<_> = users
    .iter()
    .filter(|u| u.is_vip)
    .sorted_by(|a, b| b.total_spent.cmp(&a.total_spent)) // itertools
    .map(|u| VipInfo { name: &u.name, email: &u.email })
    .take(10)
    .collect();
```

### 2. Event / Delegate -> Channel / Callback

```csharp
// C#: event —— 发布/订阅模式
public class OrderProcessor
{
    public event EventHandler<OrderEventArgs>? OrderCompleted;

    protected virtual void OnOrderCompleted(Order order)
    {
        OrderCompleted?.Invoke(this, new OrderEventArgs(order));
    }
}

// 订阅
processor.OrderCompleted += (sender, args) => {
    Console.WriteLine($"Order {args.Order.Id} completed");
};
```

```rust
// Rust: tokio broadcast channel —— 发布/订阅
use tokio::sync::broadcast;

pub struct OrderProcessor {
    tx: broadcast::Sender<OrderEvent>,
}

impl OrderProcessor {
    pub fn new(capacity: usize) -> (Self, broadcast::Receiver<OrderEvent>) {
        let (tx, rx) = broadcast::channel(capacity);
        (Self { tx }, rx)
    }

    pub fn complete_order(&self, order: Order) {
        let _ = self.tx.send(OrderEvent::Completed(order));
    }
}

// 订阅
let mut rx2 = processor.tx.subscribe(); // 每个订阅者独立
tokio::spawn(async move {
    while let Ok(event) = rx2.recv().await {
        match event {
            OrderEvent::Completed(order) => tracing::info!("Order {} completed", order.id),
        }
    }
});
```

### 3. Extension Method -> Trait Extension

```csharp
// C#: extension method —— 类型扩展
public static class StringExtensions
{
    public static bool IsValidEmail(this string str)
    {
        return str.Contains('@') && str.Length > 3;
    }
}

// 使用
if (email.IsValidEmail()) { ... }
```

```rust
// Rust: extension trait pattern
pub trait StringExt {
    fn is_valid_email(&self) -> bool;
}

impl StringExt for str {
    fn is_valid_email(&self) -> bool {
        self.contains('@') && self.len() > 3
    }
}

// 使用
if email.is_valid_email() { ... }
```

### 4. Dependency Injection -> Manual/Constructor Injection

```csharp
// C#: ASP.NET Core DI 容器
public class OrderService
{
    private readonly IOrderRepository _repo;
    private readonly IPaymentGateway _payment;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IOrderRepository repo,
        IPaymentGateway payment,
        ILogger<OrderService> logger)
    {
        _repo = repo;
        _payment = payment;
        _logger = logger;
    }
}
```

```rust
// Rust: 构造函数注入 —— 无 DI 容器
pub struct OrderService<R: OrderRepo, P: Payment> {
    repo: R,
    payment: P,
}

impl<R: OrderRepo, P: Payment> OrderService<R, P> {
    pub fn new(repo: R, payment: P) -> Self {
        Self { repo, payment }
    }

    #[tracing::instrument(skip(self))]
    pub async fn place(&self, req: &OrderRequest) -> Result<Order, OrderError> {
        self.payment.charge(&req.payment).await?;
        let order = self.repo.insert(req).await?;
        tracing::info!(order_id = %order.id, "order placed");
        Ok(order)
    }
}

// 启动时组装
let service = OrderService::new(pg_repo, stripe_gateway);
let app_state = Arc::new(AppState { order_service: service });
```

### 5. Async Stream (IAsyncEnumerable) -> Stream

```csharp
// C# 8+: IAsyncEnumerable —— 异步流
public async IAsyncEnumerable<Order> GetPendingOrdersAsync()
{
    await foreach (var batch in _db.Orders
        .Where(o => o.Status == OrderStatus.Pending)
        .AsAsyncEnumerable())
    {
        yield return batch;
    }
}
```

```rust
// Rust: Stream trait (tokio-stream / futures)
use tokio_stream::StreamExt;
use futures::stream::Stream;

fn get_pending_orders(
    pool: &PgPool,
) -> impl Stream<Item = Result<Order, sqlx::Error>> + '_ {
    // 分批查询或使用游标
    async_stream::stream! {
        let mut offset = 0i64;
        loop {
            let batch = sqlx::query_as::<_, Order>(
                "SELECT * FROM orders WHERE status = 'pending' LIMIT 100 OFFSET $1"
            )
            .bind(offset)
            .fetch_all(pool)
            .await?;

            if batch.is_empty() {
                break;
            }
            for order in batch {
                yield Ok(order);
            }
            offset += 100;
        }
    }
}
```

## FFI & Incremental Migration

### Strategy: Service Boundary First

For .NET monoliths, extract services at the HTTP/gRPC boundary and rebuild in Rust. For performance-critical libraries, use P/Invoke to call Rust CDylib from C#.

### P/Invoke: Rust Library Called from C#

```rust
// Rust 侧: cdylib 导出
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

#[no_mangle]
pub extern "C" fn validate_json(
    schema_ptr: *const c_char,
    data_ptr: *const c_char,
) -> *mut c_char {
    let schema = unsafe { CStr::from_ptr(schema_ptr) }.to_str().unwrap();
    let data = unsafe { CStr::from_ptr(data_ptr) }.to_str().unwrap();

    let result = match fast_validate(schema, data) {
        Ok(()) => r#"{"valid":true}"#.to_string(),
        Err(e) => format!(r#"{{"valid":false,"error":"{}"}}"#, e),
    };

    CString::new(result).unwrap().into_raw()
}

#[no_mangle]
pub extern "C" fn free_rust_string(ptr: *mut c_char) {
    unsafe { let _ = CString::from_raw(ptr); }
}
```

```csharp
// C# 侧: P/Invoke 调用 Rust library
public static class RustValidator
{
    private const string RustLib = "rust_validator";

    [DllImport(RustLib, CallingConvention = CallingConvention.Cdecl)]
    private static extern IntPtr validate_json(string schema, string data);

    [DllImport(RustLib, CallingConvention = CallingConvention.Cdecl)]
    private static extern void free_rust_string(IntPtr ptr);

    public static ValidationResult Validate(string schema, string data)
    {
        var ptr = validate_json(schema, data);
        var json = Marshal.PtrToStringAnsi(ptr);
        free_rust_string(ptr);
        return JsonSerializer.Deserialize<ValidationResult>(json);
    }
}
```

### Migration Order

| Phase | Scope              | Strategy                                    |
|-------|--------------------|---------------------------------------------|
| 1     | Shared types       | Protobuf / JSON schema; both sides consume   |
| 2     | Computation engine | Hot-path algorithms moved to Rust first     |
| 3     | Background workers | Hangfire/Quartz jobs -> tokio tasks         |
| 4     | Web API (read)     | Read endpoints in Rust, route via proxy     |
| 5     | Web API (write)    | Write endpoints, shared DB transactional    |
| 6     | Full cutover       | Disable C# service, keep P/Invoke for legacy |

## Common Mistakes

### Mistake 1: Trying to Port Class Hierarchies

```rust
// 错误：C# 开发者尝试用 trait 对象模拟深层继承链
// 深层继承树在 Rust 中是反模式
trait Animal { fn speak(&self); }
trait Mammal: Animal { fn feed_milk(&self); }
trait Canine: Mammal { fn bark(&self); }
struct Dog;
impl Animal for Dog { ... }
impl Mammal for Dog { ... }
impl Canine for Dog { ... }

// 正确：使用 enum 表示封闭类型集合，组合表示共享行为
enum Animal {
    Dog(DogConfig),
    Cat(CatConfig),
    Bird(BirdConfig),
}

impl Animal {
    fn speak(&self) -> &str {
        match self {
            Animal::Dog(..) => "woof",
            Animal::Cat(..) => "meow",
            Animal::Bird(..) => "chirp",
        }
    }
}
```

### Mistake 2: Mutable Reference Confusion

```rust
// 错误：C# 中所有引用默认可变；Rust 中 &T 不可变
fn update_order(order: &Order, new_status: Status) {
    order.status = new_status; // 编译错误：&Order 不可变
}

// 正确：需要 &mut 才能修改
fn update_order(order: &mut Order, new_status: Status) {
    order.status = new_status;
}

// 编译期禁止别名可变：同一时间只能有一个 &mut 或任意多个 &
let mut order = Order::new();
let r1 = &mut order;
let r2 = &mut order; // 编译错误：不能同时有两个可变借用
```

### Mistake 3: Expecting Runtime Reflection

```csharp
// C#: 运行时反射 —— 轻松但代价高
var type = obj.GetType();
foreach (var prop in type.GetProperties())
{
    Console.WriteLine($"{prop.Name} = {prop.GetValue(obj)}");
}
```

```rust
// Rust: 无运行时反射 —— 使用 serde + derive 宏进行序列化
#[derive(Serialize)]
struct Order {
    id: String,
    total: f64,
}

let json = serde_json::to_string(&order)?;
println!("{json}");

// 或用 trait + 手动实现替代反射
trait Inspect {
    fn fields(&self) -> Vec<(&str, String)>;
}
impl Inspect for Order {
    fn fields(&self) -> Vec<(&str, String)> {
        vec![
            ("id", self.id.clone()),
            ("total", self.total.to_string()),
        ]
    }
}
```

### Mistake 4: `async void` / Fire-and-Forget

```csharp
// C#: fire-and-forget —— 可能吞没异常
async void SendNotificationAsync(string message)
{
    await _emailService.SendAsync(message); // 异常丢失！
}

// 在 Rust 中没有 async void 等价物
```

```rust
// Rust: 每个 async fn 必须返回 Future，必须有明确的错误处理路径
async fn send_notification(
    email: &EmailService,
    message: &str,
) -> Result<(), EmailError> {
    email.send(message).await
}

// 如需 fire-and-forget，显式 spawn 并处理 JoinHandle
tokio::spawn(async move {
    if let Err(e) = send_notification(&email, msg).await {
        tracing::error!(error = %e, "notification failed");
    }
});
```

### Mistake 5: Property Pattern Over-Translation

```csharp
// C#: 属性是语言特性
public class Person
{
    public string Name { get; set; }       // 自动属性
    public int Age { get; private set; }   // 私有 setter

    private string _email;
    public string Email
    {
        get => _email;
        set => _email = value?.Trim().ToLowerInvariant();
    }
}
```

```rust
// Rust: 简单场景用 pub 字段，需要验证/计算时用 getter/setter 方法
pub struct Person {
    pub name: String,   // 简单公有字段
    age: u32,           // 私有字段
    email: String,
}

impl Person {
    pub fn age(&self) -> u32 {  // getter
        self.age
    }

    pub fn set_email(&mut self, email: impl Into<String>) {
        self.email = email.into().trim().to_lowercase();
    }

    pub fn email(&self) -> &str {  // getter
        &self.email
    }
}
```

## Reference Implementations

| Project                       | Description                                         | C# LOC  | Rust LOC |
|-------------------------------|-----------------------------------------------------|---------|----------|
| Turbopack (Vercel)            | Rust successor to webpack; WebAssembly bundler      | N/A     | ~80k     |
| Tauri                         | Desktop app framework; Electron alternative         | N/A     | ~50k     |
| Tantivy (full-text search)    | Lucene-equivalent; often replaces Elasticsearch .NET clients | N/A | ~60k |
| Ruff (Python linter)          | Linter/formatter; .NET Roslyn analyzer equivalent   | N/A     | ~40k     |
| Polars                        | DataFrame library; Pandas/Spark .NET replacement    | N/A     | ~150k    |
| Oso (authorization)           | Core engine migrated from C# to Rust                | ~12k    | ~10k     |
| Meilisearch                   | REST search; .NET clients migrated to Rust core     | ~30k    | ~25k     |
| Sled (embedded DB)            | Embedded KV store; often replaces LiteDB/SQLite wrappers | N/A | ~20k |
| Helix (editor)                | Modal editor; .NET tools rewritten in Rust          | N/A     | ~30k     |

## Cross-Reference

- **java-to-rust**: JVM-to-native mapping; shared enterprise service patterns with ASP.NET migration
- **go-to-rust**: Goroutine/async patterns; shared cancellation and channel primitives
- **nodejs-to-rust**: Async runtime and web framework patterns; shared NPM/NuGet-to-Cargo mapping
- **cpp-to-rust**: P/Invoke and FFI interop strategies
