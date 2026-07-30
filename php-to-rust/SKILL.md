---
name: php-to-rust
description: Use when migrating PHP codebases to Rust — covers dynamic to static typing, Laravel/Symfony to Axum/Actix, PDO to sqlx/Diesel, array shapes to Vec/HashMap, exception to Result, Composer to Cargo, and incremental migration via sidecar services. Includes canonical code patterns, common mistakes, and reference implementations.
updated: 2026-07-30
---

# PHP to Rust Migration

## Architecture Mapping

PHP's shared-nothing, request-per-process execution model (FPM/mod_php) maps to Rust's long-lived, multi-threaded process with async I/O. The PHP runtime (Zend Engine -- interpreted bytecode, reference-counting GC, dynamic typing) is replaced by AOT compilation with zero runtime overhead. A Laravel or Symfony application served by PHP-FPM behind Nginx becomes a single Rust binary serving HTTP via Actix-Web, Axum, or Rocket, using tokio for async I/O. Composer's vendor directory and autoloader map to Cargo's `Cargo.toml` dependency resolution and Rust's module system. PHP's per-request isolation (every request starts fresh) becomes connection pooling and shared application state in Rust.

| PHP Concept                 | Rust Equivalent                          | Notes                                              |
|----------------------------|------------------------------------------|----------------------------------------------------|
| Zend Engine                | None (native binary)                     | AOT compilation; no interpreter overhead           |
| Garbage collector          | Ownership + RAII                         | Reference counting replaced by static lifetimes    |
| Dynamic typing             | Static typing + generics                 | All types known at compile time                    |
| `class`                    | `struct` + `impl` + `trait`              | Data separated from behavior                       |
| Inheritance (`extends`)    | Trait + composition                      | No class hierarchy; enums for sealed variants      |
| `interface`                | `trait`                                  | Explicit implementation; no checked at runtime     |
| `trait` (PHP)              | `trait` (Rust)                           | PHP traits have state; Rust traits only behavior   |
| `abstract class`           | Trait with default methods               | Partial implementation via trait                   |
| `namespace`                | `mod` / `pub mod`                        | Module system with explicit visibility             |
| `use` statement            | `use`                                    | Import paths                                       |
| `include` / `require`      | `mod` / `use`                            | Compile-time module inclusion                      |
| `autoload` (Composer)      | Cargo dependency resolution              | No runtime autoloading; compiled in                |
| `array` (list)             | `Vec<T>`                                 | Homogeneous, typed vectors                         |
| `array` (associative)      | `HashMap<K,V>` / `BTreeMap<K,V>`         | Typed key-value maps                                |
| `array` (mixed shape)      | `struct` / `enum`                        | Structured types instead of amorphous arrays       |
| `null`                     | `Option<T>`                              | Explicit absence; no null everywhere               |
| `mixed` / `object` type    | Concrete type / `Box<dyn Trait>`         | No dynamic dispatch unless explicitly opted in     |
| `callable`                 | `Fn` / `FnMut` / `FnOnce`                | Closure trait; typed & zero-cost                   |
| `Closure`                  | `Fn` / `FnMut` / `FnOnce` trait objects  | Anonymous functions via closure syntax             |
| `Exception`                | `Result<T, E>`                           | Error as value; no try/catch stack unwinding       |
| `throw`                    | `return Err(...)` + `?` operator         | Values, not control flow                           |
| `try/catch/finally`        | `match result { Ok(v) => ..., Err(e) => ... }` | Pattern matching + `Drop` for cleanup        |
| `Generator` / `yield`      | `Iterator` / `Stream` / `async_stream`   | Iterators for sync; Stream for async laziness      |
| `$_GET` / `$_POST` / `$_SERVER` | Extractor (axum) / request method   | Typed extraction from request                      |
| `$_SESSION` / `$_COOKIE`   | `tower-sessions` / `cookie` crate        | HTTP session and cookie management                 |
| `echo` / `print`           | `println!` / tracing macros              | Structured output vs. raw write                    |
| `header()`                 | `Response::builder().header(...)`        | Typed response construction                        |
| `setcookie()`              | `Cookie::build(...)` (cookie crate)      | Typed cookie creation                              |
| `$_ENV` / `getenv()`       | `std::env::var()` / `dotenvy::var()`     | Type-safe env var access                           |
| `json_encode` / `json_decode` | `serde_json::to_string` / `from_str`   | Derive-based serialization                         |
| `PDO` / `mysqli`           | `sqlx` / `diesel`                        | Async-native, compile-time checked queries         |
| `file_get_contents`        | `std::fs::read_to_string` / `reqwest`    | Local fs or HTTP fetch                             |
| `preg_match` / `preg_replace` | `regex::Regex`                        | Regex via regex crate                              |
| `str_replace`              | `str::replace`                           | Standard library method                            |
| `explode` / `implode`      | `str::split` / `Vec::join` on `&[T]`     | Iterator-based split, slice join                    |
| `array_map` / `array_filter` | `.iter().map().filter().collect()`      | Iterator combinators                               |
| `count()`                  | `.len()` method                          | Standard library; O(1) for most collections        |
| `isset()` / `empty()`      | `Option::is_some()` / `Option::is_none()`| Explicit presence check                            |

