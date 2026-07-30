---
name: cpp-to-rust
description: Use when migrating C++ codebases to Rust — covers type mapping, template-to-generics translation, virtual dispatch to trait objects, STL-to-std mapping, move semantics, exception-to-Result conversion, and incremental replacement strategy. Includes canonical signatures, common mistakes, and reference implementations.
---

## Architecture Mapping

C++ and Rust share significant architectural DNA: both are systems-level languages with zero-cost abstractions, deterministic resource management, and strong support for generic programming. The key difference is that Rust enforces at compile time what C++ relies on convention, static analysis, and sanitizers to catch. Classes become structs with `impl` blocks; virtual dispatch becomes trait objects; template metaprogramming becomes generics with trait bounds; and RAII -- which C++ pioneered -- is Rust's universal resource management paradigm through `Drop`.

A C++ project laid out as:

```
src/
  main.cpp
  parser.h / parser.cpp
  network.h / network.cpp
  CMakeLists.txt
  conanfile.txt
```

translates to:

```
src/
  main.rs
  parser.rs
  network.rs
Cargo.toml
build.rs
cpp_src/          (code not yet ported)
```

The most natural path for C++ to Rust migration is **class-to-module**: each `.h/.cpp` pair becomes a `.rs` file, member functions become methods, and inheritance hierarchies become trait-based dispatch.

## Type System Mapping

| C++ Type | Rust Type | Notes |
|----------|-----------|-------|
| `bool` | `bool` | Direct mapping |
| `int8_t` / `uint8_t` | `i8` / `u8` | Fixed-width, identical semantics |
| `int16_t` / `uint16_t` | `i16` / `u16` | Fixed-width |
| `int32_t` / `uint32_t` | `i32` / `u32` | Fixed-width |
| `int64_t` / `uint64_t` | `i64` / `u64` | Fixed-width |
| `size_t` | `usize` | Platform word size |
| `ptrdiff_t` | `isize` | Signed pointer difference |
| `char` | `i8` or `u8`, use `char` crate for Unicode | C++ `char` may be signed or unsigned |
| `std::string` | `String` | UTF-8 validated; `&str` for borrowed slices |
| `std::string_view` | `&str` | Borrowed, non-owning view; same zero-copy semantics |
| `std::vector<T>` | `Vec<T>` | Same contiguous heap allocation |
| `std::array<T, N>` | `[T; N]` | Stack-allocated fixed-size array |
| `std::span<T>` | `&[T]` or `&mut [T]` | Borrowed view into contiguous memory |
| `std::list<T>` | `std::collections::LinkedList<T>` | Doubly-linked list; prefer `VecDeque` if possible |
| `std::deque<T>` | `std::collections::VecDeque<T>` | Ring-buffer deque |
| `std::map<K, V>` | `std::collections::BTreeMap<K, V>` | Ordered map; use `HashMap` for unordered (closer to `std::unordered_map`) |
| `std::unordered_map<K, V>` | `std::collections::HashMap<K, V>` | Same hash-table semantics |
| `std::set<T>` | `std::collections::BTreeSet<T>` | Ordered set |
| `std::unordered_set<T>` | `std::collections::HashSet<T>` | Hash-based set |
| `std::optional<T>` | `Option<T>` | `None` replaces `std::nullopt` |
| `std::variant<A, B, C>` | `enum { A(A), B(B), C(C) }` | Rust enum is more ergonomic; no `std::visit` needed |
| `std::pair<A, B>` | `(A, B)` | Tuple, destructured with pattern matching |
| `std::tuple<A, B, C>` | `(A, B, C)` | Same heterogeneous collection |
| `std::unique_ptr<T>` | `Box<T>` | Single-owner heap allocation; `Box::new(...)` |
| `std::shared_ptr<T>` | `std::sync::Arc<T>` (thread-safe) / `std::rc::Rc<T>` (single-threaded) | Reference counting; Arc is the shared_ptr equivalent |
| `std::weak_ptr<T>` | `std::sync::Weak<T>` / `std::rc::Weak<T>` | Non-owning reference; `upgrade()` mirrors `lock()` |
| `std::function<R(Args...)>` | `Box<dyn Fn(Args) -> R>` or `fn(Args) -> R` | `Fn` for immutable capture; `FnMut` for mutable; `FnOnce` for ownership transfer |
| `std::move` | Ownership transfer (implicit) | Rust moves are always destructive; `Clone` must be explicit |
| `std::forward<T>` | Not needed | Rust does not have forwarding references; use generics directly |
| `nullptr` | `None` (for `Option<&T>`) or `std::ptr::null()` | No null references; Option wraps nullable cases |
| `void` (return type) | `()` (unit type) | `()` is a zero-size value, not "nothing" |
| `T&` (lvalue ref) | `&T` (shared borrow) | Same semantics; compile-time aliasing rules |
| `const T&` | `&T` | All Rust references are immutable by default |
| `T&&` (rvalue ref) | `T` (value, moved) | Rust moves are always destructive; no need for rvalue ref syntax |
| `auto` | Type inference (let) | `let x = ...` infers type; explicit with `let x: T = ...` |
| `decltype(expr)` | `<typeof>` or trait-level associated type | Use `let` with type annotation; in generics use `type Output = ...;` |
| `enum class` | `enum` | Rust enums carry data; no need for `enum class` distinction |
| `union` | `union` (unsafe) or `enum` | Prefer enum for tagged unions; use `union` for FFI only |
| `template<typename T>` | `fn foo<T>(...)` or `struct Foo<T>` | Generics with monomorphization, same zero-overhead principle |
| `concept` / `requires` (C++20) | Trait bounds `T: Trait` | Compile-time interface constraint; trait bounds are the moral equivalent |
| `constexpr` | `const` / `const fn` | `const fn` can be evaluated at compile time |
| `thread_local` | `std::thread_local!` macro | Same per-thread storage semantics |

