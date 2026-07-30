---
name: python-to-rust
description: Use when migrating Python codebases to Rust — covers dynamic-to-static type mapping, asyncio-to-tokio translation, Django/Flask/FastAPI to Axum/Actix migration, numpy/pandas to ndarray/polars, pip/poetry to Cargo, GIL-limitations to true parallelism, exception-to-Result conversion, and incremental migration via PyO3. Includes canonical signatures, common mistakes, and reference implementations.
---

# Python to Rust Migration

## Architecture Mapping

Python's CPython interpreter (bytecode compilation, reference-counting GC with cycle detection, Global Interpreter Lock) maps to Rust's AOT-compiled native binary with no runtime overhead. Where Python relies on the GIL to serialize thread access to Python objects and `asyncio` for cooperative multitasking on a single thread, Rust provides true multi-threaded parallelism via `tokio` (async) and `rayon` (CPU-bound). A Django or FastAPI application served by uvicorn/gunicorn becomes a single Rust binary served by Axum or Actix-Web. pip/poetry's virtual environments and dependency resolution map to Cargo's `Cargo.toml` with lockfile-based reproducible builds. Python's per-interpreter isolation (virtualenvs, subprocess workers) becomes Rust's compile-time module system with crate-level visibility.

Python's "batteries included" standard library and PyPI's 500k+ packages map to Rust's `std` library and crates.io — Rust pulls in exactly what's needed rather than carrying the full runtime. The critical architectural difference: Python encourages "duck typing" and runtime introspection; Rust requires all types known at compile time with trait-based polymorphism.

| Python Concept                | Rust Equivalent                         | Notes                                              |
|-------------------------------|-----------------------------------------|----------------------------------------------------|
| CPython interpreter           | rustc + LLVM (AOT compilation)          | No interpreter overhead, no JIT warmup             |
| GIL (Global Interpreter Lock) | No GIL — true parallelism              | `Send + Sync` traits guarantee thread safety       |
| Reference counting + GC       | Ownership + RAII                        | No cycle detector; `Rc`/`Arc` for shared ownership |
| `async` / `await` (asyncio)   | `async fn` + tokio                      | Stackless coroutines, work-stealing scheduler      |
| `class`                       | `struct` + `impl` + `trait`             | Data separated from behavior                       |
| Duck typing                   | Static typing + generics                | All types checked at compile time                  |
| Multiple inheritance          | Trait composition                       | No MRO; explicit trait bounds                      |
| `@property`                   | Getter/setter methods                   | No language-level property syntax                  |
| `@staticmethod` / `@classmethod` | Associated functions / `Self` parameter | `fn method()` is inherent; `&self` for instances |
| `__init__` / `__new__`        | `fn new(...) -> Self` convention        | No magic methods; constructor is a static function |
| `__str__` / `__repr__`        | `Display` / `Debug` traits              | Implement traits for formatting                    |
| `__enter__` / `__exit__`      | `Drop` trait + RAII                     | No context manager protocol; RAII at scope exit    |
| `try/except/finally`          | `Result<T, E>` + `?` operator           | Errors as values; no exception stack unwinding     |
| `None`                        | `Option::None`                          | Type-level absence; `None` is a variant, not null  |
| `list` / `tuple`              | `Vec<T>` / `(A, B, ...)`               | Homogeneous vs. heterogeneous; fixed-size tuples   |
| `dict`                        | `HashMap<K, V>` / `BTreeMap<K, V>`      | Hash vs. ordered; `IndexMap` for insertion order   |
| `set`                         | `HashSet<T>` / `BTreeSet<T>`            | Unordered vs. ordered                              |
| `generator` / `yield`         | `Iterator` / `impl Iterator<Item = T>`  | Lazy; use `std::iter::from_fn` or custom `impl`    |
| `async generator`             | `futures::Stream` / `TryStream`         | Async iteration via `while let Some(x) = stream.next().await` |
| `@dataclass`                  | `#[derive(Debug, Clone)] struct`        | With `serde` derive for serialization              |
| `NamedTuple`                  | Struct with named fields                | Type-safe, immutable by default                    |
| `Enum` (Python 3.11+)         | `enum` (Rust, algebraic data type)      | Rust enums carry variant-specific payloads         |
| `Protocol` (typing)           | `trait`                                 | Structural subtyping vs. nominal trait             |
| `TypeVar` / Generic           | Generics `<T>` with trait bounds        | Monomorphized, zero-cost                           |
| `Callable[[A], R]`            | `Fn(A) -> R` / `Box<dyn Fn(A) -> R>`   | Closure trait; zero-cost or heap-erased            |
| `lambda`                      | Closure `\|x\| x + 1`                    | Same concise syntax; captures by reference or move |
| `decorator`                   | Attribute macros / wrapper functions    | Compile-time or runtime wrapping                   |
| `import` / `from ... import`  | `use crate::module::Item`               | Module system with absolute paths                  |
| `pip install` / `poetry add`  | `cargo add`                             | Cargo fetches, builds, links automatically         |
| `requirements.txt`            | `Cargo.toml [dependencies]`             | Declarative, with version resolution               |
| `virtualenv` / `.venv`        | `target/` directory (build artifacts)   | Cargo caches globally; no env activation needed    |
| `__name__ == "__main__"`      | `fn main()`                             | Defined entry point in `src/main.rs`               |
| `PYTHONPATH`                  | Cargo dependency graph                  | Compile-time resolution; no runtime path           |
| C extension (.pyd/.so)        | `cdylib` crate type                     | Native shared library via C ABI or PyO3            |
| Cython                        | `pyo3` crate + `maturin`                | Rust bindings to/from Python                       |

## Type System Mapping

Python's dynamic, duck-typed system means any variable can hold any type at any time. Rust requires all types known at compile time. This is the largest conceptual gap. Python 3.12+ type hints (`list[int]`, `dict[str, float]`, `Optional[str]`) provide a migration bridge — use them to inform the Rust type design.