## Type System Mapping

PHP's dynamic, weak typing is the largest conceptual gap. PHP 8.x's type declarations (`int`, `string`, `?string`) provide a bridge, but the migration to Rust requires redesigning data structures around algebraic types and generics.

| PHP Type                        | Rust Type                          | Notes                                              |
|---------------------------------|------------------------------------|----------------------------------------------------|
| `int`                           | `i64`                              | PHP int is platform-dependent signed; prefer sized  |
| `float`                         | `f64`                              | IEEE 754 double                                     |
| `string`                        | `String` / `&str`                  | Owned (heap) vs. borrowed reference                 |
| `bool`                          | `bool`                             | Identical                                           |
| `array` (indexed, homogeneous)  | `Vec<T>`                           | Typed, contiguous                                   |
| `array` (indexed, mixed)        | `Vec<EnumType>`                    | Enum wraps each possible variant                    |
| `array` (associative, `K => V`) | `HashMap<K, V>` / `BTreeMap<K, V>` | Typed key-value                                     |
| `array` (fixed shape / record)  | `struct`                           | Compile-time field list                             |
| `null`                          | `Option<T>`                        | All types optional by default; `Option` is explicit |
| `iterable`                      | `impl Iterator<Item = T>`          | Return position matters                             |
| `callable`                      | `Fn(T) -> R` / `Box<dyn Fn(T) -> R>` | Zero-cost or heap-allocated closure               |
| `Closure`                       | Closure `\|args\| { body }`          | Anonymous function literal                          |
| `object`                        | `Box<dyn Trait>`                   | Trait object for runtime polymorphism               |
| `mixed`                         | Avoid; use concrete type           | Or `serde_json::Value` for truly unknown JSON       |
| `void`                          | `()`                               | Unit type; no return value                          |
| `never`                         | `!` (never type)                   | Diverging function; signals unreachable             |
| `resource`                      | Struct wrapping handle             | Explicit type, lifetime tracked                    |
| `enum` (PHP 8.1)                | `enum` (Rust, full ADT)            | Rust enums carry variant data                       |
| `match` (PHP 8.0)               | `match` (Rust, exhaustive)         | Must cover all variants                             |
| `readonly`                      | No direct equivalent               | All bindings immutable by default; use `mut`        |
| `DateTime`                      | `chrono::DateTime<Utc>`            | Timezone-aware datetime                             |
| `DateInterval`                  | `std::time::Duration`              | Time interval                                       |

## Memory & Ownership Model

PHP's reference-counted, copy-on-write memory model with garbage collection is the hardest mental shift. In PHP, every variable is a reference to a zval structure; in Rust, every variable owns its data, and sharing requires explicit borrowing.

### The Copy-on-Write to Ownership Shift

```php
// PHP: COW（写时复制）——变量共享底层值直到被修改
$a = "hello";              // zval refcount = 1
$b = $a;                   // zval refcount = 2（共享）
$a = "world";              // zval refcount = 1 on new string
// $b 仍然是 "hello"
```

```rust
// Rust: 所有权——Move 语义，不复制的共享通过借用
let a = String::from("hello");  // a 拥有字符串
let b = a;                       // b 现在拥有字符串，a 不再有效
// println!("{a}");              // 编译错误：a 已被移动

// 需要共享时显式借用
let a = String::from("hello");
let b = &a;                      // b 借用 a
println!("{a} {b}");            // 两者都可用
```

### PHP Array Shapes -> Rust Structs / Enums

```php
// PHP: 数组被当作万能数据结构
function getUser(int $id): array {
    return [
        'id' => 1,
        'name' => 'Alice',
        'email' => 'alice@example.com',
        'roles' => ['admin', 'editor'],
        'profile' => [
            'avatar' => 'https://...',
            'bio' => 'Software engineer',
        ],
    ];
}
```

```rust
// Rust: 每个数据形状对应一个具体类型
use serde::Serialize;

#[derive(Debug, Clone, Serialize)]
pub struct User {
    pub id: i64,
    pub name: String,
    pub email: String,
    pub roles: Vec<Role>,
    pub profile: Profile,
}

#[derive(Debug, Clone, Serialize)]
pub struct Profile {
    pub avatar: Option<String>,
    pub bio: Option<String>,
}

fn get_user(id: i64) -> Result<User, UserError> {
    // 编译期保证返回类型符合结构定义
    Ok(User {
        id: 1,
        name: "Alice".into(),
        email: "alice@example.com".into(),
        roles: vec![Role::Admin, Role::Editor],
        profile: Profile {
            avatar: Some("https://...".into()),
            bio: Some("Software engineer".into()),
        },
    })
}
```