## Memory & Ownership Model

C++11+ modern memory management (RAII, smart pointers, move semantics) translates to Rust naturally because Rust's ownership model was directly inspired by C++ RAII. The critical shift is that Rust makes moves destructive and enforces the Single Owner Principle at compile time.

| C++ Pattern | Rust Pattern | Semantic Difference |
|-------------|-------------|---------------------|
| `auto p = std::make_unique<T>(args);` | `let p = Box::new(T::new(args));` | Identical semantics |
| `auto p = std::make_shared<T>(args);` | `let p = Arc::new(T::new(args));` | Identical; clone the `Arc` to share |
| `std::move(obj)` | Direct usage; moves are implicit | C++ moved-from objects are "valid but unspecified"; Rust moved-from values are inaccessible |
| `const T& param` (pass by ref-to-const) | `param: &T` | Same: no copy, read-only access |
| `T& param` (mutable output parameter) | `param: &mut T` or return value | Prefer returning a value; use `&mut` only when genuinely needed |
| `std::unique_ptr<T> release()` | `Box::into_raw(b)` + `unsafe { Box::from_raw(p) }` | Manual drop-elision; rare in idiomatic Rust |
| Copy constructor `T(const T&)` | `#[derive(Clone)]` + `fn clone(&self) -> T` | Explicit cloning; no implicit copies |
| Move constructor `T(T&&)` | Not needed | Rust moves are always memcpy + ownership transfer; no special constructor |
| Destructor `~T()` | `impl Drop for T { fn drop(&mut self) { ... } }` | Identical semantics; `Drop::drop` cannot fail |
| Rule of Five / Rule of Zero | `derive(Clone, Debug)` + optional `Drop` | Rust encourages Rule of Zero; the compiler auto-derives common traits |
| Placement new | Not directly available (nightly `box` syntax) | Use `Box::new()` or `Vec::push()`; placement is an optimization detail |
| `std::allocator<T>` | `std::alloc::GlobalAlloc` / `Allocator` (nightly) | Custom allocators via `#[global_allocator]` or per-container (nightly) |
| Small Buffer Optimization (`std::string` SSO) | `smartstring` / `smallvec` crates | Rust `String` is always heap-allocated; crates fill the SSO gap |

## Concurrency / Async Translation