| Python Type                        | Rust Type                          | Notes                                              |
|------------------------------------|------------------------------------|----------------------------------------------------|
| `int` (unbounded)                  | `i64` / `u64` / `i128`            | Python int is arbitrary precision; `num_bigint` for big ints |
| `float`                            | `f64`                              | IEEE 754 double; Python float is the same         |
| `complex`                          | `num_complex::Complex<f64>`        | Complex numbers from `num` crate                   |
| `bool`                             | `bool`                             | Identical                                          |
| `str`                              | `String` / `&str`                  | Owned vs. borrowed; Python strs are immutable      |
| `bytes`                            | `Vec<u8>` / `&[u8]` / `bytes::Bytes` | Byte buffer; `bytes` crate for zero-copy ops    |
| `bytearray`                        | `Vec<u8>`                          | Mutable byte buffer                                |
| `None`                             | `Option::None`                     | Not a standalone type; wraps `T`                   |
| `list[T]`                          | `Vec<T>`                           | Contiguous growable array                          |
| `tuple[T, U, ...]`                 | `(T, U, ...)`                      | Fixed-size heterogeneous tuple                     |
| `dict[K, V]`                       | `HashMap<K, V>` / `BTreeMap<K, V>` | Hash or ordered                                     |
| `set[T]`                           | `HashSet<T>` / `BTreeSet<T>`       | Unique elements                                    |
| `frozenset[T]`                     | Immutable reference or `im::HashSet` | Persistent hash set from `im` crate              |
| `deque[T]`                         | `VecDeque<T>`                      | Double-ended queue                                 |
| `defaultdict[K, V]`                | `HashMap<K, V>` with `.entry().or_insert_with()` | Entry API replaces defaultdict            |
| `Counter[T]`                       | `HashMap<T, usize>` + `.entry()`   | Count occurrences manually                         |
| `OrderedDict[K, V]`                | `indexmap::IndexMap<K, V>`         | Insertion-ordered map                              |
| `ChainMap`                         | Manual chained lookup              | No built-in; implement `get` chaining              |
| `Enum` (Python, str/int enum)      | `enum` (Rust, tagged union)        | Rust enums carry variant data                      |
| `Union[A, B]`                      | `enum { A(A), B(B) }`              | Discriminated union with payloads                  |
| `Optional[T]`                      | `Option<T>`                        | `None` variant, not a union type                   |
| `Any`                              | `dyn Any` / `serde_json::Value`    | Type-erased; minimize usage                        |
| `Callable[[A], R]`                 | `Fn(A) -> R` / `Box<dyn Fn(A) -> R>` | Zero-cost or heap-allocated                      |
| `Iterable[T]` / `Iterator[T]`      | `impl Iterator<Item = T>`          | Lazy; `IntoIterator` for consumable sources        |
| `Generator[T, S, R]`               | `impl Iterator<Item = T>`          | Use `std::iter::from_fn` for stateful generators   |
| `AsyncIterable[T]`                 | `impl Stream<Item = T>`            | From `futures` or `tokio-stream` crate             |
| `Awaitable[T]`                     | `impl Future<Output = T>`          | `.await` to resolve                                |
| `Type[T]`                          | `PhantomData<T>` / generics        | No runtime type parametrization                    |
| `Literal["a", "b"]`                | `enum` variant / `&'static str`    | Compile-time string discriminant                   |
| `TypedDict`                        | `struct`                           | Compile-time field list, not dict                  |
| `dataclass`                        | `#[derive(Debug, Clone)] struct`   | `serde` for serialization; `derive_builder`        |
| `Protocol` (structural typing)     | `trait` (nominal typing)           | Explicit implementation required                   |
| `NewType("Id", int)`               | Newtype pattern: `struct Id(i64)`  | Zero-cost wrapper with type safety                 |
| `Decimal`                          | `rust_decimal::Decimal`            | Fixed-point decimal from `rust_decimal` crate      |
| `Fraction`                         | `num_rational::Rational64`         | Rational numbers from `num` crate                  |
| `datetime`                         | `chrono::DateTime<Utc>`            | Timezone-aware; `NaiveDateTime` for naive          |
| `timedelta`                        | `std::time::Duration` / `chrono::TimeDelta` | Time interval                              |
| `uuid.UUID`                        | `uuid::Uuid`                       | 128-bit identifier                                 |
| `pathlib.Path`                     | `std::path::{Path, PathBuf}`       | `PathBuf` is owned; `Path` is borrowed             |
| `re.Pattern`                       | `regex::Regex`                     | Compile once; `Regex::new(r"...").unwrap()`        |

## Memory & Ownership Model

Python's reference-counting GC (with cycle-breaking trace collector) means objects live as long as something references them. Rust's ownership system replaces the GC entirely with compile-time rules. This is the hardest mental shift for Python developers — there is no more "everything is a reference on the heap."

### The Stack/Heap Inversion

```python
# Python: everything is heap-allocated, variables are references
class User:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

user = User("Alice", 30)  # User object on heap, 'user' is a reference
```

```rust
// Rust: stack allocation by default; heap requires explicit Box
struct User {
    name: String,   // String data on heap, struct field on stack
    age: u32,       // inline, on stack
}

let user = User { name: "Alice".into(), age: 30 }; // stack-allocated
// For heap: let user = Box::new(User { ... });
```

### Ownership Rules for Python Developers

| Python Pattern                         | Rust Translation                                    |
|----------------------------------------|-----------------------------------------------------|
| Pass reference, mutate object in place | `&mut T` (exclusive mutable borrow)                 |
| Pass reference, read only              | `&T` (shared immutable borrow)                      |
| Multiple threads sharing mutable state | `Arc<Mutex<T>>` or `Arc<RwLock<T>>`                 |
| GC cleans up unreachable objects       | `Drop` trait, scope-based cleanup, no finalizers    |
| `__del__`                              | `Drop::drop(&mut self)` — cannot fail               |
| `weakref.ref`                          | `Weak<T>` (from `Arc` or `Rc`)                      |
| `id(obj)` (object identity)            | `Arc::as_ptr()` / pointer comparison                |
| Circular references                    | `Weak<T>` breaks cycles; or arena-based indices     |
| `copy.copy` / `copy.deepcopy`          | `.clone()` (explicit, `Clone` trait)                |

### Pattern: List Comprehension → Iterator

```python
# Python: list comprehension — eagerly builds a list
squares = [x * x for x in range(1000) if x % 2 == 0]
```

```rust
// Rust: iterator chain — lazy, zero intermediate allocations
let squares: Vec<i32> = (0..1000)
    .filter(|x| x % 2 == 0)
    .map(|x| x * x)
    .collect(); // materialize only once
```

