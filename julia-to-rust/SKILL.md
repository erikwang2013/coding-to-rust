---
name: julia-to-rust
description: Use when migrating Julia codebases to Rust — covers JIT to AOT compilation, multiple dispatch to traits/enums, Array to ndarray, DataFrames to polars, Flux to burn/candle, and incremental replacement strategy. Includes canonical code patterns, common mistakes, and reference implementations.
updated: 2026-07-30
---

# Julia to Rust Migration Guide

## Architecture Mapping

Julia is a JIT-compiled dynamic language with multiple dispatch as its core paradigm, designed for high-performance numerical computing. Rust is an AOT-compiled systems language with a trait system, zero-cost abstractions, and explicit control over memory.

| Julia Concept | Rust Equivalent |
|---|---|
| Julia runtime + LLVM JIT | rustc + LLVM (AOT compilation) |
| Pkg (Project.toml / Manifest.toml) | Cargo.toml + Cargo.lock |
| Julia REPL | evcxr (Rust REPL) / cargo script |
| Pluto / Jupyter notebooks | evcxr (Jupyter kernel) / mdBook |
| Revise.jl (hot reloading) | Not available — requires recompilation; use `cargo watch` |
| `@time` / `@benchmark` | `criterion` / `divan` / `std::time::Instant` |
| `@code_llvm` / `@code_native` | `cargo rustc -- --emit=llvm-ir` / `cargo asm` |
| `@profview` | `samply` / `perf` / `flamegraph` |
| PackageCompiler.jl | `cargo build --release` (always AOT) |
| CUDA.jl / AMDGPU.jl | `cudarc` / `wgpu` / `rustacuda` |
| `@spawn` / `@spawnat` (Distributed) | `rayon` / `mpi` crate / `tokio::spawn` |

Julia's main advantage is the interactive, exploratory loop aided by JIT. When migrating to Rust, pair the compiled binary with evcxr for exploration or keep Julia for prototyping while Rust handles production deployment.

## Type System Mapping

Julia is dynamically typed with optional type annotations for dispatch. Rust is statically typed with generics and trait bounds.

| Julia Type | Rust Type | Notes |
|---|---|---|
| `Int64` / `Int32` | `i64` / `i32` | Choose based on domain |
| `Float64` / `Float32` | `f64` / `f32` | Julia default is Float64 |
| `Bool` | `bool` | Direct mapping |
| `String` | `String` / `&str` | |
| `Symbol` | `&'static str` (interned) / custom `Symbol` type | Use `string_cache` crate for interning |
| `Nothing` | `()` (unit type) / `Option<T>` | |
| `Missing` | `Option<T>` | Julia's sentinel for missing data |
| `Vector{T}` / `Array{T,1}` | `Vec<T>` / `ndarray::Array1<T>` | |
| `Matrix{T}` / `Array{T,2}` | `ndarray::Array2<T>` / `nalgebra::DMatrix<T>` | |
| `Array{T,N}` | `ndarray::Array<T, D>` | ndarray uses const generics for dimension |
| `Dict{K,V}` | `HashMap<K,V>` / `BTreeMap<K,V>` | |
| `Set{T}` | `HashSet<T>` / `BTreeSet<T>` | |
| `Tuple` | `(A, B, C)` native tuple | Direct mapping |
| `NamedTuple` | `(field: Type, ...)` / struct | Named fields require a struct |
| `Union{A, B, C}` | `enum` with variants | `enum MyUnion { A(A), B(B), C(C) }` |
| `AbstractArray` | `trait + generics` / `ndarray::ArrayBase` | |
| `AbstractFloat` | `num_traits::Float` trait | |
| `Any` | `Box<dyn Any>` / `dyn Trait` | |
| `Function` | `fn(...)` / `Fn` / `FnMut` / `FnOnce` | |
| `Expr` (metaprogramming) | `proc_macro` TokenStream / `syn` AST | |
| `Type{T}` | `PhantomData<T>` / `std::any::TypeId` | Compile-time type information |
| `Val{N}` | const generics `const N: usize` | |
| `Channel{T}` | `tokio::sync::mpsc::channel` / `std::sync::mpsc` | |

### Parametric Types to Generics

