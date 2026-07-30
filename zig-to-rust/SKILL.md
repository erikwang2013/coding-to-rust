---
name: zig-to-rust
description: Use when migrating Zig codebases to Rust — covers comptime to proc macros/const generics, allocator to ownership, error sets to Result, build.zig to Cargo, and incremental replacement. Includes canonical code patterns, common mistakes, and reference implementations.
updated: 2026-07-30
---

## Architecture Mapping

Zig and Rust share a core philosophy: no hidden control flow, no implicit allocations, and strong compile-time guarantees. Zig achieves this through `comptime`, explicit allocator passing, and error sets; Rust achieves the same goals through ownership, generics + trait bounds, and `Result<T, E>`. Both languages reject exceptions, both offer fine-grained memory control, and both target the same systems-programming niche. The translation is mostly structural: Zig's flat-source-file-with-imports becomes Rust's module tree, Zig's `std.mem.Allocator` interface becomes Rust's implicit ownership system (with explicit `Box`/`Vec`/`Arc` when heap allocation is needed), and Zig's `comptime` code generation becomes Rust's declarative macros, proc macros, const generics, and `build.rs`.

A Zig project:

```
src/
  main.zig
  parser.zig
  network.zig
build.zig
build.zig.zon
```

becomes:

```
src/
  main.rs
  parser.rs
  network.rs
Cargo.toml
build.rs
```

Zig's emphasis on "no hidden allocations" maps beautifully to Rust: in Zig you see every allocator passed explicitly; in Rust you see every heap allocation via `Box`, `Vec`, `String`, `Arc` -- no invisible allocations in either language.

## Type System Mapping

| Zig Type | Rust Type | Notes |
|----------|-----------|-------|
| `u8` / `i8` | `u8` / `i8` | Direct mapping |
| `u16` / `i16` | `u16` / `i16` | Direct mapping |
| `u32` / `i32` | `u32` / `i32` | Direct mapping |
| `u64` / `i64` | `u64` / `i64` | Direct mapping |
| `usize` / `isize` | `usize` / `isize` | Direct mapping |
| `f32` / `f64` | `f32` / `f64` | Direct mapping |
| `bool` | `bool` | Direct mapping |
| `noreturn` | `!` (never type) | Diverging function return type |
| `void` | `()` (unit type) | `()` is a zero-size value |
| `anytype` | Generics `T` or `impl Trait` | Zig's compile-time duck typing becomes trait-bounded generics |
| `?T` (optional) | `Option<T>` | `null` becomes `None`; unwrap with `.?` becomes `.unwrap()` |
| `!T` (error union) | `Result<T, E>` | Error set becomes an enum; `try` becomes `?` |
| `[*]T` (many-pointer) | `*const T` or `*mut T` | Raw pointer; use `NonNull<T>` for guaranteed non-null |
| `[*:0]T` (sentinel-terminated) | `*const c_char` / `CStr` / `CString` | Sentinel arrays are mainly for C FFI |
| `[N]T` (array) | `[T; N]` | Stack-allocated, size known at compile time |
| `[]T` (slice) | `&[T]` (borrowed) / `Vec<T>` (owned) | Slice is a fat pointer (ptr + len), identical runtime representation |
| `[]const T` | `&[T]` | Immutable borrow of contiguous elements |
| `[:0]T` | `&CStr` | Borrowed sentinel-terminated slice |
| `struct { ... }` | `struct { ... }` | Fields are private by default in Zig; use `pub` in Rust too |
| `packed struct` | `#[repr(C)]` / `#[repr(packed)]` | Use `#[repr(C)]` for C-compatible layout, `#[repr(packed)]` for byte-level packing |
| `extern struct` | `#[repr(C)] struct` | Same ABI-stable layout for FFI |
| `union` | `union` (unsafe) or `enum` | Prefer tagged enum; use raw union for FFI only |
| `tagged union` / `union(enum)` | `enum` with payload | Rust enums are always tagged; this is idiomatic Rust |
| `enum { ... }` | `enum { ... }` | Rust enums can carry data, unlike Zig |
| `enum(u8) { ... }` | `#[repr(u8)] enum { ... }` | Specify discriminant type |
| `*T` (single-item pointer) | `&T` / `&mut T` / `Box<T>` | Convert to reference; use `NonNull<T>` for nullable FFI pointers |
| `**T` (double pointer) | `&mut &T` or `Option<&mut T>` | Out-parameter pattern; prefer return values |
| `fn(T) void` | `fn(&T)` | Function pointer |
| `@TypeOf(x)` | Type inference in `let` / compile-time `type_name::<T>()` | Rarely needed; turbofish `::<T>` for explicit type params |
| `type` | `T: 'static` or trait objects | Type as value; Rust separates types (compile-time) and values |
| `@Vector(N, T)` | Portable SIMD via `std::simd` (nightly) or `packed_simd` crate | SIMD vector types |
| `comptime_int` / `comptime_float` | Const generics + `typenum` crate for type-level ints | No directly equivalent arbitrary-precision comptime number type |
| `@Type(.{ .Int = .{...} })` | `typenum` crate | Type-level integer arithmetic |