### Pattern: Context Manager → Drop / RAII

```python
# Python: context manager — __enter__ / __exit__
with open("data.txt") as f:
    content = f.read()
# f.close() called automatically
```

```rust
// Rust: RAII — Drop at scope exit
use std::fs;
let content = {
    let f = fs::File::open("data.txt")?;
    fs::read_to_string(f)?
}; // File::drop closes the handle here
```

## Concurrency / Async Translation

Python has multiple concurrency models: `threading` (GIL-limited), `multiprocessing` (process isolation), `asyncio` (single-threaded cooperative), and `concurrent.futures` (thread/process pools). Rust provides true multi-threaded parallelism with no GIL, plus an async runtime (tokio) for I/O-bound workloads.

### asyncio → tokio

| Python (asyncio)                    | Rust (tokio)                              | Notes                                   |
|-------------------------------------|-------------------------------------------|-----------------------------------------|
| `async def fn(): ...`               | `async fn fn() { ... }`                   | Identical syntax; Rust futures are lazy |
| `await coro`                        | `coro.await`                              | Postfix suspension syntax               |
| `asyncio.gather(a, b, c)`           | `tokio::join!(a, b, c)`                   | Concurrent execution of all             |
| `asyncio.wait(tasks, return_when=FIRST)` | `tokio::select! { r = a => ..., r = b => ... }` | Race: first to complete          |
| `asyncio.create_task(coro)`         | `tokio::spawn(async { ... })`             | Returns `JoinHandle`                    |
| `asyncio.sleep(ms / 1000)`          | `tokio::time::sleep(Duration::from_millis(ms))` | Async sleep                         |
| `asyncio.Queue`                     | `tokio::sync::mpsc::channel(n)`           | Bounded async channel                   |
| `asyncio.Event`                     | `tokio::sync::Notify`                     | Single-event notification               |
| `asyncio.Semaphore`                 | `tokio::sync::Semaphore`                  | Concurrency limit                       |
| `asyncio.Lock`                      | `tokio::sync::Mutex<T>`                   | Async mutex; data-inside-lock           |
| `asyncio.to_thread(func)`           | `tokio::task::spawn_blocking(move || func())` | Offload CPU work to blocking pool  |
| `asyncio.timeout(seconds)`          | `tokio::time::timeout(dur, future)`       | Timeout wrapping                         |
| `asyncio.TaskGroup` (3.11+)         | `tokio::task::JoinSet`                    | Structured concurrent tasks             |

### threading / multiprocessing → Rayon / std::thread

| Python                               | Rust                                     |
|--------------------------------------|------------------------------------------|
| `threading.Thread(target=fn)`        | `std::thread::spawn(move || fn())`       |
| `multiprocessing.Pool(n).map(fn, it)` | `it.into_par_iter().map(fn)` (rayon)    |
| `concurrent.futures.ThreadPoolExecutor` | `rayon::ThreadPool`                    |
| `multiprocessing.Queue`              | `crossbeam::queue::SegQueue` / channels  |
| `multiprocessing.Value` / `.Array`   | `Arc<Atomic*>` / `Arc<Mutex<T>>`         |
| `queue.Queue` (thread-safe)          | `std::sync::mpsc::channel`               |
| `threading.local()`                  | `std::thread_local!` macro               |

### asyncio Task → tokio Task

```python
# Python: asyncio — single event loop
import asyncio

async def fetch_all(urls: list[str]) -> list[dict]:
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)
```

```rust
// Rust: tokio — multi-threaded work-stealing runtime
use reqwest::Client;
use tokio::task::JoinSet;

async fn fetch_all(urls: &[String]) -> Result<Vec<serde_json::Value>, reqwest::Error> {
    let client = Client::new();
    let mut tasks = JoinSet::new();
    for url in urls {
        let client = client.clone();
        let url = url.clone();
        tasks.spawn(async move {
            client.get(&url).send().await?.json().await
        });
    }
    let mut results = Vec::new();
    while let Some(result) = tasks.join_next().await {
        results.push(result??);
    }
    Ok(results)
}
```

### GIL-Free Parallelism

```python
# Python: CPU-bound work — GIL prevents true parallelism in threads
from concurrent.futures import ProcessPoolExecutor

with ProcessPoolExecutor() as pool:
    results = pool.map(heavy_compute, data_chunks)
```

```rust
// Rust: true parallelism — no GIL, shared memory
use rayon::prelude::*;

let results: Vec<Output> = data_chunks
    .par_iter()  // parallel iterator, work-stealing
    .map(|chunk| heavy_compute(chunk))
    .collect();
```

## Build System & Dependencies

| Python Tool                       | Rust Equivalent                      | Notes                                   |
|-----------------------------------|--------------------------------------|-----------------------------------------|
| `pip install` / `pip3`            | `cargo add` / `cargo build`          | Cargo resolves, fetches, builds, links   |
| `poetry add` / `poetry install`   | `cargo add` / `cargo build`          | Same declarative dependency management   |
| `pyproject.toml`                  | `Cargo.toml`                         | Build manifest + deps + metadata         |
| `poetry.lock` / `requirements.txt` | `Cargo.lock`                        | Reproducible builds via lockfile         |
| `pip freeze > requirements.txt`   | `cargo update` (auto-updates lock)   | Lockfile updated on dependency change    |
| `virtualenv .venv`                | Cargo global cache (`~/.cargo`)      | No per-project env; globally cached deps |
| `python -m pytest`                | `cargo test`                         | Built-in test runner                     |
| `python -m mypy`                  | `cargo check` (type checking)        | Compile-time type verification           |
| `ruff check` / `flake8`           | `cargo clippy`                       | Linter with 500+ rules                   |
| `black` / `ruff format`           | `cargo fmt`                          | Opinionated formatter                    |
| `python setup.py build_ext`       | `build.rs` + `cc` crate              | Compile native C extensions              |
| `cython` / `mypyc`                | `pyo3` crate + `maturin`             | Rust bindings from/to Python             |
| `coverage run -m pytest`          | `cargo tarpaulin` / `cargo-llvm-cov` | Code coverage                            |
| `python -m timeit`                | `criterion` / `divan`                | Benchmarking                             |
| `python -m cProfile`              | `perf` / `flamegraph` / `samply`     | Profiling                                |
| `docker build` (multi-stage)      | `cargo build --release` + `scratch`  | Single binary; tiny base image           |