| C++ Pattern | Rust Pattern | Notes |
|-------------|-------------|-------|
| `std::thread` | `std::thread::spawn` | Same join/detach semantics; Rust closures capture safely |
| `std::mutex` | `std::sync::Mutex<T>` | Rust Mutex wraps and owns the guarded data |
| `std::shared_mutex` | `std::sync::RwLock<T>` | Read-write lock |
| `std::lock_guard` | RAII guard (returned by `.lock().unwrap()`) | MutexGuard auto-unlocks on drop |
| `std::condition_variable` | `std::sync::Condvar` | Paired with Mutex |
| `std::atomic<T>` | `std::sync::atomic::Atomic*` | Same memory ordering model (`Acquire`, `Release`, `SeqCst`, etc.) |
| `std::future<T>` / `std::promise<T>` | `Future` trait / `async {}` blocks | Rust futures are lazy (poll-based); C++ futures are eager unless `std::async` deferred |
| `std::async` (std::launch::async) | `tokio::spawn(async { ... })` | Eager execution on a runtime |
| `co_await` (C++20 coroutines) | `.await` | Suspension point; Rust async is stackless, same as C++20 coroutines |
| Thread pool `std::execution::par` (C++17) | `rayon::par_iter()` | Data parallelism |
| `std::jthread` (C++20) | `std::thread::spawn` + `JoinHandle` | Auto-join on drop; Rust requires explicit join or detach |
| `std::latch` / `std::barrier` (C++20) | `std::sync::Barrier` | Thread synchronization primitive |
| `std::counting_semaphore` (C++20) | `tokio::sync::Semaphore` | Async-capable semaphore |
| `boost::asio` | `tokio` runtime | Async I/O networking; tokio covers the full asio surface |
| `std::execution` senders/receivers (C++23) | `tokio` / `async-std` channels | Async channel-based communication |

## Build System & Dependencies

| C++ Tool / Concept | Rust Equivalent | Notes |
|--------------------|-----------------|-------|
| `CMakeLists.txt` | `Cargo.toml` | Declarative; Cargo resolves and fetches dependencies |
| `conanfile.txt` / `vcpkg.json` | `[dependencies]` in `Cargo.toml` | crates.io replaces package managers |
| `.h` / `.hpp` headers | `pub mod` / `pub use` / `pub fn` | Module system; no separate declaration/definition |
| `#include "foo.h"` | `mod foo;` (then `foo.rs` or `foo/mod.rs`) | Module tree replaces textual inclusion |
| `namespace foo { namespace bar { ... } }` | `pub mod foo { pub mod bar { ... } }` | Nested modules; use `pub use` to re-export paths |
| `-I/path/to/include` | Path dependencies in `Cargo.toml`: `foo = { path = "../foo" }` | Cargo resolves include paths via dependency graph |
| `-lfoo` linker flag | `build.rs` with `println!("cargo:rustc-link-lib=foo");` | Link to system libraries via build script |
| `target_link_libraries` (CMake) | Cargo links dependencies automatically | No manual linking for Rust/Rust deps; only for C/C++ native libs |
| `add_compile_definitions(FOO=1)` | `cfg(feature = "foo")` / `--cfg foo` | Conditional compilation via Cargo features |
| `#ifdef` / `#ifndef` / `#if` | `#[cfg(target_os = "linux")]` / `#[cfg(feature = "foo")]` | Attribute-based; no preprocessor |
| `#define FOO bar` | `const FOO: &str = "bar";` | Typed constants |
| `-O2` / `-O3` / `-DNDEBUG` | `[profile.release]` in `Cargo.toml` | `opt-level = 3`, `lto = true`, `codegen-units = 1`, `panic = "abort"` |
| `-fsanitize=address` | `export RUSTFLAGS="-Zsanitizer=address"` (nightly) | ASan requires nightly Rust |
| `clang-tidy` / `cppcheck` | `cargo clippy` | Linter with hundreds of rules; `clippy::pedantic` for strictness |
| `gcov` / `lcov` | `cargo-tarpaulin` / `grcov` | Code coverage tooling |
| `valgrind` | Miri (`cargo +nightly miri test`) | UB detection for unsafe code; Valgrind still useful for FFI |
| Compiler Explorer (godbolt.org) | `cargo asm` / `cargo-show-asm` | Inspect generated assembly |

## Standard Library Mapping