```rust
// Julia:
// struct MyContainer{T}
//     data::Vector{T}
// end

// Rust — 泛型 struct:
struct MyContainer<T> {
    data: Vec<T>,
}

impl<T> MyContainer<T> {
    fn new() -> Self {
        Self { data: Vec::new() }
    }

    fn push(&mut self, item: T) {
        self.data.push(item);
    }
}

// 带 trait 约束的泛型（类似 Julia 的 where T <: Number）:
use std::fmt::Display;
use std::ops::Add;

struct Calculator<T: Add<Output = T> + Display> {
    value: T,
}
```

## Memory & Ownership Model

Julia uses a tracing garbage collector with a generational design. Rust uses ownership with no GC.

| Julia Memory Pattern | Rust Translation |
|---|---|
| GC-managed heap objects | Ownership — values dropped at scope exit |
| Immutable objects (struct, Tuple) | Stack-allocated or heap via `Box` |
| Mutable objects (arrays, Dict) | Owned values with `&mut` borrowing |
| Circular references | `Weak<T>` |+
| `finalizer(f, obj)` | `Drop` trait |+
| `unsafe_wrap(Array, pointer)` | `unsafe { ... }` with raw pointer |
| `pointer_from_objref` | `&raw const` / `&raw mut` (raw references) |
| Pooled arrays / memory arenas | `bumpalo` / `typed-arena` crate |
| `deepcopy()` | `.clone()` / `serde::Serialize` round-trip |
| `@view` (array views) | `ArrayView` / `ArrayViewMut` (ndarray slices) |
| `copyto!(dest, src)` | `dest.copy_from(&src)` (ndarray) / `dest.clone_from(&src)` |

### Array Views and Ownership

```rust
// Julia:
// v = [1, 2, 3, 4, 5]
// @view v[2:4]  # 返回数组视图，不复制数据

use ndarray::{Array1, s};

fn use_views() {
    let v = Array1::from_vec(vec![1, 2, 3, 4, 5]);
    let view = v.slice(s![1..4]);  // 不可变视图，零拷贝
    // view 借用 v，在 view 的生存期内 v 不可变
    // println!("{}", view);

    let mut v_mut = v.clone();
    {
        let mut view_mut = v_mut.slice_mut(s![1..4]);
        view_mut.fill(99);  // 通过视图修改原数组
    }
    // 视图离开作用域后，v_mut 重新可用
}
```

## Concurrency / Async Translation

Julia has coroutines (Tasks), multi-threading, and distributed computing. Rust provides async/await, OS threads, and rayon.

| Julia Concurrency | Rust Equivalent |
|---|---|
| `@async` / `@sync` | Tokio tasks — `tokio::spawn(async { ... })` |
| `@spawn` (threads) | `rayon::spawn(|| { ... })` / `std::thread::spawn` |
| `@threads for` | `rayon::par_iter()` |
| `@distributed for` | `mpi` crate / rayon on cluster node |
| `Channel{T}` | `tokio::sync::mpsc` / `std::sync::mpsc` |
| `put!(ch, val)` / `take!(ch)` | `tx.send(val).await` / `rx.recv().await` |
| `@fetchfrom` / `@spawnat` | MPI `send` / `recv` |
| `Future` (Julia) | `tokio::task::JoinHandle` |
| `fetch(f::Future)` | `handle.await.unwrap()` |
| `Threads.@threads` (loop parallel) | `rayon::par_iter().for_each()` |
| `Threads.@spawn` | `std::thread::spawn` |
| `Base.Threads.atomic_add!` | `AtomicU64::fetch_add` |
| `ReentrantLock` | `std::sync::Mutex` (non-reentrant by design) |

### Task Migration Example

```rust
// Julia:
// ch = Channel{Int}(32)
// @async for i in 1:10
//     put!(ch, i^2)
// end
// @async for result in ch
//     println(result)
// end

// Rust — tokio:
use tokio::sync::mpsc;

async fn producer(tx: mpsc::Sender<i32>) {
    for i in 1..=10 {
        tx.send(i * i).await.unwrap();
    }
}

async fn consumer(mut rx: mpsc::Receiver<i32>) {
    while let Some(result) = rx.recv().await {
        println!("{result}");
    }
}

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel(32);
    let h1 = tokio::spawn(producer(tx));
    let h2 = tokio::spawn(consumer(rx));
    let _ = tokio::join!(h1, h2);
}
```

## Build System & Dependencies