## Memory & Ownership Model

Zig makes allocation explicit by requiring an `Allocator` parameter for all heap operations. Rust makes ownership explicit through the type system. This is a philosophical alignment: both languages reject hidden allocations. The key translation is mapping Zig's allocator parameter to Rust's ownership rules.

| Zig Pattern | Rust Pattern | Semantic Notes |
|-------------|-------------|----------------|
| `allocator.alloc(T, n)` | `vec![T::default(); n]` or `(0..n).map(...).collect::<Vec<T>>()` | Owned buffer; Rust Vec manages its own allocator |
| `allocator.free(slice)` | Automatic at end of scope | Drop is implicit; no manual free call |
| `allocator.create(T)` | `Box::new(T::new())` | Single heap-allocated value |
| `allocator.destroy(ptr)` | Automatic (Box::drop) | Drop on box deallocation |
| `allocator.dupe(u8, slice)` | `slice.to_vec()` or `slice.to_owned()` | Copy a slice to owned allocation |
| `allocator.realloc(slice, n)` | `vec.resize(n, default)` | Vec handles resize internally |
| `std.heap.page_allocator` | `#[global_allocator]` or default system allocator | Default global allocator |
| `std.heap.GeneralPurposeAllocator` | `mimalloc` / `jemallocator` crates | Custom global allocator |
| `std.heap.ArenaAllocator` | `bumpalo` crate / `typed_arena` crate | Arena/bump allocation for short-lived objects |
| `std.heap.FixedBufferAllocator` | `arrayvec::ArrayVec` / `smallvec::SmallVec` | Stack-allocated buffer fallback |
| Defer-based cleanup `defer allocator.free(buf)` | `Drop` trait impl | RAII: `Drop::drop` runs at scope exit |
| `errdefer` (defer on error) | `Drop` + `Result` -- Drop always runs; conditionals inside drop | `Drop` runs regardless; check state in drop body if cleanup varies |
| Passing allocator through call stack | Ownership system; no allocator parameter | Heap types (`Box`, `Vec`, `String`) carry their allocator implicitly |
| `std.mem.Allocator` interface | `std::alloc::Allocator` trait (nightly) | Can implement custom allocators; rare in practice |
| Stack allocation `var buf: [1024]u8 = undefined` | `let mut buf = [0u8; 1024];` | Fixed stack array; must be initialized in Rust |
| Undefined memory `= undefined` | `MaybeUninit<T>` | Only for FFI or performance-critical init-elision |

## Concurrency / Async Translation

Zig's concurrency model is still evolving (no stable async/await). Rust has a mature async ecosystem built on `tokio`, plus `std::thread` and `rayon` for CPU parallelism.

| Zig Pattern | Rust Pattern | Notes |
|-------------|-------------|-------|
| `std.Thread.spawn` | `std::thread::spawn` | Same: closure as entry point, `JoinHandle` to wait |
| `std.Thread.Mutex` | `std::sync::Mutex<T>` | Rust Mutex guards data |
| `std.Thread.Condition` | `std::sync::Condvar` | Condition variable |
| `std.Thread.ResetEvent` / `AutoResetEvent` (Windows) | `tokio::sync::Notify` or `std::sync::mpsc` | Notification primitive |
| `std.atomic.*` | `std::sync::atomic::*` | Same CAS, load/store; identical `Ordering` enum |
| `std.Thread.Semaphore` | `tokio::sync::Semaphore` | Async semaphore |
| `std.ChildProcess` | `std::process::Command` | Spawn and manage subprocess |
| `std.event.Loop` | `tokio` runtime | Event loop |
| `async fn` (Zig, unstable) | `async fn` (stable, tokio) | Stackless coroutine; `.await` is identical syntax |
| `suspend` / `resume` (Zig) | `Future::poll` (low-level) -- prefer `.await` | Manual poll is rare; use async/await |
| `@fieldParentPtr` for async frames | N/A -- compiler handles async frame layout | Rust compiler manages the state machine |
| `std.Thread.Pool` | `rayon::ThreadPool` | Work-stealing thread pool for CPU-bound work |
| `std.Thread.spawn` + manual join | `rayon::scope` or `tokio::task::JoinSet` | Structured concurrency |
| Shared memory between threads | `Arc<Mutex<T>>` or `Arc<Atomic*>` | Arc for shared ownership; Mutex/RwLock for mutable access |