| C++ Standard Library | Rust Equivalent | Notes |
|----------------------|-----------------|-------|
| `std::cout << x` | `println!("{x}")` or `print!("{x}")` | Format strings use `{}`; `{x}` for named arguments |
| `std::cin >> x` | `std::io::stdin().read_line(&mut buf)` | Line-based by default; use `buf.parse::<T>()` to parse |
| `std::cerr << err` | `eprintln!("{err}")` | Stderr output |
| `std::ostringstream` | `format!("{a} {b}")` | Returns `String`; no stream state |
| `std::ifstream` | `std::fs::File::open(path)` | RAII; `BufReader::new(file)` for buffered reads |
| `std::ofstream` | `std::fs::File::create(path)` | `BufWriter::new(file)` for buffered writes |
| `std::getline(stream, line)` | `reader.read_line(&mut line)` | Appends to String |
| `std::stoi` / `std::stod` | `s.parse::<i32>()` / `s.parse::<f64>()` | Returns `Result`, not exception |
| `std::to_string(x)` | `x.to_string()` or `format!("{x}")` | `Display` trait; `format!` for composition |
| `std::filesystem::path` | `std::path::{Path, PathBuf}` | `PathBuf` is owned; `Path` is borrowed; same semantics |
| `std::filesystem::exists` | `path.try_exists()` | Returns `Result<bool>` |
| `std::filesystem::create_directories` | `std::fs::create_dir_all(path)` | Same recursive creation |
| `std::sort(v.begin(), v.end())` | `v.sort()` | In-place sort; `v.sort_by(|a, b| ...)` for custom order |
| `std::stable_sort` | `v.sort()` (Rust `sort` is stable; `sort_unstable` is `std::sort`) | Rust default sort is stable (merge sort); unstable is pattern-defeating quicksort |
| `std::lower_bound` / `std::upper_bound` | `v.binary_search(&x)` / `v.partition_point(...)` | `binary_search` returns `Result<usize, usize>` |
| `std::find(v.begin(), v.end(), x)` | `v.iter().position(|e| *e == x)` | Returns `Option<usize>` |
| `std::count` | `v.iter().filter(|&e| pred(e)).count()` | Iterator adaptors |
| `std::accumulate` | `v.iter().sum()` or `v.iter().fold(init, |acc, x| ...)` | Fold is generalized accumulate |
| `std::transform` | `v.iter().map(|x| f(x)).collect()` | Lazy iterator, collect to materialize |
| `std::copy_if` | `v.iter().filter(|&x| pred(x)).cloned().collect()` | Filter + collect |
| `std::erase(v, x)` (erase-remove idiom) | `v.retain(|e| e != &x)` | In-place removal |
| `std::unique` | `v.dedup()` (adjacent only) or `v.sort(); v.dedup();` | Sort + dedup for full uniqueness |
| `std::chrono::system_clock` | `std::time::SystemTime` / `chrono` crate | `chrono` for formatting, timezone support |
| `std::chrono::duration` | `std::time::Duration` | `Duration::from_secs(5)` etc. |
| `std::this_thread::sleep_for` | `std::thread::sleep(dur)` | Same behavior |
| `std::random_device` / `std::mt19937` | `rand::thread_rng()` / `rand::rngs::StdRng` | From the `rand` crate |
| `std::uniform_int_distribution` | `rng.gen_range(0..10)` | Range syntax |
| `std::regex` | `regex` crate | `Regex::new(r"\d+").unwrap().captures(s)` |
| `std::exception` | `Result<T, E>` + `anyhow::Error` for type-erased errors | No exception hierarchy; use `thiserror` for structured errors |
| `std::optional<T>` | `Option<T>` | `unwrap()`, `expect()`, `map()`, `and_then()` compose identically |
| `std::expected<T, E>` (C++23) | `Result<T, E>` | Same monadic interface; `?` operator like `try` |
| `std::variant<A, B>` | `enum { A(A), B(B) }` | Pattern matching via `match` replaces `std::visit` |
| `std::any` | `Box<dyn Any>` or `dyn Any` trait | Type-erased value; `downcast_ref::<T>()` |
| `typeid(T)` | `std::any::TypeId::of::<T>()` | Compile-time type identity |
| `std::initializer_list<T>` | `&[T]` or `vec![a, b, c]` | Brace-init list becomes slice or vec |
| `RTTI` / `dynamic_cast` | `Any` trait `downcast_ref::<T>()` | Opt-in; no universal RTTI |
| `std::bit_cast<T>` (C++20) | `unsafe { std::mem::transmute::<U, T>(val) }` or `bytemuck` crate | Prefer `bytemuck::cast` for checked transmute |

## Canonical Patterns

### Pattern 1: Class Hierarchy → Trait + Struct

**C++:**
```cpp
// 基类,虚函数定义接口
class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual void scale(double factor) = 0;
};

class Circle : public Shape {
    double radius_;
public:
    explicit Circle(double r) : radius_(r) {}
    double area() const override { return 3.14159 * radius_ * radius_; }
    void scale(double factor) override { radius_ *= factor; }
};

// 使用:
std::vector<std::unique_ptr<Shape>> shapes;
shapes.push_back(std::make_unique<Circle>(5.0));
for (const auto& s : shapes) {
    std::cout << s->area() << "\n";
}
```

