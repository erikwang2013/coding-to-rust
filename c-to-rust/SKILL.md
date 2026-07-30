---
name: c-to-rust
description: Use when migrating C codebases to Rust — covers type mapping, ownership translation, pointer-to-reference migration, build system conversion, FFI bridging, and incremental replacement strategy. Includes canonical signatures, common mistakes, and reference implementations.
---

## Architecture Mapping

C programs follow a procedural paradigm: global mutable state, manual memory management via `malloc`/`free`, header-based interface declarations, and platform-conditional compilation via the preprocessor. Rust replaces each of these with a zero-cost-abstraction equivalent: modules replace headers, ownership replaces manual allocation, `Cargo.toml` replaces Makefiles, and `cfg` attributes replace `#ifdef`. The translation is structural -- C's flat namespace becomes Rust's hierarchical module tree, C's opaque pointer handles become newtype wrappers, and C's `errno`-plus-`goto cleanup` becomes `Result<T, E>` with RAII-based cleanup.

A C project organized as:

```
src/
  main.c
  parser.h
  parser.c
  network.h
  network.c
Makefile
```

becomes:

```
src/
  main.rs
  parser.rs
  network.rs
Cargo.toml
build.rs        (if native deps remain)
c_src/          (staging area for code not yet ported)
```

## Type System Mapping

| C Type | Rust Type | Notes |
|--------|-----------|-------|
| `int` / `unsigned int` | `i32` / `u32` | Use sized types; `c_int` for FFI only |
| `long` / `unsigned long` | `i64` / `u64` (Linux) or `i32` / `u32` (Windows) | Platform-dependent; use `std::os::raw::c_long` for FFI |
| `size_t` | `usize` | Natural word size; use `libc::size_t` for FFI bindings |
| `char` | `i8` or `u8` | C `char` is ambiguous; choose based on signedness |
| `char*` (string) | `&str` / `String` / `CStr` / `CString` | `CStr` for borrowed FFI strings, `CString` for owned |
| `void*` | `*mut c_void` or generics `T` | `NonNull<c_void>` for guaranteed non-null opaque handles |
| `T*` (pointer) | `&T` / `&mut T` / `Box<T>` | Convert to reference where lifetime is clear; `Box` where heap-allocated |
| `T[N]` (fixed array) | `[T; N]` | Stack-allocated, size known at compile time |
| `struct { ... }` | `struct { ... }` | Add `#[repr(C)]` for FFI, omit otherwise for optimizer freedom |
| `union { ... }` | `union { ... }` or `enum` | Prefer tagged enum unless FFI requires untagged union |
| `enum { ... }` | `enum { ... }` | Rust enums carry payloads; use `#[repr(C)]` only for FFI C-compat |
| `typedef` | `type Alias = ...;` | Type alias syntax |
| `void (*)(int)` (function pointer) | `fn(i32)` or `Box<dyn Fn(i32)>` | fn pointers for stateless; closures for state capture via FnMut/FnOnce |
| `_Bool` / `bool` | `bool` | Direct mapping |
| `NULL` | `None` / `std::ptr::null()` / `std::ptr::null_mut()` | Use `Option<&T>` or `Option<NonNull<T>>` for nullable pointers |
| `float` / `double` | `f32` / `f64` | Direct mapping |

## Memory & Ownership Model

C's model is **programmer-managed**: you `malloc`, you `free`. Rust's model is **compiler-enforced**: ownership is transferred or borrowed through the type system, and memory is freed when the owner goes out of scope. This is the single hardest conceptual leap for C programmers.

| C Pattern | Rust Pattern | Why |
|-----------|-------------|-----|
| `T* p = malloc(sizeof(T));` | `let p = Box::new(T::new());` | Owned heap allocation; dropped automatically |
| `free(p);` | Automatic (drop at end of scope) | No manual free needed |
| `T* arr = malloc(N * sizeof(T));` | `let arr = vec![T::default(); N];` or `let arr: Vec<T> = (0..N).map(...).collect();` | Vec owns the heap buffer |
| Shared read-only `T*` | `&T` (shared reference) | Compiler guarantees no mutable aliasing |
| Mutating through `T*` | `&mut T` (exclusive reference) | Only one `&mut` at a time, no data races |
| Reference counting `struct { int refcount; T* data; }` | `Rc<T>` (single-threaded) / `Arc<T>` (thread-safe) | No manual refcount management |
| `realloc` | `Vec::resize()` / `Vec::reserve()` | Vec manages capacity and growth automatically |
| Custom allocator | `std::alloc::{GlobalAlloc, Allocator}` | Rust nightly has the `Allocator` trait; stable uses `#[global_allocator]` |
| `memcpy(dst, src, n)` | `dst[..n].copy_from_slice(&src[..n]);` | Bounds-checked; panics on overlap or OOB |
| `memset(ptr, 0, n)` | `slice.fill(0);` or `vec.resize(N, 0);` | Type-safe, no raw pointer arithmetic |
| `memmove(dst, src, n)` | `dst.copy_within(src_range, dst_start);` | Bounds-checked, handles overlap safely |