**Cargo.toml for a migrated FastAPI service:**

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
reqwest = { version = "0.12", features = ["json"] }
validator = { version = "0.18", features = ["derive"] }
```

## Framework Mapping: Django/Flask/FastAPI → Axum/Actix-Web

| Python Framework               | Rust / Axum                           | Notes                                   |
|--------------------------------|---------------------------------------|-----------------------------------------|
| FastAPI `@app.get("/path")`    | `Router::route("/path", get(handler))` | Method + path chaining on Router        |
| `@app.post("/path")`           | `Router::route("/path", post(handler))` | Same pattern                            |
| FastAPI `Query(...)` / `Body(...)` | `Query<T>` / `Json<T>` extractor    | Type-safe extraction via serde          |
| FastAPI `Path(...)`            | `Path<T>` extractor                   | URL parameter extraction                |
| FastAPI `Depends(...)`         | `axum::extract::State` + middleware   | Shared state injection                  |
| FastAPI `HTTPException`        | Error type implementing `IntoResponse` | Typed error → HTTP response mapping     |
| FastAPI `Response(status_code=N)` | `(StatusCode::OK, Json(body))`     | Tuple return type for status + body     |
| Flask `@app.route`             | `Router::route` with method chaining   | Same pattern; no decorator syntax       |
| Flask `request.json`           | `Json<T>` extractor                   | Type at compile time, not runtime        |
| Flask `g` (request-scoped)     | Request extensions via `Extension<T>`  | Typed request-local state               |
| Django `Model` / ORM           | `sqlx::query_as` / `diesel::table!`    | Explicit SQL or query DSL               |
| Django `QuerySet.filter(...)`  | Iterator `.filter()` or query builder  | Lazy chains on query or in-memory       |
| Django `Model.objects.get(pk=)` | `.fetch_one(&pool)` / `.fetch_optional(&pool)` | Async-aware, compile-time checked |
| Django Admin                   | No built-in equivalent                 | Use `daisyui` + HTMX or custom UI       |
| Django REST Framework          | `axum` + `serde` + `utoipa` (OpenAPI)  | Serde covers serializers; axum covers views |
| DRF `Serializer`               | `serde::Serialize` / `Deserialize`     | Derive-based, not class definition      |
| Django Migrations              | `sqlx migrate` / `diesel migration`    | Versioned SQL files                     |
| Celery tasks                   | `tokio::spawn` + Redis/RabbitMQ client  | Background tasks; no dedicated worker   |
| Django Channels (WebSocket)    | `axum::extract::ws`                    | Built-in WebSocket support              |
| Flask-SQLAlchemy               | `sqlx` or `diesel`                     | Async-native or sync ORM                |
| gunicorn / uvicorn workers     | `tokio` runtime (multi-threaded)       | Built-in; no separate process manager   |
| Pydantic models                | `struct` + `serde` + `validator`       | Compile-time validated types            |
| Marshmallow schemas            | `serde` with `#[serde(rename = "...")]` | Derive-based serialization              |
| Python `logging`               | `tracing` + `tracing-subscriber`       | Structured, span-based logging          |
| structlog                      | `tracing` with JSON subscriber          | Same structured approach                |
| python-dotenv                  | `dotenvy` crate                        | Same `.env` file loading                |
| Alembic (DB migrations)        | `sqlx migrate` / `diesel migration`    | Versioned SQL migration runner          |
| pytest + pytest-asyncio        | `#[tokio::test]` + `cargo test`       | Built-in async test support             |
| pytest fixtures                | Manual setup/teardown or `rstest` crate | `#[fixture]` from `rstest`              |
| factory_boy                    | Builder pattern + `fake` crate         | `#[derive(Builder)]` from `typed-builder` |

## Standard Library & Ecosystem Mapping

### Core Builtins