**Rust:**
```rust
// trait 定义接口; struct 各自实现
pub trait Shape {
    fn area(&self) -> f64;
    fn scale(&mut self, factor: f64);
}

pub struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }

    fn scale(&mut self, factor: f64) {
        self.radius *= factor;
    }
}

// 使用: Box<dyn Shape> 替代 unique_ptr<Shape>
let mut shapes: Vec<Box<dyn Shape>> = vec![
    Box::new(Circle { radius: 5.0 }),
];
// 或使用 enum 做静态分发(多数情况性能更好):
enum ShapeEnum { Circle { radius: f64 }, Rectangle { w: f64, h: f64 } }
```

### Pattern 2: Template Function → Generic with Trait Bound

**C++:**
```cpp
// C++20 concept 约束模板
template<typename T>
requires std::integral<T> || std::floating_point<T>
T clamp(T value, T lo, T hi) {
    if (value < lo) return lo;
    if (value > hi) return hi;
    return value;
}
```

**Rust:**
```rust
// 泛型 + trait bound,等价于 concept 约束
fn clamp<T: PartialOrd>(value: T, lo: T, hi: T) -> T {
    if value < lo { lo }
    else if value > hi { hi }
    else { value }
}

// 或使用 num_traits crate 做数值泛型约束:
use num_traits::Num;
fn clamp_num<T: Num + PartialOrd + Copy>(value: T, lo: T, hi: T) -> T {
    if value < lo { lo } else if value > hi { hi } else { value }
}
```

### Pattern 3: Virtual Dispatch → Trait Object / Enum Dispatch

**C++:**
```cpp
// 运行时多态,通过 vtable 分发
class Handler {
public:
    virtual ~Handler() = default;
    virtual void handle_event(int event_id) = 0;
};

class LogHandler : public Handler {
    void handle_event(int event_id) override {
        std::cout << "log: " << event_id << "\n";
    }
};

class MetricsHandler : public Handler {
    void handle_event(int event_id) override {
        // increment counter
    }
};
```

**Rust - 方案 A (trait object,等价于虚函数):**
```rust
pub trait Handler {
    fn handle_event(&mut self, event_id: u32);
}

struct LogHandler;
impl Handler for LogHandler {
    fn handle_event(&mut self, event_id: u32) {
        println!("log: {event_id}");
    }
}

// 使用 Box<dyn Handler> 做动态分发:
let handlers: Vec<Box<dyn Handler>> = vec![
    Box::new(LogHandler),
];
```

**Rust - 方案 B (enum dispatch,零虚函数开销,多数场景推荐):**
```rust
pub enum Handler {
    Log,
    Metrics { counter: u64 },
    Custom(Box<dyn Fn(u32) + Send>),
}

impl Handler {
    pub fn handle_event(&mut self, event_id: u32) {
        match self {
            Handler::Log => println!("log: {event_id}"),
            Handler::Metrics { ref mut counter } => *counter += 1,
            Handler::Custom(f) => f(event_id),
        }
    }
}
```

### Pattern 4: Exception → Result

**C++:**
```cpp
// 异常用于错误传播
std::string read_config(const std::string& path) {
    std::ifstream file(path);
    if (!file.is_open()) {
        throw std::runtime_error("cannot open: " + path);
    }
    std::stringstream buffer;
    buffer << file.rdbuf();
    return buffer.str();
}

// 调用端:
try {
    auto config = read_config("/etc/app.conf");
    process(config);
} catch (const std::exception& e) {
    std::cerr << "error: " << e.what() << "\n";
    return 1;
}
```

**Rust:**
```rust
// Result<T, E> 替代异常,? 操作符传播错误
use std::fs;
use std::io;

fn read_config(path: &str) -> io::Result<String> {
    fs::read_to_string(path)
}

// 调用端: 使用 ? 自动传播,或 match / unwrap 处理
fn main() -> anyhow::Result<()> {
    let config = read_config("/etc/app.conf")?;
    process(&config);
    Ok(())
}
```

### Pattern 5: Move Semantics → Ownership Transfer