## Build System & Dependencies

| Zig Tool / Concept | Rust Equivalent | Notes |
|--------------------|-----------------|-------|
| `build.zig` | `Cargo.toml` | Declarative build manifest |
| `build.zig.zon` | `Cargo.toml` (`[dependencies]` section) | Package manifest; Cargo.toml combines both roles |
| `const exe = b.addExecutable(...)` | `[[bin]]` section or `src/main.rs` | Binary target |
| `const lib = b.addStaticLibrary(...)` | `[lib]` section or `src/lib.rs` | Library target |
| `exe.addModule("foo", foo_module)` | `foo = { path = "../foo" }` in `[dependencies]` | Local path dependency |
| `exe.linkSystemLibrary("ssl")` | `build.rs` with `cargo:rustc-link-lib=ssl` | Link system native libraries |
| `exe.addIncludePath("include/")` | `cc::Build` in `build.rs` | Compile and link C code |
| `exe.addCSourceFile("vendor/foo.c")` | `cc::Build::new().file("vendor/foo.c").compile("foo")` in `build.rs` | Mixed-language build |
| `b.option(T, "name", "description")` | `cfg(feature = "name")` + `[features]` in `Cargo.toml` | Feature flags |
| `target.cpu.arch` / `target.os.tag` | `#[cfg(target_arch = "x86_64")]` / `#[cfg(target_os = "linux")]` | Conditional compilation |
| `@import("foo")` | `use foo;` (module from crate or path) | Module import |
| `@import("std")` | `use std::...;` | Standard library import |
| `pub usingnamespace` | `pub use module::*;` (glob re-export) | Re-export all public items |
| `@import("root")` | `crate::` (crate root) | Reference the crate root |
| `comptime` build steps (e.g., codegen) | `build.rs` or proc macros | Build-time code execution |
| `exe.setBuildMode(.ReleaseSafe)` | `--release` flag; `[profile.release]` in `Cargo.toml` | Optimized build |
| `exe.setBuildMode(.ReleaseFast)` | `[profile.release]` with `opt-level = 3`, `lto = true` | Maximum performance |
| `exe.setBuildMode(.ReleaseSmall)` | `[profile.release]` with `opt-level = "s"`, `lto = true` | Size-optimized |
| `exe.setTarget(target)` (cross-compilation) | `--target x86_64-unknown-linux-musl` + `.cargo/config.toml` | Cross-compilation via target triple |

## Standard Library & Ecosystem Mapping

