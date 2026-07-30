---
name: ruby-to-rust
description: Use when migrating Ruby codebases to Rust — covers dynamic-to-static typing, blocks/closures to Fn traits, metaprogramming to macros/generics, Rails to Axum/Actix, ActiveRecord to Diesel/sqlx, GIL to true parallelism, garbage collection to ownership, Bundler to Cargo, and incremental migration via sidecar services or FFI. Includes canonical code patterns, common mistakes, and reference implementations.
updated: 2026-07-30
---

# Ruby to Rust Migration

## Architecture Mapping

Ruby's MRI/YARV interpreter (with generational GC, Global VM Lock, and object-oriented-everything paradigm) maps to Rust's AOT-compiled native binary with no runtime overhead. Where Ruby encourages "duck typing" and runtime metaprogramming (`method_missing`, `define_method`, `class_eval`), Rust requires compile-time type resolution with trait-based polymorphism. Ruby's GIL (Global VM Lock) is replaced by Rust's `Send + Sync` traits enabling true multi-threaded parallelism. A Rails application served by Puma becomes a single Rust binary served by Axum or Actix-Web. Bundler's `Gemfile` becomes `Cargo.toml` with lockfile-based reproducible builds.

Ruby's "developer happiness" philosophy maps to Rust's "compiler-as-teacher" approach — both prioritize ergonomics, but Rust enforces correctness at compile time. The critical paradigm shift: Ruby blocks, Procs, and Lambdas become Rust's closure traits (`Fn`, `FnMut`, `FnOnce`); Ruby's `include`/`extend` mixins become Rust's trait composition; and Ruby's exception-driven error handling becomes `Result<T, E>` with the `?` operator.

| Ruby Concept | Rust Equivalent | Notes |
|-------------|-----------------|-------|
| MRI/YARV interpreter | rustc + LLVM (AOT) | No interpreter, no warmup |
| GC (generational, compaction) | Ownership + RAII | Deterministic, zero pause |
| GIL (Global VM Lock) | `Send + Sync` traits | True multi-threaded parallelism |
| `class` / `module` | `struct` + `impl` + `trait` | Data + behavior separated |
| Mixin (`include` / `extend`) | Trait + default methods | Composition over inheritance |
| `method_missing` | Trait dispatch + generics | Compile-time, not runtime |
| `define_method` (dynamic) | Proc macros / code generation | Compile-time metaprogramming |
| `attr_accessor` | `pub` field or getter/setter | Explicit access |
| Block / `yield` | `FnMut` closure + `.call()` or `for` | Explicit closure type |
| Proc / Lambda | `Box<dyn Fn()>` / `fn()` pointer | Typed callables |
| `&:method` (symbol-to-proc) | Method reference `T::method` | Compile-time resolution |
| `unless` / `if` modifier | `if !cond { }` / regular `if` | No postfix conditionals |
| `case/when` | `match` expression | Exhaustive, expression-oriented |
| `rescue` / `ensure` | `Result<T, E>` + `?` / `Drop` | Errors as values |
| `nil` | `Option<T>` | Explicit absence everywhere |
| `true` / `false` (only falsey: nil, false) | `bool` (only `true`/`false`) | No truthy/falsey coercion |
| `Symbol` | `&'static str` / enum / `string_cache` | Interned strings |
| `Hash` | `HashMap<K,V>` / `BTreeMap<K,V>` | No symbol-key shortcut |
| `Array` | `Vec<T>` | Typed, homogeneous |
| `Range` | `start..end` / `start..=end` | Range expressions |
| `Enumerable` module | `Iterator` trait | Lazy combinators |
| `String` (mutable) | `String` (owned) / `&str` (borrowed) | Explicit mutability |
| `open class` / refinement | Extension trait (`impl T for X`) | Open extension via traits |
| `require` / `require_relative` | `mod` / `use` | Compile-time module resolution |
| `$LOAD_PATH` | Cargo dependency graph | No runtime path manipulation |

## Type System Mapping