### Reference Patterns vs. Borrowing

| PHP Pattern                | Rust Equivalent                                  |
|----------------------------|--------------------------------------------------|
| `$obj->method()`           | `obj.method()` (self is `&self` by default)       |
| `&$var` for pass-by-ref    | `&mut var` (exclusive mutable borrow)             |
| Cloning `clone $obj`       | `obj.clone()` (explicit `Clone` trait)            |
| Lazy loading properties    | `Option<T>` with lazy init or `OnceLock`          |
| Circular references        | `Weak<T>` + `Rc<T>` or restructure without cycles |
| `unset($var)`              | `drop(var)` or natural scope exit                 |
| `global $x` / `$GLOBALS`   | Thread-local or application state; avoid global mutable state |

## Concurrency / Async Translation

PHP is traditionally single-threaded per request (with extensions like Swoole or Fiber). Rust's async/await enables concurrent request handling natively. PHP 8.1 Fibers provide a limited form of cooperative multitasking; tokio provides full M:N async scheduling.

### Synchronous -> Async Execution

```php
// PHP: 同步 ORM 查询（PDO/Doctrine）
public function getOrderSummary(int $orderId): array
{
    $order = $this->db->query(
        'SELECT * FROM orders WHERE id = ?',
        [$orderId]
    )->fetch();

    $items = $this->db->query(
        'SELECT * FROM order_items WHERE order_id = ?',
        [$orderId]
    )->fetchAll();

    $customer = $this->db->query(
        'SELECT * FROM customers WHERE id = ?',
        [$order['customer_id']]
    )->fetch();

    return [
        'order' => $order,
        'items' => $items,
        'customer' => $customer,
    ];
}
```

```rust
// Rust: 并发查询——三个查询并行执行
async fn get_order_summary(
    pool: &PgPool,
    order_id: i64,
) -> Result<OrderSummary, sqlx::Error> {
    let (order, items, customer) = tokio::try_join!(
        sqlx::query_as::<_, Order>("SELECT * FROM orders WHERE id = $1")
            .bind(order_id)
            .fetch_one(pool),
        sqlx::query_as::<_, LineItem>("SELECT * FROM order_items WHERE order_id = $1")
            .bind(order_id)
            .fetch_all(pool),
        async {
            let order = sqlx::query_as::<_, Order>("SELECT customer_id FROM orders WHERE id = $1")
                .bind(order_id)
                .fetch_one(pool)
                .await?;
            sqlx::query_as::<_, Customer>("SELECT * FROM customers WHERE id = $1")
                .bind(order.customer_id)
                .fetch_one(pool)
                .await
        },
    )?;

    Ok(OrderSummary { order, items, customer })
}
```

### PHP Fiber -> Rust Async Task

```php
// PHP 8.1+: Fiber（协作式并发）
$fiber = new Fiber(function () use ($data): void {
    $result = Fiber::suspend(processPart1($data));
    $final = processPart2($result);
    Fiber::suspend($final);
});

$part1 = $fiber->start();
$final = $fiber->resume($part1);
```

```rust
// Rust: async/await（同样协作式，但有运行时调度）
async fn process(data: Data) -> Result<Final, Error> {
    let result = process_part1(&data).await?; // .await 点 = 让出点
    process_part2(result).await
}

let handle = tokio::spawn(process(data));
let final = handle.await??;
```

## Build System & Dependencies

| PHP / Composer                 | Rust / Cargo                          |
|--------------------------------|---------------------------------------|
| `composer.json`                | `Cargo.toml`                          |
| `composer.lock`                | `Cargo.lock`                          |
| `composer install`             | `cargo build` / `cargo fetch`         |
| `composer require vendor/pkg`  | `cargo add crate`                     |
| `composer update`              | `cargo update`                        |
| `vendor/autoload.php`          | Module system (no runtime autoloader)  |
| `composer dump-autoload`       | `cargo build` (generated at compile time) |
| `composer validate`            | `cargo check` / `cargo clippy`        |
| `composer scripts`             | `[package.metadata]` + custom tasks   |
| Psalm / PHPStan                | `cargo clippy` (built-in linter)      |
| PHPUnit                        | `#[test]` + `cargo test`             |
| `php -S localhost:8000`        | `cargo run`                           |
| `.env` (via vlucas/dotenv)     | `.env` + `dotenvy` crate              |
| PHP CS Fixer                   | `cargo fmt`                           |
| `php -l` (lint)                | `cargo check`                         |
| Xdebug                         | `rust-gdb` / `lldb`                   |
| PSR-4 autoload standard        | Module path conventions               |
| `config/*.php`                 | `config.rs` or `config.toml` + `serde` |

**Cargo.toml for a migrated Laravel application:**