## Concurrency / Async Translation

| C Pattern | Rust Pattern | Notes |
|-----------|-------------|-------|
| `pthread_create` | `std::thread::spawn` | Type-safe closure as entry point |
| `pthread_mutex_t` | `std::sync::Mutex<T>` | Mutex wraps and owns the data |
| `pthread_cond_t` | `std::sync::Condvar` | Paired with Mutex |
| `pthread_rwlock_t` | `std::sync::RwLock<T>` | Multiple readers or single writer |
| `sem_t` (POSIX semaphore) | `std::sync::Semaphore` (crate `tokio`), `std::sync::mpsc` | Use channels as a safer alternative |
| `pthread_once` | `std::sync::Once` or `once_cell::sync::OnceCell` | Lazy initialization |
| Thread-local `__thread` | `std::thread_local!` macro | `thread_local!` creates a `LocalKey` |
| `atomic_*` functions | `std::sync::atomic::{AtomicBool, AtomicUsize, ...}` | Same ordering semantics (`Ordering::SeqCst`, etc.) |
| `fork()` | `std::process::Command` | Avoid `fork`; use subprocess spawn |
| `epoll` / `kqueue` / `select` | `tokio::net` / `mio` for low-level | mio wraps epoll/kqueue/IOCP; tokio for ergonomic async |
| `poll` / `select` loops | `tokio::select!` macro | Structured concurrency: race multiple futures on readiness |
| `libuv` event loop | `tokio` runtime | Single-threaded or multi-threaded async runtime |
| `signal(SIGINT, handler)` | `tokio::signal::ctrl_c()` | Async signal handling within the runtime |

## Build System & Dependencies

| C Tool / Concept | Rust Equivalent | Notes |
|-----------------|-----------------|-------|
| `Makefile` / `CMakeLists.txt` | `Cargo.toml` | Declarative; Cargo manages fetch, build, link |
| `./configure` + `make` | `cargo build` | Single build command; no separate configure step |
| Header file `.h` | `pub mod` / `pub use` | Rust modules define their interface within source files |
| `#include "foo.h"` | `mod foo;` (after `foo.rs` / `foo/mod.rs`) | Module system replaces textual inclusion |
| `-I/path` include dirs | `[dependencies]` in `Cargo.toml` | External crates fetched by Cargo; local paths via `path = "..."` |
| `-lfoo` linker flag | `[build-dependencies]` + `build.rs` with `cargo:rustc-link-lib=foo` | `build.rs` for native library linking |
| `gcc -DNAME=VALUE` | `cfg(feature = "name")` in code; `--features name` on CLI | Conditional compilation via features |
| `#ifdef` / `#ifndef` | `#[cfg(target_os = "linux")]` / `#[cfg(feature = "foo")]` | Attribute-based conditional compilation |
| `#define PI 3.14` | `const PI: f64 = 3.14;` | `const` is typed and namespaced |
| `#define MAX(x,y) ((x)>(y)?(x):(y))` | `fn max<T: Ord>(a: T, b: T) -> T { a.max(b) }` or `T::max` | Generic function replaces function-like macro; typesafe |
| `pkg-config` | `system-deps` crate in `build.rs` | Declarative pkg-config dependency specification |
| Compiler warning flags | `#![deny(unsafe_code)]` / clippy lints | Opt-in lint levels in `Cargo.toml` and source attributes |
| `-O2` / `-O3` | `[profile.release]` section in `Cargo.toml` | opt-level, lto, codegen-units, panic = "abort" |

## Standard Library Mapping