| Ruby Type | Rust Type | Notes |
|-----------|-----------|-------|
| `Integer` (bigint) | `i64` / `i128` / `num_bigint::BigInt` | Ruby Integer is arbitrary-precision |
| `Float` | `f64` | IEEE 754 double |
| `Rational` | `num_rational::Rational64` | Fractional numbers |
| `Complex` | `num_complex::Complex<f64>` | Complex numbers |
| `String` | `String` / `&str` | Ruby strings are mutable byte sequences |
| `Symbol` | `&'static str` / custom enum | Interned, immutable |
| `Array` | `Vec<T>` | Ordered, indexed |
| `Hash` | `HashMap<K,V>` / `BTreeMap<K,V>` | Key-value storage |
| `Set` | `HashSet<T>` / `BTreeSet<T>` | Unique elements |
| `Range` | `Range<T>` / `RangeInclusive<T>` | Start..end |
| `true` / `false` | `bool` | Boolean values |
| `nil` | `Option::None` | Absence of value |
| `Proc` (callable) | `Box<dyn Fn(A) -> R>` / `fn(A) -> R` | Typed callable |
| `Time` / `DateTime` | `chrono::DateTime<Utc>` | Timezone-aware |
| `Date` | `chrono::NaiveDate` | Date only |
| `Regexp` | `regex::Regex` | Regular expressions |
| `File` / `IO` | `std::fs::File` / `std::io::Read` | File I/O |
| `Pathname` | `std::path::{Path, PathBuf}` | File paths |
| `Struct` (stdlib) | `struct` (language) | Named value types |
| `Data` (Ruby 3.2+) | `struct` with `#[derive(Clone, Debug)]` | Immutable value object |
| `OpenStruct` | `HashMap<String, Value>` / concrete struct | Prefer concrete types |

## Memory & Ownership Model

Ruby's generational GC (with incremental marking and compaction) means objects live as long as they are reachable. Rust's ownership system replaces the GC entirely. This is the hardest mental shift for Ruby developers — there is no `GC.start`, no `ObjectSpace`, and no `finalizer`.

| Ruby Pattern | Rust Translation |
|-------------|------------------|
| Pass object, mutate in place | `&mut T` (exclusive borrow) |
| Pass object, read only | `&T` (shared borrow) |
| GC cleans up unreachable | `Drop` trait, scope-based |
| `ObjectSpace.define_finalizer` | `impl Drop for T` | 
| `WeakRef` | `Weak<T>` (from `Rc`/`Arc`) |
| `dup` / `clone` | `.clone()` (explicit, `Clone` trait) |
| `freeze` | `let` binding (immutable by default) |
| Thread-local variables | `thread_local!` macro |
| `$SAFE` / taint levels | Not applicable — memory safety at compile time |

### The GC-Free Mindset

```ruby
# Ruby: everything is a heap-allocated object, GC handles cleanup
class OrderService
  def initialize(db, payment_gateway)
    @db = db
    @payment_gateway = payment_gateway
  end

  def place_order(request)
    total = request.items.sum(&:price)
    charge = @payment_gateway.charge(total, request.card_token)
    @db.orders.create(request.to_h.merge(charge_id: charge.id))
  end
end
```

```rust
// Rust: explicit ownership — data owns its heap allocations
pub struct OrderService<P: PaymentGateway> {
    db: PgPool,
    payment: P,
}

impl<P: PaymentGateway> OrderService<P> {
    pub async fn place_order(&self, request: &OrderRequest) -> Result<Order, OrderError> {
        let total: f64 = request.items.iter().map(|i| i.price).sum();
        let charge = self.payment.charge(total, &request.card_token).await?;
        let order = sqlx::query_as::<_, Order>("INSERT INTO orders (...) VALUES (...) RETURNING *")
            .bind(&request.customer_id)
            .bind(total)
            .bind(&charge.id)
            .fetch_one(&self.db)
            .await?;
        Ok(order)
    }
}
```

## Concurrency / Async Translation

Ruby has multiple concurrency models: threads (GIL-limited for CPU, usable for I/O), fibers (cooperative), Ractors (experimental, isolated), and async gems (`async`, `eventmachine`). Rust provides true multi-threaded parallelism via tokio for async I/O and rayon for CPU-bound work.