**C++:**
```cpp
// 移动语义转移所有权
class Buffer {
    std::vector<uint8_t> data_;
public:
    Buffer(std::vector<uint8_t> d) : data_(std::move(d)) {}
    Buffer(Buffer&& other) noexcept : data_(std::move(other.data_)) {}
    Buffer& operator=(Buffer&& other) noexcept {
        data_ = std::move(other.data_);
        return *this;
    }
    // 删除拷贝
    Buffer(const Buffer&) = delete;
    Buffer& operator=(const Buffer&) = delete;
};

Buffer create_buffer() {
    std::vector<uint8_t> data(1024);
    return Buffer(std::move(data));  // move 到返回值
}
```

**Rust:**
```rust
// 所有权转移是默认语义,无需特殊实现
pub struct Buffer {
    data: Vec<u8>,
}

impl Buffer {
    pub fn new(data: Vec<u8>) -> Self {
        Buffer { data }  // data 所有权转移进 Buffer,无需显式 move
    }
}

fn create_buffer() -> Buffer {
    let data = vec![0u8; 1024];
    Buffer::new(data)  // 所有权自然转移
}
// 如果需要阻止 Clone: 直接不 derive Clone 即可
```

### Pattern 6: Singleton → OnceCell / Lazy

**C++:**
```cpp
// Meyer's Singleton (线程安全,懒加载)
class Config {
    Config() = default;
public:
    static Config& instance() {
        static Config inst;
        return inst;
    }
    Config(const Config&) = delete;
    Config& operator=(const Config&) = delete;
};
```

**Rust:**
```rust
use std::sync::OnceLock;

// OnceLock 实现懒加载单例(线程安全)
static CONFIG: OnceLock<Config> = OnceLock::new();

pub fn config() -> &'static Config {
    CONFIG.get_or_init(|| Config::load().expect("failed to load config"))
}

pub struct Config {
    pub database_url: String,
}

impl Config {
    fn load() -> Result<Self, Box<dyn std::error::Error>> {
        // 从文件或环境变量加载
        let url = std::env::var("DATABASE_URL")?;
        Ok(Config { database_url: url })
    }
}
```

### Pattern 7: Iterator Pipeline → Iterator Adaptors

**C++:**
```cpp
// C++20 ranges pipeline
auto result = data
    | std::views::filter([](int x) { return x % 2 == 0; })
    | std::views::transform([](int x) { return x * 2; })
    | std::views::take(5)
    | std::ranges::to<std::vector>();
```

**Rust:**
```rust
// 迭代器链式调用,语义相同
let result: Vec<i32> = data.iter()
    .filter(|&&x| x % 2 == 0)
    .map(|&x| x * 2)
    .take(5)
    .collect();
```

## FFI & Incremental Migration

### Strategy Layers

| Layer | C++ Code | Rust Code | Interface |
|-------|----------|-----------|-----------|
| 1 | Module `.cpp` | Rust `rs` via `extern "C"` | C ABI bridge (extern "C") |
| 2 | `main()` | All business logic | Rust calls C++ via `extern "C"` wrappers |
| 3 | None | Full application | vendored C++ libs via `cc` crate in `build.rs` |

### Wrapping C++ in C ABI for Rust

```cpp
// cpp_lib.h -- C++头文件(纯C ABI,供Rust调用)
#ifdef __cplusplus
extern "C" {
#endif

typedef struct Engine Engine;

Engine* engine_create(const char* config_path);
void engine_destroy(Engine* e);
int engine_process(Engine* e, const char* input, char** output);
void engine_free_string(char* s);

#ifdef __cplusplus
}
#endif

// cpp_lib.cpp -- 实现
Engine* engine_create(const char* config_path) {
    return reinterpret_cast<Engine*>(new CppEngine(config_path));
}

void engine_destroy(Engine* e) {
    delete reinterpret_cast<CppEngine*>(e);
}
```

### Rust Side Binding