| Julia Tool | Rust Equivalent |
|---|---|
| `Pkg.add("Package")` | `cargo add <crate>` |
| `using Package` / `import Package` | `use crate_name;` |
| `Project.toml` / `Manifest.toml` | `Cargo.toml` / `Cargo.lock` |
| `] activate .` | Working directory auto-activated |
| `Pkg.test("Package")` | `cargo test` |
| `Pkg.update()` | `cargo update` |
| `Pkg.instantiate()` | `cargo fetch` (automatic on build) |
| `JULIA_DEPOT_PATH` | `CARGO_HOME` |
| `JULIA_LOAD_PATH` | Module path under `src/` |
| `BinaryBuilder.jl` (cross-compile) | `cross` crate / `cargo-zigbuild` |
| `PackageCompiler.jl` (AOT) | `cargo build --release` (always AOT) |
| `Test.jl` / `@test` / `@testset` | `#[cfg(test)] mod tests` / `cargo test` |
| `Documenter.jl` | `cargo doc` |
| `Aqua.jl` (quality) | `cargo clippy` / `cargo audit` |

### Cargo.toml for a Julia-to-Rust Scientific Project

```toml
[package]
name = "my-scientific-app"
version = "0.1.0"
edition = "2021"

[dependencies]
ndarray = { version = "0.16", features = ["rayon", "blas"] }
nalgebra = "0.33"
ndarray-stats = "0.6"
linregress = "0.5"
polars = { version = "0.45", features = ["lazy", "csv", "parquet"] }
rayon = "1.10"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
plotters = { version = "0.3", features = ["evcxr", "line_series"] }
burn = { version = "0.16", features = ["wgpu", "train"] }  # 深度学习
num-traits = "0.2"
tracing = "0.1"
thiserror = "2"
clap = { version = "4", features = ["derive"] }  # CLI 参数解析

[profile.release]
opt-level = 3        # Julia -O3 等价
lto = "fat"
codegen-units = 1    # 最大化优化（类似 Julia 的 @code_native 输出）
```

## Standard / Popular Library Mapping

| Julia Library | Rust Equivalent |
|---|---|
| `Base.map(f, arr)` | `arr.mapv(f)` (ndarray) / `iter.map(f)` |
| `Base.filter(f, arr)` | `iter.filter(f)` |
| `Base.reduce(f, arr)` | `iter.fold(init, f)` |
| `Base.broadcast(f, arr)` | `arr.mapv(f)` (ndarray) / `iter.map(f)` (element-wise) |
| `Base.cat(arrs...; dims=2)` | `ndarray::stack(Axis(1), &[a, b])` |
| `LinearAlgebra` | `nalgebra` / `ndarray-linalg` / `faer` |
| `LinearAlgebra.eigen(A)` | `nalgebra::DMatrix::symmetric_eigen()` |
| `LinearAlgebra.svd(A)` | `nalgebra::SVD::new(A)` |
| `LinearAlgebra.qr(A)` | `nalgebra::QR::new(A)` |
| `LinearAlgebra.cholesky(A)` | `nalgebra::Cholesky::new(A)` |
| `Statistics.mean(x)` | `ndarray_stats::QuantileExt` / manual |
| `Statistics.std(x)` | `ndarray_stats::MaybeQuantileExt` / manual |
| `Distributions.jl` | `rand_distr` crate |
| `Random.rand(n)` | `rand::random::<f64>()` / `rand::thread_rng()` |
| `DifferentialEquations.jl` | `diffsol` / `ode_solvers` crate |
| `Optim.jl` | `argmin` / `nlopt` crate |
| `JuMP.jl` (optimization modeling) | `good_lp` crate |
| `DataFrames.jl` | `polars` |
| `CSV.jl` | `csv` crate / `polars::CsvReader` |
| `JSON.jl` | `serde_json` |
| `HTTP.jl` | `reqwest` (client) / `axum` (server) |
| `Plots.jl` | `plotters` |
| `Makie.jl` | `plotters` (2D) / `kiss3d` (3D) |
| `Flux.jl` | `burn` / `candle` / `tch-rs` |
| `Zygote.jl` (AD) | `dfdx` (autodiff in Rust) |
| `Symbolics.jl` | `symbolica` crate (computer algebra) |
| `BenchmarkTools.jl` | `criterion` / `divan` |
| `Profile.jl` | `perf` / `flamegraph` / `samply` |
| `JLD2.jl` | `hdf5` crate / `serde` + custom format |
| `Arrow.jl` | `arrow` crate / polars IPC |
| `PyCall.jl` / `PythonCall.jl` | `PyO3` crate |
| `CxxWrap.jl` | `cxx` crate / `bindgen` |
| `RCall.jl` | `extendr` (from Rust to R) |
| `MPI.jl` | `mpi` crate |
| `CUDA.jl` | `cudarc` / `rustacuda` |
| `LoopVectorization.jl` | Compiler `-C target-cpu=native` (auto-vectorization) |
| `StaticArrays.jl` | const generics `[T; N]` / `nalgebra::SVector` |
| `UnPack.jl` / `Parameters.jl` | Destructuring / builder pattern |