| Python Builtin                     | Rust Equivalent                       | Notes                                   |
|------------------------------------|---------------------------------------|-----------------------------------------|
| `print(x)`                         | `println!("{x}")`                     | Format string with `{}`                 |
| `len(seq)`                         | `seq.len()`                           | Method, not function; O(1) for Vec      |
| `range(start, stop, step)`         | `(start..stop).step_by(step)`         | Range expressions; lazy                 |
| `enumerate(seq)`                   | `seq.iter().enumerate()`              | Returns `(usize, &T)` iterator          |
| `zip(a, b)`                        | `a.iter().zip(b.iter())`              | Stops at shorter; `izip!` for multi     |
| `map(fn, seq)`                     | `seq.iter().map(fn).collect()`        | Lazy, materialize with collect          |
| `filter(fn, seq)`                  | `seq.iter().filter(fn).collect()`     | Lazy filter                             |
| `sorted(seq, key=fn)`              | `seq.iter().sorted_by_key(fn)` (itertools) | Or `v.sort_by_key(fn)` if mutable  |
| `reversed(seq)`                    | `seq.iter().rev()`                    | Reversed iteration                      |
| `sum(seq)`                         | `seq.iter().sum::<T>()`               | Requires `T: Sum`                       |
| `max(seq)` / `min(seq)`            | `seq.iter().max()` / `.min()`         | Returns `Option<&T>`                    |
| `any(seq)` / `all(seq)`            | `seq.iter().any(fn)` / `.all(fn)`     | Same semantics                          |
| `isinstance(obj, cls)`             | `std::any::Any::downcast_ref::<T>()`  | Only for `dyn Any`; prefer `match`      |
| `hasattr(obj, "attr")`             | Compile-time: traits + method existence | Not needed; types known at compile time |
| `getattr(obj, "attr", default)`    | Pattern matching or trait methods     | Use `Option` combinators                |
| `type(obj)`                        | `std::any::type_name::<T>()`          | Debug-only; not for logic               |
| `f"{value:.2f}"`                   | `format!("{value:.2}")`               | Same precision specifier                |
| `open(path)`                       | `std::fs::File::open(path)`           | RAII; closes on drop                    |
| `open(path).read()`                | `std::fs::read_to_string(path)?`      | Convenient one-shot                     |
| `open(path, "wb").write(data)`     | `std::fs::write(path, data)?`         | Atomic-ish; `tokio::fs::write` for async |
| `pathlib.Path("a/b").mkdir(parents=True)` | `std::fs::create_dir_all(path)?` | Recursive creation                  |
| `os.listdir(path)`                 | `std::fs::read_dir(path)?`            | Returns iterator of `DirEntry`          |
| `os.environ["KEY"]`                | `std::env::var("KEY")?`               | Returns `Result<String, VarError>`      |
| `sys.argv`                         | `std::env::args()`                    | Iterator; `clap` for CLI parsing        |
| `json.loads(s)`                    | `serde_json::from_str::<T>(&s)?`      | Type must be known at compile time      |
| `json.dumps(obj)`                  | `serde_json::to_string(&obj)?`        | Derive `Serialize`                      |
| `json.dumps(obj)` (dynamic)        | `serde_json::to_value(&obj)?`         | Returns `serde_json::Value`             |
| `csv.reader(file)`                 | `csv::Reader::from_reader(file)`      | From `csv` crate                        |
| `hashlib.sha256(data)`             | `sha2::Sha256::digest(data)`          | From `sha2` crate                       |
| `base64.b64encode(data)`           | `base64::Engine::encode(&data)`       | From `base64` crate                     |
| `re.compile(r"\d+")`               | `Regex::new(r"\d+")?`                 | From `regex` crate                      |
| `random.randint(a, b)`             | `rng.gen_range(a..=b)`                | From `rand` crate                       |
| `random.choice(seq)`               | `seq.choose(&mut rng)`                | From `rand::seq::SliceRandom`           |
| `uuid.uuid4()`                     | `Uuid::new_v4()`                      | From `uuid` crate                       |
| `datetime.now()`                   | `chrono::Utc::now()`                  | From `chrono` crate                     |
| `str.startswith(p)` / `endswith(p)` | `s.starts_with(p)` / `s.ends_with(p)` | Same                                    |
| `str.split(d)`                     | `s.split(d).collect::<Vec<_>>()`      | Returns iterator                         |
| `str.strip()`                      | `s.trim()`                            | Same whitespace handling                |
| `",".join(seq)`                    | `seq.join(",")` (Vec) / `itertools::join` | Iterator-level join with itertools   |
| `str.replace(a, b)`                | `s.replace(a, b)`                     | Same                                    |
| `str.upper()` / `str.lower()`      | `s.to_uppercase()` / `s.to_lowercase()` | Unicode-aware                          |
| `int(s)` / `float(s)`              | `s.parse::<i64>()` / `s.parse::<f64>()` | Returns `Result`; no ValueError        |
| `bin(n)` / `hex(n)`                | `format!("{n:b}")` / `format!("{n:x}")` | Format specifiers                       |

### NumPy → ndarray

| NumPy Operation                     | Rust (ndarray)                        | Notes                                   |
|-------------------------------------|---------------------------------------|-----------------------------------------|
| `np.array([1, 2, 3])`               | `arr1(&[1.0, 2.0, 3.0])`             | `Array1` from slice                     |
| `np.zeros((M, N))`                  | `Array2::zeros((M, N))`               | Zero-initialized                        |
| `np.ones((M, N))`                   | `Array2::ones((M, N))`                | One-initialized                         |
| `np.arange(start, stop, step)`      | `Array1::range(start, stop, step)`    | Linear range                            |
| `np.linspace(a, b, n)`              | `Array1::linspace(a, b, n)`           | Evenly-spaced values                    |
| `a + b` (element-wise)              | `&a + &b`                             | Same operator; returns new array        |
| `a * b` (element-wise)              | `&a * &b`                             | Element-wise multiply                   |
| `a @ b` (matrix multiply)           | `a.dot(&b)`                           | Matrix product                          |
| `a.T` (transpose)                   | `a.t()`                               | Returns view; `.to_owned()` for copy    |
| `a.reshape((M, N))`                 | `a.into_shape((M, N))`                | Shape change; O(1) if contiguous        |
| `a.sum(axis=0)`                     | `a.sum_axis(Axis(0))`                 | Axis-specific reduction                 |
| `a.mean(axis=1)`                    | `a.mean_axis(Axis(1))`                | From `ndarray-stats`                    |
| `np.concatenate([a, b], axis=0)`    | `ndarray::stack(Axis(0), &[a, b])`    | Stack along axis                        |
| `a[a > 0]` (boolean indexing)       | Filter with iterator or mask          | No built-in boolean array indexing      |
| `np.where(cond, a, b)`              | `azip!((c in cond, a in a, b in b) if *c { a } else { b })` | Element-wise `azip!` macro |

### Pandas → Polars

```python
# Python (pandas):
# df = pd.read_csv("data.csv")
# result = (
#     df[df["value"] > 10]
#     .groupby("category")
#     .agg(mean_val=("value", "mean"), n=("value", "count"))
#     .sort_values("mean_val", ascending=False)
# )
```

```rust
// Rust (polars):
use polars::prelude::*;

fn pipeline(path: &str) -> PolarsResult<DataFrame> {
    let df = CsvReadOptions::default()
        .try_into_reader_with_file_path(Some(path.into()))?
        .finish()?;

    df.lazy()
        .filter(col("value").gt(10.0))
        .group_by(&[col("category")])
        .agg(&[col("value").mean().alias("mean_val"), col("value").count().alias("n")])
        .sort(["mean_val"], SortMultipleOptions::default().with_order_descending(true))
        .collect()
}
```

### Popular Python Libraries → Rust Crates