| Zig Standard Library | Rust Equivalent | Notes |
|---------------------|-----------------|-------|
| `std.debug.print` | `println!` / `eprintln!` | Format string uses `{}` not `{s}`; compile-time type checked |
| `std.fmt.allocPrint` | `format!` macro | Returns `String`; allocates automatically |
| `std.fmt.parseInt` | `s.parse::<T>()` | Returns `Result`; `from_str_radix` for custom bases |
| `std.fmt.parseFloat` | `s.parse::<f64>()` | Same semantics |
| `std.fmt.bufPrint` | `write!(buf, "...")` | Write formatted to buffer |
| `std.fs.cwd()` | `std::env::current_dir()` | Current working directory |
| `std.fs.openFileAbsolute` | `std::fs::File::open` | Open file by absolute path |
| `std.fs.Dir.openFile` | `std::fs::File::open` (relative to CWD) or `Path::join` | File relative to directory |
| `std.fs.Dir.createFile` | `std::fs::File::create` | Create or truncate file |
| `std.fs.Dir.readFileAlloc` | `std::fs::read_to_string` / `std::fs::read` | Read entire file to String/Vec |
| `std.fs.Dir.writeFile` | `std::fs::write` | Write entire file at once |
| `std.fs.Dir.deleteFile` | `std::fs::remove_file` | Delete a file |
| `std.fs.Dir.makeDir` | `std::fs::create_dir` / `create_dir_all` | Create directories |
| `std.fs.path.join` | `Path::join` / `PathBuf::push` | Path concatenation |
| `std.fs.path.dirname` | `Path::parent()` | Get parent directory |
| `std.fs.path.basename` | `Path::file_name()` | Get file name component |
| `std.fs.path.extension` | `Path::extension()` | Get file extension |
| `std.fs.path.realpathAlloc` | `std::fs::canonicalize` | Resolve symlinks, return absolute path |
| `std.mem.len` / `slice.len` | `slice.len()` | O(1) length |
| `std.mem.eql` | `a == b` | Equality for slices |
| `std.mem.indexOf` | `haystack.find(needle)` | Returns `Option<usize>` |
| `std.mem.containsAtLeast` | `slice.windows(N).any(|w| w == ...)` | Window-based match |
| `std.mem.concat` | `[a, b].concat()` or `vec![a, b].concat()` | Concatenate slices |
| `std.mem.join` | `items.join(separator)` from `itertools` or custom | Join iterator with separator |
| `std.mem.replace` | `s.replace(from, to)` | String replacement |
| `std.mem.startsWith` / `endsWith` | `s.starts_with(prefix)` / `s.ends_with(suffix)` | Prefix/suffix check |
| `std.mem.trim` | `s.trim()` / `s.trim_start()` / `s.trim_end()` | Whitespace trimming |
| `std.mem.split` / `tokenize` | `s.split(delim)` / `s.split_whitespace()` | String splitting; Rust returns iterator |
| `std.mem.zeroes` | `[0u8; N]` or `vec.resize(N, 0)` | Zero-initialization |
| `std.mem.copy` / `copyForwards` / `copyBackwards` | `dst[..n].copy_from_slice(&src[..n])` / `dst.copy_within(...)` | Memory copy with overlap handling |
| `std.mem.set` | `slice.fill(value)` | Fill slice with value |
| `std.sort.sort` | `slice.sort()` / `sort_unstable()` | In-place sort; Rust sort is stable by default |
| `std.sort.binarySearch` | `slice.binary_search(&x)` | Returns `Result<usize, usize>` |
| `std.rand.Random` | `rand::Rng` trait from `rand` crate | Random number generation |
| `std.rand.DefaultPrng` | `rand::rngs::StdRng` | Default PRNG (ChaCha-based) |
| `std.crypto.*` | `ring` crate / `RustCrypto` traits | Cryptography primitives |
| `std.json.parse` | `serde_json::from_str::<T>(s)` | JSON deserialization |
| `std.json.stringify` | `serde_json::to_string(&value)` | JSON serialization |
| `std.process.args` | `std::env::args()` | CLI arguments iterator |
| `std.process.getEnvVarOwned` | `std::env::var("KEY")` | Returns `Result<String, VarError>` |
| `std.process.Child` | `std::process::Child` | Subprocess handle |
| `std.process.Child.exec` | `std::process::Command::new("cmd").spawn()` | Spawn subprocess |
| `std.os.argv` | `std::env::args_os()` | Raw OS args |
| `std.time.milliTimestamp` | `std::time::Instant::now()` | Monotonic clock |
| `std.time.sleep` | `std::thread::sleep(dur)` | Block for duration |
| `std.time.timestamp` | `std::time::SystemTime::now()` | Wall clock |
| `std.net.Server` (TCP listen) | `std::net::TcpListener` or `tokio::net::TcpListener` | TCP server socket |
| `std.net.Stream` (TCP connect) | `std::net::TcpStream` or `tokio::net::TcpStream` | TCP client stream |
| `std.net.Address` | `std::net::SocketAddr` | IP + port |
| `std.net.parseIp4` | `"1.2.3.4".parse::<Ipv4Addr>()` | Parse IPv4 |
| `std.ArrayList(T)` | `Vec<T>` | Growable array; Vec has more methods |
| `std.ArrayListUnmanaged(T)` | `Vec<T>` (always uses global allocator) | No allocator parameter needed |
| `std.BufMap` | `std::collections::HashMap<String, String>` | String key-value map |
| `std.StringHashMap(T)` | `std::collections::HashMap<String, T>` | String-keyed hash map |
| `std.AutoHashMap(K, V)` | `std::collections::HashMap<K, V>` | Auto-context hash map |
| `std.HashMap(K, V, Context)` | `HashMap<K, V, BuildHasher>` | Custom-hashed map |
| `std.BoundedArray(T, N)` | `arrayvec::ArrayVec<T, N>` | Stack-bounded array |
| `std.SinglyLinkedList(T)` | `Option<Box<Node<T>>>` or `std::collections::LinkedList<T>` | Linked list |
| `std.DoublyLinkedList(T)` | `std::collections::LinkedList<T>` | Doubly-linked list |
| `std.PriorityQueue(T)` | `std::collections::BinaryHeap<T>` | Max-heap; use `Reverse` for min-heap |
| `std.ComptimeStringMap` | `phf` crate (perfect hash) or `match` on &str | Compile-time string→value map |
| `std.StaticStringMap` | `phf::Map` | Runtime-constructed, compile-time-optimized string map |
| `std.meta.trait.isInteger` | Not directly; rely on trait bounds | Type introspection is compile-time via traits |
| `std.meta.FieldEnum` | Derive macro + manually list fields | Struct field reflection via `strum` crate |
| `@compileError("msg")` | `compile_error!("msg")` | Compile-time error |
| `@panic("msg")` | `panic!("msg")` or `unreachable!("msg")` | Runtime assertion failure |