### Broadcasting Translation

```rust
// Julia:
// result = sin.(x) .+ cos.(y) .* 2

// Rust — ndarray mapv（逐元素）:
use ndarray::Array1;
fn broadcast_op(x: &Array1<f64>, y: &Array1<f64>) -> Array1<f64> {
    x.mapv(|v| v.sin()) + &y.mapv(|v| v.cos()) * 2.0
}

// 使用 azip 宏（更高效，融合循环）:
fn broadcast_fused(x: &Array1<f64>, y: &Array1<f64>) -> Array1<f64> {
    ndarray::azip!((x in x, y in y) x.sin() + y.cos() * 2.0)
}
```

## Canonical Patterns

### Pattern 1: Multiple Dispatch to Trait / Enum

```rust
// Julia — 多重分派:
// process(x::Int) = x * 2
// process(x::String) = uppercase(x)
// process(x::Vector) = sum(x)
// process(x::Float64) = round(x, digits=2)

// Rust — 方案 A: trait-based (静态分派，编译时):
trait Process {
    fn process(&self) -> Self;
}

impl Process for i32 {
    fn process(&self) -> Self { self * 2 }
}

impl Process for String {
    fn process(&self) -> Self { self.to_uppercase() }
}

// Rust — 方案 B: enum dispatch (动态，运行时):
#[derive(Debug)]
enum Value {
    Int(i32),
    Float(f64),
    Str(String),
    Vec(Vec<f64>),
}

impl Value {
    fn process(&self) -> Result<Value, &'static str> {
        match self {
            Value::Int(x) => Ok(Value::Int(x * 2)),
            Value::Float(x) => Ok(Value::Float((x * 100.0).round() / 100.0)),
            Value::Str(s) => Ok(Value::Str(s.to_uppercase())),
            _ => Err("不支持的操作"),
        }
    }
}

// 推荐：大多数 Julia 多分派场景用 trait-based，少数需运行时切换的用 enum
```

### Pattern 2: Macro to Declarative/Proc Macro

```rust
// Julia:
// macro twice(ex)
//   quote
//     $(esc(ex))
//     $(esc(ex))
//   end
// end
// @twice println("hello")

// Rust — declarative macro:
macro_rules! twice {
    ($expr:expr) => {
        $expr;
        $expr;
    };
}
// twice!(println!("hello"));

// Rust — 更复杂的宏（proc macro，等价于 Julia 的 generated functions）:
// Julia @generated 函数用 proc_macro 实现，在编译时生成代码
```

### Pattern 3: Lazy Evaluation with Iterators

```rust
// Julia — 惰性求值:
// result = 1:10 |> (x -> x .|> abs2) |> sum

// Rust — 迭代器链（零中间分配）:
fn pipeline() -> i32 {
    (1..=10)
        .map(|x| x * x)    // abs2 => 平方
        .sum()
}

// 更复杂的链:
fn complex_pipeline(data: &[f64]) -> Option<f64> {
    data.iter()
        .filter(|&&x| x > 0.0)         // 只处理正值
        .map(|&x| x.sqrt() * 2.0)       // 逐元素变换
        .reduce(|acc, x| acc + x)       // 折叠求和
}
```

### Pattern 4: Type-Stable Struct with Validation

```rust
// Julia:
// struct Config
//     host::String
//     port::Int
//     function Config(host, port)
//         @assert port > 0 && port < 65536
//         new(host, port)
//     end
// end

// Rust — 构造函数验证:
#[derive(Debug, Clone)]
pub struct Config {
    host: String,
    port: u16,  // u16 保证 0-65535
}

impl Config {
    pub fn new(host: impl Into<String>, port: u16) -> Result<Self, &'static str> {
        if port == 0 {
            return Err("端口不能为 0");
        }
        Ok(Self { host: host.into(), port })
    }
}
```

### Pattern 5: Differential Equation Solver