| Python Library                | Rust Crate                            | Notes                                   |
|-------------------------------|---------------------------------------|-----------------------------------------|
| `requests` / `httpx`          | `reqwest`                             | Async HTTP client with TLS              |
| `aiohttp`                     | `reqwest`                             | Same; axum for server side              |
| `flask` / `fastapi`           | `axum` / `actix-web`                  | High-performance web frameworks         |
| `django`                      | `axum` + `sqlx` + templates           | Assembly of crates, not one framework   |
| `sqlalchemy`                  | `diesel` (sync) / `sea-orm` (async)   | ORM with migrations                     |
| `psycopg2` / `asyncpg`        | `sqlx` (postgres feature)             | Async, compile-time SQL verification   |
| `pymongo` / `motor`           | `mongodb` crate                       | Official MongoDB driver                 |
| `redis-py` / `aioredis`       | `redis` crate                         | Async Redis client                      |
| `celery`                      | `tokio::spawn` + Redis / `apalis`     | Background job processing               |
| `click` / `typer` / `argparse` | `clap` (derive mode)                 | Declarative CLI argument parsing        |
| `pydantic` / `attrs` / `dataclasses` | `serde` + `validator`           | Derive-based serialization + validation |
| `marshmallow`                 | `serde` with attributes               | Serialization via derive macros         |
| `tenacity` / `backoff`        | `backoff` crate                       | Retry with exponential backoff          |
| `structlog` / `loguru`        | `tracing` + `tracing-subscriber`      | Structured, span-based logging          |
| `pytest`                      | `#[test]` + `cargo test`              | Built-in test framework                 |
| `pytest-cov` / `coverage`     | `cargo-tarpaulin` / `cargo-llvm-cov`  | Source-based code coverage              |
| `unittest.mock` / `pytest-mock` | `mockall` / `mock_instant`          | Mock frameworks                         |
| `hypothesis` / `pytest-check`  | `proptest`                            | Property-based testing                  |
| `pillow` / `PIL`              | `image` crate                         | Image decode/resize/encode              |
| `opencv-python`               | `opencv` crate                        | Computer vision                         |
| `pyyaml`                      | `serde_yaml`                          | YAML serialization                      |
| `lxml` / `beautifulsoup4`      | `scraper` / `tl`                      | HTML parsing; `scraper` for CSS selectors |
| `cryptography` / `pycryptodome` | `ring` / `aes-gcm` / `rsa`          | Cryptographic primitives                |
| `jwt` / `python-jose`         | `jsonwebtoken` crate                  | JWT encode/decode                       |
| `bcrypt` / `passlib`          | `bcrypt` / `argon2` crate             | Password hashing                        |
| `jinja2` / `mako` / `django-templates` | `tera` / `askama` / `minijinja` | Template engines; askama is compile-time |
| `websockets`                  | `axum::extract::ws` / `tokio-tungstenite` | WebSocket support                      |
| `grpcio` / `grpc`             | `tonic`                               | gRPC server/client                      |
| `protobuf`                    | `prost` + `prost-build`               | Protocol Buffers codegen                |
| `msgpack`                     | `rmp-serde`                           | MessagePack serialization               |
| `numpy`                       | `ndarray`                             | N-dimensional arrays                    |
| `scipy`                       | `ndarray-stats` / `nalgebra` / `faer` | Scientific computing pieces             |
| `pandas`                      | `polars`                              | DataFrames; lazy + eager APIs           |
| `scikit-learn`                | `smartcore` / `linfa`                 | ML algorithms                           |
| `pytorch`                     | `tch-rs` / `candle` / `burn`          | Deep learning frameworks                |
| `tensorflow`                  | `tensorflow` crate (FFI)              | TF C API bindings                       |
| `matplotlib`                  | `plotters`                            | Plotting and charting                   |
| `seaborn`                     | `plotters` + custom styling           | Statistical plotting                    |
| `plotly`                      | `plotly` crate                        | Interactive plots (WASM target)         |
| `jupyter`                     | `evcxr` (Rust REPL in Jupyter)        | Interactive exploration                 |
| `ruff` / `flake8` / `pylint`  | `clippy` + `rustfmt`                  | Linting and formatting (built-in)       |
| `black` / `isort`             | `cargo fmt`                           | Built-in formatter                      |

## Canonical Patterns

### 1. Class with Methods → Struct + impl

```python
# Python: class-based design
class OrderService:
    def __init__(self, db: Database, payment: PaymentGateway):
        self.db = db
        self.payment = payment

    async def place_order(self, request: OrderRequest) -> Order:
        total = sum(item.price for item in request.items)
        charge = await self.payment.charge(total, request.card_token)
        order = await self.db.orders.create(request, charge.id)
        return order
```

```rust
// Rust: struct + impl with trait bounds for dependencies
use async_trait::async_trait;

#[async_trait]
pub trait PaymentGateway: Send + Sync {
    async fn charge(&self, amount: f64, token: &str) -> Result<Charge, PaymentError>;
}

pub struct OrderService<P: PaymentGateway> {
    db: PgPool,
    payment: P,
}

impl<P: PaymentGateway> OrderService<P> {
    pub fn new(db: PgPool, payment: P) -> Self {
        Self { db, payment }
    }

    pub async fn place_order(
        &self,
        request: &OrderRequest,
    ) -> Result<Order, OrderError> {
        let total: f64 = request.items.iter().map(|i| i.price).sum();
        let charge = self.payment.charge(total, &request.card_token).await?;
        let order = sqlx::query_as::<_, Order>(
            "INSERT INTO orders (...) VALUES (...) RETURNING *"
        )
        .bind(&request.customer_id)
        .bind(total)
        .bind(&charge.id)
        .fetch_one(&self.db)
        .await?;
        Ok(order)
    }
}
```

### 2. Exception → Result

```python
# Python: try/except — control flow via exceptions
def divide(a: float, b: float) -> float:
    if b == 0:
        raise ValueError("division by zero")
    return a / b

try:
    result = divide(10, 0)
except ValueError as e:
    print(f"error: {e}")
```

```rust
// Rust: Result — error as value, not control flow
fn divide(a: f64, b: f64) -> Result<f64, &'static str> {
    if b == 0.0 {
        Err("division by zero")
    } else {
        Ok(a / b)
    }
}

match divide(10.0, 0.0) {
    Ok(result) => println!("{result}"),
    Err(e) => eprintln!("error: {e}"),
}

// With thiserror for structured errors:
#[derive(Error, Debug)]
enum MathError {
    #[error("division by zero")]
    DivisionByZero,
    #[error("overflow: {0}")]
    Overflow(String),
}

fn safe_divide(a: f64, b: f64) -> Result<f64, MathError> {
    if b == 0.0 { return Err(MathError::DivisionByZero); }
    Ok(a / b)
}
```

### 3. Generator → Iterator

```python
# Python: generator — lazy, stateful iteration
def fibonacci(limit: int):
    a, b = 0, 1
    while a < limit:
        yield a
        a, b = b, a + b

for n in fibonacci(100):
    print(n)
```