```rust
// ffi.rs -- Rust侧的 C ABI 绑定
use std::ffi::{c_char, CStr, CString};

#[repr(C)]
pub struct Engine {
    _private: [u8; 0],  // 不透明类型
}

extern "C" {
    fn engine_create(config_path: *const c_char) -> *mut Engine;
    fn engine_destroy(engine: *mut Engine);
    fn engine_process(
        engine: *mut Engine,
        input: *const c_char,
        output: *mut *mut c_char,
    ) -> i32;
    fn engine_free_string(s: *mut c_char);
}

// 安全封装
pub struct SafeEngine {
    raw: *mut Engine,
}

impl SafeEngine {
    pub fn new(config: &str) -> Result<Self, String> {
        let cfg = CString::new(config).map_err(|e| e.to_string())?;
        let raw = unsafe { engine_create(cfg.as_ptr()) };
        if raw.is_null() {
            Err("failed to create engine".into())
        } else {
            Ok(SafeEngine { raw })
        }
    }

    pub fn process(&mut self, input: &str) -> Result<String, i32> {
        let c_in = CString::new(input).unwrap();
        let mut c_out: *mut c_char = std::ptr::null_mut();
        let rc = unsafe {
            engine_process(self.raw, c_in.as_ptr(), &mut c_out)
        };
        if rc != 0 {
            return Err(rc);
        }
        let result = unsafe { CStr::from_ptr(c_out) }.to_string_lossy().into_owned();
        unsafe { engine_free_string(c_out); }
        Ok(result)
    }
}

impl Drop for SafeEngine {
    fn drop(&mut self) {
        unsafe { engine_destroy(self.raw); }
    }
}
```

### CXX Crate (Type-Safe C++ FFI)

For tighter C++ integration, use the `cxx` crate:

```rust
// 使用 cxx crate 直接在 Rust 中调用 C++ 函数
#[cxx::bridge]
mod ffi {
    unsafe extern "C++" {
        include!("cpp_lib/include/engine.h");

        type Engine;

        fn create_engine(path: &CxxString) -> UniquePtr<Engine>;
        fn process(&self, input: &CxxString) -> Result<CxxString>;
    }
}

fn use_engine() -> Result<(), cxx::Exception> {
    let engine = ffi::create_engine(&cxx::CxxString::new("config.toml"))?;
    let result = engine.process(&cxx::CxxString::new("input"))?;
    Ok(())
}
```

## Common Mistakes

### Mistake 1: Emulating Inheritance with Nested Structs

**Wrong:**
```rust
// 不要: 试图用组合模拟 C++ 继承
struct Base {
    x: i32,
}
struct Derived {
    base: Base,  // "继承" Base
    y: i32,
}
// 然后手动转发方法调用
```

**Right:**
```rust
// 正确: 使用 trait 定义接口,没继承层次就用组合 + trait
trait Entity { fn id(&self) -> u64; }

struct Base { x: i32 }
impl Entity for Base { fn id(&self) -> u64 { self.x as u64 } }

struct Derived { x: i32, y: i32 }
impl Entity for Derived { fn id(&self) -> u64 { (self.x + self.y) as u64 } }

// 需要扩展已有 struct?用 Deref 或直接组合:
struct Enhanced {
    base: Base,
    extra: String,
}
impl std::ops::Deref for Enhanced {
    type Target = Base;
    fn deref(&self) -> &Base { &self.base }
}
```

### Mistake 2: Overusing `unsafe` to Bypass Lifetime Rules

**Wrong:**
```rust
// 不要: 用 unsafe 绕过生命周期,模拟 C++ 的悬垂引用习惯
struct Widget<'a> { data: &'a str }
fn broken() -> Widget<'static> {
    let s = String::from("hello");
    unsafe { std::mem::transmute(Widget { data: &s }) }  // s在函数结束时释放!
}
```

**Right:**
```rust
// 正确: 让生命周期正确表达关系,或使用 owned 类型
struct Widget { data: String }  // 拥有数据,不借用
fn correct() -> Widget {
    Widget { data: String::from("hello") }
}

// 或让调用方提供存储:
fn correct_borrow<'a>(storage: &'a str) -> Widget<'a> {
    Widget { data: storage }
}
```

### Mistake 3: Converting `reinterpret_cast` to `transmute`

**Wrong:**
```rust
// 不要: 将 reinterpret_cast 直译为 transmute
let bytes: [u8; 4] = [0, 0, 128, 63];
let float: f32 = unsafe { std::mem::transmute::<[u8; 4], f32>(bytes) };
// transmute 要求大小完全一致,容易 UB
```

**Right:**
```rust
// 正确: 使用 bytemuck crate 做安全的位级转换
use bytemuck;
let bytes: [u8; 4] = [0, 0, 128, 63];
let float: f32 = bytemuck::cast(bytes);
// bytemuck 在编译期验证大小和对齐

// 或使用标准库的安全方法:
let float = f32::from_ne_bytes(bytes);  // 标准库原生支持
```