```rust
// Julia:
// using DifferentialEquations
// f(u, p, t) = -0.5 * u
// u0 = 1.0
// prob = ODEProblem(f, u0, (0.0, 10.0))
// sol = solve(prob, Tsit5())

// Rust — 使用 ode_solvers:
use ode_solvers::{Dopri5, OdeSolverState, System};

struct ExponentialDecay { pub lambda: f64 }

impl System<f64, OdeSolverState<f64>> for ExponentialDecay {
    fn system(&self, _t: f64, y: &[f64], dy: &mut [f64]) {
        dy[0] = -self.lambda * y[0];
    }
}

fn solve_ode() {
    let system = ExponentialDecay { lambda: 0.5 };
    let mut solver = Dopri5::new(
        system,
        0.0,    // t0
        10.0,   // t_end
        0.01,   // dt
        vec![1.0], // u0
        1e-6,   // rtol
        1e-8,   // atol
    );
    solver.integrate().unwrap();
    // solver.x_out() — time points
    // solver.y_out() — solution values
}
```

### Pattern 6: DataFrames.jl to Polars

```rust
// Julia:
// df = CSV.read("data.csv", DataFrame)
// filtered = filter(:value => v -> v > 10, df)
// transformed = transform(filtered, :value => (v -> v .* 2) => :doubled)
// grouped = groupby(transformed, :category)
// result = combine(grouped, :doubled => mean)

// Rust polars:
use polars::prelude::*;

fn process_dataframe(path: &str) -> PolarsResult<DataFrame> {
    let df = CsvReadOptions::default()
        .try_into_reader_with_file_path(Some(path.into()))?
        .finish()?;

    df.lazy()
        .filter(col("value").gt(10.0))
        .with_column((col("value") * 2.0).alias("doubled"))
        .group_by(&[col("category")])
        .agg(&[col("doubled").mean()])
        .collect()
}
```

## FFI & Incremental Migration

| Strategy | Tool | When to Use |
|---|---|---|
| Call Rust from Julia | `jlrs` crate / `ccall` with Rust cdylib | Replace hot Julia functions |
| Call Julia from Rust | `jlrs` crate (embed Julia runtime) | Keep complex Julia ecosystem packages |
| Shared memory via Arrow | `arrow` crate in both | Zero-copy data exchange |
| IPC via REST/gRPC | axum/tonic | Micro-service architecture |
| Standalone binary | Pure Rust | Batch processing pipelines |
| Python bridge | `PyO3` (Julia calls Python → calls Rust) | Legacy integration |

### jlrs: Rust Function Callable from Julia

```rust
// Rust — 编译为 Julia 可调用的共享库:
use jlrs::prelude::*;

fn fast_sum(arr: ArrayView1<f64>) -> f64 {
    arr.iter().sum()
}

// 通过 ccall 从 Julia 调用:
// Julia:
// result = ccall((:fast_sum, "libmyrust"), Float64, (Ptr{Float64}, Int64), data, length(data))
```

### Migration Order for Julia Projects

1. Profile with `@profview` to identify bottlenecks. JIT overhead is typically in first-call latency and type instability.
2. Rewrite type-unstable Julia functions as type-stable Rust functions callable via jlrs or ccall.
3. Replace numerical kernels (matrix ops, ODE solvers) with Rust equivalents, using ndarray/nalgebra.
4. Port data pipeline logic (CSV/JSON processing) from DataFrames.jl to polars.
5. Extract ML training loops from Flux.jl to burn/candle for deployment.
6. Replace the CLI/entry point with a Rust binary; keep Julia only for interactive notebooks.

## Common Mistakes

### Mistake 1: Expecting JIT-Like Start-up Speed

```rust
// Julia — 第一次调用时 JIT 编译，后续调用极快:
// julia> @time f(100)  ->  首先 ~0.5s JIT，之后 ~ns 级

// Rust — AOT 编译，启动极快，但无运行时优化:
// 不会因为调用上下文而重新编译或优化
// 解决方案: cargo build --release 中 lto = "fat" 和 codegen-units = 1
// 最大化静态优化，类似 Julia 的稳态性能
```

### Mistake 2: Trying to Reproduce Multiple Dispatch Exactly

```rust
// 错误 — 试图复制完全动态的多重分派:
// 使用 Any 类型和运行时类型检查 → 失去 Rust 优势

// 正确 — 识别实际的分派模式:
// 1. 如果分派目标在编译时已知 → trait + generics
// 2. 如果只有少数几种类型 → enum
// 3. 如果类型在运行时才知道 → Box<dyn Trait>
// 大多数 Julia 代码中，多分派其实是静态可确定的
```