| Ruby | Rust / Tokio |
|------|--------------|
| `Thread.new { }` | `std::thread::spawn(move || { })` |
| `Thread.new { }.join` | `handle.join().unwrap()` |
| `Fiber.new { }.resume` | `async fn` + `.await` (stackless coroutines) |
| `Async { }` (async gem) | `tokio::spawn(async { })` |
| `Async::Barrier` | `tokio::sync::Barrier` |
| `sleep(n)` | `tokio::time::sleep(Duration::from_secs(n)).await` |
| `Timeout.timeout(n) { }` | `tokio::time::timeout(dur, future)` |
| `Queue` (stdlib, thread-safe) | `std::sync::mpsc::channel` |
| `Concurrent::Future` (concurrent-ruby) | `tokio::task::JoinHandle<T>` |
| `Concurrent::Promise` | `futures::future::Map` combinators |
| Ractor | `std::thread::spawn` + `mpsc` channel |
| `Mutex` (stdlib) | `std::sync::Mutex<T>` |
| `Monitor` / `MonitorMixin` | `std::sync::Mutex<T>` + `Condvar` |
| `ConditionVariable` | `std::sync::Condvar` |
| `Concurrent::Array` / `Concurrent::Hash` | `Arc<Mutex<Vec<T>>>` / `Arc<Mutex<HashMap<K,V>>>` |
| `EventMachine` | `tokio` runtime (work-stealing M:N scheduler) |
| Sidekiq / background jobs | `tokio::spawn` + Redis / `apalis` crate |

### async.rb → tokio

```ruby
# Ruby: async gem — reactor-pattern concurrency
require 'async'

Async do
  user = Async { fetch_user(id) }
  orders = Async { fetch_orders(id) }
  Dashboard.new(user.wait, orders.wait)
end
```

```rust
// Rust: tokio join — concurrent futures
async fn load_dashboard(id: &str) -> Result<Dashboard, AppError> {
    let (user, orders) = tokio::join!(
        fetch_user(id),
        fetch_orders(id),
    );
    Ok(Dashboard::new(user?, orders?))
}
```

## Build System & Dependencies

| Ruby Tool | Rust Equivalent |
|-----------|-----------------|
| `Gemfile` + `Gemfile.lock` | `Cargo.toml` + `Cargo.lock` |
| `bundle install` | `cargo build` |
| `bundle exec` | `cargo run` |
| Bundler / RubyGems | Cargo / crates.io |
| `rbenv` / `rvm` (Ruby versions) | `rustup` (Rust toolchain) |
| `rake` (task runner) | `cargo` subcommands / `just` / `xtask` |
| RuboCop / Standard | `cargo clippy` / `cargo fmt` |
| RSpec / Minitest | `#[test]` + `cargo test` |
| SimpleCov | `cargo tarpaulin` / `cargo-llvm-cov` |
| `pry` / `irb` (REPL) | `evcxr` (Rust REPL) / `cargo script` |
| Spring / Bootsnap (preloader) | Not needed — AOT compilation |
| Sorbet / RBS (type checking) | Built-in static type system |

**Cargo.toml for a migrated Rails API service:**

```toml
[package]
name = "web-api"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"
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
reqwest = { version = "0.12", features = ["json"] }
jsonwebtoken = "9"
argon2 = "0.5"
```

## Framework Mapping: Rails → Axum/Actix