## Canonical Patterns

### Pattern 1: Comptime → Const Generics / Proc Macros

**Zig:**
```zig
// comptime 泛型: 编译期根据类型生成代码
fn Matrix(comptime T: type, comptime rows: comptime_int, comptime cols: comptime_int) type {
    return struct {
        data: [rows * cols]T,

        pub fn at(self: *const @This(), r: usize, c: usize) T {
            return self.data[r * cols + c];
        }
    };
}

const Mat3x4 = Matrix(f32, 3, 4);
```

**Rust:**
```rust
// 使用 const generics 实现编译期维度参数化
#[derive(Debug, Clone)]
pub struct Matrix<T, const ROWS: usize, const COLS: usize> {
    data: [T; ROWS * COLS],
}

impl<T: Copy + Default, const ROWS: usize, const COLS: usize> Matrix<T, ROWS, COLS> {
    pub fn new() -> Self {
        Matrix { data: [T::default(); ROWS * COLS] }
    }

    pub fn at(&self, r: usize, c: usize) -> T {
        self.data[r * COLS + c]
    }
}

type Mat3x4 = Matrix<f32, 3, 4>;
```

### Pattern 2: Error Sets → Enum-Based Error Types

**Zig:**
```zig
// Zig 错误集: 隐式枚举,自动合并
const ParserError = error{
    UnexpectedEof,
    InvalidToken,
    StackOverflow,
};

fn parse(input: []const u8) ParserError!Ast {
    if (input.len == 0) return error.UnexpectedEof;
    // ...
    return Ast{};
}

// 错误传播:
const ast = try parse(data);
```

**Rust:**
```rust
// 使用 thiserror 定义结构化错误枚举
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ParserError {
    #[error("unexpected end of input")]
    UnexpectedEof,

    #[error("invalid token at position {pos}: '{found}'")]
    InvalidToken { pos: usize, found: char },

    #[error("stack overflow at depth {depth}")]
    StackOverflow { depth: usize },
}

fn parse(input: &str) -> Result<Ast, ParserError> {
    if input.is_empty() {
        return Err(ParserError::UnexpectedEof);
    }
    // ...
    Ok(Ast {})
}

// 错误传播: ? 操作符等价于 try
let ast = parse(data)?;
```

### Pattern 3: Defer → Drop

**Zig:**
```zig
// defer 保证退出作用域时执行清理代码
fn processFile(path: []const u8) !void {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();

    var buf: [4096]u8 = undefined;
    const n = try file.read(&buf);

    var arena = std.heap.ArenaAllocator.init(allocator);
    defer arena.deinit();

    // 使用 file 和 arena ...
    // 退出时 file.close() 和 arena.deinit() 自动执行
}
```

**Rust:**
```rust
// Drop trait 实现 RAII: 退出作用域自动清理
use std::fs::File;
use std::io::Read;

fn process_file(path: &str) -> std::io::Result<()> {
    let mut file = File::open(path)?;       // Drop 时自动关闭
    let mut buf = [0u8; 4096];
    let n = file.read(&mut buf)?;

    let arena = bumpalo::Bump::new();       // Drop 时自动释放全部内存

    // 使用 file 和 arena ...
    // 作用域退出时,arena 和 file 的 Drop 自动执行
    Ok(())
}
```

### Pattern 4: Optional Unwrapping

**Zig:**
```zig
// 可选类型和错误联合的处理
fn getConfig(key: []const u8) ?[]const u8 {
    // 返回 null 表示不存在
}

fn lookupOrDefault(key: []const u8, default: []const u8) []const u8 {
    return getConfig(key) orelse default;
}

fn requireConfig(key: []const u8) ![]const u8 {
    return getConfig(key) orelse error.ConfigMissing;
}
```

**Rust:**
```rust
// Option 和 Result 组合子链式处理
fn get_config(key: &str) -> Option<&str> {
    // 返回 None 表示不存在
}

fn lookup_or_default(key: &str, default: &str) -> &str {
    get_config(key).unwrap_or(default)
}

fn require_config(key: &str) -> Result<&str, ConfigError> {
    get_config(key).ok_or(ConfigError::Missing { key: key.to_string() })
}
```

### Pattern 5: Allocator Interface → Ownership