### Mistake 3: Over-allocating in Loops (Julia's GC Pattern)

```rust
// 错误 — Julia 风格的自由分配（依赖 GC）:
fn compute_bad(n: usize) -> Vec<f64> {
    let mut result = Vec::new();
    for i in 0..n {
        let temp = vec![i as f64; 100];  // 每次循环分配，重复释放
        result.push(temp.iter().sum());
    }
    result
}

// 正确 — 预分配并重用缓冲区:
fn compute_good(n: usize) -> Vec<f64> {
    let mut result = Vec::with_capacity(n);
    let mut buffer = vec![0f64; 100];  // 重用同一个缓冲区
    for i in 0..n {
        buffer.fill(i as f64);
        result.push(buffer.iter().sum());
    }
    result
}
```

### Mistake 4: Treating Rust Arrays as 1-Indexed

```rust
// 错误 — Julia 的 1-index 习惯:
fn get_element_bad(matrix: &ndarray::Array2<f64>, i: usize, j: usize) -> f64 {
    matrix[(i, j)]  // Julia 习惯: (1,1) 是第一个元素
    // Rust 中 (1,1) 实际上是第二行第二列！
}

// 正确:
fn get_element_good(matrix: &ndarray::Array2<f64>, i: usize, j: usize) -> Option<&f64> {
    if i == 0 || j == 0 { return None; }
    matrix.get((i - 1, j - 1))  // 将 1-based 索引转换为 0-based
}
```

### Mistake 5: Unnecessary Mutable Patterns

```rust
// 错误 — Julia 的 in-place 习惯:
fn add_noise_bad(data: &mut [f64]) {
    for i in 0..data.len() {
        data[i] += rand::random::<f64>() * 0.01;
    }
}

// 更好的 Rust 风格 — 函数式 + 允许编译器优化:
fn add_noise_good(data: &[f64]) -> Vec<f64> {
    use rand::Rng;
    let mut rng = rand::thread_rng();
    data.iter()
        .map(|&v| v + rng.gen::<f64>() * 0.01)
        .collect()
}
// 只有在性能关键的内部循环中才用 &mut 版本
```

### Mistake 6: Ignoring rustc Auto-Vectorization

```rust
// Julia 开发者习惯显式使用 @simd / LoopVectorization.jl
// Rust 编译器在 release 模式下自动向量化

// 无需手动标 simd，编译器已处理:
fn dot_product(a: &[f64], b: &[f64]) -> f64 {
    a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
}
// cargo build --release 自动使用 SSE/AVX/NEON
// RUSTFLAGS="-C target-cpu=native" 可启用所有本地 CPU 特性
```

## Reference Implementations

| Project | Description | Migration Pattern |
|---|---|---|
| [jlrs](https://github.com/Taaitaaiger/jlrs) | Julia-Rust interop toolkit | Bidirectional FFI; embed Julia in Rust |
| [polars](https://github.com/pola-rs/polars) | DataFrame library | DataFrames.jl alternative written in Rust |
| [nalgebra](https://github.com/dimforge/nalgebra) | Linear algebra library | General purpose; similar to Julia's LinearAlgebra |
| [faer](https://github.com/sarah-ek/faer) | High-performance linear algebra | BLAS/LAPACK in pure Rust; similar scope to Julia arrays |
| [burn](https://github.com/tracel-ai/burn) | Deep learning framework | Flux.jl alternative; multi-backend (wgpu, CUDA, ROCm) |
| [candle](https://github.com/huggingface/candle) | Minimalist ML framework | Lightweight Flux.jl alternative by HuggingFace |
| [ode_solvers](https://github.com/srenevey/ode-solvers) | ODE solvers in Rust | DifferentialEquations.jl subset |
| [symbolica](https://github.com/benruijl/symbolica) | Computer algebra system | Symbolics.jl analogue |
| [evcxr](https://github.com/evcxr/evcxr) | Rust REPL for Jupyter | Interactive exploration, similar to Julia REPL/Pluto |

## Cross-Reference

- **python-to-rust** — For migrating numpy/scipy workflows similar to Julia's numerical stack
- **r-to-rust** — For migrating statistical modeling and data frame operations
- **go-to-rust** — For migrating concurrent service code that Julia calls via HTTP/IPC
- **c-to-rust** — For migrating the C/Fortran libraries that Julia wraps via ccall
- **cpp-to-rust** — For migrating C++ scientific libraries used as Julia dependencies