| C Standard Library | Rust Equivalent | Notes |
|--------------------|-----------------|-------|
| `printf` / `fprintf` | `println!` / `write!` / `eprintln!` | Format strings use `{}` not `%d`; compile-time checked |
| `snprintf` | `format!` macro | Returns `String`; no fixed buffer required |
| `scanf` / `sscanf` | `text.split_whitespace()` / `str::parse()` / `regex` crate | Manual parsing is safer and more explicit |
| `fopen / fread / fclose` | `std::fs::File` / `std::io::Read` | RAII file handles; `File` closes on drop |
| `fwrite` | `std::io::Write` trait | `file.write_all(&buf)` |
| `fseek / ftell` | `file.seek(SeekFrom::Start(n))` | Enum-based seek positions |
| `errno` / `perror` | `std::io::Error` / `Result<T, io::Error>` | Error carries kind (`ErrorKind`) and message |
| `strerror(errno)` | `io::Error::from_raw_os_error(errno)` | Wrap OS errors in `io::Error` |
| `strlen(s)` | `s.len()` | O(1) length; Rust slices always know their length |
| `strcmp / strncmp` | `a == b` (for `&str`) or `a.starts_with(b)` | String comparison is built-in |
| `strcpy / strncpy` | `dst = src.to_string()` or `dst.clone_from(&src)` | No buffer overflow risk |
| `strcat` | `s.push_str(&other)` or `format!("{a}{b}")` | `String::push_str` appends in-place |
| `strdup` | `s.to_string()` | Returns an owned `String` |
| `strstr` | `s.contains(needle)` or `s.find(needle)` | `contains` for bool; `find` for `Option<usize>` |
| `strtok` | `s.split(delimiter)` or `s.split_whitespace()` | Returns iterator; no global state |
| `atoi / atof / strtol` | `s.parse::<i32>()` / `s.parse::<f64>()` | Returns `Result` on failure; no silent truncation |
| `qsort` | `slice.sort()` / `slice.sort_by(|a, b| a.field.cmp(&b.field))` | Type-safe; no void* comparator needed |
| `bsearch` | `slice.binary_search(&key)` | Returns `Result<usize, usize>` |
| `rand()` | `rand::random::<T>()` or `rand::thread_rng()` | From the `rand` crate |
| `srand(seed)` | `let mut rng = StdRng::seed_from_u64(seed);` | Seeded RNG is explicit and separate |
| `calloc(n, sz)` | `vec![0u8; n * sz]` | Zero-initialized Vec |
| `exit(code)` | `std::process::exit(code)` | Same semantics; `fn main() -> ExitCode` is an alternative |
| `atexit(func)` | `Drop` impl on a sentinel value | RAII-based cleanup is safer and deterministic |
| `setjmp / longjmp` | `Result<T, E>` propagation or `panic!` | No non-local goto needed; use `?` operator for error propagation |
| `getenv` | `std::env::var("KEY")` | Returns `Result<String, VarError>` |
| `system(cmd)` | `std::process::Command::new("cmd").status()` | Structured subprocess API |
| `gmtime / localtime` | `chrono` crate; `std::time::SystemTime` | Timezone-aware via `chrono`; UTC via `SystemTime` |

## Canonical Patterns

### Pattern 1: Owned Heap Object

**C:**
```c
// 手动分配和释放
typedef struct {
    int x;
    int y;
} Point;

Point* point_new(int x, int y) {
    Point* p = malloc(sizeof(Point));
    p->x = x;
    p->y = y;
    return p;
}

void point_free(Point* p) {
    free(p);
}
```

**Rust:**
```rust
// 所有权自动管理,无需手动释放
pub struct Point {
    pub x: i32,
    pub y: i32,
}

impl Point {
    pub fn new(x: i32, y: i32) -> Box<Self> {
        Box::new(Point { x, y })
    }
}
// Drop 自动调用,不需要 point_free 等价函数
```

### Pattern 2: Opaque Handle / Newtype

**C:**
```c
// handle.h -- 不透明指针,隐藏实现细节
typedef struct Context Context;
Context* context_create(void);
void context_destroy(Context* ctx);
int context_do_work(Context* ctx, int input);
```

**Rust:**
```rust
// 使用 newtype 包装,无需指针技巧即可隐藏内部结构
pub struct Context {
    // 内部字段默认私有
    state: InnerState,
}

impl Context {
    pub fn new() -> Self {
        Context { state: InnerState::default() }
    }

    pub fn do_work(&mut self, input: i32) -> Result<i32, Error> {
        self.state.process(input)
    }
}
```

### Pattern 3: Error Handling with errno → Result