```toml
[package]
name = "web-api"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.7", features = ["macros"] }
tokio = { version = "1", features = ["full"] }
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace", "compression-gzip", "auth"] }
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
cookie = "0.18"
tower-sessions = "0.12"
argon2 = "0.5"  # PHP password_hash 等价物
validator = { version = "0.18", features = ["derive"] }
```

## Framework Mapping: Laravel / Symfony -> Axum / Actix / Rocket

| Laravel / Symfony            | Rust / Axum                           | Notes                                   |
|------------------------------|---------------------------------------|-----------------------------------------|
| `Route::get('/path', fn)`    | `Router::route("/path", get(handler))` | Explicit method + path binding          |
| `Route::post('/path', fn)`   | `Router::route("/path", post(handler))` | Same pattern                            |
| `Route::resource('users')`   | Manual CRUD routes                    | No convention-over-config; explicit     |
| Controller class             | Module of handler functions            | No base class needed                    |
| FormRequest / `$request->validate()` | `validator` crate + `Json<T>` extractor | Type-safe validation before handler |
| Eloquent Model               | `sqlx::FromRow` derive + query functions | Active Record to typed queries         |
| `User::find($id)`            | `sqlx::query_as::<_, User>("SELECT ...").bind(id).fetch_optional(&pool)` | Explicit SQL |
| `User::where('status', 1)->get()` | Iterators or query builder methods    | Compose queries manually or use Diesel  |
| Query Builder (`DB::table()`) | `sqlx::query_as` / Diesel DSL         | Raw SQL or type-safe query DSL          |
| Eloquent Relationships       | Manual JOIN or separate queries        | No hasMany/belongsTo; explicit loading  |
| Migrations                   | `sqlx migrate` / `diesel migration`    | Versioned SQL migration files           |
| Seeder                       | `sqlx::query` in a binary or test      | No dedicated seeder; raw inserts        |
| Middleware                   | `tower::Layer` / `axum::middleware`    | Layered, composable middleware           |
| Service Provider             | Startup-time initialization in `main`  | Explicit registration at boot           |
| `auth()->user()`             | Middleware-extracted `AuthUser` type    | Extract from request; no global helper  |
| `Gate` / `Policy`            | Manual authorization check function    | Simple boolean check before action      |
| Blade / Twig templates       | `askama` / `tera` / `minijinja`       | Server-side rendering or separate SPA   |
| `@csrf`                      | CSRF token middleware (e.g., `tower-sessions` CSRF) | Middleware-based protection   |
| `Session::get` / `Session::put` | `tower-sessions` / `axum-sessions` | Typed session store                   |
| `Cache::get` / `Cache::put`  | `moka` / Redis client                  | Direct cache crate usage                |
| Queues (`Job`, `dispatch`)   | Background task via `tokio::spawn`      | No dedicated queue; use Redis/RabbitMQ  |
| Scheduled commands           | `tokio::time::interval` + `tokio::spawn` | Explicit scheduling loop               |
| `Log::info` / `Log::error`   | `tracing::info!` / `tracing::error!`    | Structured, span-based logging          |
| `.env` file                  | `dotenvy` crate + `std::env::var`       | Same pattern                            |
| `config('app.name')`         | `AppConfig::from_env()` or config file  | Type-safe config structs                |
| Artisan Console (`make:`)    | No equivalent; manual module creation   | Create files, add `mod` declaration     |
| `php artisan serve`          | `cargo run`                             | Direct binary execution                 |

## Standard Library & Utility Mapping