```rust
// Rust: Iterator impl — same lazy semantics
struct Fibonacci {
    curr: u64,
    next: u64,
    limit: u64,
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        if self.curr >= self.limit {
            return None;
        }
        let current = self.curr;
        self.curr = self.next;
        self.next = current + self.next;
        Some(current)
    }
}

// Or with std::iter::from_fn for simpler cases:
fn fibonacci(limit: u64) -> impl Iterator<Item = u64> {
    let mut state = (0u64, 1u64);
    std::iter::from_fn(move || {
        if state.0 >= limit { return None; }
        let current = state.0;
        state = (state.1, state.0 + state.1);
        Some(current)
    })
}
```

### 4. Dependency Injection → Trait Objects

```python
# Python: duck typing — "if it looks like a duck"
async def send_invoice_email(
    email_service,  # any object with .send()
    invoice: Invoice,
) -> None:
    body = render_invoice_html(invoice)
    await email_service.send(
        to=invoice.customer_email,
        subject=f"Invoice #{invoice.id}",
        body=body,
    )
```

```rust
// Rust: trait bounds — compile-time duck typing
use async_trait::async_trait;

#[async_trait]
pub trait EmailService: Send + Sync {
    async fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), EmailError>;
}

pub async fn send_invoice_email(
    email: &impl EmailService,
    invoice: &Invoice,
) -> Result<(), AppError> {
    let body = render_invoice_html(invoice)?;
    email
        .send(
            &invoice.customer_email,
            &format!("Invoice #{}", invoice.id),
            &body,
        )
        .await?;
    Ok(())
}

// Or use Arc<dyn EmailService> for runtime polymorphism
pub async fn send_invoice_email_dyn(
    email: &dyn EmailService,
    invoice: &Invoice,
) -> Result<(), AppError> {
    // ...
}
```

### 5. Context Manager → RAII Guard

```python
# Python: context manager with __enter__/__exit__
from contextlib import contextmanager

@contextmanager
def transaction(db: Database):
    tx = db.begin()
    try:
        yield tx
        tx.commit()
    except Exception:
        tx.rollback()
        raise
```

```rust
// Rust: RAII guard — Drop handles cleanup regardless of exit path
pub struct Transaction<'a> {
    conn: &'a mut sqlx::PgConnection,
    committed: bool,
}

impl<'a> Transaction<'a> {
    pub async fn begin(conn: &'a mut sqlx::PgConnection) -> Result<Self, sqlx::Error> {
        sqlx::query("BEGIN").execute(&mut *conn).await?;
        Ok(Self { conn, committed: false })
    }

    pub async fn commit(mut self) -> Result<(), sqlx::Error> {
        sqlx::query("COMMIT").execute(&mut *self.conn).await?;
        self.committed = true;
        Ok(())
    }
}

impl Drop for Transaction<'_> {
    fn drop(&mut self) {
        if !self.committed {
            // In real code, spawn a blocking task for ROLLBACK
        }
    }
}
```

### 6. Dataclass + Validation → Typed Struct + Constructor

```python
# Python: pydantic — runtime validation
from pydantic import BaseModel, EmailStr, field_validator

class CreateUserRequest(BaseModel):
    name: str
    email: EmailStr
    age: int

    @field_validator("age")
    @classmethod
    def age_must_be_positive(cls, v: int) -> int:
        if v <= 0:
            raise ValueError("age must be positive")
        return v
```

```rust
// Rust: typed builder + validation at construction time
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
        Ok(ValidatedUser {
            name: self.name,
            email: self.email,
            age: self.age,
        })
    }
}
```

## FFI & Incremental Migration

### Strategy: PyO3 Bridge (Recommended)

PyO3 enables bidirectional Rust-Python interop. For existing Python codebases, the most practical approach is to replace hot paths with Rust extensions while keeping the Python application shell.

| Strategy                 | Tool                    | When to Use                              |
|--------------------------|-------------------------|------------------------------------------|
| PyO3 extension module    | `pyo3` + `maturin`     | Replace performance-critical functions   |
| Rust sidecar binary      | JSON over stdin/stdout  | Batch processing, queue workers          |
| HTTP/gRPC microservice   | axum / tonic            | Extract service at network boundary      |
| Shared library (C ABI)   | `cdylib` + `ctypes`/`cffi` | Minimal Python changes, drop-in .so    |
| Full rewrite             | Pure Rust               | Greenfield, small-to-medium codebases    |

### PyO3: Calling Rust from Python

```rust
// Rust — compiled as Python extension via maturin
use pyo3::prelude::*;
use std::collections::HashMap;

/// Fast word frequency counter, replacing Python's collections.Counter
#[pyfunction]
fn word_frequency(text: &str) -> HashMap<String, usize> {
    let mut counts = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word.to_lowercase()).or_insert(0) += 1;
    }
    counts
}

/// Parse and validate JSON against a schema (fast path)
#[pyfunction]
fn validate_json(schema: &str, data: &str) -> PyResult<String> {
    let schema: serde_json::Value = serde_json::from_str(schema)
        .map_err(|e| PyErr::new::<pyo3::exceptions::PyValueError, _>(e.to_string()))?;
    let data: serde_json::Value = serde_json::from_str(data)
        .map_err(|e| PyErr::new::<pyo3::exceptions::PyValueError, _>(e.to_string()))?;
    // validation logic...
    Ok(serde_json::to_string(&data).unwrap())
}

#[pymodule]
fn my_rust_lib(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(word_frequency, m)?)?;
    m.add_function(wrap_pyfunction!(validate_json, m)?)?;
    Ok(())
}
```

```python
# Python — using the Rust extension
from my_rust_lib import word_frequency, validate_json

counts = word_frequency("the quick brown fox the fox jumps")
# {'the': 2, 'quick': 1, 'brown': 1, 'fox': 2, 'jumps': 1}

result = validate_json('{"type": "object"}', '{"name": "Alice"}')
```

### Migration Order

| Phase | Scope              | Strategy                                        |
|-------|--------------------|-------------------------------------------------|
| 1     | Type definitions   | Define shared types via JSON Schema / Protobuf  |
| 2     | Hot-path functions | Rewrite CPU-intensive functions as PyO3 modules |
| 3     | Data validation    | Move Pydantic/marshmallow validation to Rust    |
| 4     | Background workers | Replace Celery tasks with Rust sidecar binaries |
| 5     | API read endpoints | Rewrite in axum, route via reverse proxy        |
| 6     | API write endpoints| Full mutation handling in Rust; shared DB       |
| 7     | Full cutover       | Remove Python dependency; keep PyO3 for scripting |