**C:**
```c
// 返回特殊值 + 检查 errno
int read_file(const char* path, char** out_data, size_t* out_len) {
    FILE* f = fopen(path, "rb");
    if (!f) return -1;

    fseek(f, 0, SEEK_END);
    long size = ftell(f);
    rewind(f);

    char* buf = malloc(size + 1);
    if (!buf) { fclose(f); return -1; }

    size_t n = fread(buf, 1, size, f);
    fclose(f);

    if (n != (size_t)size) { free(buf); return -1; }
    buf[size] = '\0';
    *out_data = buf;
    *out_len = size;
    return 0;
}

// 调用端:
char* data;
size_t len;
if (read_file("config.txt", &data, &len) != 0) {
    fprintf(stderr, "read failed: %s\n", strerror(errno));
    return 1;
}
```

**Rust:**
```rust
// 使用 Result 和 ? 操作符,错误自动向上传播
use std::fs;
use std::io;

fn read_file(path: &str) -> io::Result<String> {
    fs::read_to_string(path)
}

// 调用端:
fn main() -> anyhow::Result<()> {
    let data = read_file("config.txt")?;
    println!("{}", data);
    Ok(())
}
```

### Pattern 4: Dynamic Array (Vec)

**C:**
```c
// 手动管理动态数组的容量和长度
typedef struct {
    int* data;
    size_t len;
    size_t cap;
} IntVec;

void vec_push(IntVec* v, int val) {
    if (v->len == v->cap) {
        v->cap = v->cap ? v->cap * 2 : 8;
        v->data = realloc(v->data, v->cap * sizeof(int));
    }
    v->data[v->len++] = val;
}
```

**Rust:**
```rust
// Vec 自动管理容量增长
let mut v: Vec<i32> = Vec::new();
v.push(42);
v.extend(&[1, 2, 3]);
// 容量在 push 时按 2x 策略自动增长,realloc 完全透明
```

### Pattern 5: Callback → Closure / Channel

**C:**
```c
// 回调函数指针,通过 void* 传递上下文
typedef void (*event_cb)(int event_id, void* user_data);

void register_handler(event_cb cb, void* user_data) {
    g_callback = cb;
    g_user_data = user_data;
}

// 使用端:
void my_handler(int event_id, void* user_data) {
    MyContext* ctx = (MyContext*)user_data;
    ctx->count++;
}
register_handler(my_handler, &ctx);
```

**Rust:**
```rust
// 闭包携带上下文,无需 void* 转换;或用 channel 解耦
use std::sync::mpsc;

// 方案 A: 闭包 (类型安全)
fn register_handler<F>(handler: F)
where
    F: Fn(i32) + Send + 'static,
{
    // 存储 handler ...
}

// 方案 B: channel (解耦生产者和消费者)
let (tx, rx) = mpsc::channel::<i32>();
std::thread::spawn(move || {
    tx.send(42).unwrap(); // 事件通知
});

for event in rx {
    println!("received event: {}", event);
}
```

### Pattern 6: Linked List / Tree Node

**C:**
```c
// 使用原始指针构建链表
typedef struct Node {
    int value;
    struct Node* next;
} Node;

void list_free(Node* head) {
    while (head) {
        Node* tmp = head;
        head = head->next;
        free(tmp);
    }
}
```

**Rust:**
```rust
// 使用 Box 表达所有权;Option 表达可空性
pub struct Node {
    pub value: i32,
    pub next: Option<Box<Node>>,
}

impl Node {
    pub fn new(value: i32) -> Self {
        Node { value, next: None }
    }

    pub fn push(&mut self, value: i32) {
        match self.next {
            Some(ref mut next) => next.push(value),
            None => self.next = Some(Box::new(Node::new(value))),
        }
    }
}
// Drop 递归自动释放整个链表;长链可用迭代循环替代

// 对于超长链表(栈溢出风险),使用显式迭代 Drop:
impl Drop for Node {
    fn drop(&mut self) {
        let mut curr = self.next.take();
        while let Some(mut node) = curr {
            curr = node.next.take();
            // node 在此作用域结束时被释放
        }
    }
}
```

## FFI & Incremental Migration

The recommended strategy is **leaf-to-root**: port the leaf modules (those with no internal dependencies) to Rust first, expose them via C-compatible FFI, and have the remaining C code call into them. Then move upward through the call graph until `main()` is Rust.

### Exposing Rust to C