| PHP Function / Pattern          | Rust Equivalent                       | Notes                                   |
|---------------------------------|---------------------------------------|-----------------------------------------|
| `array_map(fn, $arr)`           | `arr.iter().map(f).collect()`         | Iterator-based                           |
| `array_filter($arr, fn)`        | `arr.iter().filter(f).collect()`      | Same pattern                            |
| `array_reduce($arr, fn, init)`  | `arr.iter().fold(init, f)`            | Classical fold                          |
| `array_key_exists($k, $arr)`    | `map.contains_key(&k)`                | HashMap/BTreeMap method                 |
| `in_array($v, $arr)`            | `arr.contains(&v)`                    | Slice/Vec method                        |
| `array_unique($arr)`            | `arr.iter().unique().collect()` (itertools) | Requires itertools                |
| `array_merge($a, $b)`           | `a.extend(b)` or `[a, b].concat()`    | In-place or new allocation              |
| `array_slice($arr, 0, 10)`      | `&arr[..10]` or `arr[0..10].to_vec()` | Slice syntax; bounds checked            |
| `array_push($arr, $v)`          | `arr.push(v)`                         | Vec method                              |
| `array_pop($arr)`               | `arr.pop()` -> `Option<T>`            | Returns Option, not false/null          |
| `count($arr)`                   | `arr.len()`                           | O(1) for Vec/HashMap                    |
| `empty($var)`                   | `var.is_empty()` / `var.is_none()`    | Type-specific emptiness checks          |
| `isset($var)`                   | `Option::is_some()` / pattern match   | Explicit presence checks                |
| `is_null($var)`                 | `var.is_none()`                       | Option pattern                          |
| `json_encode($data)`            | `serde_json::to_string(&data)?`       | Derive Serialize                        |
| `json_decode($json, true)`      | `serde_json::from_str::<T>(&json)?`   | Deserialize to concrete type            |
| `json_decode($json)` (as object)| `serde_json::from_str::<SerdeJsonValue>(&json)?` | Generic value type              |
| `file_get_contents($path)`      | `std::fs::read_to_string(path)?`      | Nearly identical                        |
| `file_get_contents($url)`       | `reqwest::get(url).await?.text().await?` | HTTP fetch via reqwest                |
| `file_put_contents($p, $d)`    | `std::fs::write(path, data)?`         | Atomic-ish write                        |
| `file_exists($path)`            | `Path::new(path).try_exists()?`       | Semantic equivalent                     |
| `is_dir($path)`                 | `path.is_dir()`                       | Path method                             |
| `mkdir($path, 0755, true)`      | `std::fs::create_dir_all(path)?`      | Recursive directory creation            |
| `unlink($path)`                 | `std::fs::remove_file(path)?`         | File deletion                           |
| `basename($path)`               | `Path::new(path).file_name()`         | Returns Option<&OsStr>                  |
| `dirname($path)`                | `Path::new(path).parent()`            | Returns Option<&Path>                   |
| `sprintf($fmt, ...)`            | `format!(fmt, ...)`                   | Almost identical                        |
| `str_replace($s, $r, $str)`     | `str.replace(s, r)`                   | String method                           |
| `str_contains($h, $n)`          | `h.contains(n)`                       | String method                           |
| `str_starts_with($h, $n)`       | `h.starts_with(n)`                    | String method                           |
| `str_ends_with($h, $n)`         | `h.ends_with(n)`                      | String method                           |
| `strtolower($s)`                | `s.to_lowercase()`                    | Unicode-aware (std)                     |
| `strtoupper($s)`                | `s.to_uppercase()`                    | Unicode-aware (std)                     |
| `trim($s)`                      | `s.trim()`                            | Whitespace trim                         |
| `explode(',', $s)`              | `s.split(',').collect::<Vec<_>>()`    | Iterator returns &str slices            |
| `implode(',', $a)`              | `a.join(",")` or `itertools::join`    | Vec/slice method                        |
| `substr($s, 0, 5)`              | `&s[..5]` or `s.get(..5)`             | Safe with `.get()` returning `Option`   |
| `strlen($s)`                    | `s.len()` / `s.chars().count()`       | Byte length vs. char count              |
| `md5($s)`                       | `md5::compute(s)` (md-5 crate)        | For legacy compat; prefer SHA-256       |
| `sha1($s)`                      | `sha1::Sha1::digest(s)`               | For legacy compat                        |
| `password_hash($p)`             | `argon2::hash_encoded(p.as_bytes(), ...)` | argon2 crate for same algorithm      |
| `password_verify($p, $h)`       | `argon2::verify_encoded(&h, p.as_bytes())` | Verify against stored hash            |
| `random_bytes($n)`              | `rand::random::<[u8; N]>()` (rand)     | System CSPRNG via rand crate            |
| `uniqid()`                      | `uuid::Uuid::new_v4()`                | Proper UUID v4                          |
| `time()` / `microtime(true)`    | `SystemTime::now()` / `Instant::now()` | Unix timestamp vs. monotonic clock      |
| `date('Y-m-d')`                 | `chrono::Local::now().format("%Y-%m-%d")` | chrono formatting                    |
| `header('Content-Type: ...')`   | Response builder `.header(...)`        | Part of response construction           |
| `http_response_code(404)`       | `StatusCode::NOT_FOUND`               | Enum variant in response                |
| `parse_url($url)`               | `url::Url::parse(url)?` (url crate)   | URL parsing and manipulation            |
| `http_build_query($data)`       | `serde_urlencoded::to_string(&data)?`  | Query string serialization              |
| `parse_str($qs, $arr)`          | `serde_urlencoded::from_str::<T>(qs)?` | Query string parsing                    |
| `htmlspecialchars($s)`          | `html_escape::encode_text(s)`          | HTML entity encoding                    |

## Canonical Patterns

### 1. Dynamic Array Shape -> Struct

```php
// PHP: 数组作为万能数据结构——无编译期保证
function formatUser(array $user): array {
    return [
        'display' => $user['first_name'] . ' ' . $user['last_name'],
        'member_since' => date('Y', $user['created_at']),
        'is_admin' => in_array('admin', $user['roles'] ?? []),
    ];
}
```