**Zig:**
```zig
// Zig: 显式传递分配器
fn buildTree(allocator: std.mem.Allocator, depth: u32) !*Node {
    const node = try allocator.create(Node);
    node.value = depth;
    if (depth > 0) {
        node.left = try buildTree(allocator, depth - 1);
        node.right = try buildTree(allocator, depth - 1);
    }
    return node;
}

fn freeTree(allocator: std.mem.Allocator, node: *Node) void {
    if (node.left) |left| freeTree(allocator, left);
    if (node.right) |right| freeTree(allocator, right);
    allocator.destroy(node);
}
```

**Rust:**
```rust
// Rust: 所有权系统自动管理分配和释放
fn build_tree(depth: u32) -> Box<Node> {
    let mut node = Box::new(Node { value: depth, left: None, right: None });
    if depth > 0 {
        node.left = Some(build_tree(depth - 1));
        node.right = Some(build_tree(depth - 1));
    }
    node
}
// 整个树在 Box<Node> 离开作用域时自动递归释放
// 如需显式迭代释放(防止栈溢出):
impl Drop for Node {
    fn drop(&mut self) {
        let mut stack = vec![self.left.take(), self.right.take()];
        while let Some(Some(mut node)) = stack.pop() {
            stack.push(node.left.take());
            stack.push(node.right.take());
            // node 在此处被释放
        }
    }
}
```

### Pattern 6: Switch / Pattern Matching

**Zig:**
```zig
// Zig switch: 穷举匹配,编译期检查完整性
const Event = union(enum) {
    click: struct { x: i32, y: i32 },
    keypress: struct { code: u8, modifiers: u8 },
    quit,
};

fn handle(ev: Event) void {
    switch (ev) {
        .click => |c| std.debug.print("click at {}, {}\n", .{c.x, c.y}),
        .keypress => |k| std.debug.print("key {} mod {}\n", .{k.code, k.modifiers}),
        .quit => std.debug.print("quitting\n", .{}),
    }
}
```

**Rust:**
```rust
// Rust match: 穷举模式匹配,编译器保证完整性
pub enum Event {
    Click { x: i32, y: i32 },
    KeyPress { code: u8, modifiers: u8 },
    Quit,
}

fn handle(ev: Event) {
    match ev {
        Event::Click { x, y } => println!("click at {x}, {y}"),
        Event::KeyPress { code, modifiers } => println!("key {code} mod {modifiers}"),
        Event::Quit => println!("quitting"),
    }
    // 如果漏掉某个变体,编译器报错 -- 与 Zig 一样穷举检查
}
```

### Pattern 7: Packed Struct / C ABI Layout

**Zig:**
```zig
// packed struct: 位精确布局,适合协议头和硬件寄存器
const TCPHeader = packed struct {
    src_port: u16,
    dst_port: u16,
    seq_num: u32,
    ack_num: u32,
    data_offset: u4,
    reserved: u3,
    flags: u9,
    window: u16,
    checksum: u16,
    urgent_ptr: u16,
};
```

**Rust:**
```rust
// 使用 bitflags 或手工位操作; #[repr(C)] 保证 C 兼容布局
// 方案 A: 使用 deku crate 做位级序列化
use deku::prelude::*;

#[derive(Debug, PartialEq, DekuRead, DekuWrite)]
#[deku(endian = "big")]
pub struct TcpHeader {
    pub src_port: u16,
    pub dst_port: u16,
    pub seq_num: u32,
    pub ack_num: u32,
    #[deku(bits = "4")]  pub data_offset: u8,
    #[deku(bits = "3")]  pub reserved: u8,
    #[deku(bits = "9")]  pub flags: u16,
    pub window: u16,
    pub checksum: u16,
    pub urgent_ptr: u16,
}

// 方案 B: 使用 bitfield crate 或手工位运算
// 对于简单场景,直接读字节然后用 >> 和 & 提取位域
```

## FFI & Incremental Migration

### Strategy: Leaf-to-Root Porting

| Stage | Zig | Rust | Bridge |
|-------|-----|------|--------|
| 1 - Baseline | Full application | None | N/A |
| 2 - Library extraction | Core algorithms | Utility crates called via C ABI | `extern "C"` from Zig to Rust |
| 3 - Mid port | I/O, allocator plumbing | Business logic | Bidirectional C ABI |
| 4 - Top port | Main entry point, CLI arg parsing | All modules | Zig `main.zig` calls Rust; eventually Rust becomes main |
| 5 - Complete | None | Entire application | Vendored C deps via `build.rs` |

### Exposing Rust to Zig (via C ABI)