| Rails Component | Rust Equivalent | Notes |
|-----------------|-----------------|-------|
| `config/routes.rb` | `Router::route(path, method(handler))` | Route definitions |
| `ApplicationController` | Shared middleware / `Extension` layers | Base behavior |
| `before_action` callback | Middleware layer / `tower::ServiceBuilder` | Pre-handler logic |
| `params` (ActionController) | `Json<T>` / `Query<T>` / `Path<T>` extractor | Typed extraction |
| `render json:` | `Json(response)` | Serde-based serialization |
| Active Record model | `sqlx::FromRow` / Diesel schema | ORM mapping |
| Active Record query | `sqlx::query_as` / `diesel::dsl` | Compile-time checked |
| Active Record migration | `sqlx migrate` / `diesel migration` | Versioned SQL |
| Active Record validation | `validator` crate / constructor validation | Typed validation |
| Active Job / Sidekiq | `tokio::spawn` + Redis | Background processing |
| Action Cable (WebSocket) | `axum::extract::ws` | Built-in WebSocket |
| Action Mailer | `lettre` crate / `aws-sdk-ses` | Email delivery |
| Active Storage | `aws-sdk-s3` / object_store crate | File storage |
| Devise (auth) | `axum-login` / custom JWT middleware | Authentication |
| Pundit / CanCanCan | Custom guard trait or middleware | Authorization |
| Rails Console | `evcxr` REPL | Interactive Rust |
| `rails db:migrate` | `sqlx migrate run` / `diesel migration run` | |
| View / ERB / Slim / Haml | `askama` / `tera` / `minijinja` | Template engines |
| `config/database.yml` | `.env` / config struct | Database URL |
| `Rails.cache` | `redis` crate / `moka` (in-memory) | Caching |

## Standard Library & Ecosystem Mapping

### Core Extensions

| Ruby | Rust |
|------|------|
| `[1,2,3].map { |x| x * 2 }` | `vec![1,2,3].iter().map(|x| x * 2).collect()` |
| `[1,2,3].select(&:even?)` | `vec.iter().filter(|&&x| x % 2 == 0).collect()` |
| `[1,2,3].reject { |x| x.zero? }` | `vec.iter().filter(|&&x| x != 0).collect()` |
| `[1,2,3].reduce(0, &:+)` | `vec.iter().sum()` |
| `[1,2,3].each { |x| puts x }` | `for x in &vec { println!("{x}"); }` |
| `[1,2,3].flat_map { |x| [x, x*2] }` | `vec.iter().flat_map(|x| [*x, x*2]).collect()` |
| `[1,2,3].find { |x| x > 1 }` | `vec.iter().find(|&&x| x > 1)` returns `Option` |
| `[1,2,3].any?(&:even?)` | `vec.iter().any(|x| x % 2 == 0)` |
| `[1,2,3].all? { |x| x > 0 }` | `vec.iter().all(|&x| x > 0)` |
| `[1,2,3].include?(2)` | `vec.contains(&2)` |
| `[1,2,3].first(2)` / `.last(2)` | `&vec[..2]` / `vec.iter().rev().take(2)` |
| `(1..10).step(2)` | `(1..=10).step_by(2)` |
| `"hello".upcase` / `.downcase` | `"hello".to_uppercase()` / `.to_lowercase()` |
| `"hello".reverse` | `"hello".chars().rev().collect::<String>()` |
| `"hello".gsub(/l/, 'w')` | `regex.replace_all("hello", "w")` |
| `"hello".start_with?("he")` | `"hello".starts_with("he")` |
| `"a,b,c".split(",")` | `"a,b,c".split(',')` returns iterator |
| `["a","b"].join(",")` | `["a","b"].join(",")` or itertools |
| `"  trim  ".strip` | `"  trim  ".trim()` |
| `"%05d" % 42` / `sprintf` | `format!("{n:05}")` |
| `hash = { a: 1, b: 2 }` | `let mut map = HashMap::new(); map.insert("a", 1);` |
| `hash[:key]` / `hash.fetch(:key, default)` | `map["key"]` / `map.get("key").unwrap_or(&default)` |
| `hash.transform_keys(&:to_s)` | `map.into_iter().map(|(k,v)| (k.to_string(), v)).collect()` |
| `hash.merge(other_hash)` | `map.extend(other_map)` |
| `Digest::SHA256.hexdigest(data)` | `sha2::Sha256::digest(data)` |
| `Base64.encode64(data)` | `base64::Engine::encode(&data)` |
| `JSON.parse(str)` / `JSON.generate(obj)` | `serde_json::from_str(&str)` / `to_string(&obj)` |
| `YAML.safe_load(str)` | `serde_yaml::from_str(&str)` |
| `CSV.read(path)` | `csv::Reader::from_path(path)` |