```rust
// Rust: 每个形状都是具体类型——编译期保证正确性
struct UserInput {
    first_name: String,
    last_name: String,
    created_at: chrono::NaiveDateTime,
    roles: Vec<String>,
}

struct UserDisplay {
    display: String,
    member_since: i32,
    is_admin: bool,
}

fn format_user(user: &UserInput) -> UserDisplay {
    UserDisplay {
        display: format!("{} {}", user.first_name, user.last_name),
        member_since: user.created_at.year(),
        is_admin: user.roles.iter().any(|r| r == "admin"),
    }
}
```

### 2. PHP Exception -> Rust Result

```php
// PHP: try/catch 异常——非显式控制流
public function transferFunds(Account $from, Account $to, Money $amount): void
{
    if ($from->balance < $amount) {
        throw new InsufficientFundsException($from->id);
    }

    try {
        $this->gateway->debit($from, $amount);
    } catch (PaymentException $e) {
        throw new TransferFailedException('debit failed', 0, $e);
    }

    try {
        $this->gateway->credit($to, $amount);
    } catch (PaymentException $e) {
        // 需要回滚 debit——异常控制流使这变得脆弱
        $this->gateway->refund($from, $amount);
        throw new TransferFailedException('credit failed', 0, $e);
    }
}
```

```rust
// Rust: Result —— 错误是值，不是控制流
#[derive(Error, Debug)]
enum TransferError {
    #[error("insufficient funds in account {0}")]
    InsufficientFunds(String),
    #[error("payment gateway error: {0}")]
    Gateway(#[from] PaymentError),
}

async fn transfer_funds(
    gateway: &dyn PaymentGateway,
    from: &Account,
    to: &Account,
    amount: Money,
) -> Result<(), TransferError> {
    if from.balance < amount {
        return Err(TransferError::InsufficientFunds(from.id.clone()));
    }

    gateway.debit(from, amount).await?; // ? 传播错误

    match gateway.credit(to, amount).await {
        Ok(()) => Ok(()),
        Err(e) => {
            // 明确的补偿操作
            let _ = gateway.refund(from, amount).await;
            Err(TransferError::Gateway(e))
        }
    }
}
```

### 3. Class Inheritance -> Trait + Composition

```php
// PHP: 类继承——重用行为
abstract class Mailer
{
    abstract protected function transport(): Transport;

    public function send(Message $message): void
    {
        $this->transport()->send($message->toArray());
        $this->log($message);
    }

    protected function log(Message $message): void
    {
        // 记录发送日志
    }
}

class SmtpMailer extends Mailer
{
    protected function transport(): Transport
    {
        return new SmtpTransport($this->config);
    }
}
```

```rust
// Rust: trait 定义行为，组合替代继承
#[async_trait]
trait Transport: Send + Sync {
    async fn send(&self, message: &Message) -> Result<(), TransportError>;
}

trait Mailer {
    type TransportImpl: Transport;

    fn transport(&self) -> &Self::TransportImpl;

    async fn send(&self, message: &Message) -> Result<(), MailerError>
    where
        Self: Loggable,
    {
        self.transport().send(message).await?;
        self.log(message);
        Ok(())
    }
}

struct SmtpMailer {
    transport: SmtpTransport,
    config: SmtpConfig,
}

impl Mailer for SmtpMailer {
    type TransportImpl = SmtpTransport;
    fn transport(&self) -> &Self::TransportImpl { &self.transport }
}
```

### 4. Composer Autoload -> Module System

```php
// PHP: PSR-4 autoload —— 文件名映射到类名
// src/Services/Payment/Gateway.php
namespace App\Services\Payment;

use App\Contracts\GatewayInterface;
use App\Models\Transaction;

class Gateway implements GatewayInterface
{
    public function charge(Transaction $txn): void { ... }
}

// 调用
use App\Services\Payment\Gateway;
$gw = new Gateway();
```

```rust
// Rust: 显式模块树 —— 模块在文件中声明
// src/services/payment.rs
pub mod gateway;

// src/services/payment/gateway.rs
use crate::contracts::GatewayInterface;
use crate::models::Transaction;

pub struct Gateway { ... }

impl GatewayInterface for Gateway {
    fn charge(&self, txn: &Transaction) -> Result<(), GatewayError> { ... }
}

// 调用 —— 路径对应模块层次结构
use crate::services::payment::gateway::Gateway;
let gw = Gateway::new(config);
```

### 5. Middleware / Request Pipeline

```php
// Laravel: 中间件管道
class Authenticate
{
    public function handle(Request $request, Closure $next): Response
    {
        if (!Auth::check()) {
            return redirect('/login');
        }
        return $next($request);
    }
}
```