```rust
// rust_lib.rs -- 通过 extern "C" 将 Rust 函数暴露给 C 代码
use std::ffi::CString;
use std::os::raw::c_char;

/// 解析配置,返回堆分配的 C 字符串(调用方负责释放)
#[no_mangle]
pub extern "C" fn parse_config(path: *const c_char) -> *mut c_char {
    // 安全的 Rust 实现
    let path = unsafe { std::ffi::CStr::from_ptr(path) }
        .to_str()
        .unwrap_or("");
    let result = rust_config_parser::parse(path).unwrap_or_default();
    CString::new(result).unwrap().into_raw()
}

/// 释放由 parse_config 返回的字符串
#[no_mangle]
pub extern "C" fn free_config_string(s: *mut c_char) {
    if !s.is_null() {
        unsafe { drop(CString::from_raw(s)); }
    }
}
```

### Calling C from Rust

```rust
// build.rs -- 链接现有的 C 代码
fn main() {
    cc::Build::new()
        .file("c_src/legacy_parser.c")
        .compile("legacy_parser");
    println!("cargo:rerun-if-changed=c_src/legacy_parser.c");
}

// 对应的 Rust 绑定:
extern "C" {
    fn legacy_parse(input: *const c_char, len: usize) -> i32;
}

pub fn safe_parse(input: &str) -> Result<i32, String> {
    let c_input = std::ffi::CString::new(input).map_err(|e| e.to_string())?;
    let result = unsafe { legacy_parse(c_input.as_ptr(), input.len()) };
    if result < 0 {
        Err(format!("legacy parse error: {}", result))
    } else {
        Ok(result)
    }
}
```

### Migration Stages

| Stage | C Remaining | Rust Introduced | Integration |
|-------|-------------|-----------------|-------------|
| 1 - Baseline | Entire codebase | None | N/A |
| 2 - Leaf port | Core logic | Utility modules (parsing, serialization) | Rust → C via `extern "C"` + `cbindgen` headers |
| 3 - Mid-level port | I/O wrappers, main loop | Business logic, data structures | Bidirectional FFI |
| 4 - Top-level port | Legacy drivers, OS-specific glue | All business logic, networking | C → Rust direction; C becomes statically linked lib |
| 5 - Complete | None (or vendored libs) | Entire application | `build.rs` for remaining C deps |

### Generating C Headers from Rust

Use `cbindgen` to auto-generate `.h` files:

```toml
# cbindgen.toml
language = "C"
include_guard = "RUST_LIB_H"
autogen_warning = "/* Auto-generated by cbindgen. DO NOT EDIT. */"

[export]
include = ["parse_config", "free_config_string"]
```

```bash
cbindgen --config cbindgen.toml --crate my_crate --output rust_lib.h
```

## Common Mistakes

### Mistake 1: Translating `malloc` to `unsafe` allocation directly

**Wrong:**
```rust
// 不要：直接用 unsafe 模拟 malloc
let ptr: *mut Point = unsafe { std::alloc::alloc(std::alloc::Layout::new::<Point>()) as *mut Point };
unsafe { std::alloc::dealloc(ptr as *mut u8, std::alloc::Layout::new::<Point>()); }
```

**Right:**
```rust
// 正确：使用 Box 或 Vec,让所有权系统管理内存
let point = Box::new(Point { x: 0, y: 0 });
// 离开作用域时自动释放,无需手动 dealloc

let points: Vec<Point> = (0..100).map(|i| Point { x: i, y: i * 2 }).collect();
// Vec 管理整个数组的内存
```

### Mistake 2: Using raw pointers where references would work

**Wrong:**
```rust
// 不要：继续使用原始指针传递数据
fn process(data: *const u8, len: usize) -> i32 {
    unsafe {
        let slice = std::slice::from_raw_parts(data, len);
        // ...
    }
    // 调用方必须确保 data 有效,容易出错
}
```

**Right:**
```rust
// 正确：使用 &[u8] 切片,借用检查器保证安全
fn process(data: &[u8]) -> i32 {
    // 无需 unsafe,编译器保证 data 有效
    data.iter().map(|&b| b as i32).sum()
}
```

### Mistake 3: Translating `goto cleanup` as explicit error checks

**Wrong:**
```rust
// 不要：逐层检查错误并手动回滚
fn complex_operation() -> Result<(), Error> {
    let res1 = step1();
    if res1.is_err() { return Err(res1.unwrap_err()); }

    let res2 = step2();
    if res2.is_err() {
        step1_rollback();
        return Err(res2.unwrap_err());
    }
    Ok(())
}
```