## Canonical Patterns

### 1. Class → Struct + impl

```ruby
# Ruby: class-based, duck-typed
class OrderNotifier
  def initialize(email_service, template_engine)
    @email = email_service
    @templates = template_engine
  end

  def notify_shipped(order)
    body = @templates.render('shipped', order: order)
    @email.send(to: order.customer_email, subject: "Shipped!", body: body)
  end
end
```

```rust
// Rust: struct + impl with trait bounds
pub struct OrderNotifier<E: EmailService, T: TemplateEngine> {
    email: E,
    templates: T,
}

impl<E: EmailService, T: TemplateEngine> OrderNotifier<E, T> {
    pub fn new(email: E, templates: T) -> Self {
        Self { email, templates }
    }

    pub async fn notify_shipped(&self, order: &Order) -> Result<(), AppError> {
        let body = self.templates.render("shipped", &order.into_context())?;
        self.email
            .send(&order.customer_email, "Shipped!", &body)
            .await?;
        Ok(())
    }
}
```

### 2. Block → Closure / Iterator

```ruby
# Ruby: block-based iteration
def active_user_names(users)
  users
    .select { |u| u.active? }
    .map(&:name)
    .sort
end
```

```rust
// Rust: iterator combinators
fn active_user_names(users: &[User]) -> Vec<String> {
    let mut names: Vec<_> = users
        .iter()
        .filter(|u| u.active)
        .map(|u| u.name.clone())
        .collect();
    names.sort();
    names
}
```

### 3. Exception → Result

```ruby
# Ruby: begin/rescue/ensure
def transfer(from, to, amount)
  raise ArgumentError, "invalid amount" if amount <= 0
  raise InsufficientFundsError unless from.balance >= amount

  from.withdraw(amount)
  to.deposit(amount)
rescue InsufficientFundsError => e
  log_error(e)
  raise
ensure
  audit_log.record(from, to, amount)
end
```

```rust
// Rust: Result with ? operator; Drop replaces ensure
fn transfer(from: &mut Account, to: &mut Account, amount: f64) -> Result<(), TransferError> {
    if amount <= 0.0 { return Err(TransferError::InvalidAmount); }
    if from.balance < amount { return Err(TransferError::InsufficientFunds); }

    let _guard = AuditGuard::new(from.id, to.id, amount); // Drop logs on any exit
    from.withdraw(amount)?;
    to.deposit(amount)?;
    Ok(())
}
```

### 4. DSL → Builder Pattern

```ruby
# Ruby: DSL-style configuration
Rails.application.configure do
  config.cache_classes = true
  config.log_level = :info
end
```

```rust
// Rust: builder pattern
use typed_builder::TypedBuilder;

#[derive(TypedBuilder)]
pub struct AppConfig {
    #[builder(default = true)]
    cache_classes: bool,
    #[builder(default = "info")]
    log_level: String,
}

let config = AppConfig::builder()
    .cache_classes(true)
    .log_level("info".into())
    .build();
```

### 5. Mixin → Trait

```ruby
# Ruby: module mixin via include
module Timestampable
  def touch
    @updated_at = Time.now
  end
end

class Post
  include Timestampable
end
```

```rust
// Rust: trait with default implementation
pub trait Timestampable {
    fn touch(&mut self) {
        // default implementation
    }
}

impl Timestampable for Post {
    // inherits default touch() or override
}
```

## FFI & Incremental Migration

| Strategy | Tool | When |
|----------|------|------|
| Sidecar binary | JSON/stdin-stdout or HTTP localhost | Batch processing, workers |
| FFI via C ABI | `extern "C"` + FFI gem | Performance-critical functions |
| HTTP/gRPC extraction | Axum web service + reverse proxy | API boundary migration |
| rubsys / magnus | `magnus` crate — Ruby extensions in Rust | Embed Rust in Ruby gems |
| Helix / Rutie | Rust-native Ruby extensions | Replace C extensions |
| Shared database | Both read/write same DB | Transitional phase |

### Magnus: Rust Extensions for Ruby (Recommended)