```rust
// Rust (axum): middleware as async layer
use axum::{
    extract::Request,
    middleware::Next,
    response::{Response, Redirect},
};

async fn authenticate(
    mut request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    // 从 header 或 session 提取用户信息
    let user = extract_user(&request).ok_or(StatusCode::UNAUTHORIZED)?;
    request.extensions_mut().insert(user);
    Ok(next.run(request).await)
}

// 注册中间件
let app = Router::new()
    .route("/dashboard", get(dashboard))
    .layer(middleware::from_fn(authenticate));
```

### 6. Dynamic Method Call -> Enum Dispatch

```php
// PHP: 动态方法调用——运行时决定
class NotificationService
{
    public function notify(string $channel, string $message): void
    {
        $method = 'sendVia' . ucfirst($channel);
        if (method_exists($this, $method)) {
            $this->$method($message); // 运行时分派
        }
    }

    private function sendViaEmail(string $msg): void { ... }
    private function sendViaSms(string $msg): void { ... }
    private function sendViaPush(string $msg): void { ... }
}
```

```rust
// Rust: enum + trait 实现——编译期安全的分派
enum Channel {
    Email,
    Sms,
    Push,
}

struct NotificationService {
    email: EmailClient,
    sms: SmsClient,
    push: PushClient,
}

impl NotificationService {
    async fn notify(&self, channel: Channel, message: &str) -> Result<(), NotifyError> {
        match channel {  // 穷举匹配，编译器保证所有分支已处理
            Channel::Email => self.email.send(message).await?,
            Channel::Sms => self.sms.send(message).await?,
            Channel::Push => self.push.send(message).await?,
        }
        Ok(())
    }
}
```

## FFI & Incremental Migration

PHP-to-Rust migration typically happens at service boundaries or via PHP extensions. The recommended strategy is the "strangler fig" pattern: route traffic incrementally to Rust services.

### Strategies

| Strategy                    | Complexity | Risk      | When to Use                               |
|-----------------------------|------------|-----------|-------------------------------------------|
| Sidecar microservice        | Low        | Low       | Existing API gateway / reverse proxy      |
| PHP extension (C-ABI)       | High       | Medium    | Performance-critical library in monolith  |
| Event-driven decoupling     | Medium     | Low       | Async jobs, queues, event listeners       |
| HTTP reverse proxy split    | Low        | Low       | Nginx/Caddy route splitting               |
| Read-replica migration      | Medium     | Medium    | Both PHP and Rust read same DB, migrate writes last |

### Sidecar Pattern (Recommended)

```
        +---------+
        | Nginx   |
        +----+----+
             |
    +--------+---------+
    |                  |
    v                  v
+---+----+      +-----+------+
| PHP App|      | Rust Axum  |
| (legacy)|      | (new API) |
+---+----+      +-----+------+
    |                  |
    +-----+------+-----+
          |      |
          v      v
      +---+------+---+
      |  PostgreSQL  |
      +--------------+
```

**Nginx split configuration:**

```nginx
location /api/v2/ {
    proxy_pass http://rust-service:3000;
}

location /api/ {
    proxy_pass http://php-app:9000;
}
```

### PHP Extension via C-ABI (for Library Migration)

```rust
// Rust 侧: cdylib 导出
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

#[no_mangle]
pub extern "C" fn rustium_validate_email(ptr: *const c_char) -> i32 {
    let email = match unsafe { CStr::from_ptr(ptr) }.to_str() {
        Ok(s) => s,
        Err(_) => return 0,
    };
    // 使用邮箱验证逻辑
    if email.contains('@') && email.len() <= 254 {
        1
    } else {
        0
    }
}
```

```php
// PHP 侧: FFI 调用
$ffi = FFI::cdef(
    "int rustium_validate_email(const char* email);",
    "lib/librustium.so"
);

$valid = $ffi->rustium_validate_email("test@example.com");
```

### Migration Order

| Phase | Scope              | Strategy                                      |
|-------|--------------------|-----------------------------------------------|
| 1     | Data types         | Define types in Rust, share via JSON schema    |
| 2     | Validation logic   | Move form/input validation to Rust (faster, safer) |
| 3     | Background jobs    | Migrate queue workers to Rust for throughput   |
| 4     | Read-only API      | Rewrite GET endpoints in Rust behind proxy     |
| 5     | Write API          | Mutations go to Rust; PHP becomes read-only    |
| 6     | Full cutover       | Decommission PHP-FPM entirely                 |

## Common Mistakes

### Mistake 1: Porting Arrays as `serde_json::Value` Universally

```rust
// 错误：把 PHP 数组习惯直接翻译为动态 JSON 类型
use serde_json::Value;
fn get_user(id: i64) -> Value {
    json!({         // 编译期无类型检查
        "id": 1,
        "name": "Alice",
        "roles": ["admin"]
    })
}

// 正确：定义具体类型
#[derive(Serialize)]
struct User {
    id: i64,
    name: String,
    roles: Vec<String>,
}
fn get_user(id: i64) -> Result<User, Error> {
    // 编译期保证所有字段类型正确
    Ok(User { id: 1, name: "Alice".into(), roles: vec!["admin".into()] })
}
```