## Common Mistakes

### Mistake 1: Overusing `clone()` Instead of Borrowing

```rust
// WRONG: Python devs .clone() everything to "keep a reference"
fn process(items: Vec<Item>) -> Vec<Output> {
    let mut results = Vec::new();
    for item in &items {
        let item_copy = item.clone();  // unnecessary clone
        results.push(transform(item_copy));
    }
    results
}

// CORRECT: borrow what you can, move what you own
fn process(items: &[Item]) -> Vec<Output> {
    items.iter().map(|item| transform(item)).collect()
}
```

### Mistake 2: Using `Box<dyn Error>` as Catch-All

```rust
// WRONG: like Python's `except Exception as e`
fn handle_request(req: Request) -> Result<Response, Box<dyn std::error::Error>> {
    // all callers lose type information
}

// CORRECT: define structured error types with thiserror
#[derive(Error, Debug)]
pub enum ApiError {
    #[error("validation: {0}")]
    Validation(String),
    #[error("not found: {0}")]
    NotFound(String),
    #[error("internal error")]
    Internal(#[from] anyhow::Error),
}

impl axum::response::IntoResponse for ApiError {
    fn into_response(self) -> axum::response::Response {
        let (status, message) = match &self {
            ApiError::Validation(m) => (StatusCode::BAD_REQUEST, m.clone()),
            ApiError::NotFound(m) => (StatusCode::NOT_FOUND, m.clone()),
            ApiError::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "internal error".into()),
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}
```

### Mistake 3: Using `Vec<Vec<T>>` Instead of ndarray

```rust
// WRONG: Python lists of lists habit for numerical data
let data: Vec<Vec<f64>> = vec![vec![1.0; 1000]; 1000];
let sum: f64 = data.iter().flatten().sum(); // cache-unfriendly, no SIMD

// CORRECT: use ndarray for numerical arrays
use ndarray::Array2;
let data = Array2::from_elem((1000, 1000), 1.0);
let sum: f64 = data.sum(); // contiguous memory, auto-SIMD
```

### Mistake 4: Holding a MutexGuard Across `.await`

```rust
// WRONG: Python devs used to GIL serializing access
async fn update_cache(cache: &Mutex<HashMap<String, Item>>, key: &str) {
    let mut guard = cache.lock().unwrap();
    // ... holding lock across await point — will deadlock or panic
    let item = fetch_from_db(key).await?;
    guard.insert(key.to_string(), item);
}

// CORRECT: scope the lock to the minimal critical section
async fn update_cache(cache: &Mutex<HashMap<String, Item>>, key: &str) {
    let item = fetch_from_db(key).await?;  // I/O outside the lock
    let mut guard = cache.lock().unwrap();
    guard.insert(key.to_string(), item);   // lock held only here
} // guard dropped here
```

### Mistake 5: Expecting Runtime Introspection

```python
# Python: runtime attribute access is normal
data = json.loads(response)
if "user" in data:
    name = data["user"].get("name", "anonymous")
```

```rust
// WRONG: using serde_json::Value everywhere loses type safety
let value: serde_json::Value = serde_json::from_str(&response)?;
let name = value["user"]["name"].as_str().unwrap_or("anonymous");

// CORRECT: define typed structs at the boundary
#[derive(Deserialize)]
struct ApiResponse {
    user: Option<User>,
}

#[derive(Deserialize)]
struct User {
    #[serde(default = "default_name")]
    name: String,
}

fn default_name() -> String { "anonymous".into() }

let response: ApiResponse = serde_json::from_str(&response)?;
let name = response.user.map(|u| u.name).unwrap_or_else(default_name);
// compiler catches field name typos, type mismatches, missing required fields
```

### Mistake 6: Treating Strings Like Python str

```rust
// WRONG: Python devs slice strings by character index
let s = "你好世界";  // "hello world" in Chinese
let first = &s[..1];  // PANIC: byte index 1 is not a char boundary

// CORRECT: respect UTF-8 byte boundaries
let s = "你好世界";
let first_char = s.chars().next().unwrap();  // '你'
// For slicing: s.char_indices() or the unicode-segmentation crate
```

## Reference Implementations

| Project                          | Description                                         | Python LOC | Rust LOC |
|----------------------------------|-----------------------------------------------------|------------|----------|
| Ruff                             | Python linter + formatter (10-100x faster than Flake8) | N/A      | ~40k     |
| uv                               | Python package installer (replaces pip, 10-100x faster) | N/A    | ~100k    |
| Polars                           | DataFrame library (replaces pandas for large data)   | N/A       | ~150k    |
| Pydantic V2 (core)               | Validation engine rewritten in Rust                   | ~15k      | ~10k     |
| Granian                          | HTTP server (replaces gunicorn/uvicorn)               | N/A       | ~15k     |
| Robyn                            | Python web framework with Rust runtime                | ~5k       | ~20k     |
| tokenizers (HuggingFace)         | Fast NLP tokenizers, Rust core with Python bindings   | ~10k      | ~15k     |
| orjson                           | Fast JSON library (replaces stdlib json)              | ~2k       | ~8k      |
| Cryptography (Rust parts)        | Cryptographic recipes, Rust core with Python CFFI     | ~50k      | ~10k     |
| Watchfiles                       | File watching (replaces watchdog), Rust core          | ~1k       | ~5k      |
| PyO3                             | Rust bindings for Python — THE bridge                 | N/A       | ~30k     |
| Maturin                          | Build tool for Rust Python packages                   | N/A       | ~15k     |

## Cross-Reference

- **go-to-rust**: Async runtime patterns; shared M:N scheduling concepts with asyncio migration
- **java-to-rust**: Enterprise service patterns; shared DI-to-trait-injection approaches
- **nodejs-to-rust**: Web framework migration; shared npm/pip-to-Cargo dependency mapping
- **r-to-rust**: Scientific computing and data frame migration (pandas/numpy patterns)
- **julia-to-rust**: Numerical computing; shared scientific Python → Rust patterns
- **php-to-rust**: Dynamic-to-static typing patterns; shared framework migration strategies
- **c-to-rust**: C extension replacement via PyO3; FFI bridging patterns