### Mistake 4: Using `Box<dyn Any>` for Everything That Was `std::any`

**Wrong:**
```rust
// 不要: 所有类型擦除都用 dyn Any
fn process(items: Vec<Box<dyn Any>>) {
    for item in items {
        if let Some(s) = item.downcast_ref::<String>() {
            println!("{s}");
        }
        // 每个类型都要写一次 downcast,不可扩展
    }
}
```

**Right:**
```rust
// 正确: 用 enum 做封闭集合的静态分发
enum Item { Text(String), Number(i32), Flag(bool) }

fn process(items: Vec<Item>) {
    for item in items {
        match item {
            Item::Text(s) => println!("{s}"),
            Item::Number(n) => println!("{n}"),
            Item::Flag(f) => println!("{f}"),
        }
        // 编译器保证所有分支都被处理
    }
}

// 需要开放集合时才用 trait object:
trait Processable { fn process(&self); }
fn process(items: Vec<Box<dyn Processable>>) {
    for item in items { item.process(); }
}
```

### Mistake 5: Keeping `mutable` Data Members as `Cell` / `RefCell` Everywhere

**Wrong:**
```rust
// 不要: 在 const 方法中修改状态用 RefCell 包装所有字段
struct Cache {
    data: RefCell<HashMap<String, String>>,
    hits: RefCell<u64>,
}
```

**Right:**
```rust
// 正确: 区分逻辑可变和物理可变
// 方案 A: 设计时让需要修改的方法接收 &mut self
struct Cache {
    data: HashMap<String, String>,
    hits: u64,
}

impl Cache {
    fn get(&mut self, key: &str) -> Option<&str> {
        self.hits += 1;
        self.data.get(key).map(|s| s.as_str())
    }
}

// 方案 B: 如果确实需要共享可变(如多线程缓存),用 Arc + 合适的并发原语
use std::sync::Arc;
struct Cache {
    data: dashmap::DashMap<String, String>,  // 并发 HashMap
    hits: std::sync::atomic::AtomicU64,
}

impl Cache {
    fn get(&self, key: &str) -> Option<String> {
        self.hits.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
        self.data.get(key).map(|r| r.clone())
    }
}
```

## Reference Implementations

| Project | Description | Relevant Patterns |
|---------|-------------|-------------------|
| [servo](https://github.com/servo/servo) | Browser engine (replaced C++ layout engines) | Trait-based layout system, parallel CSS styling with rayon |
| [swc](https://github.com/swc-project/swc) | JS/TS compiler (replaced Babel, written in C++ concepts) | AST manipulation, visitor pattern, plugin system |
| [deno](https://github.com/denoland/deno) | JS runtime (replaced C++ V8 embedding patterns) | Async I/O, FFI with C++, multi-threaded event loop |
| [rust-analyzer](https://github.com/rust-lang/rust-analyzer) | IDE backend (replaced C++ clangd architecture) | LSP server, incremental computation, salsa framework |
| [pest](https://github.com/pest-parser/pest) | PEG parser (replaced Boost.Spirit / ANTLR patterns) | Macro-based grammar definition, zero-copy parsing |
| [fish-shell](https://github.com/fish-shell/fish-shell) | Shell (C++ to Rust incremental port) | Incremental migration, C++ FFI strategy |
| [ruff](https://github.com/astral-sh/ruff) | Python linter (replaced C++/Python linters) | AST-walk based linting, caching layer |
| [leftright](https://github.com/jonhoo/leftright) | Concurrent map (replaced C++ concurrent containers) | Lock-free data structures, `Arc`-based ownership |
| [cxx](https://github.com/dtolnay/cxx) | Safe C++ FFI bridge | The canonical tool for C++/Rust interop |
| [rg3d](https://github.com/rg3dengine/rg3d) | Game engine (replaced C++ engines like Unreal/Unity patterns) | ECS architecture, GPU resource management |

## Cross-References

- `c-to-rust`: For C codebases with manual memory management and raw pointers
- `nodejs-to-rust`: For Node.js backends with async event loop patterns
- For C++ FFI details: `cxx` crate documentation at https://cxx.rs
- For build integration: `cc` crate for compiling C++ from `build.rs`; `cmake` crate for CMake-based deps
- For migrating large class hierarchies: consider the `enum_dispatch` crate for zero-cost dynamic dispatch
- For serialization replacing Boost.Serialization: `serde` crate with `serde_json`, `bincode`, `rmp-serde`