```rust
// rust_engine.rs -- Rust 侧导出 C ABI
use std::ffi::{c_char, CStr, CString};

#[derive(Default)]
pub struct Engine {
    state: i32,
}

#[no_mangle]
pub extern "C" fn engine_create() -> *mut Engine {
    Box::into_raw(Box::new(Engine::default()))
}

#[no_mangle]
pub extern "C" fn engine_process(
    engine: *mut Engine,
    input: *const c_char,
) -> *mut c_char {
    let eng = unsafe { &mut *engine };
    let input = unsafe { CStr::from_ptr(input) }
        .to_str().unwrap_or("");
    let result = format!("processed: {input} (state={})", eng.state);
    eng.state += 1;
    CString::new(result).unwrap().into_raw()
}

#[no_mangle]
pub extern "C" fn engine_destroy(engine: *mut Engine) {
    if !engine.is_null() {
        unsafe { drop(Box::from_raw(engine)); }
    }
}

#[no_mangle]
pub extern "C" fn engine_free_string(s: *mut c_char) {
    if !s.is_null() {
        unsafe { drop(CString::from_raw(s)); }
    }
}
```

```zig
// zig端调用Rust导出的C ABI函数
const c = @cImport({
    @cInclude("rust_engine.h");
});

pub fn main() !void {
    const engine = c.engine_create();
    defer c.engine_destroy(engine);

    const input = "hello from zig";
    const output = c.engine_process(engine, input);
    defer c.engine_free_string(output);
    // 使用 output ...
}
```

### Calling Zig from Rust

```rust
// build.rs
fn main() {
    // 前提: 需要 zig build-exe 或 zig build-lib 生成 .a / .so
    println!("cargo:rustc-link-lib=zig_lib");
    println!("cargo:rustc-link-search=native=/path/to/zig/build");
}

// ffi.rs -- Rust 侧绑定
extern "C" {
    fn zig_parse_protocol(data: *const u8, len: usize) -> i32;
    fn zig_serialize(data: *const u8, len: usize, out: *mut *mut u8, out_len: *mut usize) -> i32;
}
```

## Common Mistakes

### Mistake 1: Keeping `defer` Patterns via Manual Cleanup

**Wrong:**
```rust
// 不要: 在 Rust 中手动调用清理函数模拟 Zig 的 defer
fn process() -> Result<(), Error> {
    let file = File::open("data.bin")?;
    // ... 使用 file ...
    // 错误: 每个返回点都需要记得清理
    // 容易在早退时遗漏
}
```

**Right:**
```rust
// 正确: 依赖 Drop 自动清理; 用 scope 精确控制生命周期
fn process() -> Result<(), Error> {
    let file = File::open("data.bin")?;
    // Drop 自动关闭文件句柄,无论函数如何退出
    // ...

    // 需要提前释放: 用 scope
    {
        let temp = File::create("temp.bin")?;
        // temp 在此作用域结束时被关闭
    }
    // temp 已经被释放
    Ok(())
}
```

### Mistake 2: Translating `anytype` as `Box<dyn Any>`

**Wrong:**
```rust
// 不要: 每个 anytype 都翻译成类型擦除
fn process_any(input: Box<dyn std::any::Any>) {
    // 类型信息丢失,需要到处 downcast_ref
}
```

**Right:**
```rust
// 正确: 使用泛型 + trait bound 实现编译期多态
fn process<T: Processable>(input: T) {
    input.process();
}

trait Processable {
    fn process(&self);
}

// 或者: 如果真的是开放集合,用 enum 而非 Any
enum Input {
    Text(String),
    Binary(Vec<u8>),
    Json(serde_json::Value),
}
```

### Mistake 3: Direct `@intFromPtr` → `as` Pointer Casts

**Wrong:**
```rust
// 不要: 将 Zig 的 @intFromPtr/@ptrFromInt 直译成 as 转换
let addr = 0x1000usize;
let ptr = addr as *const u8;
let bytes = unsafe { std::slice::from_raw_parts(ptr, 16) };
// 只在非常特定的场景(内存映射IO)中合法
```

**Right:**
```rust
// 正确: 内存映射 IO 使用专门的 crate
// 方案 A: 如果是 mmap,使用 memmap2 crate
use memmap2::MmapOptions;
let file = File::open("data.bin")?;
let mmap = unsafe { MmapOptions::new().map(&file)? };
let bytes: &[u8] = &mmap;

// 方案 B: 如果真的是硬件地址(嵌入式/Legacy),隔离在 unsafe 模块
#[cfg(target_os = "none")]
mod hardware {
    const PERIPHERAL_BASE: usize = 0x4000_0000;
    pub fn read_register(offset: usize) -> u32 {
        unsafe {
            let ptr = (PERIPHERAL_BASE + offset) as *const u32;
            ptr.read_volatile()
        }
    }
}
```