```ruby
# Ruby: calling Rust via magnus
require 'my_rust_lib'
result = MyRustLib.heavy_computation(large_dataset)
```

```rust
// Rust: magnus extension
use magnus::{function, prelude::*, Error, Ruby};

fn heavy_computation(data: Vec<i64>) -> Vec<i64> {
    data.into_iter().map(|x| x * 2).collect()
}

#[magnus::init]
fn init(ruby: &Ruby) -> Result<(), Error> {
    let module = ruby.define_module("MyRustLib")?;
    module.define_singleton_method("heavy_computation", function!(heavy_computation, 1))?;
    Ok(())
}
```

### Migration Order

1. **Schema & types**: Define shared JSON/Protobuf schema; both Ruby and Rust consume
2. **Hot-path functions**: Replace CPU-intensive Ruby methods with magnus-extended Rust
3. **Background jobs**: Replace Sidekiq workers with Rust binaries reading the same Redis
4. **Read endpoints**: Build Rust service reading the same database; route via nginx
5. **Write endpoints**: Migrate mutation handlers; maintain transactional consistency
6. **Full cutover**: Remove Ruby runtime; keep magnus bridges for scripting

## Common Mistakes

### Mistake 1: `unwrap()` Everywhere (Nil-Check Mindset)

```rust
// WRONG: Ruby devs used to nil checks, translate to unwrap
let user = db.find(id).unwrap();  // panics if not found!
let name = user.name.unwrap();    // panics if None!

// CORRECT: propagate with ? or match
let user = db.find(id)?.ok_or(AppError::NotFound)?;
let name = user.name.unwrap_or("anonymous".into());
```

### Mistake 2: Overusing `clone()` (GC Mindset)

```ruby
# Ruby: GC handles copying, freely create new objects
items.each { |item| cache.set(item.id, item.dup) }
```

```rust
// WRONG: cloning everything
for item in &items {
    cache.set(item.id.clone(), item.clone()); // unnecessary clones
}

// CORRECT: borrow or move
for item in &items {
    cache.set(&item.id, item); // borrow key, reference value
}
```

### Mistake 3: Trying to Recreate `method_missing`

```rust
// WRONG: trying to create dynamic dispatch like method_missing
// Rust has no runtime method_missing equivalent for production use

// CORRECT: use enums + match for closed dynamic sets
enum Action { Create, Update, Delete }

impl Handler {
    fn handle(&self, action: Action) -> Result<(), AppError> {
        match action {
            Action::Create => self.create()?,
            Action::Update => self.update()?,
            Action::Delete => self.delete()?,
        }
    }
}
```

### Mistake 4: Mutable String Confusion

```ruby
# Ruby: strings are mutable
name = "Alice"
name << " Smith"  # modifies in place
name.upcase!      # modifies in place
```

```rust
// Rust: explicit mutability
let mut name = String::from("Alice");
name.push_str(" Smith");  // mutates
let upper = name.to_uppercase();  // new String, name unchanged
```

## Reference Implementations

| Project | Description | Pattern |
|---------|-------------|---------|
| YJIT (CRuby JIT) | Ruby JIT compiler; C → Rust core | Incremental replacement in MRI |
| Parser (Prism) | Ruby parser; C+Ruby → Rust | Performance-critical library rewrite |
| Artichoke | Ruby interpreter in Rust | Full runtime reimplementation |
| magnus | Ruby C extension replacement | FFI crate for Ruby↔Rust |
| Meilisearch | Ruby SDK + Rust core | Shared wire protocol |
| Sorbet (partial) | Ruby type checker | Static analysis in Rust |

## Cross-Reference

- **python-to-rust**: Shared dynamic-to-static migration patterns; Django/Rails framework parallels
- **php-to-rust**: Web framework migration (Laravel/Rails); shared Composer/Bundler patterns
- **lua-to-rust**: Embedded scripting and DSL migration; dynamic typing to struct/enum
- **nodejs-to-rust**: Async runtime migration; shared EventMachine/tokio patterns
- **java-to-rust**: Enterprise patterns; Spring Boot/Rails service architecture