**Right:**
```rust
// 正确：使用 ? 操作符和 Drop/Raii 自动回滚
struct ResourceGuard { /* ... */ }
impl Drop for ResourceGuard {
    fn drop(&mut self) {
        // 自动回滚逻辑
        self.rollback();
    }
}

fn complex_operation() -> Result<(), Error> {
    let guard = ResourceGuard::acquire()?;  // 失败时自动返回 Err
    step1()?;
    step2()?;  // 失败时 guard 的 Drop 自动执行回滚
    std::mem::forget(guard);  // 成功时取消回滚
    Ok(())
}
```

### Mistake 4: Translating `#define` constants as `static mut`

**Wrong:**
```rust
// 不要：用 static mut 替代 #define 全局可变状态
static mut LOG_LEVEL: i32 = 1;
```

**Right:**
```rust
// 正确：使用 const,或者需要可变时使用 Atomic + OnceCell
const LOG_LEVEL: i32 = 1;  // 编译期常量

// 需要运行时可变时:
use std::sync::atomic::{AtomicI32, Ordering};
static LOG_LEVEL_RT: AtomicI32 = AtomicI32::new(1);
LOG_LEVEL_RT.store(2, Ordering::Relaxed);
```

### Mistake 5: Using `unsafe` to bypass the borrow checker for concurrency

**Wrong:**
```rust
// 不要：用 unsafe 和原始指针在线程间共享可变数据
let mut data = vec![1, 2, 3];
let ptr: *mut Vec<i32> = &mut data;
std::thread::spawn(move || {
    unsafe { (*ptr).push(4); }  // 数据竞争!
});
```

**Right:**
```rust
// 正确：使用 Arc<Mutex<T>> 安全地在线程间共享数据
let data = std::sync::Arc::new(std::sync::Mutex::new(vec![1, 2, 3]));
let data_clone = data.clone();
std::thread::spawn(move || {
    data_clone.lock().unwrap().push(4);
});
```

### Mistake 6: Keeping `void*` for generic code

**Wrong:**
```rust
// 不要：继续使用 void* / *mut c_void 假装泛型
fn generic_push(vec: *mut c_void, value: *const c_void, elem_size: usize) {
    unsafe { /* 手动 memcpy */ }
}
```

**Right:**
```rust
// 正确：使用真正的泛型,编译器做代码生成
fn generic_push<T>(vec: &mut Vec<T>, value: T) {
    vec.push(value);
}
// 或者用 trait 约束:
fn generic_push_trait<T: Clone>(vec: &mut Vec<T>, value: &T) {
    vec.push(value.clone());
}
```

## Reference Implementations

| Project | Description | Relevant Patterns |
|---------|-------------|-------------------|
| [ripgrep](https://github.com/BurntSushi/ripgrep) | Replaced `grep` / `ack` (C) | File I/O, regex, parallelism with `ignore` crate |
| [fd](https://github.com/sharkdp/fd) | Replaced `find` (C) | Walkdir, thread pool, CLI argument parsing |
| [bat](https://github.com/sharkdp/bat) | Replaced `cat` (C) | Syntax highlighting, pager integration |
| [bottom](https://github.com/ClementTsang/bottom) | Replaced `top` / `htop` (C) | TUI rendering, process stats parsing |
| [zoxide](https://github.com/ajeetdsouza/zoxide) | Replaced `autojump` / `z` | Path matching, database, shell integration |
| [lsd](https://github.com/lsd-rs/lsd) | Replaced `ls` (C) | File metadata, Nerd Font icons, colorized output |
| [starship](https://github.com/starship/starship) | Shell prompt (replaced C/bash scripts) | Cross-shell integration, module system |
| [exa](https://github.com/ogham/exa) | Replaced `ls` (C) | Grid layout rendering, git integration |
| [coreutils](https://github.com/uutils/coreutils) | Rewrite of GNU coreutils in Rust | Direct C-to-Rust function-level porting |
| [fish-shell](https://github.com/fish-shell/fish-shell) | Migrating from C++ to Rust | Incremental port strategy, FFI between C++ and Rust |

## Cross-References

- `cpp-to-rust`: For C++ codebases with templates, virtual dispatch, and RAII patterns
- `zig-to-rust`: For comptime-heavy code and custom allocator patterns
- For FFI details: see `std::ffi` module documentation and the `bindgen` / `cbindgen` crate docs
- For build integration: see the `cc` crate for compiling C sources from `build.rs`