### Mistake 4: Using `unsafe` Everywhere as Zig-idiomatic "I Know What I'm Doing"

**Wrong:**
```rust
// 不要: Zig 程序员习惯用 unsafe 标记所有"我知道这安全"的代码
unsafe fn fast_copy(src: &[u8], dst: &mut [u8]) {
    let len = src.len();
    std::ptr::copy_nonoverlapping(src.as_ptr(), dst.as_mut_ptr(), len);
}
// 到处 unsafe,失去了 Rust 的安全检查价值
```

**Right:**
```rust
// 正确: 将 unsafe 封装在最小的安全抽象内
// copy_from_slice 已经是安全的,编译器会优化为 memcpy
fn fast_copy(src: &[u8], dst: &mut [u8]) {
    dst[..src.len()].copy_from_slice(src);
}

// 如果确实需要 unsafe(自定义SIMD等),隔离在安全接口内:
mod simd_impl {
    #[cfg(target_arch = "x86_64")]
    pub fn copy_aligned(src: &[u8], dst: &mut [u8]) {
        // SAFETY: 调用方保证对齐
        unsafe { /* SSE/AVX 实现 */ }
    }

    #[cfg(not(target_arch = "x86_64"))]
    pub fn copy_aligned(src: &[u8], dst: &mut [u8]) {
        dst[..src.len()].copy_from_slice(src);
    }
}
```

### Mistake 5: Build-System Over-Engineering (Replicating `build.zig` Complexity in `build.rs`)

**Wrong:**
```rust
// 不要: 将 build.zig 的灵活性完整复制到 build.rs
// build.rs: 实现自定义交叉编译逻辑、目标检测、代码生成
fn main() {
    let target = std::env::var("TARGET").unwrap();
    if target.contains("x86_64") {
        // 手动设置几十个编译选项...
    }
    // 手动调用 cc, 手动链接...
}
```

**Right:**
```rust
// 正确: 利用 Cargo 的内置功能; build.rs 只处理 Cargo 做不到的事
// build.rs
fn main() {
    // 只编译无法通过 Cargo 管理的 C/ASM 依赖
    cc::Build::new()
        .file("vendor/legacy_parser.c")
        .compile("legacy_parser");
    println!("cargo:rerun-if-changed=vendor/legacy_parser.c");
}

// Cargo.toml 使用 cfg 和 features 替代 build.zig 的大部分功能:
// [target.'cfg(target_os = "linux")'.dependencies]
// mio = { version = "1", features = ["os-poll"] }
```

## Reference Implementations

| Project | Description | Relevant Patterns |
|---------|-------------|-------------------|
| [tigerbeetle](https://github.com/tigerbeetle/tigerbeetle) | Financial accounting DB -- Zig native; comparable Rust implementations exist for protocol handling | Custom allocator patterns, SIMD, IO_uring |
| [bun](https://github.com/oven-sh/bun) | JS runtime in Zig; comparable to Deno (Rust) | Build system complexity, comptime codegen, allocator strategies |
| [ghostty](https://github.com/ghostty-org/ghostty) | GPU-accelerated terminal emulator (Zig) | Similar to Alacritty (Rust): SIMD, GPU rendering, cross-platform |
| [river](https://github.com/riverwm/river) | Wayland compositor in Zig | Similar compositors in Rust (smithay-based): IPC protocols, layout engine |
| [ziglings](https://github.com/ratfactor/ziglings) | Zig exercises; comparable to Rustlings | Language idioms, test-driven learning |
| [ncbi-rs](https://github.com/stain/ncbi-rs) | Bio-informatics (Zig to Rust pattern) | Error union → Result mapping, allocator → ownership |
| [zap](https://github.com/zigzap/zap) | HTTP server in Zig (wraps C facil.io) | Comparable to hyper/actix: C FFI wrapping, async I/O |
| [capy](https://github.com/capy-ui/capy) | GUI framework in Zig | Comparable to egui/iced: immediate mode rendering, cross-platform |

## Cross-Reference

- `c-to-rust`: For C codebases -- similar memory model but C lacks comptime and error sets
- `cpp-to-rust`: For C++ codebases with templates and RAII patterns similar in spirit to Zig comptime and defer
- For allocator patterns: `bumpalo` crate for arena allocation; `typed-arena` for typed arenas
- For comptime codegen replacement: `syn` + `quote` + `proc-macro2` for proc macros; `build.rs` with `codegen` patterns
- For error set patterns: `thiserror` for derive macros; `anyhow` for application-level error type erasure
- For packed struct / protocol parsing: `deku`, `binrw`, `nom` crates