### Mistake 2: Copying PHP's Defensive Null Checks

```php
// PHP: 习惯到处检查 null/empty
$name = $user['name'] ?? 'Unknown';
$email = isset($user['email']) ? $user['email'] : '';
$avatar = $user['profile']['avatar'] ?? null;
```

```rust
// Rust: Option 类型在类型层面处理缺失
struct User {
    name: String,                  // 保证非空
    email: Option<String>,         // 类型系统表明可空
    profile: Option<Profile>,
}

struct Profile {
    avatar: Option<String>,        // 可能没有头像
}

// 不需要 ?? 操作符；使用模式匹配或组合器
let display_name = user.name;  // 总是有效
let email = user.email.unwrap_or_default();
let avatar = user.profile
    .as_ref()
    .and_then(|p| p.avatar.as_deref())
    .unwrap_or("/default-avatar.png");
```

### Mistake 3: Runtime Flexibility Over Type Safety

```rust
// 错误：PHP 开发者用 trait 对象模拟动态分发，丢失类型信息
fn process_payment(method: Box<dyn Any>) -> Result<(), String> {
    if let Some(stripe) = method.downcast_ref::<StripePayment>() {
        stripe.charge();
    } else {
        return Err("unknown payment method".into());
    }
    Ok(())
}

// 正确：使用 enum + trait，编译期穷举
enum PaymentMethod {
    Stripe(StripePayment),
    Paypal(PaypalPayment),
    BankTransfer(BankTransferPayment),
}

impl PaymentMethod {
    async fn charge(&self, amount: Money) -> Result<ChargeResult, PaymentError> {
        match self {
            PaymentMethod::Stripe(s) => s.charge(amount).await,
            PaymentMethod::Paypal(p) => p.charge(amount).await,
            PaymentMethod::BankTransfer(b) => b.charge(amount).await,
        }
        // 添加新支付方式时编译器会报错此处漏分支
    }
}
```

### Mistake 4: Expecting Function-Level Scope Cleanup

```php
// PHP: 函数结束时自动清理（引用计数归零）
function process(): void {
    $file = fopen('/tmp/data.csv', 'r');
    // ... 使用文件 ...
    // 函数结束时 $file 自动关闭（或 GC 回收）
}
```

```rust
// Rust: 作用域退出时 Drop，不是函数退出时
fn process() -> Result<(), Error> {
    {
        let file = File::open("/tmp/data.csv")?;
        // ... 使用 file ...
    } // file 在此 Drop，不是在函数末尾

    // file 已经关闭，这里可以安全打开同一个文件
    let file2 = File::open("/tmp/data.csv")?;
    Ok(())
} // 不是 PHP 风格的"函数结束时清理"而是"作用域结束时清理"
```

### Mistake 5: Over-Using `unwrap()` / `expect()`

```rust
// 错误：PHP 开发者用 unwrap() 替代 PHP 的隐式 null 忽略
let user = db.get_user(id).unwrap();             // 可能 panic
let email = user.email.unwrap();                 // 可能 panic
let name = config.get("app.name").unwrap();      // 可能 panic

// 正确：传播错误或提供合理的默认值/错误处理
let user = db.get_user(id)
    .map_err(|e| AppError::NotFound { id, source: e })?;

let email = user.email
    .ok_or_else(|| AppError::Validation("email is required".into()))?;

let name = config.get("app.name")
    .unwrap_or_else(|| String::from("My App"));
```

## Reference Implementations

| Project                     | Description                                        | PHP LOC | Rust LOC |
|-----------------------------|----------------------------------------------------|---------|----------|
| Glommio (async runtime)     | Purpose-built I/O-uring runtime; replaces php-fpm for network services | N/A | ~30k |
| Meilisearch                 | Originally PHP search engine; Rust core for perf   | ~30k    | ~25k     |
| Toshi (search engine)       | Tantivy-based; replaces Elasticsearch + PHP client | N/A     | ~15k     |
| Blog OS (Philipp Oppermann) | OS tutorial; pattern for PHP devs learning systems | N/A     | ~5k      |
| youki (OCI runtime)         | Docker container runtime; replaces PHP Docker SDKs | N/A     | ~25k     |
| Fuchsia OS (components)     | OS-level; Rust components replacing PHP admin panels | N/A   | ongoing  |

## Cross-Reference

- **java-to-rust**: Class-to-trait mapping; shared enterprise patterns with Laravel/Symfony migration
- **go-to-rust**: Async runtime patterns; PHP Fiber/Goroutine parallelism concepts
- **nodejs-to-rust**: Web framework migration; shared NPM/Composer-to-Cargo dependency mapping
- **c-to-rust**: PHP extension (C-ABI) interop if migrating via extension approach
